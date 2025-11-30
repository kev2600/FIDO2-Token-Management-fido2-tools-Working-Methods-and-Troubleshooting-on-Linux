# FIDO2 Token Management (fido2-tools): Working Methods and Troubleshooting on Linux

This guide provides a comprehensive command-line reference and confirmed working procedures for administering FIDO2 hardware tokens using the `fido2-tools` package on Linux (e.g., Fedora, Nobara).

> ⚠️ **CRITICAL NOTE ON DELETION**  
> The command-line deletion of single passkeys (`fido2-token -D`) is often **UNRELIABLE** due to a persistent `FIDO_ERR_NO_CREDENTIALS` bug in the underlying library when dealing with specific key types. The only reliable way to remove a stuck credential is the **Factory Reset (`-R`)**.

## Setup Note

All commands use a placeholder for the device path. You MUST replace `"${DEVICE_PATH}"` with the actual path found by `fido2-token -L` (e.g., `/dev/hidraw9`).

```bash
# Placeholder variable for easier copy/paste. Replace /dev/hidrawX with your actual path.
DEVICE_PATH="/dev/hidrawX"
```

## 1. Installation

Installation command for Fedora/Nobara systems:

```bash
sudo dnf install fido2-tools
```

## 2. Device Discovery and Information

### Find Device Path

Identify the key's device path (e.g., `/dev/hidraw9`):

```bash
fido2-token -L
```

### Get Detailed Key Information

Outputs firmware version, AAGUID, and key capabilities:

```bash
fido2-token -I "${DEVICE_PATH}"
# Example: fido2-token -I /dev/hidraw9
```

### Check Passkey Capacity

Requires PIN. Shows the count of resident keys stored and remaining capacity:

```bash
fido2-token -I -c "${DEVICE_PATH}"
# Example: fido2-token -I -c /dev/hidraw9
```

### List Stored Passkeys (Resident Keys)

Requires PIN. Displays list index, Base64 ID, and RP ID:

```bash
fido2-token -L -r "${DEVICE_PATH}"
# Example: fido2-token -L -r /dev/hidraw9
```

> **Note:** If no credentials exist (e.g., after Factory Reset), the command prompts for PIN and returns **no output**, confirming a clean key.

## 3. PIN Management

### Set Initial PIN (New or Reset Key)

This procedure is **time-sensitive** due to FIDO protocol security requirements.

> ❗ **TIMING IS CRITICAL:** Requires UNPLUGGING and REPLUGGING the key. Run the command **within ~5 seconds** after replugging.

```bash
fido2-token -S "${DEVICE_PATH}"
# Example: fido2-token -S /dev/hidraw9
```

### Change Existing PIN

Operates while the key is in normal state:

```bash
# Does NOT require replugging. Requires entering the existing PIN.
fido2-token -C "${DEVICE_PATH}"
# Example: fido2-token -C /dev/hidraw9
```

## 4. Passkey Deletion & Factory Reset

### UNRELIABLE Deletion (For Testing Only)

Attempts to delete a single passkey by ID. Often fails:

```bash
# Replace [BASE64_ID] and [RP_ID] with values from the list command.
fido2-token -D -i '[BASE64_ID]' -r [RP_ID] "${DEVICE_PATH}"
# Example: fido2-token -D -i '1MnZAnMmJx...=' -r google.com /dev/hidraw9
```

### Factory Reset (The Reliable Solution)

Use when single credential deletion fails or to restore factory state:

> ⚠️ **WARNING:** Deletes **ALL** credentials and PINs.  
> **Procedure:** Unplug key, plug in, and run command immediately (within 10 seconds).

```bash
fido2-token -R "${DEVICE_PATH}"
```

### Post-Reset Initialization

After reset, you **MUST** set a new PIN using the `-S` command (Section 3) immediately to make the key usable again.
