# Complete FIDO2 Hardware Token Management on Linux (2025 Edition)  
Tested on Fedora, Nobara, Arch, Ubuntu, Debian with YubiKey, Nitrokey 3, SoloKeys 2, Google Titan, Feitian, OnlyKey, etc.

### Critical Warnings Up Front
- `fido2-token -D` (single credential delete) is **almost always broken** for resident/discoverable credentials → use factory reset or vendor tool instead.
- Most operations that change state (set PIN first time, factory reset) are **time-critical** (5–10 seconds after plug-in).
- You **must touch** the key when it blinks — commands will hang forever otherwise.
- Always know your device node: `fido2-token -L` or `ls /dev/hidraw*` + `journalctl -k | tail`
nitrokey-app
### 1. Permissions & udev Rules (Do This First!)
```bash
# Fedora / Nobara / RHEL
sudo dnf install libfido2 fido2-tools yubikey-manager nitrokey-app yubikey-manager

# Ubuntu / Debian
sudo apt install libfido2-1 fido2-tools yubikey-manager pcscd libnitrokey-dev

# Add your user to the plugdev (most distros)
sudo usermod -aG plugdev $USER
# Log out and back in (or reboot)
```

If you still get “Permission denied”, copy the built-in udev rules:
```bash
sudo cp 70-u2f.rules /etc/udev/rules.d/   # most distros already ship this
sudo udevadm control --reload-rules && udevadm trigger
```

### 2. Find Your Device Path
```bash
fido2-token -L
# Example output → /dev/hidraw9
# Or watch dmesg
watch -n 1 "dmesg | tail -n 8"
```

Set it once for the whole session:
```bash
DEVICE="/dev/hidraw9"   # ← change every time you replug!
```

You can begin by checking if your security key is recognized by the system:

Check for device recognition (if a key is plugged in):
```bash
lsusb
```

### 3. Core fido2-token Commands (Generic, works on all keys)

| Purpose                              | Command                                                                 | Notes                                                                 |
|--------------------------------------|-------------------------------------------------------------------------|-----------------------------------------------------------------------|
| List all connected tokens            | `fido2-token -L`                                                        |                                                                       |
| Full device info                     | `sudo fido2-token -I "${DEVICE}"`                                            | Shows AAGUID, firmware, capabilities                                  |
| Remaining PIN attempts               | `sudo fido2-token -I "${DEVICE}" | grep pinRetries`                           | Critical before you get locked out                                    |
| Verify current PIN (no side effects) | `sudo fido2-token -V "${DEVICE}"`                                            | Just checks PIN, very useful                                          |
| List resident credentials            | `sudo fido2-token -L -r "${DEVICE}"`                                         | Requires PIN                                                          |
| List with non-interactive PIN        | `echo -n "123456" | fido2-token -L -r -k "${DEVICE}"`                    | Great for scripts                                                     |
| Show used / remaining slots          | `sudo fido2-token -I -c "${DEVICE}"`                                         | Requires PIN                                                          |

### 4. PIN Management (The Tricky Part)

**First-time PIN or after factory reset (time-critical!)**
```bash
# 1. Unplug key
# 2. Plug key back in
# 3. Run within ~5 seconds
fido2-token -S "${DEVICE}"
```
If you see “Operation not permitted” → you were too slow → repeat.

**Change existing PIN (no timing issue)**
```bash
fido2-token -C "${DEVICE}"    # asks old PIN → new PIN → verify
```

### 5. Credential Deletion

| Method                              | Command                                                                                           | Reliability | Notes                                                      |
|-------------------------------------|---------------------------------------------------------------------------------------------------|-------------|------------------------------------------------------------|
| Single credential (almost never works) | `fido2-token -D -i 'BASE64ID' -r 'example.com' "${DEVICE}"`                                      | 5–10 %      | Works only on some very old keys                           |
| Factory reset (100 % reliable)      | Unplug → plug → run within 10 s: <br>`fido2-token -R "${DEVICE}"`                                 | 100 %       | Wipes EVERYTHING (all credentials + PIN)                   |
| YubiKey reliable single delete      | `ykman fido credentials delete --rp-id example.com <credential-id>`                              | 100 %       | Best option for YubiKeys                                   |
| Nitrokey 3 reliable delete       | `nitropy nk3 fido2 delete-credential <credential-id>`                                            | 100 %       |                                                            |
| Solo 2 reliable delete              | `solokey credentials delete <credential-id>`                                                      | 100 %       |                                                            |

### 6. Vendor-Specific Tools (Use These When fido2-token Isn’t Enough)

| Key Brand          | Recommended Tool                               | Install Command (Fedora)                     | Most Useful Commands                                                          |
|---------------------|------------------------------------------------|----------------------------------------------|-------------------------------------------------------------------------------|
| YubiKey (all)       | ykman (YubiKey Manager CLI)                    | `sudo dnf install yubikey-manager`           | `ykman fido info` <br> `ykman fido credentials list` <br> `ykman fido credentials delete …` <br> `ykman fido reset` |
| Nitrokey 3          | nitropy                                        | `pip3 install --user nitropy`                | `nitropy nk3 fido2 list` <br> `nitropy nk3 fido2 delete-credential <id>`      |
| SoloKeys 2          | solokey                                        | Cargo: `cargo install solokey`               | `solokey info` <br> `solokey credentials list` <br> `solokey credentials delete <id>` |
| Google Titan        | Only factory reset works reliably              | —                                            | Use `fido2-token -R` only                                             |
| Feitian ePass       | Usually only factory reset                     | —                                            | reset works                                                           |

### 7. Fingerprint / Biometrics (YubiKey Bio Series & 5.7+)

```bash
# List enrolled fingerprints
ykman fido fingerprints list

# Add a new fingerprint (touch sensor many times)
ykman fido fingerprints add "Left index"

# Delete one
ykman fido fingerprints delete "Left index"
```

### 8. Common Failure Modes & Fixes

| Symptom                                          | Fix                                                                                     |
|--------------------------------------------------|-----------------------------------------------------------------------------------------|
| Command hangs forever                            | Touch the key when it blinks!                                                           |
| “Device not found” or “Permission denied”        | Check udev rules + plugdev group + replug                                               |
| “Operation not permitted” on -S or -R            | You missed the 5–10 second window → unplug → plug → try again immediately              |
| `-L -r` shows nothing even though keys exist     | PIN not verified → run `fido2-token -V` first                                            |
| Key permanently locked after too many wrong PINs | Consumer keys (YubiKey, Solo, Titan) are bricked forever. Only enterprise YubiKeys have PUK. |
| Multiple /dev/hidraw entries                     | Use `dmesg to see which one appears on plug-in                                         |

### 9. Quick One-Liners You’ll Use All the Time

```bash
# Full status dump (my favorite)
fido2-token -I /dev/hidraw9 && fido2-token -L -r /dev/hidraw9

# Factory reset in one go (copy-paste friendly)
sudo fido2-token -R /dev/hidraw9 && echo "Now set new PIN immediately!"

# List credentials non-interactively with PIN from file (safe for scripts)
echo -n "mysuperpin" | fido2-token -L -r -k /dev/hidraw9
```

This version reflects real-world Linux usage in 2025.  
Keep it bookmarked — you’ll need it every time a key gets a stuck credential!
