# Security Assessment & Language Portability

This document analyzes the security characteristics of Deepgram's uinput approach and how it could be ported to other languages.

## Security Assessment

The implementation is **reasonably secure** for its purpose. Here's the breakdown:

### What's Good

#### 1. Minimal Privilege Window

Root access is held only for milliseconds - just long enough to open `/dev/uinput` and create the device:

```
Create keyboard (as root) → Drop privileges → Everything else runs as user
```

The actual privileged operations are in `main.rs:157-171`:

```rust
// Step 1: Create virtual keyboard while we have root privileges
let hardware = RealKeyboardHardware::new(device_name)?;
let mut keyboard = VirtualKeyboard::new(hardware);

// Step 2: Drop root privileges before initializing audio
original_user.drop_privileges()?;
```

#### 2. Proper Privilege Drop Order

The code correctly drops GID before UID (required by POSIX - doing it backwards can leave elevated privileges):

```rust
// From main.rs:67-69
// Drop group first, then user (required order!)
setgid(self.gid).context("Failed to drop group privileges")?;
setuid(self.uid).context("Failed to drop user privileges")?;
```

#### 3. No Persistent Privileged Daemon

Unlike `ydotool` which keeps a root daemon (`ydotoold`) running continuously, this approach has no long-lived privileged process. The attack surface is minimal.

#### 4. Selective Environment Preservation

Only preserves environment variables needed for audio functionality:

```rust
// From main.rs:62-65
let pulse_runtime_path = env::var("PULSE_RUNTIME_PATH").ok();
let xdg_runtime_dir = env::var("XDG_RUNTIME_DIR").ok();
let display = env::var("DISPLAY").ok();
let wayland_display = env::var("WAYLAND_DISPLAY").ok();
```

No unnecessary environment leakage from root context.

### Potential Concerns

#### 1. File Descriptor Inheritance

Once created, the virtual keyboard FD could theoretically be inherited by child processes. Any process with access to this FD can inject keystrokes.

**Mitigation**: This is inherent to the design. The code doesn't spawn untrusted child processes, so this is acceptable.

#### 2. USB Device Spoofing

The virtual device identifies as USB with fake vendor/product IDs:

```rust
// From virtual_keyboard.rs:72-75
uidev.id.bustype = 0x03; // USB
uidev.id.vendor = 0x1234;
uidev.id.product = 0x5678;
```

Some enterprise security software might flag this, but it's harmless in practice.

#### 3. No Input Validation

The code types whatever the STT service returns. A malicious or compromised STT server could inject arbitrary keystrokes.

**Mitigation**: This is an architectural concern for any STT system, not specific to this implementation. Users should only connect to trusted STT endpoints.

#### 4. Race Condition Window

There's a brief window between device creation and privilege drop where the process runs as root with an open uinput device. In practice, this window is too short to exploit.

### Security Comparison: Deepgram vs ydotool

| Aspect | Deepgram Approach | ydotool |
|--------|-------------------|---------|
| Privileged runtime | Milliseconds | Continuous daemon |
| Attack surface | Single process, brief | Daemon + socket + client |
| IPC mechanism | None | Unix socket |
| Privilege model | Escalate → drop | Persistent root daemon |

**Verdict**: The Deepgram approach has a significantly smaller attack surface.

---

## Language Portability

The core mechanism uses **standard Linux syscalls** - nothing Rust-specific. Any language that can:

1. Call `open()` on `/dev/uinput`
2. Make `ioctl()` calls
3. `write()` binary structs to a file descriptor
4. Call `setuid()`/`setgid()`

...can implement this pattern.

### Language Feasibility Matrix

| Language | Feasibility | Binary Size | Ease of Implementation | Notes |
|----------|-------------|-------------|------------------------|-------|
| **C** | Excellent | Tiny | Easy | Most natural fit - these are C APIs |
| **Go** | Excellent | ~5-10MB | Easy | `syscall` package has everything needed |
| **Python** | Good | N/A (interpreted) | Medium | Use `ctypes` or `python-evdev` |
| **Zig** | Excellent | Tiny | Easy | Direct C interop, similar to Rust |
| **C++** | Excellent | Small | Easy | Same as C, wrap in classes |
| **Node.js** | Possible | Large | Hard | Via `ffi-napi` - awkward |
| **Ruby** | Possible | N/A | Medium | Via FFI gem |

### Key Data Structures to Port

#### 1. input_event (24 bytes on 64-bit Linux)

This is the structure written to the FD for each key event:

```c
struct input_event {
    struct timeval time;  // 16 bytes
        // long tv_sec;    // 8 bytes
        // long tv_usec;   // 8 bytes
    __u16 type;           // 2 bytes (EV_KEY = 0x01, EV_SYN = 0x00)
    __u16 code;           // 2 bytes (key code, e.g., KEY_A = 30)
    __s32 value;          // 4 bytes (1 = press, 0 = release)
};
```

#### 2. uinput_user_dev (~340 bytes)

Used during device setup:

```c
struct uinput_user_dev {
    char name[UINPUT_MAX_NAME_SIZE];  // 80 bytes
    struct input_id id;                // 8 bytes
        // __u16 bustype;
        // __u16 vendor;
        // __u16 product;
        // __u16 version;
    __u32 ff_effects_max;              // 4 bytes
    __s32 absmax[ABS_CNT];             // 64 * 4 = 256 bytes
    __s32 absmin[ABS_CNT];             // 256 bytes (not used for keyboard)
    __s32 absfuzz[ABS_CNT];            // 256 bytes (not used for keyboard)
    __s32 absflat[ABS_CNT];            // 256 bytes (not used for keyboard)
};
```

#### 3. ioctl Constants

```c
#define UI_SET_EVBIT   _IOW('U', 100, int)  // 0x40045564
#define UI_SET_KEYBIT  _IOW('U', 101, int)  // 0x40045565
#define UI_DEV_CREATE  _IO('U', 1)          // 0x5501
#define UI_DEV_DESTROY _IO('U', 2)          // 0x5502
```

### Python Implementation Sketch

```python
#!/usr/bin/env python3
"""
Minimal virtual keyboard using uinput.
Must be run with sudo, will drop privileges after device creation.
"""

import os
import struct
import fcntl
import time

# ioctl constants
UI_SET_EVBIT = 0x40045564
UI_SET_KEYBIT = 0x40045565
UI_DEV_CREATE = 0x5501
UI_DEV_DESTROY = 0x5502

# Event types
EV_SYN = 0x00
EV_KEY = 0x01
SYN_REPORT = 0

# Key codes (subset)
KEY_A = 30
KEY_ENTER = 28
KEY_SPACE = 57
KEY_LEFTSHIFT = 42

class VirtualKeyboard:
    def __init__(self, name="Python Virtual Keyboard"):
        # Must be root to open uinput
        self.fd = os.open("/dev/uinput", os.O_WRONLY | os.O_NONBLOCK)

        # Enable EV_KEY events
        fcntl.ioctl(self.fd, UI_SET_EVBIT, EV_KEY)

        # Enable all key codes (1-255)
        for keycode in range(1, 256):
            fcntl.ioctl(self.fd, UI_SET_KEYBIT, keycode)

        # Build uinput_user_dev struct
        name_bytes = name.encode('utf-8')[:79].ljust(80, b'\x00')

        # struct: name[80] + id(bustype,vendor,product,version) + ff_effects_max + abs arrays
        # We only need name + id + ff_effects_max, abs arrays are zeroed
        uidev = struct.pack(
            '80s HHH H I',  # + 256 zeros for abs arrays
            name_bytes,
            0x03,   # bustype = USB
            0x1234, # vendor
            0x5678, # product
            1,      # version
            0       # ff_effects_max
        )
        # Pad with zeros for abs arrays (64 * 4 * 4 = 1024 bytes)
        uidev += b'\x00' * 1024

        os.write(self.fd, uidev)

        # Create the device
        fcntl.ioctl(self.fd, UI_DEV_CREATE)

        # Small delay for device to be ready
        time.sleep(0.1)

    def _send_event(self, event_type, code, value):
        """Send a single input event."""
        # struct input_event: timeval(16) + type(2) + code(2) + value(4) = 24 bytes
        now = time.time()
        sec = int(now)
        usec = int((now - sec) * 1000000)

        event = struct.pack('llHHi', sec, usec, event_type, code, value)
        os.write(self.fd, event)

    def _send_key(self, keycode, pressed):
        """Send a key press or release with sync."""
        self._send_event(EV_KEY, keycode, 1 if pressed else 0)
        self._send_event(EV_SYN, SYN_REPORT, 0)

    def press_key(self, keycode):
        """Press and release a key."""
        self._send_key(keycode, True)
        self._send_key(keycode, False)
        time.sleep(0.01)  # Small delay between keys

    def type_char(self, char):
        """Type a single character."""
        keycode, needs_shift = char_to_keycode(char)
        if keycode is None:
            return

        if needs_shift:
            self._send_key(KEY_LEFTSHIFT, True)

        self.press_key(keycode)

        if needs_shift:
            self._send_key(KEY_LEFTSHIFT, False)

    def type_text(self, text):
        """Type a string of text."""
        for char in text:
            self.type_char(char)

    def close(self):
        """Destroy the virtual device."""
        fcntl.ioctl(self.fd, UI_DEV_DESTROY)
        os.close(self.fd)


def char_to_keycode(char):
    """Map character to (keycode, needs_shift)."""
    # Lowercase letters
    if 'a' <= char <= 'z':
        return (KEY_A + ord(char) - ord('a'), False)
    # Uppercase letters
    if 'A' <= char <= 'Z':
        return (KEY_A + ord(char) - ord('A'), True)
    # Space
    if char == ' ':
        return (KEY_SPACE, False)
    # ... add more mappings as needed
    return (None, False)


def drop_privileges(target_uid, target_gid):
    """Drop root privileges to specified user."""
    if os.getuid() == 0:
        os.setgid(target_gid)  # Group first!
        os.setuid(target_uid)


# Example usage
if __name__ == "__main__":
    import pwd

    # Get original user from SUDO_UID
    original_uid = int(os.environ.get('SUDO_UID', os.getuid()))
    original_gid = int(os.environ.get('SUDO_GID', os.getgid()))

    # Create keyboard as root
    kb = VirtualKeyboard()

    # Drop privileges
    drop_privileges(original_uid, original_gid)
    print(f"Dropped to UID {os.getuid()}")

    # Now type something (still works with dropped privileges!)
    time.sleep(2)  # Give user time to focus a text field
    kb.type_text("hello world")

    kb.close()
```

### Go Implementation Sketch

```go
package main

import (
    "encoding/binary"
    "os"
    "syscall"
    "time"
    "unsafe"
)

const (
    UI_SET_EVBIT  = 0x40045564
    UI_SET_KEYBIT = 0x40045565
    UI_DEV_CREATE = 0x5501

    EV_SYN = 0x00
    EV_KEY = 0x01

    KEY_A     = 30
    KEY_SPACE = 57
)

type InputEvent struct {
    Sec   int64
    Usec  int64
    Type  uint16
    Code  uint16
    Value int32
}

type VirtualKeyboard struct {
    fd int
}

func NewVirtualKeyboard(name string) (*VirtualKeyboard, error) {
    fd, err := syscall.Open("/dev/uinput", syscall.O_WRONLY|syscall.O_NONBLOCK, 0)
    if err != nil {
        return nil, err
    }

    // Enable EV_KEY
    syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), UI_SET_EVBIT, EV_KEY)

    // Enable key codes
    for i := 1; i <= 255; i++ {
        syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), UI_SET_KEYBIT, uintptr(i))
    }

    // Write device info and create...
    // (struct packing code here)

    syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), UI_DEV_CREATE, 0)

    return &VirtualKeyboard{fd: fd}, nil
}

func (kb *VirtualKeyboard) SendKey(keycode uint16, pressed bool) {
    var value int32 = 0
    if pressed {
        value = 1
    }

    event := InputEvent{
        Sec:   time.Now().Unix(),
        Usec:  int64(time.Now().Nanosecond() / 1000),
        Type:  EV_KEY,
        Code:  keycode,
        Value: value,
    }

    // Write event to fd...
}
```

### Recommendations by Use Case

| Use Case | Recommended Language |
|----------|---------------------|
| Standalone CLI tool | Go or Rust (single binary, easy distribution) |
| Integration with existing Python STT | Python with ctypes |
| Maximum performance | C or Rust |
| Rapid prototyping | Python |
| Embedded/minimal systems | C or Zig |

---

## Summary

The Deepgram implementation is:

1. **Secure enough** for its purpose - brief privilege escalation, proper drop, no daemon
2. **Easily portable** - standard Linux syscalls work from any language
3. **Well-designed** - clean separation of concerns, testable architecture

The key insight is that `/dev/uinput` is a kernel interface, not a Rust-specific feature. Any language that can make syscalls can implement this pattern.
