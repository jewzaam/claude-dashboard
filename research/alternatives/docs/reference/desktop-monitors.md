# Desktop Monitoring Tools

## Python Stack (What claude-dashboard Uses)

| Library | Purpose | Cross-Platform | Citation |
|---------|---------|---------------|----------|
| psutil | Process discovery, PID validation | Win + Linux + macOS | [50] |
| pystray | System tray icon | Win + Linux + macOS | [51] |
| Tkinter | UI framework (bundled with Python) | Win + Linux + macOS | stdlib |
| Pillow | Tray icon image generation | Win + Linux + macOS | stdlib |

This stack is the lightest viable option for a cross-platform desktop overlay. No build tools, no runtime downloads, no browser.

## Alternative Frameworks

| Framework | Pros | Cons |
|-----------|------|------|
| Electron | Rich UI, web tech | 100MB+ runtime, high memory |
| Tauri | Smaller than Electron, Rust backend | Requires Rust toolchain |
| PyQt/PySide | Mature, feature-rich | GPL/LGPL licensing, heavier than Tkinter |
| ttkbootstrap [52] | Modern Tkinter themes | Add-on library, not a framework |

## Always-on-Top Pattern

Standard Tkinter approach: `root.attributes('-topmost', True)` with `overrideredirect(True)` for borderless. Works on Windows and Linux. Some Wayland compositors may ignore `-topmost` but GNOME respects it.

## Wayland Considerations

On GNOME/Wayland, window management requires the `window-calls` GNOME Shell extension for listing and activating windows via D-Bus. No equivalent of X11's `xdotool` or `wmctrl`. See `claude_dashboard/platform/linux.py` for the implementation.
