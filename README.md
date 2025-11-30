# FIDO2 Security Key Management Cheat Sheet (libfido2 tools)

These commands use `fido2-token` and `fido2-cred` from the official `libfido2` toolkit (Yubico/maintained FIDO2 tools on Linux). Accurate as of libfido2 1.16.0 (2025-05-06). Works with YubiKeys, OnlyKey, Nitrokey 3, SoloKeys, etc.

**First step – always identify your device:**
```bash
fido2-token -L
# Example output: /dev/hidraw6 Yubikey 5 NFC
export DEVICE=/dev/hidraw6 # set once per session
```

**Notes on PIN handling:**  
- If your device is PIN-protected, operations like `-V`, `-I` (for full info), `-L -r`, or `-L -r` (cred) require authentication.  
- Use `echo "your-pin" | command` for scripting, or omit for interactive prompts (default if TTY available).  
- Replace `"PIN"` in examples with your actual PIN. For security in scripts, use `read -s PIN` or a keyring.  
- No PIN set? Commands like `-V` will fail until you set one with `-S`.

| Purpose | Command | Notes / Output |
|---------|---------|----------------|
| List all connected tokens | `fido2-token -L` | Shows device path and model (e.g., `/dev/hidraw6 Yubikey 5 NFC`). No PIN needed. Set `$DEVICE` from output. |
| Device info (AAGUID, capabilities, PIN attempts…) | `fido2-token -I "$DEVICE"` | Human-readable details: VID/PID, AAGUID, versions, options, remaining PIN attempts. PIN required if protected. |
| Firmware version | `echo "PIN" \| fido2-token -V "$DEVICE"` | Outputs protocol and firmware versions (e.g., `2.1 5.7.3.4`). PIN required if protected. |
| Remaining PIN attempts | `fido2-token -I "$DEVICE" \| grep -i attempts` | Extracts "PIN (remaining attempts = X)" from info output. Requires `-I` (PIN if protected). |
| List resident keys / passkeys (recommended) | `echo "PIN" \| fido2-cred -L -r "$DEVICE"` | Lists RP ID, username, and cred ID for discoverable credentials. Clean for modern passkeys. PIN required. |
| List resident keys (legacy) | `echo "PIN" \| fido2-token -L -r "$DEVICE"` | Raw credential IDs only. Use `-cred` for fuller details. PIN required. |
| Verify PIN (script-friendly) | `echo "PIN" \| fido2-token -V "$DEVICE" && echo "PIN OK"` | Exits 0 on success (prints version), non-zero on failure. Ideal for if-statements in scripts. PIN input required. |
| Factory reset (wipe everything) | `sudo fido2-token -R "$DEVICE"` | Irreversible: Deletes all credentials, PIN, etc. No PIN needed; no confirmation prompt. Use with extreme caution. |
| Set first PIN (no PIN yet) | `fido2-token -S "$DEVICE"` | Interactive: Prompts for new PIN (min 4 chars, max 63 for most devices). No old PIN needed. |
| Change existing PIN | `fido2-token -C "$DEVICE"` | Interactive: Prompts for old PIN, then new PIN. Fails if old PIN wrong (locks after attempts). |

### Quick script-friendly one-liners
```bash
# Replace PIN with your actual PIN or use a secure method (read -s, etc.)
PIN="123456"
echo "$PIN" | fido2-cred -L -r "$DEVICE" # list passkeys (RP ID + user + cred ID)
echo "$PIN" | fido2-token -V "$DEVICE" # verify PIN + get versions
echo "$PIN" | fido2-token -L -r "$DEVICE" # legacy resident key list (cred IDs only)

# Example: Check PIN and list keys if OK
if echo "$PIN" | fido2-token -V "$DEVICE" >/dev/null 2>&1; then
  echo "PIN OK"
  echo "$PIN" | fido2-cred -L -r "$DEVICE"
else
  echo "PIN failed"
fi
```

**Troubleshooting tips:**  
- **Permission errors?** Add your user to `plugdev` group (`sudo usermod -aG plugdev $USER`) and reload udev rules. Or use `sudo` for testing.  
- **No output or errors?** Ensure libfido2 is installed (`fido2-token --version`) and device is inserted/unlocked.  
- **Interactive mode:** Drop `echo "PIN" |` for TTY prompts (hides PIN entry).  
- **More help:** Run `man fido2-token` or `man fido2-cred` for full options. For credential management (make/delete), see `fido2-cred -M` or `fido2-cred -D`.

**Tip:** For production scripts, avoid hardcoding PINs—use `read -s PIN` or integrate with a password manager/keyring.  

Enjoy your hardware-bound passwordless future!  
