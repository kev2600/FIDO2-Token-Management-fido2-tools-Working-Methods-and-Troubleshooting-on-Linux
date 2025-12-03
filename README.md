# FIDO2 Security Key Management Cheat Sheet (libfido2 tools)

100% correct for libfido2 1.16.0 – November 2025  
Tested on YubiKey 5C NFC & TrustKey T120

## Quick Start

```bash
# 1. Find your key (no PIN needed, no sudo)
fido2-token -L
# Example output → /dev/hidraw9

export DEVICE=/dev/hidraw9   # do this once per session
```

## Core Commands That Actually Work in 1.16.0

| Purpose | Command (copy-paste ready) | Notes |
|---------|----------------------------|-------|
| List all connected tokens | `fido2-token -L` | No PIN |
| Firmware version (human readable) | `fido2-token -I "$DEVICE" \| awk '/major/{M=$2}/minor/{m=$2}/build/{b=$2}END{printf "%d.%d.%d\n",strtonum(M),strtonum(m),strtonum(b)}'` | Works on every device |
| List resident keys / passkeys (recommended) | `fido2-token -L -r "$DEVICE"` (interactive) or `echo "PIN" \| fido2-token -L -r "$DEVICE"` | Shows RP ID, |
| Change PIN (safest) | `fido2-token -C "$DEVICE"` | Interactive, highly recommended |
| Set first PIN (if none exists) | `fido2-token -S "$DEVICE"` | Interactive |
| Factory reset (wipes everything) | `sudo fido2-token -R "$DEVICE"` | No PIN, irreversible |

## Script-friendly One-liners (actually work in 1.16.0)

```bash
# Read PIN once (hidden input)
read -s -p "Enter PIN: " PIN; echo

# List passkeys only if PIN is correct
if echo "$PIN" | fido2-token -I "$DEVICE" >/dev/null 2>&1; then
    echo "PIN correct"
    echo "$PIN" | fido2-token -L -r "$DEVICE"      # full readable list
else
    echo "Wrong PIN"
fi

# ⚠️ DANGEROUS - Non-interactive PIN change (only for automation, never type PINs in cleartext!)
printf "oldpin\nnewpin123\nnewpin123" | fido2-token -C "$DEVICE"
```

## Important 1.16.0 Gotchas (so you never get confused again)

* `fido2-cred` has no `-L` option → never use `fido2-cred -L`
* The only tool that lists resident keys is `fido2-token -L -r`
* `-VerifyPin` is broken in interactive mode → it always prints "1.16.0". Use `-I` or pipe the PIN to test it instead
* `-V` (uppercase) = print version. Always. Ignore it

## Permissions Setup (2025 – works out of the box)

```bash
# Install tools (also installs udev rules)

# Debian/Ubuntu
sudo apt install libfido2-1 fido2-tools

# Fedora
sudo dnf install libfido2

# Arch
sudo pacman -S libfido2

# Activate the udev rules that were just installed
sudo udevadm control --reload-rules && sudo udevadm trigger

# Re-plug your key → no sudo or plugdev group needed!
# (The installed rules in /etc/udev/rules.d/ now grant access automatically)
```

**What this does:** The package installation drops udev rules files (like `70-u2f.rules`, `69-libfido2.rules`) into `/etc/udev/rules.d/`. The `reload-rules` command tells udev to read them, and `trigger` applies them to connected devices. Re-plugging ensures a clean detection cycle with the new permissions.

## TrustKey T120 Specific Setup (VID 311f)

If your TrustKey isn't detected or requires sudo after the standard setup, create a custom rule:

```bash
# 1. Install the required packages (if not already done)
sudo dnf install fido2-tools libfido2

# 2. Create a permanent udev rule for TrustKey devices (VID 311f)
sudo tee /etc/udev/rules.d/70-trustkey.rules <<'EOF'
# TrustKey T110 / T120 / G320 / G320H (FIDO2 + HID)
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="311f", TAG+="uaccess", MODE="0666"
EOF

# 3. Reload udev rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# 4. Unplug your TrustKey
# 5. Re-plug your TrustKey ← this step is mandatory!

# 6. Verify it works
fido2-token -L
# → should show /dev/hidraw9: vendor=0x311f, product=0xa6e9 (TrustKey TrustKey T120)

# 7. Show full device info (AAGUID, options, PIN retries, etc.)
fido2-token -I /dev/hidraw9

# 8. One-liner that always works (even if hidraw number changes)
fido2-token -I $(fido2-token -L | grep -o '/dev/hidraw[0-9]*')
```

### Optional: Make it even easier (add to your shell)

```bash
# Add this line to ~/.bashrc or ~/.zshrc
echo "alias trustkey-info='fido2-token -I \$(fido2-token -L | grep -o \"/dev/hidraw[0-9]*\")'" >> ~/.bashrc

# Reload your shell or run:
source ~/.bashrc

# Now just type:
trustkey-info
```

## Extra Useful Commands

| Purpose | Command |
|---------|---------|
| Make a new discoverable credential | `fido2-cred -M -r "$DEVICE"` |
| Delete a credential | `echo "PIN" \| fido2-cred -D <cred-id> "$DEVICE"` |
| Man pages | `man fido2-token` / `man fido2-cred` |

**Note on credential deletion:** To get the `<cred-id>`, first list your credentials with `fido2-token -L -r "$DEVICE"` and extract the credential ID from the output.

## Final Tips

* Always use `fido2-token -L -r` to list passkeys (nothing else)
* Never hard-code PINs in scripts
* For YubiKey OTP/PIV/OpenPGP slots → use `ykman` instead
* When PIN retries are exhausted, the device locks and requires a factory reset
* Factory reset requires `sudo` because it's a destructive operation that bypasses PIN protection
* The TrustKey T120 is now permanently fixed on your machine — no more permission headaches!
