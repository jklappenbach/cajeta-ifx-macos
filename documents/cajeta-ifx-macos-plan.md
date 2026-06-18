# cajeta-ifx-macos — Plan

Derived from `cajeta-ifx-macos-spec.md`. Outline numbering `1` / `a` / `1`.
Checkbox legend: `[x]` done · `[~]` partial · `[ ]` not started.

## 1. Window + Surface backend
macOS window creation via Cocoa / AppKit (NSWindow + CAMetalLayer), producing the opaque `Surface` for `VK_EXT_metal_surface (MoltenVK)`.

   **TDD**
   a. [ ] `probe()` returns true under macOS, false elsewhere (env-forced).
   b. [ ] `createWindow` + `surfaceOf` yield a non-headless `Surface` with the requested extent.

   **Deliverables**
   a. [ ] `CocoaWindowBackend implements WindowBackend` + self-registration.
   b. [ ] FFI bindings for Cocoa / AppKit (NSWindow + CAMetalLayer) (create / destroy / poll) + the surface handle.

   **Acceptance Criteria**
   a. [ ] A window opens on macOS and the gfx swapchain presents to its `Surface`.

## 2. Input
   **TDD**
   a. [ ] Synthetic GameController + NSEvent / IOKit HID events surface as `WindowEvent` / `InputDevice` state.

   **Deliverables**
   a. [ ] keyboard/mouse via the window queue; gamepads via GameController + NSEvent / IOKit HID.

   **Acceptance Criteria**
   a. [ ] Input round-trips on a real macOS device.

## 3. Audio
   **TDD**
   a. [ ] An output stream plays a known buffer; capture reads a known input.

   **Deliverables**
   a. [ ] `AudioBackend` over CoreAudio (AVAudioEngine) (output + capture).

   **Acceptance Criteria**
   a. [ ] Audio output is audible / captured PCM matches a reference within tolerance.

## 4. Registration & dispatch
   **TDD**
   a. [ ] Registers at load; the dispatcher selects it on macOS; `CAJETA_IFX_*` override works.

   **Deliverables**
   a. [ ] static-init registration into `BackendRegistry`; `priority()` == 100.

   **Acceptance Criteria**
   a. [ ] On macOS with this library linked, `ifx` binds it with no app changes.

## Dependencies & sequencing
- Gated on the stdlib `cajeta.ifx` contract landing (`runtime/src/cajeta/ifx/`).
- The Surface/WSI hand-off is gated on the gfx swapchain (`cajeta.gpu.gfx`, spec Part GP-1 §4.2).
- Versioned independently of the other backends (own macOS API cadence).

---

## Appendix A — Binding & capability-gap work (from vendor research)

### 5. Binding (Objective-C shim + C FFI)
   **TDD**
   a. [ ] The `extern "C"` shim brings up `NSApplication`/`NSWindow`/`NSView` + `CAMetalLayer` on the
      **main thread** and pumps events.
   b. [ ] GameController connect/value callbacks marshal through the shim into C callbacks.

   **Deliverables**
   a. [ ] An Obj-C/Obj-C++ (`.m`/`.mm`) shim for AppKit windowing + GameController, flat C surface.
   b. [ ] Direct C FFI for Core Audio / Audio Unit (`AUHAL`, output+capture) and IOKit HID.
   c. [ ] MoltenVK `VK_EXT_metal_surface` from the shim's `CAMetalLayer`.

   **Acceptance Criteria**
   a. [ ] No Obj-C types cross into Cajeta; the engine sees only the C shim + C audio/HID APIs.

### 6. Capability gaps & fallbacks (vs spec §9.7)
   **TDD**
   a. [ ] Window/event work asserted on the main thread; off-main calls rejected.

   **Deliverables**
   a. [ ] main-thread run-loop integration; mic `NSMicrophoneUsageDescription` + entitlement; the
      Hardened-Runtime/notarization (and App-Sandbox entitlement) build path.
   b. [ ] `supports()` = full desktop set (multi-window, positioning, warp, rumble/adaptive-triggers/
      gyro via GameController, low-latency audio); loopback capture partial (ScreenCaptureKit).

   **Acceptance Criteria**
   a. [ ] A window + DualSense input + low-latency Audio-Unit output work; build notarizes.

---

## Appendix B — Interop implementation (Obj-C)

### 7. Obj-C runtime FFI core
   **TDD**
   a. [ ] A typed `objc_msgSend` cast calls a known selector (e.g. `[NSProcessInfo processInfo]`)
      and returns the right value on arm64 + x86_64.
   b. [ ] A runtime-built delegate's IMP receives `applicationDidFinishLaunching:`.

   **Deliverables**
   a. [ ] `@Native` bindings: `objc_getClass`, `sel_registerName`, typed `objc_msgSend` (+ `_stret`/
      `_fpret` on x86_64); cached selectors/classes.
   b. [ ] explicit retain/release + `objc_autoreleasePoolPush/Pop` per frame/callback.
   c. [ ] runtime delegate builder (`objc_allocateClassPair`+`class_addMethod`, IMP = exported C fn).

   **Acceptance Criteria**
   a. [ ] Ordinary AppKit/GameController calls work via FFI with correct memory behavior.

### 8. Clang shim + build integration
   **Deliverables**
   a. [ ] `.m`/`.mm` shim (`-fobjc-arc`) for blocks (`valueChangedHandler`), the `[NSApp run]`
      bootstrap (owns thread 0, callbacks into Cajeta), and heavy delegates; `extern "C"` surface.
   b. [ ] build-tool step: compile the shim with clang, link `-framework AppKit/GameController/
      AVFoundation/QuartzCore`; notarization/Hardened-Runtime path.

   **Acceptance Criteria**
   a. [ ] Window + DualSense (block handler) + main-thread loop work; the shim is the only Obj-C TU.
