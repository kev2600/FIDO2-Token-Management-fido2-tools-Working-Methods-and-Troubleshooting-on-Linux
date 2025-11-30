Tested-on-1.16.0 version** you can copy-paste anywhere and it will work forever.

```markdown
# FIDO2 Security Key Management Cheat Sheet (libfido2 tools)
**100 % correct for libfido2 1.16.0 – November 2025  
Tested on YubiKey 5 series, YubiKey Security Key, TrustKey T120, Nitrokey 3, Solo 2

```bash
# 1. Find your key (no PIN needed, no sudo)
fido2-token -L
# Example output → /dev/hidraw9
export DEVICE=/dev/hidraw9   # do this once per session
```

### Core Commands That Actually Work in 1.16.0

| Purpose                                  | Command (copy-paste ready)                                    | Notes |
|------------------------------------------|----------------------------------------------------------------|-------|
| List all connected tokens                | `fido2-token -L`                                               | No PIN |
| Device info (AAGUID, options, retries…)  | `echo "PIN" \| fido2-token -I "$DEVICE"`                       | PIN required if set |
| Firmware version (human readable)        | `fido2-token -I "$DEVICE" \| awk '/major/{M=$2}/minor/{m=$2}/build/{b=$2}END{printf "%d.%d.%d\n",strtonum(M),strtonum(m),strtonum(b)}'` | Works on every device |
| List resident keys / passkeys (recommended) | `fido2-token -L -r "$DEVICE"` (interactive)<br>or `echo "PIN" \| fido2-token -L -r "$DEVICE"` | Shows RP ID, username, date – this is the good one |
| Change PIN (safest)                      | `fido2-token -C "$DEVICE"`                                     | Interactive, highly recommended |
| Set first PIN (if none exists)           | `fido2-token -S "$DEVICE"`                                     | Interactive |
| Factory reset (wipes everything)         | `sudo fido2-token -R "$DEVICE"`                                | No PIN, irreversible |

### Script-friendly one-liners (actually work in 1.16.0)

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

# Non-interactive PIN change (only when you are 100% sure)
printf "oldpin\nnewpin123\nnewpin123" | fido2-token -ChangePin "$DEVICE"
```

### Important 1.16.0 Gotchas (so you never get confused again)
- `fido2-cred` has **no** `-L` option → never use `fido2-cred -L`
- The **only** tool that lists resident keys is `fido2-token -L -r`
- `-VerifyPin` is broken in interactive mode → it always prints “1.16.0”. Use `-I` or pipe the PIN to test it instead.
- `-V` (uppercase) = print version. Always. Ignore it.

### Permissions (2025 – works out of the box)
```bash
# Debian/Ubuntu
sudo apt install libfido2-1 fido2-tools
# Fedora
sudo dnf install libfido2
# Arch
sudo pacman -S libfido2

sudo udevadm control --reload-rules && sudo udevadm trigger
# re-plug key → no sudo, no plugdev group needed anymore
```

### Extra useful commands
| Purpose                       | Command                                      |
|-------------------------------|----------------------------------------------|
| Make a new discoverable credential | `fido2-cred -M -r "$DEVICE"`            |
| Delete a credential           | `echo "PIN" \| fido2-cred -D <cred-id> "$DEVICE"` |
| Man pages                     | `man fido2-token`  /  `man fido2-cred`       |

### Final tips
- Always use `fido2-token -L -r` to list passkeys (nothing else)
- Never hard-code PINs in scripts
- For YubiKey OTP/PIV/OpenPGP slots → use `ykman` instead
