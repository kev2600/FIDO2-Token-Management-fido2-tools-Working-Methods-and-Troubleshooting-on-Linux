Here’s a clean, ready-to-use version of the cheat sheet — perfect for dropping into a personal wiki, README, git repository notes, or your own dotfiles.

```markdown
# FIDO2 Security Key Management Cheat Sheet (libfido2 tools)

These commands use `fido2-token` and `fido2-cred` from the official `libfido2` toolkit  
(Yubico/maintained FIDO2 tools on Linux).

**First step – always identify your device:**
```bash
fido2-token -L
# Example output: /dev/hidraw6   Yubikey 5 NFC
export DEVICE=/dev/hidraw6   # set once per session
```

| Purpose                        | Command                                                                      | Notes / Output                                                                 |
|--------------------------------|------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| List all connected tokens      | `fido2-token -L`                                                             | Shows device path → set `$DEVICE`                                              |
| Device info (AAGUID, caps…)    | `fido2-token -I "$DEVICE"`                                                  | Human-readable info; includes remaining PIN attempts                           |
| Firmware version               | `echo "PIN" \| fido2-token -V "$DEVICE"`                                      | Requires PIN if protected                                                      |
| Remaining PIN attempts         | `fido2-token -I "$DEVICE" \| grep -i attempts`                                | Quick one-liner                                                                |
| List resident keys / passkeys (recommended) | `echo "PIN" \| fido2-cred -L -r "$DEVICE"`                          | Clean output with RP ID + username (best for modern passkeys)                 |
| List resident keys (legacy)    | `echo "PIN" \| fido2-token -L -r "$DEVICE"`                                   | Raw credential IDs only                                                        |
| Verify PIN (script-friendly)   | `echo "PIN" \| fido2-token -V "$DEVICE" && echo "PIN OK"`                    | Returns 0 on success → perfect for scripts                                     |
| Factory reset (wipe everything)| `sudo fido2-token -R "$DEVICE"`                                              | Works even when PIN-locked; **irreversible**                                   |
| Set first PIN (no PIN yet)     | `fido2-token -S "$DEVICE"`                                                  | Interactive prompt; min 4 characters                                           |
| Change existing PIN            | `fido2-token -C "$DEVICE"`                                                  | Prompts old → new PIN                                                          |

### Quick script-friendly one-liners
```bash
# Replace PIN with your actual PIN or use a secure method (read -s, etc.)
PIN="123456"

echo "$PIN" | fido2-cred -L -r "$DEVICE"                  # list passkeys
echo "$PIN" | fido2-token -V "$DEVICE"                    # verify + get version
echo "$PIN" | fido2-token -L -r "$DEVICE"                 # legacy resident key list
```

**Tip:** For production scripts, avoid `echo "$PIN"` — use `read -s PIN` or a keyring instead.

Enjoy your hardware-bound passwordless future!
```

Just paste this into your git repo (e.g., `docs/fido2-cheatsheet.md` or `notes/fido2.md`) and commit. It’s accurate as of libfido2 1.14+ (2025) and works with YubiKeys, OnlyKey, Nitrokey 3, SoloKeys, etc.
