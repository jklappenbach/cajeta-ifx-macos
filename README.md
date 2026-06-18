# cajeta-ifx-macos

`dev.cajeta.ifx.macos` — the **macOS** backend for **ifx**, the Cajeta interface-framework
facade for window / input / audio. It implements the `ifx` backend SPI by FFI-binding macOS system
APIs and registers itself so the `ifx` dispatcher selects it at runtime on macOS.

Part of the Cajeta graphics stack: stdlib **`ifx`** owns the portable, write-once contract; this
repo is one **optional, vendor-versioned backend**, pulled in (target-conditionally) via the
**`cajeta-ifx-backend`** melt. App code never imports this directly — it codes against `ifx`.

## What it binds
- **Window / surface:** Cocoa / AppKit (NSWindow + CAMetalLayer) → `VK_EXT_metal_surface (MoltenVK)`
- **Audio:** CoreAudio (AVAudioEngine)
- **Input:** GameController + NSEvent / IOKit HID

> **Notes.** The event loop must run on the main thread (an AppKit requirement). Present via MoltenVK now; native Metal later.

## Status
**v0.1 — library skeleton** (cajeta build-tool project). No implementation yet — see
[`documents/`](documents/) for the spec and plan.

## Build
```sh
cajeta build      # emits the .cja library
cajeta test       # unit tests
```

## License
MIT © 2026 Julian Klappenbach. See [LICENSE](LICENSE).
