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

## Analysis Documents

- [analysis/](analysis/) - Detailed technical analysis of the Deepgram implementation

## Key Questions to Answer

- How does the virtual keyboard device get created?
- What kernel interfaces are used?
- How is privilege escalation handled?
- What makes this approach more reliable than `ydotool`?

## Source Project

- **Repository**: https://github.com/deepgram/voice-keyboard-linux/
- **License**: Check the original repository for licensing terms

## Notes

This analysis is for educational purposes to understand how reliable text input can be achieved on Wayland for speech-to-text applications.
