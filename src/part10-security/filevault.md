# FileVault and Disk Encryption

FileVault is macOS's full-disk encryption technology that protects all data on your startup disk. Using XTS-AES-128 encryption with a 256-bit key, FileVault ensures that your data remains unreadable without proper authentication, even if someone physically removes your drive or steals your Mac. For anyone handling sensitive data, FileVault is essential.

## How FileVault Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    FileVault Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User Password ──┐                                             │
│                   ├── Unlocks ──▶ Volume Master Key             │
│   Recovery Key ───┘                      │                      │
│                                          │                      │
│   iCloud Recovery ─── Optionally ──▶ Recovery via Apple ID     │
│                                          │                      │
│                                          ▼                      │
│                           ┌──────────────────────────┐          │
│                           │   XTS-AES-128 Encrypted  │          │
│                           │      APFS Volume         │          │
│                           │                          │          │
│                           │   macOS System + Data    │          │
│                           │                          │          │
│                           └──────────────────────────┘          │
│                                                                  │
│   Hardware Keys ───▶ Secure Enclave (T2/Apple Silicon)          │
│   (Additional layer on supported hardware)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### FileVault 2 vs Original FileVault

FileVault 2 (introduced in OS X Lion):
- Encrypts entire startup disk
- Uses XTS-AES-128 encryption
- On-the-fly encryption/decryption
- Recovery key support
- Institutional key support for enterprises

Original FileVault (deprecated):
- Only encrypted home folders
- Used AES-128 in CBC mode
- Created encrypted disk images

## The fdesetup Command

The `fdesetup` command is the primary tool for managing FileVault from the terminal.

### Checking FileVault Status

```bash
# Check if FileVault is enabled
$ fdesetup status
FileVault is On.

# Or if disabled
$ fdesetup status
FileVault is Off.

# Check encryption progress (during initial encryption)
$ fdesetup status
FileVault is On.
Encryption in progress: Percent complete = 45.00

# Detailed status with verbose output
$ sudo fdesetup status -v
FileVault is On.
Volume /dev/disk1s1 is encrypted
```

### Alternative Status Checks

```bash
# Using diskutil
$ diskutil apfs list | grep -A10 "FileVault"

# Check APFS volume encryption status
$ diskutil info disk1s1 | grep "FileVault"
FileVault:                 Yes

# Detailed APFS encryption info
$ diskutil apfs listCryptoUsers disk1s1
Cryptographic users for disk1s1 (3 found)
|
+-- 12345678-1234-1234-1234-123456789012
|   Type: Local Open Directory User
|   User: david
|
+-- ABCDEF01-2345-6789-ABCD-EF0123456789
|   Type: Personal Recovery User
|
+-- FEDCBA98-7654-3210-FEDC-BA9876543210
|   Type: Institutional Recovery User
```

## Enabling FileVault

### Enable with Recovery Key

```bash
# Enable FileVault (prompts for credentials)
$ sudo fdesetup enable

# Enable and output recovery key
$ sudo fdesetup enable -outputplist

# Example output:
# <?xml version="1.0" encoding="UTF-8"?>
# <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
# <plist version="1.0">
# <dict>
#     <key>RecoveryKey</key>
#     <string>XXXX-XXXX-XXXX-XXXX-XXXX-XXXX</string>
# </dict>
# </plist>

# Save recovery key to file
$ sudo fdesetup enable -outputplist > ~/Desktop/recovery-key.plist
```

### Enable with User List

```bash
# Create input plist specifying users
$ cat > /tmp/fv-enable.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Username</key>
    <string>david</string>
    <key>Password</key>
    <string>user-password</string>
</dict>
</plist>
EOF

# Enable with input plist
$ sudo fdesetup enable -inputplist < /tmp/fv-enable.plist

# Clean up (don't leave password in file)
$ rm /tmp/fv-enable.plist
```

### Enable with Institutional Recovery Key

For enterprise environments:

```bash
# Create certificate for institutional recovery
$ sudo fdesetup enable \
    -certificate /path/to/FileVaultMaster.cer \
    -outputplist

# The institutional key allows IT to recover encrypted Macs
# without individual user passwords
```

### Deferred Enablement

Enable FileVault at next login instead of immediately:

```bash
# Enable deferred enablement
$ sudo fdesetup enable -defer /path/to/recovery-key.plist

# Check deferred status
$ sudo fdesetup status
Deferred enablement appears to be active for user "david".

# The user will be prompted to enable FileVault at next login
# Recovery key will be written to specified plist
```

## Disabling FileVault

```bash
# Disable FileVault (starts decryption)
$ sudo fdesetup disable

# Monitor decryption progress
$ fdesetup status
FileVault is Off.
Decryption in progress: Percent complete = 35.00

# Decryption continues in background
# Mac remains usable during decryption
```

**Warning:** Disabling FileVault leaves data unencrypted. Only disable when necessary.

## Managing FileVault Users

FileVault-enabled users can unlock the disk at boot:

### List Enabled Users

```bash
# List FileVault-enabled users
$ sudo fdesetup list
david,12345678-1234-1234-1234-123456789012
admin,ABCDEF01-2345-6789-ABCD-EF0123456789

# More detailed listing
$ diskutil apfs listCryptoUsers disk1s1

# Check if specific user is enabled
$ sudo fdesetup isactive -user david
true
```

### Add FileVault User

```bash
# Add user to FileVault (prompts for passwords)
$ sudo fdesetup add -usertoadd newuser

# Add with input plist
$ cat > /tmp/fv-add.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Username</key>
    <string>admin</string>
    <key>Password</key>
    <string>admin-password</string>
    <key>AdditionalUsers</key>
    <array>
        <dict>
            <key>Username</key>
            <string>newuser</string>
            <key>Password</key>
            <string>newuser-password</string>
        </dict>
    </array>
</dict>
</plist>
EOF

$ sudo fdesetup add -inputplist < /tmp/fv-add.plist
$ rm /tmp/fv-add.plist
```

### Remove FileVault User

```bash
# Remove user from FileVault
$ sudo fdesetup remove -user username

# The user's account remains, but they can't unlock at boot
```

### Synchronize User Password

When a user's password changes, FileVault needs to be synchronized:

```bash
# Sync user's FileVault credentials after password change
$ sudo fdesetup sync

# Or for specific user
$ sudo fdesetup changerecovery -user david
```

## Recovery Key Management

### View Recovery Key Type

```bash
# Check if personal or institutional recovery key is set
$ sudo fdesetup haspersonalrecoverykey
true

$ sudo fdesetup hasinstitutionalrecoverykey
false
```

### Change Recovery Key

```bash
# Generate new personal recovery key
$ sudo fdesetup changerecovery -personal -outputplist

# Output new key
# <?xml version="1.0" encoding="UTF-8"?>
# ...
#     <key>RecoveryKey</key>
#     <string>NEW-XXXX-XXXX-XXXX-XXXX-XXXX</string>
# ...

# Change to institutional recovery key
$ sudo fdesetup changerecovery -institutional -certificate /path/to/cert.cer
```

### Using Recovery Key

If you forget your password, use the recovery key at the login screen:
1. At the login window, enter wrong password three times
2. Option to use recovery key appears
3. Enter the 24-character recovery key

From command line (Recovery Mode):

```bash
# Boot to Recovery Mode, open Terminal
$ diskutil apfs unlockVolume disk1s1 -passphrase "XXXX-XXXX-XXXX-XXXX-XXXX-XXXX"
```

## FileVault with T2 and Apple Silicon

On Macs with T2 chip or Apple Silicon, FileVault integrates with hardware security:

### T2 Macs

```bash
# T2 provides hardware encryption keys
# FileVault adds additional software layer

# Check secure boot status (affects FileVault)
$ sudo bputil -d

# On T2, encryption keys are tied to Secure Enclave
# Data is always encrypted at hardware level
# FileVault adds authentication requirement
```

### Apple Silicon Macs

```bash
# Apple Silicon always encrypts data
# FileVault adds authentication layer

# Check encryption status
$ diskutil apfs list

# Data Protection Class:
# Class A - Protected Until First User Authentication
# Class B - Protected Unless Open
# Class C - Protected Until First User Authentication (default)
# Class D - No Protection
```

## Monitoring Encryption Progress

```bash
# Monitor initial encryption
$ while true; do
    status=$(fdesetup status)
    echo "$(date): $status"
    if echo "$status" | grep -q "complete"; then
        break
    fi
    sleep 60
done

# More detailed progress
$ diskutil apfs list | grep -A5 "Encryption"

# Watch encryption progress
$ watch -n 30 'fdesetup status'
```

## Scripting FileVault Management

### Enable FileVault Silently

```bash
#!/bin/bash
# enable-filevault.sh - Enable FileVault programmatically

USERNAME="admin"
PASSWORD="$1"  # Pass password as argument
OUTPUT_DIR="/var/log/filevault"

if [[ -z "$PASSWORD" ]]; then
    echo "Usage: $0 <admin-password>"
    exit 1
fi

# Check if already enabled
if fdesetup status | grep -q "FileVault is On"; then
    echo "FileVault is already enabled"
    exit 0
fi

# Create output directory
mkdir -p "$OUTPUT_DIR"

# Create input plist
PLIST=$(mktemp)
cat > "$PLIST" << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Username</key>
    <string>$USERNAME</string>
    <key>Password</key>
    <string>$PASSWORD</string>
</dict>
</plist>
EOF

# Enable FileVault
sudo fdesetup enable -inputplist < "$PLIST" -outputplist > "$OUTPUT_DIR/recovery-key.plist"

# Secure the recovery key file
chmod 600 "$OUTPUT_DIR/recovery-key.plist"

# Clean up
rm -f "$PLIST"

# Verify
fdesetup status

echo "Recovery key saved to $OUTPUT_DIR/recovery-key.plist"
echo "IMPORTANT: Secure this file immediately!"
```

### Audit FileVault Status

```bash
#!/bin/bash
# filevault-audit.sh - Audit FileVault configuration

echo "=== FileVault Security Audit ==="
echo "Date: $(date)"
echo "Hostname: $(hostname)"
echo

echo "--- FileVault Status ---"
fdesetup status

echo
echo "--- Enabled Users ---"
sudo fdesetup list 2>/dev/null || echo "Unable to list users (requires admin)"

echo
echo "--- Recovery Key Status ---"
echo -n "Personal Recovery Key: "
sudo fdesetup haspersonalrecoverykey 2>/dev/null || echo "Unknown"
echo -n "Institutional Recovery Key: "
sudo fdesetup hasinstitutionalrecoverykey 2>/dev/null || echo "Unknown"

echo
echo "--- Volume Encryption ---"
diskutil apfs list | grep -A3 "FileVault"

echo
echo "--- Hardware Security ---"
if system_profiler SPiBridgeDataType 2>/dev/null | grep -q "T2"; then
    echo "T2 Security Chip: Present"
elif [[ $(uname -m) == "arm64" ]]; then
    echo "Apple Silicon: Yes"
else
    echo "Hardware Security: Not detected"
fi

echo
echo "=== Audit Complete ==="
```

## Troubleshooting FileVault

### FileVault Won't Enable

```bash
# Check for issues
$ sudo fdesetup status -v

# Verify secure token
$ sysadminctl -secureTokenStatus admin
Secure token is ENABLED for user "admin"

# Users need secure token to enable FileVault
# Grant secure token
$ sudo sysadminctl -adminUser admin -adminPassword - \
    -secureTokenOn newuser -password -
```

### Secure Token Issues

```bash
# Check which users have secure tokens
$ sudo fdesetup list

# Bootstrap token (for MDM environments)
$ sudo profiles status -type bootstraptoken
Bootstrap Token escrowed to server: NO

# Create bootstrap token
$ sudo profiles install -type bootstraptoken
```

### Encryption Stalled

```bash
# Check for errors
$ sudo fdesetup status

# View system logs
$ log show --predicate 'subsystem == "com.apple.corestorage"' --last 1h

# Force encryption to continue
$ sudo fdesetup haspersonalrecoverykey
# This can sometimes restart stalled encryption
```

### Recovery Mode Unlock

```bash
# Boot to Recovery Mode
# Open Terminal from Utilities menu

# List volumes
$ diskutil list

# Unlock FileVault volume
$ diskutil apfs unlockVolume disk1s1

# Or with recovery key
$ diskutil apfs unlockVolume disk1s1 -passphrase "XXXX-XXXX-XXXX-XXXX-XXXX-XXXX"
```

## Best Practices

### Enable FileVault

Every Mac with sensitive data should have FileVault enabled:

```bash
# Verify FileVault is on
$ fdesetup status
FileVault is On.
```

### Secure Recovery Key

- Store recovery key in secure location (password manager, safe)
- For enterprises, escrow to MDM or use institutional key
- Never store recovery key on the encrypted disk

### Multiple FileVault Users

- Enable FileVault for all users who need to boot the Mac
- Remove departed employees from FileVault
- Regularly audit enabled users

```bash
# Regular audit
$ sudo fdesetup list
```

### Test Recovery

Periodically verify recovery key works:

1. Boot to Recovery Mode
2. Open Terminal
3. Test unlock with recovery key
4. Lock volume again
5. Boot normally

## Summary

FileVault provides essential full-disk encryption for macOS:

| Command | Purpose |
|---------|---------|
| `fdesetup status` | Check FileVault status |
| `fdesetup enable` | Enable FileVault |
| `fdesetup disable` | Disable FileVault |
| `fdesetup list` | List enabled users |
| `fdesetup add` | Add FileVault user |
| `fdesetup remove` | Remove FileVault user |
| `fdesetup changerecovery` | Change recovery key |

Key concepts:

- FileVault encrypts the entire startup disk
- Users must be enabled to unlock at boot
- Recovery key allows access if password forgotten
- T2/Apple Silicon adds hardware encryption layer
- Always securely store recovery key off-device

```bash
# Quick reference
$ fdesetup status                          # Check status
$ sudo fdesetup enable -outputplist        # Enable and get key
$ sudo fdesetup list                       # List enabled users
$ sudo fdesetup add -usertoadd newuser     # Add user
$ diskutil apfs list | grep FileVault      # Check APFS encryption
```

FileVault should be enabled on every Mac containing sensitive data. With proper key management, it provides strong protection against data theft.
