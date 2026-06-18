# cajeta-ifx-macos — Spec

**Status:** Draft. Requirements for the **macOS** backend of `ifx` (window / input / audio).
This is a requirements document (the *what* and *why*); the task breakdown is the plan.

## 1. Overview & why
`dev.cajeta.ifx.macos` is the macOS realization of the portable `ifx` contract
(`cajeta.ifx`). It FFI-binds macOS system APIs and registers `WindowBackend` / `AudioBackend` /
`InputBackend` implementations through `BackendRegistry`. The app codes once against `ifx`; this
library is selected at runtime on macOS (registry + probe + priority, with `CAJETA_IFX_*`
overrides). Umbrella design: the cajeta repo's `documents/cajeta-gfx/cajeta-gfx-spec.md` §9.

## 2. Capabilities (what it binds)
- **Window + Surface:** Cocoa / AppKit (NSWindow + CAMetalLayer); produces the opaque `Surface` the gfx swapchain pairs with
  `VK_EXT_metal_surface (MoltenVK)`.
- **Input:** GameController + NSEvent / IOKit HID — keyboard/mouse arrive via the window event queue; gamepads via the input
  API.
- **Audio:** CoreAudio (AVAudioEngine) — output and capture.

> **Notes.** The event loop must run on the main thread (an AppKit requirement). Present via MoltenVK now; native Metal later.

## 3. Goals
- Implement the full `ifx` SPI for macOS behind the one portable API surface — no macOS concept
  leaks to callers.
- Register + `probe()`/`priority()` so the dispatcher binds it on macOS; honor `CAJETA_IFX_*`.
- Hand a correct opaque `Surface` to the gfx swapchain (`VK_EXT_metal_surface (MoltenVK)`).
- Track macOS API versioning **here**, isolated from the other backends.

## 4. Non-goals
- No portable-contract changes (those live in stdlib `ifx`).
- No graphics/render code — gfx owns the swapchain; this only supplies the `Surface`.
- No UI toolkit, input action-mapping, or audio mixing/DSP (engine / Glorias concerns).
- No codecs — capture/recording is the `cajeta-ifx-harness` backend + optional `VideoSink` providers.

## 5. References
- Umbrella design: `cajeta-gfx-spec.md` §9 (platform layer — §9.1 codegen-vs-HAL, §9.3 packaging,
  §9.4 binding model).
- Contract: `cajeta.ifx` — `Backend` / `WindowBackend` / `AudioBackend` / `InputBackend`.
- Selector: the `cajeta-ifx-backend` melt (target → backend).

---

## Appendix A — Vendor SDK details (2025-2026 research)

### Binding: the Objective-C wall (the dominant constraint)
Apple frameworks split two ABI families:
- **C-ABI (direct FFI):** Core Audio / Audio Unit / AudioToolbox, **IOKit HID**, Core Foundation,
  and `CAMetalLayer` interop via MoltenVK (`VK_EXT_metal_surface` takes a `CAMetalLayer*`). `metal-cpp`
  gives Metal a C++ binding.
- **Obj-C only (NO C API):** **AppKit** (`NSApplication`/`NSWindow`/`NSView`), **GameController**
  (`GCController`/`GCKeyboard`/`GCMouse`), `NSEvent`, **AVFoundation/AVAudioEngine**.
→ **This backend ships a thin Obj-C/Obj-C++ (`.m`/`.mm`) shim** exposing `extern "C"` for the AppKit
windowing bootstrap + GameController; binds Core Audio + IOKit directly via C FFI. (Alternative:
`objc_msgSend` runtime FFI — brittle, every call site cast to its exact prototype on arm64.)

### SDK reference
| Domain | Modern (framework) | C-ABI? | Legacy / fallback | Deprecated | Min |
|---|---|---|---|---|---|
| Window | **AppKit** `NSWindow`/`NSView` + **CAMetalLayer** | Obj-C (shim) | — | OpenGL (4.1, 2018) | macOS 11+ |
| Surface | **MoltenVK** `VK_EXT_metal_surface` (`vkCreateMetalSurfaceEXT`, `CAMetalLayer*`); native Metal/`metal-cpp` | C interop | — | `VK_MVK_macos_surface` | — |
| Input | **GameController** (Xbox/DualSense incl. haptics/adaptive triggers/gyro; `GCKeyboard`/`GCMouse` macOS 11+); `NSEvent` for UI | Obj-C (shim) | **IOKit HID** (`IOHIDManager`) | — | 10.15+ |
| Audio | **Audio Unit / Core Audio** (`AUHAL`/`kAudioUnitSubType_HALOutput`, pull-model real-time callback — output + capture) | **C (direct)** | `AVAudioEngine` (Obj-C convenience) | OpenAL (2018) | — |

**No native Vulkan on Apple** — only MoltenVK (Vulkan 1.4 subset, `VK_KHR_portability_subset`).

### Capability support & gaps (vs spec §9.7)
- **Main-thread requirement:** `NSApplication` + all NSWindow/NSView/Core-Animation work + event
  processing MUST run on the main thread; the framework owns the run loop. The portable contract's
  "event loop on the window's creating thread" maps to "main thread" here.
- **Supported (strengths):** multi/resizable windows, positioning, cursor warp, first-class
  DualSense/Xbox controllers (haptics, adaptive triggers, gyro, touchpad), low-latency audio (Audio
  Unit, C). Loopback capture = partial (ScreenCaptureKit).
- **Gaps:** Obj-C binding tax (the shim); Vulkan only via MoltenVK subset; mic capture needs
  `NSMicrophoneUsageDescription` + entitlement; **notarization + Hardened Runtime / App-Sandbox
  entitlements** for distribution.

### References
- Game window for Metal (macOS): https://developer.apple.com/documentation/Metal/managing-your-game-window-for-metal-in-macos
- NSApplication (main-thread run loop): https://developer.apple.com/documentation/appkit/nsapplication
- GameController: https://developer.apple.com/documentation/gamecontroller · Core Audio: https://developer.apple.com/documentation/coreaudio
- MoltenVK / `VK_EXT_metal_surface`: https://github.com/KhronosGroup/MoltenVK
- OpenGL deprecation → Metal: https://developer.apple.com/documentation/Metal/migrating-opengl-code-to-metal

---

## Appendix B — Interop mechanism (Obj-C via C runtime + clang shim)

**Feasibility: yes.** Apple's frameworks are reachable through `libobjc`'s **C ABI**, and Cajeta's
`@Native("symbol")` already binds typed methods to C symbols over a stable `extern "C"` ABI (it
lowers through LLVM, same toolchain family as clang). No Obj-C *language* support is required in
Cajeta. Two paths, used together:

### 1. Direct `libobjc` FFI (the floor — ~90% of calls)
- `@Native`-bind `objc_getClass(const char*)`, `sel_registerName(const char*)`, and **typed casts of
  `objc_msgSend`**. `objc_msgSend` is **not variadic** — each call site is a function pointer cast to
  the **exact** callee prototype (args + return widths). **arm64:** `objc_msgSend` for *everything*
  (small structs in regs, large via x8). **x86_64:** route struct returns → `objc_msgSend_stret`,
  `long double` → `objc_msgSend_fpret`. (This is the Rust `objc2` / Zig `zig-objc` pattern — a
  codegen-time monomorphic `msgSend<Ret,Args>` + cached interned selectors/classes.)
- **Memory (explicit — Cajeta is not ARC-aware):** `objc_retain`/`objc_release`/`objc_autorelease`;
  wrap each frame/callback in `objc_autoreleasePoolPush`/`Pop`. Honor `alloc`/`new`/`copy` = +1.
- **Delegates at runtime:** `objc_allocateClassPair(NSObject,"…",0)` + `class_addMethod(sel, IMP,
  "v@:@")` where IMP is a **Cajeta-exported `(id self, SEL _cmd, …)` C function** + `class_addProtocol`
  + `objc_registerClassPair`. Stash a C context via an ivar / `objc_setAssociatedObject`. Covers
  `NSApplicationDelegate`, target/action, `NSNotificationCenter` observers (GameController connect).

### 2. A thin clang `.m`/`.mm` shim (for the painful parts)
Compiled `clang -fobjc-arc -c shim.m`, linked `-framework AppKit -framework GameController
-framework AVFoundation -framework QuartzCore`. Encapsulates:
- **Blocks** — `GCControllerAxisInput.valueChangedHandler`, AVFoundation completion handlers. The
  block ABI (`_NSConcreteStackBlock`, copy/dispose helpers) is error-prone by hand; let clang's `^{}`
  wrap a passed-in C fn-ptr + `void* ctx`.
- **App/run-loop bootstrap** — `[NSApplication sharedApplication]` + `[NSApp run]` own **thread 0**;
  the shim takes `main()` and calls back into Cajeta via a C callback once the run loop is live
  (sidesteps main-thread affinity + "never returns").
- **Heavy delegates** with many `@encode` signatures.

Direct C FFI already works for **Core Audio / Audio Unit (`AUHAL`)** and **IOKit HID** — no shim.

### Optional language ergonomic
An `@ObjC` / `@Native(objc="ClassName","selector:")` form the compiler lowers to
selector-registration + a correctly-typed `objc_msgSend` cast — **sugar over the FFI floor above**,
not a new capability. Lets backend code read `app.run()` instead of hand-written msgSend casts.

### Sources
- objc4 `message.h` (msgSend variants, arm64): https://opensource.apple.com/source/objc4/objc4-781/runtime/message.h.auto.html
- objc_msgSend must-cast: https://www.mikeash.com/pyblog/objc_msgsends-new-prototype.html
- Runtime classes from C: https://www.mikeash.com/pyblog/friday-qa-2010-11-6-creating-classes-at-runtime-in-objective-c.html
- Block ABI: https://clang.llvm.org/docs/Block-ABI-Apple.html · Rust objc2: https://docs.rs/objc2 · zig-objc: https://github.com/mitchellh/zig-objc
