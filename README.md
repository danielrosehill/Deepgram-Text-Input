# Deepgram Voice Keyboard Analysis

Analysis of how Deepgram's Linux voice keyboard achieves reliable text input on Wayland without depending on `ydotool`.

## Background

Getting speech-to-text to work on Wayland/Linux is challenging. The typical stumbling block is `ydotool` - a tool that simulates keyboard input but:

- Hasn't been updated in quite some time
- Requires building from source with many workarounds
- Often fails to work reliably on Wayland

Many dictation tools that work flawlessly for speech recognition fail at the final step: actually typing the transcribed text into the active application.

## The Deepgram Approach

Deepgram released an open-source [voice-keyboard-linux](https://github.com/deepgram/voice-keyboard-linux/) project that works on Wayland essentially out of the box.

From initial analysis, the project:
- Uses privilege escalation
- Creates its own virtual keyboard directly at the kernel level
- Bypasses problematic dependencies like `ydotool`

## Purpose of This Repository

This is a **research/analysis repository** - not a fork or reimplementation.

The goal is to:
1. Analyze the Deepgram codebase to understand exactly how it achieves reliable virtual keyboard input
2. Document the technical approach with code snippets and explanations
3. Provide insights that could inform other speech-to-text projects on Linux/Wayland

## Analysis

See the full technical analysis: **[Virtual Keyboard Implementation Analysis](analysis/virtual-keyboard-implementation.md)**

### Summary of Findings

The Deepgram approach works by:

1. **Direct kernel-level access** via `/dev/uinput` - bypassing userspace input tools entirely
2. **Privilege escalation with immediate drop** - runs as root only long enough to create the virtual device, then drops to user privileges
3. **Pure Rust implementation** - no external dependencies for keyboard simulation

### Why This Works on Wayland

Unlike `ydotool` which requires a persistent daemon with socket communication (prone to permission issues and race conditions), Deepgram's approach:

- Opens `/dev/uinput` directly as root
- Creates a virtual keyboard device via ioctl calls
- Drops privileges immediately after device creation
- Keeps the file descriptor open - it remains usable after privilege drop
- Writes input events directly to the kernel, below the display server layer

This makes it **protocol-agnostic** - works identically on X11 and Wayland because it operates at the kernel level.

### Key Code Locations

| File | Purpose |
|------|---------|
| `src/virtual_keyboard.rs` | Creates/manages the uinput virtual keyboard |
| `src/input_event.rs` | Linux input event structures and key codes |
| `src/main.rs` | Privilege escalation and dropping |

### Implications for Other STT Projects

Any speech-to-text project could adopt this pattern:

1. Run as root initially (via sudo)
2. Open `/dev/uinput` and create virtual keyboard
3. Drop privileges back to user
4. Keep the file descriptor - it remains usable
5. Write input events directly to the FD

This completely bypasses `ydotool`, `xdotool`, and any display-server-specific input injection.

## Source Project

- **Repository**: https://github.com/deepgram/voice-keyboard-linux/
- **License**: Check the original repository for licensing terms

## Notes

This analysis is for educational purposes to understand how reliable text input can be achieved on Wayland for speech-to-text applications.
