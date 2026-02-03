# Gatekeeper and Code Signing

Code signing is the foundation of macOS application security. Every application, framework, plugin, and script that runs on modern macOS should be signed to verify its authenticity and integrity. Gatekeeper enforces these requirements, blocking unsigned or improperly signed software from running. Understanding code signing is essential for developers distributing software and administrators managing which applications can run.

## How Code Signing Works

Code signing creates a cryptographic seal over an application's contents:

```
┌──────────────────────────────────────────────────────────────────┐
│                     Signed Application                            │
├──────────────────────────────────────────────────────────────────┤
│  Code Directory                                                   │
│  ├── Hash of each code page                                      │
│  ├── Hash of Info.plist                                          │
│  ├── Hash of embedded resources                                  │
│  └── Hash of entitlements                                        │
├──────────────────────────────────────────────────────────────────┤
│  CMS Signature                                                    │
│  ├── Developer certificate                                       │
│  ├── Certificate chain to Apple Root CA                          │
│  └── Digital signature of Code Directory                         │
└──────────────────────────────────────────────────────────────────┘
```

When macOS loads signed code, it verifies:
1. The signature matches the code content
2. The signing certificate is valid
3. The certificate chains to a trusted root
4. The certificate hasn't been revoked

## The codesign Command

### Examining Signatures

View basic signature information:

```bash
# Basic signature check
$ codesign -v /Applications/Safari.app
/Applications/Safari.app: valid on disk

# Detailed signature information
$ codesign -dv /Applications/Safari.app
Executable=/Applications/Safari.app/Contents/MacOS/Safari
Identifier=com.apple.Safari
Format=app bundle with Mach-O universal (x86_64 arm64)
CodeDirectory v=20500 size=1012 flags=0x10000(runtime) hashes=21+7 location=embedded
Signature size=4442
Authority=Apple Mac OS Application Signing
Authority=Apple Worldwide Developer Relations Certification Authority
Authority=Apple Root CA
Timestamp=Jan 15, 2024 at 2:30:00 PM
Info.plist entries=35
TeamIdentifier=not applicable
Runtime Version=14.0.0
Sealed Resources version=2 rules=2 files=0
Internal requirements count=1 size=64

# Very verbose output
$ codesign -dv --verbose=4 /Applications/Safari.app

# Display signing requirements
$ codesign -dr - /Applications/Safari.app
```

### Examining Entitlements

Entitlements define what a signed application is allowed to do:

```bash
# View entitlements
$ codesign -d --entitlements - /Applications/Safari.app
Executable=/Applications/Safari.app/Contents/MacOS/Safari
[Dict]
    [Key] com.apple.private.webkit.webinspector.allow
    [Value]
        [Bool] true
    [Key] com.apple.security.app-sandbox
    [Value]
        [Bool] true
    ...

# Extract entitlements to XML
$ codesign -d --entitlements :- /Applications/Safari.app > entitlements.plist

# View as XML (more readable)
$ codesign -d --entitlements - --xml /Applications/Safari.app | plutil -convert xml1 -o - -
```

### Verifying Signatures

```bash
# Quick verification
$ codesign -v /Applications/Safari.app
/Applications/Safari.app: valid on disk

# Verify at a deeper level
$ codesign -vv /Applications/Safari.app
/Applications/Safari.app: valid on disk
satisfies its Designated Requirement

# Strict verification (checks all resources)
$ codesign --verify --strict /Applications/Safari.app

# Verbose verification with details
$ codesign --verify --verbose=4 /Applications/Safari.app
```

### Signing Your Own Code

For ad-hoc signing (no Apple developer account):

```bash
# Sign a binary for local use (ad-hoc signature)
$ codesign -s - /path/to/mybinary

# Sign with specific identifier
$ codesign -s - -i com.example.mytool /path/to/mybinary

# Force re-sign (overwrite existing signature)
$ codesign -f -s - /path/to/mybinary
```

With a Developer ID certificate:

```bash
# List available signing identities
$ security find-identity -v -p codesigning
  1) ABC123... "Developer ID Application: Your Name (TEAMID)"
  2) DEF456... "Apple Development: your@email.com (TEAMID)"
     2 valid identities found

# Sign with Developer ID
$ codesign -s "Developer ID Application: Your Name (TEAMID)" \
  --timestamp \
  --options runtime \
  /path/to/MyApp.app

# Sign with specific entitlements
$ codesign -s "Developer ID Application: Your Name (TEAMID)" \
  --timestamp \
  --options runtime \
  --entitlements entitlements.plist \
  /path/to/MyApp.app
```

### Signing Options

```bash
# Enable hardened runtime (required for notarization)
$ codesign -s "Developer ID Application" \
  --options runtime \
  /path/to/app

# Available options (can be combined with comma)
# runtime    - Enable hardened runtime
# library    - Library validation
# kill       - Kill process on signature invalidation
# hard       - Hard library validation

# Add timestamp (proves when signing occurred)
$ codesign -s "Developer ID Application" \
  --timestamp \
  /path/to/app

# Sign preserving other signatures
$ codesign -s "Developer ID Application" \
  --preserve-metadata=identifier,entitlements,flags \
  /path/to/app
```

### Removing Signatures

```bash
# Remove signature from a binary
$ codesign --remove-signature /path/to/binary

# This is sometimes needed before re-signing
$ codesign --remove-signature /path/to/app && \
  codesign -s "Developer ID Application" /path/to/app
```

## Gatekeeper

Gatekeeper is the macOS subsystem that enforces code signing policy at launch time.

### Checking Gatekeeper Status

```bash
# Check if Gatekeeper is enabled
$ spctl --status
assessments enabled

# If disabled
$ spctl --status
assessments disabled
```

### Managing Gatekeeper

```bash
# Disable Gatekeeper (requires admin, not recommended)
$ sudo spctl --master-disable

# Enable Gatekeeper
$ sudo spctl --master-enable
```

### Gatekeeper Assessments

```bash
# Assess an application
$ spctl --assess -v /Applications/Safari.app
/Applications/Safari.app: accepted
source=Apple System

# Assess with type specification
$ spctl --assess --type execute -v /Applications/SomeApp.app
/Applications/SomeApp.app: accepted
source=Notarized Developer ID

# Possible results:
# accepted - Allowed to run
# rejected - Blocked by Gatekeeper
# source values: Apple System, Apple, Notarized Developer ID, Developer ID, etc.

# Check an installer package
$ spctl --assess --type install -v /path/to/installer.pkg

# Detailed rejection reason
$ spctl -a -t exec -vvv /path/to/app.app
```

### Adding Rules

You can create custom Gatekeeper rules:

```bash
# Add an application to the whitelist
$ spctl --add --label "Approved" /path/to/app.app

# Add by hash (more secure)
$ spctl --add --hash $(codesign -dv --verbose=2 /path/to/app 2>&1 | grep CDHash | cut -d= -f2)

# Add by requirement
$ spctl --add --requirement 'anchor apple generic and certificate leaf[subject.CN] = "Developer ID"'

# List current rules
$ spctl --list

# Remove a rule
$ spctl --remove --label "Approved"
```

### Kernel Extension Consent

```bash
# Check kext consent status
$ spctl kext-consent status
Kernel Extension User Consent: ENABLED

# List approved team IDs
$ spctl kext-consent list

# Add a team ID (requires Recovery Mode)
$ sudo spctl kext-consent add TEAMID123

# Disable kext consent (requires Recovery Mode)
$ spctl kext-consent disable
```

## The Quarantine System

When you download files from the internet, macOS adds a quarantine extended attribute that triggers Gatekeeper assessment on first launch.

### Viewing Quarantine Attributes

```bash
# Check if a file is quarantined
$ xattr /path/to/downloaded.app
com.apple.quarantine

# View quarantine attribute details
$ xattr -p com.apple.quarantine /path/to/downloaded.app
0083;65a12345;Safari;12345678-1234-1234-1234-123456789012

# Format: flags;timestamp_hex;application;UUID
# Flags:
#   0001 = downloaded
#   0002 = do not trigger assessment
#   0040 = user approved (Gatekeeper OK)
#   0080 = App Translocation applied

# View all extended attributes
$ xattr -l /path/to/downloaded.app
```

### Managing Quarantine

```bash
# Remove quarantine (bypass Gatekeeper assessment)
$ xattr -d com.apple.quarantine /path/to/downloaded.app

# Remove quarantine recursively from an app bundle
$ xattr -dr com.apple.quarantine /path/to/MyApp.app

# Check if quarantine attribute exists before removing
$ xattr /path/to/file | grep -q quarantine && \
  xattr -d com.apple.quarantine /path/to/file

# Add quarantine (for testing)
$ xattr -w com.apple.quarantine "0001;$(printf '%x' $(date +%s));Terminal;12345678-1234-1234-1234-123456789012" /path/to/file
```

### App Translocation

When a quarantined app is run from certain locations (like Downloads), macOS may run it from a randomized read-only path (App Translocation):

```bash
# Check if an app is translocated
$ xattr -p com.apple.quarantine /path/to/app
# Look for 0080 flag

# Apps in these locations may be translocated:
# - ~/Downloads
# - Any location opened from a quarantined disk image

# To prevent translocation, move to /Applications:
$ mv ~/Downloads/MyApp.app /Applications/

# Or remove quarantine attribute
$ xattr -dr com.apple.quarantine ~/Downloads/MyApp.app
```

## Code Signing for Scripts

Scripts can be signed too, though it's less common:

```bash
# Sign a shell script
$ codesign -s - /path/to/script.sh

# Verify script signature
$ codesign -v /path/to/script.sh

# Sign with identifier
$ codesign -s - -i com.example.myscript /path/to/script.sh
```

For Python scripts packaged as applications:

```bash
# After using py2app or similar
$ codesign -s "Developer ID Application" \
  --deep \
  --strict \
  --options runtime \
  /path/to/MyPythonApp.app
```

## Certificate Types

Different certificates serve different purposes:

| Certificate Type | Purpose | Distribution |
|------------------|---------|--------------|
| Apple Development | Testing on your devices | Not distributable |
| Apple Distribution | App Store submission | App Store only |
| Developer ID Application | Direct distribution | Outside App Store |
| Developer ID Installer | Signed packages | Outside App Store |

```bash
# View certificate details
$ security find-certificate -c "Developer ID" -p | openssl x509 -noout -text

# List all code signing certificates
$ security find-identity -v -p codesigning
```

## Troubleshooting Code Signing

### Common Errors

```bash
# "not signed at all"
$ codesign -v unsigned.app
unsigned.app: code object is not signed at all

# Solution: Sign the application
$ codesign -s - unsigned.app

# "a sealed resource is missing or invalid"
$ codesign -v damaged.app
damaged.app: a sealed resource is missing or invalid

# Check what resources are problematic
$ codesign --verify --verbose=4 damaged.app 2>&1 | grep -i resource

# Solution: Re-sign after ensuring all resources are present
$ codesign -f -s "Developer ID Application" damaged.app

# "the signature is invalid"
# Usually means the binary was modified after signing
$ codesign --remove-signature app.app
$ codesign -s "Developer ID Application" app.app
```

### Deep Signing

For app bundles with nested code:

```bash
# Sign nested components first, then the main app
$ codesign -s "Developer ID Application" \
  MyApp.app/Contents/Frameworks/Helper.framework

$ codesign -s "Developer ID Application" \
  MyApp.app/Contents/MacOS/helper-tool

$ codesign -s "Developer ID Application" MyApp.app

# Or use --deep (less recommended, can miss components)
$ codesign -s "Developer ID Application" --deep MyApp.app
```

### Signature Verification Script

```bash
#!/bin/bash
# verify-signature.sh - Comprehensive signature verification

APP="$1"

if [[ -z "$APP" ]]; then
    echo "Usage: $0 /path/to/app"
    exit 1
fi

echo "=== Signature Verification ==="
echo "Application: $APP"
echo

echo "--- Basic Verification ---"
codesign -v "$APP"
echo

echo "--- Signature Details ---"
codesign -dv "$APP" 2>&1
echo

echo "--- Entitlements ---"
codesign -d --entitlements - "$APP" 2>&1
echo

echo "--- Gatekeeper Assessment ---"
spctl --assess -v "$APP" 2>&1
echo

echo "--- Quarantine Status ---"
QUARANTINE=$(xattr -p com.apple.quarantine "$APP" 2>/dev/null)
if [[ -n "$QUARANTINE" ]]; then
    echo "Quarantined: $QUARANTINE"
else
    echo "Not quarantined"
fi
```

## Security Audit Commands

```bash
# Find unsigned applications in /Applications
$ for app in /Applications/*.app; do
    codesign -v "$app" 2>/dev/null || echo "Unsigned: $app"
done

# Check signature validity of all running applications
$ for pid in $(pgrep -x '^[A-Z]'); do
    path=$(ps -p $pid -o comm= 2>/dev/null)
    if [[ -n "$path" ]]; then
        codesign -v "$path" 2>/dev/null || echo "Invalid: $path (PID: $pid)"
    fi
done

# List all quarantined files in Downloads
$ mdfind "kMDItemWhereFroms == '*'" -onlyin ~/Downloads 2>/dev/null | while read f; do
    xattr -p com.apple.quarantine "$f" 2>/dev/null && echo "$f"
done

# Check all kexts are properly signed
$ kextstat | awk 'NR>1 {print $6}' | while read bundle; do
    kextpath=$(find /System/Library/Extensions /Library/Extensions -name "$bundle.kext" 2>/dev/null | head -1)
    if [[ -n "$kextpath" ]]; then
        codesign -v "$kextpath" 2>/dev/null || echo "Issue with: $bundle"
    fi
done
```

## Summary

Code signing and Gatekeeper form the first line of defense for macOS application security:

| Tool | Purpose |
|------|---------|
| `codesign` | Sign and verify code signatures |
| `spctl` | Manage Gatekeeper policies |
| `xattr` | View/modify quarantine attributes |
| `security find-identity` | List signing certificates |

Key commands:

```bash
# Verify signature
$ codesign -v /path/to/app

# Assess with Gatekeeper
$ spctl --assess -v /path/to/app

# View signature details
$ codesign -dv --verbose=4 /path/to/app

# View entitlements
$ codesign -d --entitlements - /path/to/app

# Remove quarantine
$ xattr -dr com.apple.quarantine /path/to/app

# Sign your own code
$ codesign -s "Developer ID Application" --options runtime /path/to/app
```

Code signing ensures that software comes from a known source and hasn't been modified. Combined with notarization (covered in the next chapter), it provides a robust trust system for macOS software distribution.
