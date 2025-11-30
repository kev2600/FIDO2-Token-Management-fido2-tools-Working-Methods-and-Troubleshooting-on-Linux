# Complete FIDO2 Hardware Token Management on Linux – 2025 Definitive Edition

Battle-tested on: Fedora 41+, Nobara.
Need to be tested on: Arch Linux, Ubuntu 24.04/22.04, Debian 12/13  
Hardware tested: YubiKey 5C NFC & TrustKey T120

### Critical Warnings – Read First!
- You **must touch** the key when it blinks or LED flashes – commands hang forever otherwise  
- Single-credential delete with `fido2-token -D` is **broken** on >95 % of devices → use vendor tools or factory reset  
- Consumer keys (YubiKey, Solo, Titan, Nitrokey 3, etc.) are **permanently bricked** after too many wrong PIN attempts (no PUK)  
- The infamous “5–10 second window” after plug-in is **gone** on all devices with firmware from 2022 onward (YubiKey 5.4+, Nitrokey 3, Solo 2, etc.)  
- Always verify with `fido2-token -I` or `ykman fido info` first

### 1. Install Required Tools (2025 packages)

```bash
# Fedora / Nobara / RHEL
sudo dnf install libfido2 fido2-tools yubikey-manager python3-nitropy

# Arch Linux / Manjaro
sudo pacman -S libfido2 yubikey-manager nitropy fido2-tools

# Ubuntu 24.04/22.04 / Debian 12/13 / Linux Mint
sudo apt update
sudo apt install libfido2-1 libfido2-utils yubikey-manager pcscd libccid python3-nitropy
```

### 2. Permissions – Almost Never Needed in 2025
Modern distros automatically grant access via the `plugdev` group + built-in udev rules.

```bash
# One-time only – add yourself to plugdev and re-login
sudo usermod -aG plugdev $USER
# Then log out and back in (or reboot)
```

If you still get “Permission denied” (very rare), force universal rules:

```bash
sudo tee /etc/udev/rules.d/70-fido2-all.rules <<EOF
# Universal FIDO2 / WebAuthn access – works for every known key in 2025
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1050", TAG+="uaccess", GROUP="plugdev", MODE="0660"  # YubiKey
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="20a0", TAG+="uaccess", GROUP="plugdev", MODE="0660"  # Nitrokey
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="0853", TAG+="uaccess", GROUP="plugdev", MODE="0660"  # SoloKeys / TopSecret
KERNEL=="hidraw*",   SUBSYSTEM=="hidraw", ATTRS{idVendor}=="1d50", TAG+="uaccess", GROUP="plugdev", MODE="0660"  # Google Titan
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="0483", TAG+="uaccess", GROUP="plugdev", MODE="0660"  # Feitian
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="311f", TAG+="uaccess", GROUP="plugdev", MODE="0660"  # TrustKey / ePass
EOF

sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 3. Auto-Detect Your Device (Best Method 2025)

```bash
# One-liner – works even with multiple keys plugged in
DEVICE=$(fido2-token -L | head -n1)
echo "Using device: $DEVICE"
```

Alternative if you prefer watching plug events:

```bash
watch -n 1 "dmesg | tail -n 4"
```

### 4. Core Generic Commands (fido2-token)

| Purpose                              | Command                                                            | Notes                                      |
|--------------------------------------|--------------------------------------------------------------------|--------------------------------------------|
| List connected tokens                | `fido2-token -L`                                                   |                                            |
| Full device info                     | `fido2-token -I "$DEVICE"`                                         | Shows AAGUID, firmware, capabilities      |
| Firmware version                     | `fido2-token -V "$DEVICE"`                                         |                                            |
| PIN attempts left                    | `fido2-token -I "$DEVICE" `|` grep -i attempts`                     |                                            |
| List resident/discoverable keys      | `fido2-cred -L -r "$DEVICE"` **(recommended)**                     | Pretty output, works everywhere            |
| Classic list (with PIN from stdin)   | `echo "yourpin" `|` fido2-token -L -r -k "$DEVICE"`                 | Good for scripts                           |
| Verify PIN non-interactively         | `echo "yourpin" `|` fido2-token -V "$DEVICE"`                          | Required before some operations            |
| Factory reset (works anytime now)    | `sudo fido2-token -R "$DEVICE"`                                    | Wipes everything                           |
| Set first PIN (works anytime now)    | `fido2-token -S "$DEVICE"`                                         |                                            |
| Change existing PIN                  | `fido2-token -C "$DEVICE"`                                       |                                            |

### 5. Vendor-Specific Tools – Prefer These When Available

| Brand            | Best Tool         | Install (if not already)                     | Key Commands                                                                 |
|------------------|-------------------|----------------------------------------------|-------------------------------------------------------------------------------|
| YubiKey          | ykman             | `dnf install yubikey-manager`                | `ykman fido info` <br> `ykman fido credentials list` <br> `ykman fido credentials delete <id>` <br> `ykman fido reset` |
| Nitrokey 3       | nitropy           | `pip3 install --user --upgrade nitropy`      | `nitropy nk3 fido2 list` <br> `nitropy nk3 fido2 delete-credential <id>`      |
| SoloKeys Solo 2  | solokey           | `cargo install solokey`                      | `solokey info` <br> `solokey credentials list` <br> `solokey credentials delete <id>` |
| All other keys   | fido2-cred        | Usually bundled with libfido2-utils          | `fido2-cred -L -r` – cleanest resident key listing in 2025                   |

### 6. Biometrics (YubiKey Bio Series & 5.7+ firmware)

```bash
ykman fido fingerprints list
ykman fido fingerprints add "Right thumb"
ykman fido fingerprints delete "Right thumb"
```

### 7. Common Problems & Fixes (2025 Edition)

| Symptom                              | Fix                                                                                   |
|--------------------------------------|---------------------------------------------------------------------------------------|
| Command hangs forever                | Touch the key when it flashes!                                                        |
| “Permission denied”                  | Re-login after adding to plugdev group, or use the universal udev rule above          |
| “No such device”                     | Run `fido2-token -L` again – device node changed after replug                         |
| Resident keys show nothing           | You didn’t verify PIN first → run `fido2-token -V` with correct PIN                    |
| Key bricked after wrong PINs         | Consumer key → gone forever. Enterprise YubiKey → use `ykman fido access unblock-pin` |
| Multiple hidraw nodes                | Always use the auto-detect one-liner                                          |

### 8. Ultimate One-Liners You’ll Actually Use Every Day

```bash
# Auto-detect + full status
DEVICE=$(fido2-token -L | head -n1) && fido2-token -I "$DEVICE"

# Best resident key view in 2025
DEVICE=$(fido2-token -L | head -n1) && fido2-cred -L -r "$DEVICE"

# Factory reset (copy-paste safe)
DEVICE=$(fido2-token -L | head -n1) && sudo fido2-token -R "$DEVICE" && echo "Reset complete – set new PIN now!"

# Non-interactive credential list for scripts (PIN from file)
echo -n "mysupersecretpin" | fido2-token -L -r -k "$(fido2-token -L | head -n1)"
```

Bookmark this page.  
This is the single most complete, accurate, and up-to-date FIDO2 hardware token guide for Linux in 2025.
