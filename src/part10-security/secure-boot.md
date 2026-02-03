# Secure Boot on T2 and Apple Silicon

Modern Macs include dedicated security hardware that ensures the boot process is protected from tampering. Starting with the T2 Security Chip (2017) and continuing with Apple Silicon (2020), Macs verify every stage of the boot process cryptographically. Understanding these security features is essential for IT administrators managing Mac fleets and developers who need to load custom kernel extensions.

## Hardware Security Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Mac Security Architecture                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────┐    ┌─────────────────────┐           │
│   │    Main Processor   │    │   Security Chip     │           │
│   │    (Intel/Apple)    │    │   (T2/Secure Encl)  │           │
│   │                     │    │                     │           │
│   │   macOS / Apps      │    │  - Boot ROM         │           │
│   │                     │    │  - Secure Boot      │           │
│   │                     │◄───│  - Touch ID         │           │
│   │                     │    │  - Encryption Keys  │           │
│   │                     │    │  - System Integrity │           │
│   └─────────────────────┘    └─────────────────────┘           │
│                                                                  │
│   Secure Boot Chain:                                            │
│   Boot ROM ─▶ iBoot ─▶ macOS Kernel ─▶ System Extensions       │
│      │          │           │                │                  │
│      └──────────┴───────────┴────────────────┘                  │
│           Each stage verifies the next                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## T2 Security Chip

The T2 chip (found in Intel Macs from 2018-2020) provides:

### Features

| Feature | Description |
|---------|-------------|
| Secure Boot | Verifies boot process integrity |
| Encrypted Storage | Hardware encryption keys |
| Touch ID | Biometric authentication |
| Secure Enclave | Isolated security processor |
| Image Signal Processor | Camera processing |
| Audio Controller | Microphone disconnect |

### Checking T2 Status

```bash
# Check if Mac has T2 chip
$ system_profiler SPiBridgeDataType
Apple T2 Security Chip:

  Model Name: Apple T2 Security Chip
  Model Identifier: T2Chip1,1
  ...

# Alternative check
$ ioreg -l | grep "T2" | head -1

# Check T2 firmware version
$ system_profiler SPiBridgeDataType | grep "Firmware Version"
```

## Apple Silicon Security

Apple Silicon Macs (M1, M2, M3, etc.) integrate security directly into the main chip:

### Features

| Feature | Description |
|---------|-------------|
| Secure Enclave | Isolated security coprocessor |
| Boot ROM | Hardware root of trust |
| Secure Boot | Cryptographic verification |
| Pointer Authentication | Code integrity protection |
| Kernel Integrity Protection | Runtime kernel verification |
| Device Isolation | Hardware memory protection |

### Checking Apple Silicon

```bash
# Check processor type
$ uname -m
arm64

# View Apple Silicon details
$ system_profiler SPHardwareDataType | grep -i "chip"
      Chip: Apple M1 Pro

# View security features
$ sysctl -a | grep -i "arm64"
```

## Boot Security Modes

Both T2 and Apple Silicon Macs support different security levels:

### Security Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| Full Security | Only Apple-signed software | Default, recommended |
| Reduced Security | Allows notarized kexts | Third-party kexts |
| Permissive Security | No verification (Apple Silicon) | Development only |

### Checking Security Mode

```bash
# On Apple Silicon
$ sudo bputil -d
Current OS environment:
  OS Type: macOS
  Boot Mode: Normal
  Security Mode: Full Security
  ...

# Alternative (works on both)
$ csrutil status
System Integrity Protection status: enabled.

# Detailed Apple Silicon security
$ sudo bputil -s
```

## Startup Security Utility

### Accessing on Intel T2 Macs

1. Restart Mac
2. Hold **Command + R** during startup
3. Select **Utilities > Startup Security Utility**
4. Authenticate with admin credentials

### Accessing on Apple Silicon Macs

1. Shut down Mac completely
2. Press and hold **Power button** until "Loading startup options"
3. Click **Options** > **Continue**
4. Select **Utilities > Startup Security Utility**

### Command-Line Equivalents

```bash
# View current security policy (Apple Silicon)
$ sudo bputil -d

# Change security policy (Recovery Mode required)
# From Terminal in Recovery Mode:

# Set Full Security
$ sudo bputil --full-security

# Set Reduced Security
$ sudo bputil --reduced-security

# Enable kernel extension loading (Reduced Security)
$ sudo bputil --reduced-security --allow-kexts

# Enable MDM management
$ sudo bputil --reduced-security --allow-mdm-enrollment

# Reset to defaults
$ sudo bputil --default
```

## Firmware Password

A firmware password prevents unauthorized users from starting from different disks or entering Recovery Mode.

### Setting Firmware Password (Intel Macs)

```bash
# Check firmware password status
$ sudo firmwarepasswd -check
Password Enabled: No

# Set firmware password (prompts for password)
$ sudo firmwarepasswd -setpasswd
Enter new password:
Re-enter new password:
Password set successfully.

# Verify
$ sudo firmwarepasswd -check
Password Enabled: Yes

# Delete firmware password
$ sudo firmwarepasswd -delete
Enter password:
Password removed successfully.

# Change firmware password
$ sudo firmwarepasswd -setpasswd
```

### Firmware Password Modes

```bash
# Set to command mode (requires password for startup key combos)
$ sudo firmwarepasswd -setmode command
# Protects: Option boot, Recovery Mode, etc.

# Set to full mode (requires password for any boot)
$ sudo firmwarepasswd -setmode full
# Protects: All startup scenarios
```

### Apple Silicon Recovery Password

Apple Silicon Macs don't have traditional firmware passwords. Instead:

1. Recovery Mode requires authentication with a user account
2. Activation Lock (tied to Apple ID) provides additional protection
3. MDM can set restrictions

```bash
# On Apple Silicon, view security settings
$ sudo bputil -d

# Activation Lock status (check in System Information)
$ system_profiler SPHardwareDataType | grep "Activation Lock"
```

## Secure Boot Policies

### Local Policy (Apple Silicon)

Each bootable OS has its own security policy:

```bash
# View all boot policies
$ sudo bputil -l

# View policy for specific OS
$ sudo bputil -d --volume-path /Volumes/MacintoshHD

# OS security policies include:
# - Signature requirements
# - Auxiliary kernel extensions
# - MDM enrollment status
```

### Boot Policy Changes

```bash
# In Recovery Mode Terminal:

# Allow user management of kernel extensions
$ sudo bputil -g

# The -g flag enables:
# - Loading third-party kexts
# - Reduced security for the OS

# After running, restart and:
# System Preferences > Security > Allow apps from identified developers
```

## Kernel Extension Loading

Modern Macs restrict kernel extension loading for security:

### Kext Loading on T2 Macs

```bash
# Check kext loading policy
$ sudo spctl kext-consent status
Kernel Extension User Consent: ENABLED

# Add team ID to allowed list (must be in Recovery Mode)
$ sudo spctl kext-consent add TEAMID12345

# List allowed team IDs
$ sudo spctl kext-consent list

# Disable user consent (not recommended)
$ sudo spctl kext-consent disable
```

### Kext Loading on Apple Silicon

```bash
# Must be in Reduced Security mode
$ sudo bputil --reduced-security

# Then from normal boot, approve kexts in:
# System Preferences > Security & Privacy

# Check loaded kexts
$ kextstat | head -10

# View kext loading errors
$ sudo kextutil -v /path/to/kext.kext
```

### System Extensions (Modern Alternative)

Apple encourages System Extensions over kernel extensions:

```bash
# List system extensions
$ systemextensionsctl list

# System extensions run in user space
# Safer than kernel extensions
# Required for App Store apps

# Uninstall system extension
$ systemextensionsctl uninstall TEAMID com.example.extension
```

## External Boot

Control whether the Mac can boot from external devices:

### T2 Macs

```bash
# In Startup Security Utility (Recovery Mode):
# "Allowed Boot Media":
# - Full Security: Only internal disk
# - Medium Security: Allows Apple-signed external media
# - No Security: Allows any bootable external media
```

### Apple Silicon

```bash
# External boot requires:
# 1. Reduced Security mode
# 2. Explicit approval for external media

# In Recovery Mode:
$ sudo bputil --reduced-security

# Then select "Security Policy" for external volume
# and approve in Startup Security Utility
```

## Startup Key Combinations

Key combinations available at boot (may require firmware password):

| Keys | Function |
|------|----------|
| **Option (Alt)** | Startup Manager |
| **Command + R** | Recovery Mode |
| **Command + Option + R** | Internet Recovery |
| **Shift** | Safe Mode |
| **D** | Apple Diagnostics |
| **N** | Network Startup |
| **T** | Target Disk Mode (Intel) |
| **Command + V** | Verbose Mode |

```bash
# Note: On Apple Silicon, hold Power button instead for Recovery
```

## Security Audit Commands

```bash
#!/bin/bash
# boot-security-audit.sh - Audit Mac boot security

echo "=== Boot Security Audit ==="
echo "Date: $(date)"
echo

echo "--- Hardware Type ---"
if [[ $(uname -m) == "arm64" ]]; then
    echo "Platform: Apple Silicon"
    system_profiler SPHardwareDataType | grep "Chip"
elif system_profiler SPiBridgeDataType 2>/dev/null | grep -q "T2"; then
    echo "Platform: Intel with T2"
    system_profiler SPiBridgeDataType | grep "Model Name"
else
    echo "Platform: Intel (No T2)"
fi

echo
echo "--- SIP Status ---"
csrutil status

echo
echo "--- Secure Boot Policy ---"
if [[ $(uname -m) == "arm64" ]]; then
    sudo bputil -d 2>/dev/null || echo "Run with sudo for detailed info"
fi

echo
echo "--- Firmware Password ---"
if [[ $(uname -m) != "arm64" ]]; then
    sudo firmwarepasswd -check 2>/dev/null || echo "Unknown"
else
    echo "N/A (Apple Silicon uses Recovery authentication)"
fi

echo
echo "--- Kext Consent ---"
sudo spctl kext-consent status 2>/dev/null

echo
echo "--- Gatekeeper ---"
spctl --status

echo
echo "--- FileVault ---"
fdesetup status

echo
echo "--- System Extensions ---"
systemextensionsctl list 2>/dev/null | head -10

echo
echo "=== Audit Complete ==="
```

## Troubleshooting Boot Security

### Can't Boot from External Drive

```bash
# 1. Check security settings in Recovery Mode
# 2. Ensure external drive is properly formatted (APFS)
# 3. For Apple Silicon, approve external boot policy

# Format external drive
$ diskutil eraseDisk APFS "External" GPT disk2
```

### Kext Won't Load

```bash
# Check kext is properly signed
$ codesign -dv /path/to/kext.kext

# Check team ID is allowed
$ sudo spctl kext-consent list

# View kext loading errors
$ log show --predicate 'process == "kernel"' --last 5m | grep -i kext

# Ensure proper security mode
$ csrutil status
$ sudo bputil -d  # Apple Silicon
```

### Boot Stuck or Failing

```bash
# Boot to Recovery Mode
# Use Disk Utility to verify/repair disk
# Check Console for boot errors

# Safe Mode boot (Shift key at startup)
# Disables non-essential kexts

# Verbose Mode (Command + V)
# Shows boot messages for diagnosis
```

## Enterprise Management

### MDM and Secure Boot

```bash
# Check MDM enrollment
$ profiles status -type enrollment
Enrolled via DEP: Yes
MDM enrollment: Yes (User Approved)

# Bootstrap token status (for FileVault and kext management)
$ sudo profiles status -type bootstraptoken

# MDM can configure:
# - Boot security policy
# - Allowed kernel extensions
# - FileVault escrow
# - Activation Lock
```

### Automated Security Configuration

```bash
#!/bin/bash
# enterprise-security-setup.sh - Configure enterprise security

# Verify MDM enrollment
if ! profiles status -type enrollment | grep -q "Yes"; then
    echo "Warning: Device not MDM enrolled"
    exit 1
fi

# Enable FileVault (with MDM escrow)
if fdesetup status | grep -q "Off"; then
    sudo fdesetup enable -defer /tmp/fv-key.plist
fi

# Verify SIP is enabled
if ! csrutil status | grep -q "enabled"; then
    echo "Warning: SIP is disabled"
fi

# Verify Gatekeeper
if ! spctl --status | grep -q "enabled"; then
    sudo spctl --master-enable
fi

echo "Security configuration complete"
```

## Summary

Secure Boot provides hardware-backed protection for Mac startup:

| Component | T2 Macs | Apple Silicon |
|-----------|---------|---------------|
| Security Chip | T2 | Integrated |
| Secure Boot | Yes | Yes |
| Boot ROM | T2 | Main chip |
| Firmware Password | Yes | Recovery auth |
| Security Modes | Full/Medium/No | Full/Reduced/Permissive |
| External Boot Control | Yes | Yes |

Key commands:

```bash
# Check security status
$ csrutil status                       # SIP status
$ sudo bputil -d                       # Boot policy (Apple Silicon)
$ sudo firmwarepasswd -check           # Firmware password (Intel)
$ sudo spctl kext-consent status       # Kext consent

# Recovery Mode operations
$ sudo bputil --full-security          # Set full security
$ sudo bputil --reduced-security       # Allow kexts
$ sudo spctl kext-consent add TEAMID   # Allow kext developer
```

Secure Boot ensures that your Mac starts with verified, trusted software from the moment it powers on, forming the foundation of macOS security.
