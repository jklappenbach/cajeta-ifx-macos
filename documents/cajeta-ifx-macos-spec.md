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
