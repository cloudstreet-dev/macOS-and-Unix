# Notarization Requirements

Notarization is Apple's automated security scanning service for software distributed outside the Mac App Store. When you notarize your software, Apple scans it for malware and known security issues, then issues a "ticket" that Gatekeeper recognizes. Since macOS 10.15 Catalina, notarization is required for all Developer ID signed software to pass Gatekeeper without warnings.

## What Notarization Does

```
┌─────────────────────────────────────────────────────────────────┐
│                    Notarization Process                          │
├─────────────────────────────────────────────────────────────────┤
│  1. Developer uploads signed app to Apple                        │
│                         ↓                                        │
│  2. Apple performs automated security scans:                     │
│     - Malware detection                                          │
│     - Hardened runtime verification                              │
│     - Signature validation                                       │
│     - Unsafe API usage detection                                 │
│                         ↓                                        │
│  3. Apple issues notarization ticket                             │
│                         ↓                                        │
│  4. Developer staples ticket to software (optional)              │
│                         ↓                                        │
│  5. User downloads software                                      │
│                         ↓                                        │
│  6. Gatekeeper verifies:                                         │
│     - Valid Developer ID signature                               │
│     - Notarization ticket (from staple or Apple servers)         │
└─────────────────────────────────────────────────────────────────┘
```

## Notarization Requirements

To successfully notarize software, it must meet these requirements:

### Code Signing Requirements

1. **Signed with Developer ID certificate** (not Apple Development)
2. **Hardened runtime enabled** (`--options runtime`)
3. **Secure timestamp included** (`--timestamp`)
4. **All nested code must be signed** (frameworks, helpers, plugins)

### Technical Requirements

```bash
# The app must be signed with these minimum options:
$ codesign -s "Developer ID Application: Your Name (TEAMID)" \
  --timestamp \
  --options runtime \
  MyApp.app

# Verify hardened runtime is enabled
$ codesign -dv MyApp.app 2>&1 | grep flags
CodeDirectory v=20500 size=1234 flags=0x10000(runtime) hashes=42+7 location=embedded
```

### Common Notarization Blockers

| Issue | Solution |
|-------|----------|
| No hardened runtime | Add `--options runtime` to codesign |
| Missing timestamp | Add `--timestamp` to codesign |
| Unsigned nested code | Sign all frameworks and helpers |
| Invalid signature | Re-sign with valid Developer ID |
| Forbidden entitlements | Remove or get Apple approval |
| Insecure library loading | Use @rpath or hardcoded paths |

## The notarytool Command

Apple's current notarization tool is `notarytool`, included with Xcode command-line tools. It replaced the older `altool` starting with Xcode 13.

### Storing Credentials

Create an app-specific password at appleid.apple.com, then store credentials:

```bash
# Store credentials in keychain (recommended for scripts)
$ xcrun notarytool store-credentials "notarization-profile" \
  --apple-id "your@email.com" \
  --team-id "TEAMID123" \
  --password "app-specific-password"

# Credentials are stored in keychain
# Profile name can be anything meaningful to you

# Verify stored credentials
$ security find-generic-password -l "notarization-profile" -w 2>/dev/null && \
  echo "Credentials stored successfully"
```

### Submitting for Notarization

Notarization works with zip archives, disk images, or installer packages:

```bash
# Create a zip archive of your app
$ ditto -c -k --keepParent MyApp.app MyApp.zip

# Submit for notarization using stored credentials
$ xcrun notarytool submit MyApp.zip \
  --keychain-profile "notarization-profile" \
  --wait

# Output shows submission progress:
# Conducting pre-submission checks for MyApp.zip and initiating connection to the Apple notary service...
# Submission ID received
#   id: 12345678-1234-1234-1234-123456789012
# Successfully uploaded file
# Waiting for processing to complete.
# ....
# Processing complete
#   id: 12345678-1234-1234-1234-123456789012
#   status: Accepted

# Submit without waiting (returns immediately)
$ xcrun notarytool submit MyApp.zip \
  --keychain-profile "notarization-profile"
# Save the submission ID to check status later
```

### Checking Submission Status

```bash
# Check status of a submission
$ xcrun notarytool info 12345678-1234-1234-1234-123456789012 \
  --keychain-profile "notarization-profile"

# Get detailed log (useful when submission fails)
$ xcrun notarytool log 12345678-1234-1234-1234-123456789012 \
  --keychain-profile "notarization-profile"

# Save log to file for analysis
$ xcrun notarytool log 12345678-1234-1234-1234-123456789012 \
  --keychain-profile "notarization-profile" \
  developer_log.json

# View submission history
$ xcrun notarytool history --keychain-profile "notarization-profile"
```

### Notarization Log Analysis

When notarization fails, the log provides details:

```bash
# Download and examine the log
$ xcrun notarytool log SUBMISSION_ID \
  --keychain-profile "notarization-profile" \
  log.json

# View formatted log
$ cat log.json | python3 -m json.tool

# Common issues in logs:
# - "The signature does not include a secure timestamp"
# - "The executable does not have the hardened runtime enabled"
# - "The binary is not signed with a valid Developer ID certificate"
# - "The signature of the binary is invalid"
```

## Stapling

Stapling attaches the notarization ticket directly to your software, allowing offline verification:

```bash
# Staple ticket to an app
$ xcrun stapler staple MyApp.app
Processing: /path/to/MyApp.app
Processing: /path/to/MyApp.app
The staple and validate action worked!

# Staple to a disk image
$ xcrun stapler staple MyApp.dmg

# Staple to an installer package
$ xcrun stapler staple MyInstaller.pkg

# Verify stapling
$ xcrun stapler validate MyApp.app
Processing: /path/to/MyApp.app
The validate action worked!

# Check if an app has a stapled ticket
$ spctl -a -vvv MyApp.app 2>&1 | grep -i "ticket"
```

### Why Stapling Matters

Without stapling:
- Gatekeeper must contact Apple's servers on first launch
- Won't work if user is offline
- Slower first-launch experience

With stapling:
- Offline verification possible
- Faster first launch
- Better user experience

## Complete Notarization Workflow

Here's a complete script for signing, notarizing, and stapling:

```bash
#!/bin/bash
# notarize.sh - Complete notarization workflow

set -e

APP_NAME="MyApp"
APP_PATH="./build/${APP_NAME}.app"
DEVELOPER_ID="Developer ID Application: Your Name (TEAMID)"
KEYCHAIN_PROFILE="notarization-profile"
BUNDLE_ID="com.example.myapp"

echo "=== Step 1: Sign the application ==="
# Sign all nested components first
find "$APP_PATH" -name "*.framework" -exec \
  codesign -s "$DEVELOPER_ID" --timestamp --options runtime {} \;

find "$APP_PATH" -name "*.dylib" -exec \
  codesign -s "$DEVELOPER_ID" --timestamp --options runtime {} \;

# Sign helper tools
if [[ -d "$APP_PATH/Contents/Library/LoginItems" ]]; then
  find "$APP_PATH/Contents/Library/LoginItems" -name "*.app" -exec \
    codesign -s "$DEVELOPER_ID" --timestamp --options runtime {} \;
fi

# Sign the main app
codesign -s "$DEVELOPER_ID" \
  --timestamp \
  --options runtime \
  --entitlements entitlements.plist \
  "$APP_PATH"

echo "=== Step 2: Verify signature ==="
codesign -vvv --deep --strict "$APP_PATH"

echo "=== Step 3: Create ZIP for notarization ==="
ZIP_PATH="./build/${APP_NAME}.zip"
ditto -c -k --keepParent "$APP_PATH" "$ZIP_PATH"

echo "=== Step 4: Submit for notarization ==="
SUBMIT_OUTPUT=$(xcrun notarytool submit "$ZIP_PATH" \
  --keychain-profile "$KEYCHAIN_PROFILE" \
  --wait 2>&1)

echo "$SUBMIT_OUTPUT"

# Check if successful
if echo "$SUBMIT_OUTPUT" | grep -q "status: Accepted"; then
  echo "=== Step 5: Staple the ticket ==="
  xcrun stapler staple "$APP_PATH"

  echo "=== Step 6: Verify final product ==="
  spctl -a -vvv "$APP_PATH"

  echo "=== Notarization complete! ==="
else
  echo "=== Notarization failed ==="
  # Extract submission ID and get log
  SUBMISSION_ID=$(echo "$SUBMIT_OUTPUT" | grep "id:" | head -1 | awk '{print $2}')
  if [[ -n "$SUBMISSION_ID" ]]; then
    echo "Fetching detailed log..."
    xcrun notarytool log "$SUBMISSION_ID" \
      --keychain-profile "$KEYCHAIN_PROFILE"
  fi
  exit 1
fi
```

## Notarizing Disk Images

For distributing via DMG:

```bash
# Create the DMG
$ hdiutil create -volname "MyApp" \
  -srcfolder MyApp.app \
  -ov -format UDZO \
  MyApp.dmg

# Sign the DMG
$ codesign -s "Developer ID Application: Your Name (TEAMID)" \
  --timestamp \
  MyApp.dmg

# Submit for notarization
$ xcrun notarytool submit MyApp.dmg \
  --keychain-profile "notarization-profile" \
  --wait

# Staple the ticket
$ xcrun stapler staple MyApp.dmg
```

## Notarizing Installer Packages

```bash
# Build the package with pkgbuild
$ pkgbuild --root ./payload \
  --identifier com.example.myapp \
  --version 1.0 \
  --install-location /Applications \
  MyApp-unsigned.pkg

# Sign with Developer ID Installer certificate
$ productsign --sign "Developer ID Installer: Your Name (TEAMID)" \
  MyApp-unsigned.pkg \
  MyApp.pkg

# Submit for notarization
$ xcrun notarytool submit MyApp.pkg \
  --keychain-profile "notarization-profile" \
  --wait

# Staple
$ xcrun stapler staple MyApp.pkg
```

## Notarizing Command-Line Tools

Command-line tools can also be notarized:

```bash
# Sign the binary
$ codesign -s "Developer ID Application: Your Name (TEAMID)" \
  --timestamp \
  --options runtime \
  mytool

# Create a zip for submission
$ zip mytool.zip mytool

# Submit
$ xcrun notarytool submit mytool.zip \
  --keychain-profile "notarization-profile" \
  --wait

# Note: Can't staple directly to a binary
# Distribute as signed zip or embed in pkg/dmg
```

## Checking Notarization Status

Verify that distributed software is properly notarized:

```bash
# Check if an app is notarized
$ spctl -a -vvv /Applications/SomeApp.app
/Applications/SomeApp.app: accepted
source=Notarized Developer ID
origin=Developer ID Application: Company Name (TEAMID)

# If not notarized, you'll see:
# source=Developer ID
# (without "Notarized" prefix)

# Check for stapled ticket
$ stapler validate /Applications/SomeApp.app
Processing: /Applications/SomeApp.app
The validate action worked!

# If no ticket stapled:
# The validate action worked!
# (but without the Processing line showing the ticket)
```

## Troubleshooting Notarization

### Common Issues and Solutions

**"The signature does not include a secure timestamp"**

```bash
# Always use --timestamp when signing
$ codesign -s "Developer ID Application" --timestamp MyApp.app
```

**"The executable does not have the hardened runtime enabled"**

```bash
# Enable hardened runtime
$ codesign -s "Developer ID Application" \
  --timestamp \
  --options runtime \
  MyApp.app
```

**"The binary uses an SDK older than the 10.9 SDK"**

```bash
# Rebuild with a newer SDK
# In Xcode, set Deployment Target to 10.9 or later
```

**"Found an unsigned library"**

```bash
# Find unsigned components
$ codesign -vvv --deep --strict MyApp.app 2>&1 | grep "not signed"

# Sign each component
$ codesign -s "Developer ID Application" --timestamp path/to/unsigned.dylib
```

### Debugging Script

```bash
#!/bin/bash
# check-notarization-ready.sh - Verify app is ready for notarization

APP="$1"

if [[ -z "$APP" ]]; then
    echo "Usage: $0 /path/to/app"
    exit 1
fi

echo "=== Checking $APP for notarization readiness ==="

# Check signature exists
echo -n "Signature: "
if codesign -v "$APP" 2>/dev/null; then
    echo "Valid"
else
    echo "INVALID or MISSING"
    codesign -vvv "$APP"
    exit 1
fi

# Check hardened runtime
echo -n "Hardened Runtime: "
FLAGS=$(codesign -dv "$APP" 2>&1 | grep flags | grep -o "0x[0-9a-f]*")
if [[ "$FLAGS" == *"10000"* ]] || [[ "$FLAGS" == *"runtime"* ]]; then
    echo "Enabled"
else
    echo "DISABLED"
    echo "  Add --options runtime to codesign"
fi

# Check timestamp
echo -n "Secure Timestamp: "
if codesign -dv "$APP" 2>&1 | grep -q "Timestamp="; then
    echo "Present"
else
    echo "MISSING"
    echo "  Add --timestamp to codesign"
fi

# Check Developer ID
echo -n "Developer ID: "
AUTHORITY=$(codesign -dv "$APP" 2>&1 | grep "Authority=Developer ID")
if [[ -n "$AUTHORITY" ]]; then
    echo "Yes"
else
    echo "NO"
    echo "  Must be signed with Developer ID certificate"
fi

# Check for nested unsigned code
echo "Checking nested code..."
UNSIGNED=$(codesign -vvv --deep --strict "$APP" 2>&1 | grep -i "not signed")
if [[ -n "$UNSIGNED" ]]; then
    echo "WARNING: Unsigned components found:"
    echo "$UNSIGNED"
fi

# Test Gatekeeper assessment
echo -n "Gatekeeper Assessment: "
spctl -a -vvv "$APP" 2>&1 | head -2

echo "=== Check complete ==="
```

## Entitlements for Notarization

Some entitlements require special handling for notarization:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Required for JIT (like JavaScriptCore) -->
    <key>com.apple.security.cs.allow-jit</key>
    <true/>

    <!-- Allow unsigned executable memory (avoid if possible) -->
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>

    <!-- Disable library validation (for plugins) -->
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>

    <!-- Allow DYLD environment variables -->
    <key>com.apple.security.cs.allow-dyld-environment-variables</key>
    <true/>
</dict>
</plist>
```

Certain entitlements (like `com.apple.security.cs.allow-unsigned-executable-memory`) may cause notarization to fail or require additional review.

## Summary

Notarization is mandatory for Developer ID signed software on modern macOS:

| Component | Purpose |
|-----------|---------|
| `notarytool submit` | Upload software for scanning |
| `notarytool info` | Check submission status |
| `notarytool log` | Get detailed scan results |
| `stapler staple` | Attach ticket to software |
| `spctl -a -vvv` | Verify notarization |

Key requirements:
- Developer ID certificate (not Apple Development)
- Hardened runtime enabled (`--options runtime`)
- Secure timestamp (`--timestamp`)
- All nested code signed
- No blocked entitlements

```bash
# Quick notarization workflow
$ codesign -s "Developer ID Application" --timestamp --options runtime MyApp.app
$ ditto -c -k --keepParent MyApp.app MyApp.zip
$ xcrun notarytool submit MyApp.zip --keychain-profile "profile" --wait
$ xcrun stapler staple MyApp.app
```

Without notarization, your users will see scary Gatekeeper warnings or be unable to run your software at all. Proper notarization ensures a smooth, trusted experience.
