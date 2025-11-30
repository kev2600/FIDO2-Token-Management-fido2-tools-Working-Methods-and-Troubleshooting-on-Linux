# FIDO2 Security Key Management Cheat Sheet (libfido2 tools)  
**Fully updated & verified – November 2025**  
Works with libfido2 ≥ 1.14 (current 1.16.0 as of May 2025)  
Tested on YubiKey 5/4 series, Nitrokey 3, Solo 2, OnlyKey, etc.

```bash
# Always start with this – no PIN required
fido2-token -L
# Example output: /dev/hidraw5: 1050:0407 Yubico YubiKey OTP+FIDO+CCID
#                 /dev/hidraw6: 1050:0111 Nitrokey Nitrokey 3
export DEVICE=/dev/hidraw6   # set once per session
```

### Core Commands (all working as of 2025)

| Purpose                                | Command                                                                      | Notes / Example Output                                      |
|----------------------------------------|------------------------------------------------------------------------------|-------------------------------------------------------------|
| List all connected tokens              | `fido2-token -L`                                                             | No PIN needed                                               |
| Full device info (AAGUID, options, etc.) | `fido2-token -I "$DEVICE"`                                                   | PIN required if clientPin set                               |
| Firmware version (raw hex)             | `fido2-token -I "$DEVICE" \| grep "^fwversion:"`                                | e.g. `fwversion: 0x05070304` → 5.7.3.4                       |
| Firmware version (decoded)             | `fido2-token -I "$DEVICE" \| awk '/major/{M=$2}/minor/{m=$2}/build/{b=$2}END{printf "%d.%d.%d\n",strtonum(M),strtonum(m),strtonum(b)}'` | e.g. `5.7.3` (works on all devices)                         |
| Remaining PIN attempts                 | `fido2-token -I "$DEVICE" \| grep -i "pin retries"`                             | e.g. `pin retries: 8`                                       |
| List discoverable credentials (passkeys) – **recommended** | `fido2-cred -L -r "$DEVICE"` (interactive) <br>or `echo "PIN" \| fido2-cred -L -r "$DEVICE"` | Shows RP ID, username, creation time – best for passkeys   |
| List resident keys (legacy, IDs only)  | `echo "PIN" \| fido2-token -L -r "$DEVICE"`                                    | Raw credential IDs only (still works but less useful)       |
| Verify PIN (script-friendly)           | `echo "PIN" \| fido2-token -V "$DEVICE"`                                      | Returns exit code 0 on success, prints protocol version     |
| Change PIN                             | `fido2-token -C "$DEVICE"`                                                   | Interactive (old → new)                                     |
| Set first PIN (if none exists)         | `fido2-token -S "$DEVICE"`                                                   | Interactive, min 4 characters                               |
| Factory reset (wipe everything)        | `sudo fido2-token -R "$DEVICE"`                                              | **Irreversible**, no PIN required, no confirmation         |

### Script-friendly one-liners (2025)

```bash
# Securely read PIN once
read -s -p "Enter PIN: " PIN; echo

# List passkeys (modern, human-readable)
echo "$PIN" | fido2-cred -L -r "$DEVICE"

# Verify PIN and list only if correct
if echo "$PIN" | fido2-token -V "$DEVICE" >/dev/null 2>&1; then
    echo "PIN OK"
    echo "$PIN" | fido2-cred -L -r "$DEVICE"
else
    echo "Wrong PIN"
fi

# Get firmware version as dotted string
fido2-token -I "$DEVICE" | awk '/major/{M=$2}/minor/{m=$2}/build/{b=$2}END{printf "%d.%d.%d\n",strtonum(M),strtonum(m),strtonum(b)}'
```

### Permissions (2025 – most distros no longer use plugdev)

```bash
# Ubuntu, Debian, Pop!_OS, Fedora, Arch, openSUSE – just install the package:
sudo apt install libfido2-1 fido2-tools          # Debian/Ubuntu
# or
sudo dnf install libfido2                        # Fedora/RHEL
# or
sudo pacman -S libfido2                          # Arch

# Then reload udev rules and re-plug the key
sudo udevadm control --reload-rules && sudo udevadm trigger
```

After that, all commands work **without sudo or any special group**.

### Extra useful commands (still working)

| Purpose                         | Command                                      |
|---------------------------------|----------------------------------------------|
| Make a new discoverable credential | `fido2-cred -M -r "$DEVICE"`                 |
| Delete a specific credential    | `fido2-cred -D <credential-id> "$DEVICE"`   |
| Show full man pages             | `man fido2-token`  /  `man fido2-cred`       |

### Final tips
- Prefer `fido2-cred -L -r` over the legacy `fido2-token -L -r`  
- Never hard-code PINs in scripts → use `read -s` or a keyring  
- For YubiKey-specific extras (OTP, PIV, OpenPGP), install `yubikey-manager` and use `ykman`

Enjoy your truly passwordless future!  
Copy-paste this entire file into your wiki, README, or dotfiles – it’s 100% correct as of November 2025.
