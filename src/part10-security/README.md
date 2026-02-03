# Security on macOS

macOS employs a multi-layered security model that goes far beyond traditional Unix permissions. While it retains the foundational user/group/world permission system from its BSD heritage, Apple has built an extensive security architecture on top, including code signing, sandboxing, encrypted storage, and hardware-backed security features. Understanding these layers is essential for anyone administering macOS systems or developing software for the platform.

## The Security Philosophy

Apple's approach to macOS security follows a defense-in-depth strategy:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hardware Security                             │
│          (Secure Enclave, T2/Apple Silicon, Secure Boot)        │
├─────────────────────────────────────────────────────────────────┤
│                    Disk Encryption                               │
│                      (FileVault 2)                               │
├─────────────────────────────────────────────────────────────────┤
│                    Kernel Protection                             │
│            (SIP, Kernel Extension Signing, KTRR)                │
├─────────────────────────────────────────────────────────────────┤
│                    Application Security                          │
│      (Gatekeeper, Notarization, Sandboxing, Hardened Runtime)   │
├─────────────────────────────────────────────────────────────────┤
│                    Privacy Controls                              │
│          (TCC, Privacy Preferences, Transparency)               │
├─────────────────────────────────────────────────────────────────┤
│                    Traditional Unix Security                     │
│          (Users, Groups, Permissions, ACLs)                     │
└─────────────────────────────────────────────────────────────────┘
```

Each layer provides protection even if another layer is compromised.

## Key Security Components

### Code Signing and Trust

macOS verifies that software comes from identified developers:

| Component | Purpose |
|-----------|---------|
| Code Signing | Cryptographically verifies software identity |
| Gatekeeper | Enforces signature requirements for app execution |
| Notarization | Apple's automated security check for distributed software |
| Hardened Runtime | Restricts dangerous operations in signed code |

### Privacy and Sandboxing

Applications are isolated and must request permission for sensitive operations:

| Component | Purpose |
|-----------|---------|
| App Sandbox | Restricts app file system and resource access |
| TCC (Transparency, Consent, Control) | Manages privacy permissions |
| Entitlements | Declares capabilities an app needs |

### Hardware-Backed Security

Modern Macs include dedicated security hardware:

| Component | Purpose |
|-----------|---------|
| Secure Enclave | Isolated processor for cryptographic operations |
| T2 / Apple Silicon | Secure boot, encryption keys, biometrics |
| Touch ID | Biometric authentication |

## Command-Line Security Tools

macOS provides extensive security tools for the terminal:

### Code Signing and Verification

```bash
# Verify an application's signature
$ codesign -dv --verbose=4 /Applications/Safari.app

# Check Gatekeeper assessment
$ spctl --assess -v /Applications/Safari.app

# View extended attributes (quarantine)
$ xattr -l ~/Downloads/installer.pkg
```

### Keychain and Credentials

```bash
# List keychains
$ security list-keychains

# Find a password
$ security find-generic-password -a "account" -s "service" -w

# Add a certificate to keychain
$ security add-certificates /path/to/cert.cer
```

### Privacy and TCC

```bash
# Reset privacy permissions for an app
$ tccutil reset Camera com.example.app

# Check privacy database (requires Full Disk Access)
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT * FROM access"
```

### Disk Encryption

```bash
# Check FileVault status
$ fdesetup status
FileVault is On.

# List enabled users
$ fdesetup list

# Check encryption progress
$ diskutil apfs list | grep -A5 "FileVault"
```

### System Security

```bash
# Check SIP status
$ csrutil status

# Check Gatekeeper status
$ spctl --status

# View firmware password status
$ sudo firmwarepasswd -check
```

## Quick Security Audit

A rapid assessment of system security posture:

```bash
# Check SIP
$ csrutil status

# Check FileVault
$ fdesetup status

# Check Gatekeeper
$ spctl --status

# Check firewall
$ /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# Check auto-login status
$ defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser 2>/dev/null || echo "Auto-login disabled"

# Check remote access services
$ sudo systemsetup -getremotelogin
$ sudo launchctl list | grep -E "screensharing|vnc|ARD"
```

## Security on Apple Silicon vs Intel

Apple Silicon Macs have enhanced security features:

| Feature | Intel (T2) | Intel (No T2) | Apple Silicon |
|---------|------------|---------------|---------------|
| Secure Boot | Yes | No | Yes |
| Hardware Encryption | Yes | Software only | Yes |
| Secure Enclave | Yes | No | Yes |
| Boot Security Modes | Limited | No | Full |
| Kernel Extension Loading | Restricted | Allowed | Very Restricted |

## What You'll Learn in This Part

**[Gatekeeper and Code Signing](./gatekeeper-signing.md)** explains how macOS verifies software authenticity using `codesign`, `spctl`, and the quarantine system, including how to sign your own tools and scripts.

**[Notarization Requirements](./notarization.md)** covers Apple's notarization process for distributed software, using `xcrun notarytool` and stapling tickets to your applications.

**[App Sandboxing](./sandboxing.md)** explores how sandboxing isolates applications, including the `sandbox-exec` command, sandbox profiles, and entitlements.

**[Keychain Services from Terminal](./keychain-cli.md)** shows how to manage passwords, certificates, and keys using the `security` command for automation and scripting.

**[FileVault and Disk Encryption](./filevault.md)** covers full-disk encryption management using `fdesetup`, including enabling, key management, and recovery scenarios.

**[Privacy Controls and TCC Database](./tcc-privacy.md)** explains the Transparency, Consent, and Control system, including `tccutil` and programmatic permission handling.

**[Secure Boot and T2/Apple Silicon](./secure-boot.md)** details hardware security features, boot security policies, and firmware password configuration.

**[Security Best Practices](./best-practices.md)** provides a comprehensive hardening checklist and ongoing security monitoring strategies.

## Common Security Tasks

### Allow an App Blocked by Gatekeeper

```bash
# Remove quarantine attribute
$ xattr -d com.apple.quarantine /path/to/app

# Or add to Gatekeeper whitelist
$ spctl --add --label "Approved" /path/to/app
```

### Securely Store a Password in Script

```bash
# Store in keychain
$ security add-generic-password -a "$USER" -s "myservice" -w "secret"

# Retrieve in script
PASSWORD=$(security find-generic-password -a "$USER" -s "myservice" -w)
```

### Check If an App Is Notarized

```bash
$ spctl -a -vvv /Applications/SomeApp.app
/Applications/SomeApp.app: accepted
source=Notarized Developer ID
```

### Grant Terminal Full Disk Access

1. Open System Preferences > Privacy & Security > Full Disk Access
2. Click the lock to make changes
3. Add Terminal.app (or your terminal emulator)
4. Restart Terminal

This is required for many security-related terminal operations.

The following chapters provide in-depth coverage of each security component, with practical examples for both security auditing and system hardening.
