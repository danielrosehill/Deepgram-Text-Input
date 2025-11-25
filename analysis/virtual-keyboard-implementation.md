# Deepgram Voice Keyboard: Virtual Keyboard Implementation Analysis

This document analyzes how Deepgram's `voice-keyboard-linux` achieves reliable text input on Wayland without depending on `ydotool`.

## Executive Summary

The Deepgram approach works by:

1. **Direct kernel-level access** via `/dev/uinput` - bypassing userspace input tools entirely
2. **Privilege escalation with immediate drop** - runs as root only long enough to create the virtual device, then drops to user privileges
3. **Pure Rust implementation** - no external dependencies for the keyboard simulation itself

This is fundamentally different from `ydotool` which requires a persistent daemon and has various reliability issues on Wayland.

## Core Architecture

### The Key Files

| File | Purpose |
|------|---------|
| `virtual_keyboard.rs` | Creates and manages the uinput virtual keyboard device |
| `input_event.rs` | Defines Linux input event structures and key codes |
| `main.rs` | Handles privilege escalation/dropping |
| `run.sh` | Wrapper script for proper privilege handling |

## How It Works: Step by Step

### Step 1: Privilege Escalation

The application must run as root to access `/dev/uinput`. The `run.sh` script handles this:

```bash
# From run.sh
sudo -E ./target/debug/voice-keyboard "$@"
```

The `-E` flag preserves environment variables (critical for audio access later).

### Step 2: Capture Original User Info

Before doing anything privileged, the code captures the original user's UID/GID:

```rust
// From main.rs:29-52
impl OriginalUser {
    fn capture() -> Self {
        // If running under sudo, get the original user info
        let uid = if let Ok(sudo_uid) = env::var("SUDO_UID") {
            Uid::from_raw(sudo_uid.parse().unwrap_or_else(|_| getuid().as_raw()))
        } else {
            getuid()
        };
        // ... similar for gid
    }
}
```

### Step 3: Create Virtual Keyboard (As Root)

This is the critical part that requires root. The code opens `/dev/uinput` and creates a virtual keyboard device:

```rust
// From virtual_keyboard.rs:36-103
impl RealKeyboardHardware {
    pub fn new(device_name: &str) -> Result<Self> {
        // Open uinput device - REQUIRES ROOT
        let fd = open(
            "/dev/uinput",
            OFlag::O_WRONLY | OFlag::O_NONBLOCK,
            Mode::empty(),
        ).context("Failed to open /dev/uinput")?;

        // Enable key events via ioctl
        unsafe {
            ui_set_evbit(fd, EV_KEY as u64)?;
        }

        // Enable all key codes (1-255)
        for keycode in get_all_keycodes() {
            unsafe {
                ui_set_keybit(fd, keycode as u64)?;
            }
        }

        // Set up device info (name, vendor, product IDs)
        let mut uidev = UInputUserDev::default();
        uidev.name[..copy_len].copy_from_slice(&name_bytes[..copy_len]);
        uidev.id.bustype = 0x03; // USB
        uidev.id.vendor = 0x1234;
        uidev.id.product = 0x5678;

        // Write device info and create
        file.write_all(uidev_bytes)?;
        unsafe { ui_dev_create(fd)?; }

        Ok(Self { fd, name: device_name.to_string() })
    }
}
```

### Step 4: Drop Privileges Immediately

Once the virtual keyboard is created, root is no longer needed. The code drops back to the original user:

```rust
// From main.rs:54-102
fn drop_privileges(&self) -> Result<()> {
    if getuid().is_root() {
        // Preserve environment variables needed for audio
        let pulse_runtime_path = env::var("PULSE_RUNTIME_PATH").ok();
        let xdg_runtime_dir = env::var("XDG_RUNTIME_DIR").ok();
        let wayland_display = env::var("WAYLAND_DISPLAY").ok();

        // Drop group first, then user (required order!)
        setgid(self.gid)?;
        setuid(self.uid)?;

        // Restore environment for audio access
        if let Some(xdg_path) = xdg_runtime_dir {
            env::set_var("XDG_RUNTIME_DIR", xdg_path);
        }
        // ... etc
    }
    Ok(())
}
```

### Step 5: Send Key Events

With the file descriptor still open (it persists after privilege drop), the code can send key events:

```rust
// From virtual_keyboard.rs:106-145
fn send_event(&self, event: InputEvent) -> Result<()> {
    let event_bytes = unsafe {
        std::slice::from_raw_parts(
            &event as *const _ as *const u8,
            std::mem::size_of::<InputEvent>(),
        )
    };

    unsafe {
        libc::write(self.fd, event_bytes.as_ptr() as *const libc::c_void, event_bytes.len());
    }
    Ok(())
}

fn send_key(&self, keycode: u16, pressed: bool) -> Result<()> {
    // Send key event
    let key_event = InputEvent::key_event(keycode, pressed);
    self.send_event(key_event)?;

    // Send synchronization event (required!)
    let syn_event = InputEvent::syn_event();
    self.send_event(syn_event)?;

    Ok(())
}
```

## The Linux Input Event Structure

The code defines the exact structure the kernel expects:

```rust
// From input_event.rs:3-39
#[repr(C)]
pub struct InputEvent {
    pub time: libc::timeval,   // Timestamp
    pub type_: u16,            // Event type (EV_KEY, EV_SYN, etc.)
    pub code: u16,             // Key code
    pub value: i32,            // 1 = pressed, 0 = released
}

// Event types
pub const EV_SYN: u16 = 0x00;  // Synchronization
pub const EV_KEY: u16 = 0x01;  // Key press/release

// Example key codes
pub const KEY_A: u16 = 30;
pub const KEY_ENTER: u16 = 28;
pub const KEY_LEFTSHIFT: u16 = 42;
```

## Why This Works on Wayland (When ydotool Doesn't)

### The ydotool Problem

`ydotool` uses a daemon (`ydotoold`) that:
1. Creates a virtual input device
2. Listens on a socket for commands
3. Requires the daemon to be running with correct permissions

This architecture has several failure points on Wayland:
- Daemon permission issues
- Socket communication failures
- Race conditions between daemon and client
- Not maintained (stale project)

### The Deepgram Solution

By contrast, Deepgram's approach:

1. **No daemon** - single process creates and uses the device
2. **Direct kernel communication** - writes directly to the file descriptor
3. **File descriptor persists** - once `/dev/uinput` is opened as root, the FD remains usable after privilege drop
4. **Protocol-agnostic** - works identically on X11 and Wayland because it operates at the kernel level, below the display server

## Key ioctl Calls

The code defines these ioctl macros for uinput device setup:

```rust
// From virtual_keyboard.rs:14-19
nix::ioctl_write_int!(ui_set_evbit, b'U', 100);   // Enable event types
nix::ioctl_write_int!(ui_set_keybit, b'U', 101);  // Enable key codes
nix::ioctl_none!(ui_dev_create, b'U', 1);         // Create device
nix::ioctl_none!(ui_dev_destroy, b'U', 2);        // Destroy device
```

These correspond to:
- `UI_SET_EVBIT` (0x40045564) - Enable an event type
- `UI_SET_KEYBIT` (0x40045565) - Enable a specific key code
- `UI_DEV_CREATE` (0x5501) - Create the virtual device
- `UI_DEV_DESTROY` (0x5502) - Destroy the virtual device

## Character to Keycode Mapping

The code includes a complete mapping from characters to Linux key codes:

```rust
// From input_event.rs:172-240
pub fn char_to_keycode(c: char) -> Option<(u16, bool)> {
    match c {
        'a' | 'A' => Some((KEY_A, c.is_uppercase())),
        // ... all letters ...
        '!' => Some((KEY_1, true)),  // Shift+1
        '@' => Some((KEY_2, true)),  // Shift+2
        // ... all symbols ...
    }
}
```

The tuple returns `(keycode, needs_shift)` - if shift is needed, the code presses shift, then the key, then releases both.

## Dependencies

Minimal external dependencies (from `Cargo.toml`):

```toml
nix = { version = "0.27", features = ["user", "fs", "ioctl"] }  # Unix syscalls
libc = "0.2"                                                     # C library bindings
```

No `ydotool`, no `xdotool`, no display-server-specific libraries.

## Security Considerations

1. **Temporary root access** - Root is only used to create the device, then immediately dropped
2. **No persistent daemon** - No long-running privileged process
3. **File descriptor isolation** - Once created, the FD is owned by the process

## Implications for Other STT Projects

Any speech-to-text project on Linux/Wayland could use this approach:

1. **Run as root initially** (via sudo)
2. **Open `/dev/uinput`** and create virtual keyboard
3. **Drop privileges** back to user
4. **Keep the file descriptor** - it remains usable
5. **Write input events** directly to the FD

This completely bypasses the need for:
- `ydotool` / `ydotoold`
- `xdotool` (X11 only anyway)
- Any display-server-specific input injection

## Code Quality Notes

The implementation is well-structured:
- Clean separation between hardware abstraction (`KeyboardHardware` trait) and business logic
- Mock implementation for testing
- Comprehensive test suite (800+ lines of tests)
- Proper error handling with `anyhow`
- Logging with `tracing`
