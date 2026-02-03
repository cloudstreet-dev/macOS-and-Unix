# Privacy Controls and TCC Database

TCC (Transparency, Consent, and Control) is the macOS framework that manages privacy permissions. When an application wants to access your camera, microphone, contacts, location, or other sensitive data, TCC mediates the request and prompts you for consent. Understanding TCC is essential for troubleshooting permission issues, managing privacy settings programmatically, and auditing application access.

## How TCC Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    TCC Architecture                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Application                                                    │
│       │                                                         │
│       ▼                                                         │
│   Framework API Request                                         │
│   (e.g., AVCaptureDevice for camera)                            │
│       │                                                         │
│       ▼                                                         │
│   ┌────────────────────────────────────┐                       │
│   │          tccd daemon               │                       │
│   │  (TCC database + policy engine)    │                       │
│   └────────────────────────────────────┘                       │
│       │                                                         │
│       ├── Check TCC.db for existing permission                  │
│       │                                                         │
│       ├── If no permission: Show consent dialog                 │
│       │                                                         │
│       └── Return: Granted / Denied / Not Determined             │
│                                                                  │
│   TCC Databases:                                                 │
│   - User: ~/Library/Application Support/com.apple.TCC/TCC.db   │
│   - System: /Library/Application Support/com.apple.TCC/TCC.db  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## TCC Protected Services

TCC protects numerous categories of sensitive data:

| Service | Description |
|---------|-------------|
| Camera | Access to camera device |
| Microphone | Access to microphone |
| Screen Recording | Capture screen content |
| Accessibility | Control UI of other apps |
| Full Disk Access | Access all files |
| Contacts | Address book data |
| Calendar | Calendar events |
| Reminders | Reminders data |
| Photos | Photo library access |
| Location Services | Geographic location |
| System Events | Apple Events automation |
| Files and Folders | Specific folder access |
| Automation | Control other applications |
| Input Monitoring | Monitor keyboard/mouse |
| Media & Apple Music | Apple Music library |
| Developer Tools | Debugging other apps |

## The tccutil Command

The `tccutil` command can reset TCC permissions for applications.

### Resetting Permissions

```bash
# Reset all permissions for an application
$ tccutil reset All com.example.app

# Reset specific service
$ tccutil reset Camera com.example.app
$ tccutil reset Microphone com.example.app
$ tccutil reset ScreenCapture com.example.app
$ tccutil reset Accessibility com.example.app

# Reset for all applications (service-wide)
$ tccutil reset Camera

# Reset all services for all apps (use carefully)
$ tccutil reset All
```

### Available Services

```bash
# Common service identifiers for tccutil
AddressBook              # Contacts
Calendar                 # Calendar
Reminders                # Reminders
Photos                   # Photo Library
Camera                   # Camera
Microphone               # Microphone
Accessibility            # Accessibility
PostEvent                # System Events
ScreenCapture            # Screen Recording
MediaLibrary             # Media & Apple Music
ListenEvent              # Input Monitoring
SystemPolicyAllFiles     # Full Disk Access
SystemPolicySysAdminFiles # Administer files
SystemPolicyDeveloperFiles # Developer Files
AppleEvents              # Automation
```

### Reset Examples

```bash
# App has wrong camera permissions
$ tccutil reset Camera com.zoom.us
# Next time Zoom requests camera, user will be prompted

# Reset Terminal's Full Disk Access
$ tccutil reset SystemPolicyAllFiles com.apple.Terminal

# Clear all automation permissions
$ tccutil reset AppleEvents

# Clear all permissions for a malware removal
$ tccutil reset All com.suspicious.app
```

## Viewing TCC Database

The TCC database stores permission decisions. Accessing it requires Full Disk Access.

### User TCC Database

```bash
# Location
$ ls ~/Library/Application\ Support/com.apple.TCC/TCC.db

# View with sqlite3 (requires Full Disk Access)
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, service, auth_value FROM access"

# More readable output
$ sqlite3 -header -column ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, service, auth_value,
          datetime(last_modified, 'unixepoch') as modified
   FROM access
   ORDER BY service, client"

# auth_value meanings:
# 0 = Denied
# 1 = Unknown (ask next time)
# 2 = Allowed
# 3 = Limited

# Filter by service
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, auth_value FROM access WHERE service = 'kTCCServiceCamera'"
```

### System TCC Database

```bash
# Requires admin privileges
$ sudo sqlite3 /Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, service, auth_value FROM access"

# System-level permissions include:
# - Accessibility permissions
# - Input monitoring
# - Screen recording (some cases)
```

### TCC Database Schema

```sql
-- Main access table structure (simplified)
CREATE TABLE access (
    service TEXT NOT NULL,        -- Service identifier
    client TEXT NOT NULL,         -- App bundle ID or path
    client_type INTEGER NOT NULL, -- 0=bundle, 1=path
    auth_value INTEGER NOT NULL,  -- Permission state
    auth_reason INTEGER NOT NULL, -- How granted
    auth_version INTEGER NOT NULL,
    csreq BLOB,                   -- Code signing requirement
    policy_id INTEGER,
    indirect_object_identifier TEXT,
    indirect_object_code_identity BLOB,
    flags INTEGER,
    last_modified INTEGER,        -- Unix timestamp
    PRIMARY KEY (service, client, client_type, indirect_object_identifier)
);
```

### Querying Specific Permissions

```bash
#!/bin/bash
# check-tcc.sh - Check TCC permissions for an app

APP_ID="$1"
if [[ -z "$APP_ID" ]]; then
    echo "Usage: $0 <bundle-identifier>"
    exit 1
fi

DB="$HOME/Library/Application Support/com.apple.TCC/TCC.db"

echo "=== TCC Permissions for $APP_ID ==="
sqlite3 -header -column "$DB" \
  "SELECT service,
          CASE auth_value
              WHEN 0 THEN 'Denied'
              WHEN 1 THEN 'Unknown'
              WHEN 2 THEN 'Allowed'
              WHEN 3 THEN 'Limited'
              ELSE auth_value
          END as status,
          datetime(last_modified, 'unixepoch') as modified
   FROM access
   WHERE client = '$APP_ID'
   ORDER BY service"
```

## Privacy Preferences in System Settings

### Check via Profiles

```bash
# Export privacy preferences (MDM-managed)
$ sudo profiles -P | grep -i privacy

# View privacy configuration profiles
$ sudo profiles list -verbose
```

### Using System Preferences CLI

```bash
# Open Privacy settings
$ open "x-apple.systempreferences:com.apple.preference.security?Privacy"

# Open specific privacy pane
$ open "x-apple.systempreferences:com.apple.preference.security?Privacy_Camera"
$ open "x-apple.systempreferences:com.apple.preference.security?Privacy_Microphone"
$ open "x-apple.systempreferences:com.apple.preference.security?Privacy_AllFiles"
$ open "x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility"
```

## Granting TCC Permissions

### For Terminal/CLI Tools

Terminal needs Full Disk Access for many operations:

1. Open System Preferences > Security & Privacy > Privacy
2. Select "Full Disk Access"
3. Click the lock to make changes
4. Add Terminal.app (or your terminal emulator)
5. Restart Terminal

```bash
# After granting Full Disk Access, you can:
$ ls ~/Library/Mail/              # Access Mail data
$ sqlite3 ~/Library/Messages/chat.db  # Access Messages
$ cat ~/Library/Safari/History.db  # Access Safari history
```

### For Automation/AppleScript

```bash
# Script that requires automation permission
$ osascript -e 'tell application "System Events" to keystroke "a"'
# Will prompt for Accessibility permission

# Check if permission exists
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT auth_value FROM access
   WHERE service='kTCCServiceAppleEvents'
   AND client='com.apple.Terminal'"
```

### MDM-Based Permission Grants

For enterprise deployment, use Privacy Preferences Policy Control profiles:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>PayloadType</key>
            <string>com.apple.TCC.configuration-profile-policy</string>
            <key>Services</key>
            <dict>
                <key>SystemPolicyAllFiles</key>
                <array>
                    <dict>
                        <key>Allowed</key>
                        <true/>
                        <key>CodeRequirement</key>
                        <string>identifier "com.example.app" and anchor apple generic</string>
                        <key>IdentifierType</key>
                        <string>bundleID</string>
                        <key>Identifier</key>
                        <string>com.example.app</string>
                    </dict>
                </array>
            </dict>
        </dict>
    </array>
</dict>
</plist>
```

## Programmatic Permission Requests

### Checking Permission Status

```bash
# From an app's perspective, permissions have states:
# - Not Determined: Never asked
# - Restricted: Disabled by policy (e.g., parental controls)
# - Denied: User denied permission
# - Authorized: User granted permission

# Example: Check camera authorization (from app code)
# AVAuthorizationStatus status = [AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
```

### Permission States in TCC

```bash
# Query authorization states
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client,
          CASE auth_value
              WHEN 0 THEN 'Denied'
              WHEN 1 THEN 'Not Determined'
              WHEN 2 THEN 'Authorized'
              WHEN 3 THEN 'Limited (Photos)'
          END as status
   FROM access
   WHERE service = 'kTCCServiceCamera'"
```

## Automation Permissions

Automation (controlling other apps) has its own TCC category:

```bash
# Check automation permissions
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, indirect_object_identifier, auth_value
   FROM access
   WHERE service = 'kTCCServiceAppleEvents'"

# Example: Terminal controlling System Events
# client = com.apple.Terminal
# indirect_object_identifier = com.apple.systemevents
# auth_value = 2 (allowed)
```

### Script Requiring Automation

```bash
#!/bin/bash
# This script requires automation permission

# Tell Finder to empty trash
osascript << 'EOF'
tell application "Finder"
    empty trash
end tell
EOF

# First run will prompt for permission
# Grant Terminal permission to control Finder
```

## Privacy Audit Script

```bash
#!/bin/bash
# tcc-audit.sh - Audit TCC permissions

USER_DB="$HOME/Library/Application Support/com.apple.TCC/TCC.db"
SYSTEM_DB="/Library/Application Support/com.apple.TCC/TCC.db"

echo "=== TCC Privacy Audit ==="
echo "Date: $(date)"
echo

# Function to decode auth_value
decode_auth() {
    case $1 in
        0) echo "Denied" ;;
        1) echo "Unknown" ;;
        2) echo "Allowed" ;;
        3) echo "Limited" ;;
        *) echo "$1" ;;
    esac
}

echo "--- User TCC Database ---"
if [[ -r "$USER_DB" ]]; then
    sqlite3 -header -column "$USER_DB" \
      "SELECT service, client, auth_value as auth,
              datetime(last_modified, 'unixepoch') as modified
       FROM access
       ORDER BY service, client" 2>/dev/null || echo "Cannot read (need Full Disk Access)"
else
    echo "Cannot access (need Full Disk Access for Terminal)"
fi

echo
echo "--- System TCC Database ---"
sudo sqlite3 -header -column "$SYSTEM_DB" \
  "SELECT service, client, auth_value as auth
   FROM access
   ORDER BY service, client" 2>/dev/null || echo "Cannot read"

echo
echo "--- Camera Access ---"
sqlite3 "$USER_DB" \
  "SELECT client FROM access WHERE service='kTCCServiceCamera' AND auth_value=2" 2>/dev/null

echo
echo "--- Microphone Access ---"
sqlite3 "$USER_DB" \
  "SELECT client FROM access WHERE service='kTCCServiceMicrophone' AND auth_value=2" 2>/dev/null

echo
echo "--- Screen Recording Access ---"
sqlite3 "$USER_DB" \
  "SELECT client FROM access WHERE service='kTCCServiceScreenCapture' AND auth_value=2" 2>/dev/null

echo
echo "--- Full Disk Access ---"
sqlite3 "$USER_DB" \
  "SELECT client FROM access WHERE service='kTCCServiceSystemPolicyAllFiles' AND auth_value=2" 2>/dev/null

echo
echo "--- Accessibility Access ---"
sudo sqlite3 "$SYSTEM_DB" \
  "SELECT client FROM access WHERE service='kTCCServiceAccessibility' AND auth_value=2" 2>/dev/null

echo
echo "=== Audit Complete ==="
```

## Troubleshooting TCC Issues

### App Not Prompting for Permission

```bash
# Reset permission to force re-prompt
$ tccutil reset Camera com.example.app

# Verify the app is code signed
$ codesign -dv /Applications/SomeApp.app

# Check if app has required Info.plist keys
$ /usr/libexec/PlistBuddy -c "Print :NSCameraUsageDescription" \
    /Applications/SomeApp.app/Contents/Info.plist
```

### Permission Granted But Not Working

```bash
# Verify permission in database
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT * FROM access WHERE client='com.example.app'"

# Check code signing requirement matches
$ codesign -dr - /Applications/SomeApp.app

# Restart the tccd daemon
$ sudo killall -9 tccd
# tccd restarts automatically
```

### Full Disk Access Not Working

```bash
# Ensure Terminal is in FDA list
# Check TCC database
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT * FROM access WHERE service='kTCCServiceSystemPolicyAllFiles' AND client='com.apple.Terminal'"

# If using different terminal (iTerm, etc.)
# Add that terminal app to Full Disk Access

# Restart Terminal after granting permission
```

### Viewing TCC Logs

```bash
# View TCC-related logs
$ log show --predicate 'subsystem == "com.apple.TCC"' --last 10m

# Stream TCC logs
$ log stream --predicate 'subsystem == "com.apple.TCC"'

# Filter for denials
$ log show --predicate 'subsystem == "com.apple.TCC" AND eventMessage CONTAINS "denied"' --last 1h
```

## Security Implications

### Malware and TCC

Malware may attempt to bypass TCC:
- Exploit vulnerabilities in tccd
- Abuse accessibility permissions
- Use synthetic clicks to grant permissions

```bash
# Check for suspicious TCC entries
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT * FROM access WHERE auth_value=2" | grep -v "com.apple"

# Look for unknown apps with sensitive permissions
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client FROM access
   WHERE service='kTCCServiceAccessibility' AND auth_value=2"
```

### Hardening TCC

1. Regularly audit TCC permissions
2. Remove unused application permissions
3. Be cautious granting Accessibility access
4. Use MDM to control permissions in enterprise

## Summary

TCC manages privacy permissions on macOS:

| Command | Purpose |
|---------|---------|
| `tccutil reset` | Reset permissions |
| `sqlite3 TCC.db` | Query permission database |
| System Preferences | GUI permission management |
| MDM Profiles | Enterprise permission control |

Key concepts:

- TCC mediates access to sensitive data
- Permissions stored in SQLite database
- User and system databases exist
- Apps must declare usage descriptions
- Permissions can be reset via tccutil

```bash
# Quick reference
$ tccutil reset Camera com.example.app     # Reset camera permission
$ tccutil reset All com.example.app        # Reset all permissions
$ sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db "SELECT * FROM access"  # View permissions
$ log show --predicate 'subsystem == "com.apple.TCC"' --last 10m  # View TCC logs
```

TCC is a critical privacy protection layer. Understanding it helps diagnose permission issues and audit what data applications can access.
