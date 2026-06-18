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
