# App Sandboxing

Sandboxing is macOS's containment mechanism that restricts what resources an application can access. A sandboxed app operates in a constrained environment with limited access to the file system, network, hardware, and system services. While App Store apps are required to be sandboxed, understanding sandboxing is valuable for security testing, developing distributed software, and restricting untrusted processes.

## How Sandboxing Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    macOS Kernel (XNU)                            │
│                          │                                       │
│              ┌──────────────────────────┐                       │
│              │   Sandbox Kernel Module   │                       │
│              │       (Seatbelt)          │                       │
│              └──────────────────────────┘                       │
│                          │                                       │
│         ┌────────────────┼────────────────┐                     │
│         ↓                ↓                ↓                     │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐              │
│  │ Sandbox    │   │ Sandbox    │   │ Sandbox    │              │
│  │ Container  │   │ Container  │   │ Container  │              │
│  │            │   │            │   │            │              │
│  │  App A     │   │  App B     │   │  App C     │              │
│  │            │   │            │   │            │              │
│  │ ~/Library/ │   │ ~/Library/ │   │ ~/Library/ │              │
│  │ Containers/│   │ Containers/│   │ Containers/│              │
│  │ com.a.app/ │   │ com.b.app/ │   │ com.c.app/ │              │
│  └────────────┘   └────────────┘   └────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

Each sandboxed app gets its own container directory and can only access:
- Its container directory
- Resources explicitly granted via entitlements
- User-selected files (via Open/Save dialogs)
- Resources granted through security-scoped bookmarks

## Sandbox Containers

Sandboxed apps store their data in containers:

```bash
# List sandbox containers
$ ls ~/Library/Containers/
com.apple.Notes
com.apple.Safari
com.example.myapp

# View a container's structure
$ ls ~/Library/Containers/com.apple.Safari/
Data

$ ls ~/Library/Containers/com.apple.Safari/Data/
Documents
Library

# Each container has standard directories
$ ls ~/Library/Containers/com.apple.Safari/Data/Library/
Application Support
Caches
HTTPStorages
Preferences
Saved Application State

# Container metadata
$ ls -la ~/Library/Containers/com.apple.Safari/
total 0
drwx------  3 david  staff   96 Jan 15 10:00 .
drwx------  5 david  staff  160 Jan 15 10:00 ..
drwx------  5 david  staff  160 Jan 15 10:00 Data
lrwxr-xr-x  1 david  staff   72 Jan 15 10:00 .com.apple.containermanagerd.metadata.plist -> ...
```

### Group Containers

Apps can share data through group containers:

```bash
# Group containers
$ ls ~/Library/Group\ Containers/
group.com.apple.notes
group.com.example.shared

# Apps with the same app group entitlement can access shared containers
```

## The sandbox-exec Command

The `sandbox-exec` command runs processes with sandbox restrictions. While primarily used for testing, it demonstrates sandbox capabilities:

### Basic Usage

```bash
# Run a command with no network access
$ sandbox-exec -n no-network curl https://example.com
curl: (6) Could not resolve host: example.com

# Run with no file write access
$ sandbox-exec -n no-write touch /tmp/test
touch: /tmp/test: Operation not permitted

# List built-in profiles (may vary by macOS version)
$ ls /System/Library/Sandbox/Profiles/
# Note: Many profiles are embedded and not visible as files
```

### Built-in Sandbox Profiles

macOS includes several built-in profiles:

| Profile | Description |
|---------|-------------|
| `no-internet` | Blocks all network access |
| `no-network` | Blocks network access |
| `no-write` | Blocks file writes |
| `no-write-except-temporary` | Allows writes only to temp |
| `pure-computation` | Minimal access for computation only |

```bash
# Examples using built-in profiles
$ sandbox-exec -n no-internet python3 -c "import urllib.request; print(urllib.request.urlopen('https://example.com').read())"
# Fails with network error

$ sandbox-exec -n no-write-except-temporary bash -c 'echo test > /tmp/ok.txt && echo test > ~/fail.txt'
# First write succeeds, second fails
```

### Custom Sandbox Profiles

Create custom profiles using Scheme-like syntax (SBPL - Sandbox Profile Language):

```scheme
;; my-sandbox.sb - Custom sandbox profile
(version 1)

;; Start with deny-all
(deny default)

;; Allow reading most files
(allow file-read*)

;; Allow writing to specific directories
(allow file-write*
    (subpath "/tmp")
    (subpath (param "TMPDIR")))

;; Allow specific network access
(allow network-outbound
    (remote tcp "example.com:443"))

;; Allow process execution
(allow process-exec)
(allow process-fork)

;; Allow reading system libraries
(allow file-read-data
    (subpath "/usr/lib")
    (subpath "/System/Library"))

;; Allow mach services for basic functionality
(allow mach-lookup
    (global-name "com.apple.system.logger"))
```

Use the custom profile:

```bash
$ sandbox-exec -f my-sandbox.sb /path/to/command
```

### Profile Variables

Pass variables to sandbox profiles:

```bash
# Profile using parameter
# (allow file-write* (subpath (param "WRITEPATH")))

$ sandbox-exec -f my-profile.sb -D WRITEPATH=/tmp/myapp /path/to/command
```

### Debugging Sandbox Profiles

```bash
# Enable sandbox tracing (verbose)
$ sandbox-exec -f my-profile.sb -D _TRACE=1 /path/to/command

# View sandbox violations in system log
$ log show --predicate 'subsystem == "com.apple.sandbox"' --last 5m

# Stream sandbox violations
$ log stream --predicate 'subsystem == "com.apple.sandbox"'
```

## App Sandbox Entitlements

Apps declare their sandbox capabilities through entitlements:

### Common Sandbox Entitlements

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Enable App Sandbox -->
    <key>com.apple.security.app-sandbox</key>
    <true/>

    <!-- File Access -->
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>

    <key>com.apple.security.files.downloads.read-write</key>
    <true/>

    <!-- Network Access -->
    <key>com.apple.security.network.client</key>
    <true/>

    <key>com.apple.security.network.server</key>
    <true/>

    <!-- Hardware Access -->
    <key>com.apple.security.device.camera</key>
    <true/>

    <key>com.apple.security.device.microphone</key>
    <true/>

    <key>com.apple.security.device.usb</key>
    <true/>

    <!-- Personal Information -->
    <key>com.apple.security.personal-information.addressbook</key>
    <true/>

    <key>com.apple.security.personal-information.calendars</key>
    <true/>

    <key>com.apple.security.personal-information.location</key>
    <true/>

    <!-- Temporary Exception (for legacy code) -->
    <key>com.apple.security.temporary-exception.files.absolute-path.read-write</key>
    <array>
        <string>/usr/local/</string>
    </array>
</dict>
</plist>
```

### Viewing App Entitlements

```bash
# View entitlements of a running app
$ codesign -d --entitlements - /Applications/Safari.app

# Check if an app is sandboxed
$ codesign -d --entitlements - /Applications/Notes.app 2>&1 | grep "app-sandbox"
[Key] com.apple.security.app-sandbox
[Value]
    [Bool] true

# View entitlements as XML
$ codesign -d --entitlements :- /Applications/Safari.app
```

### File Access Entitlements

| Entitlement | Access Granted |
|-------------|----------------|
| `files.user-selected.read-only` | Read files user opens |
| `files.user-selected.read-write` | Read/write user-opened files |
| `files.downloads.read-only` | Read ~/Downloads |
| `files.downloads.read-write` | Read/write ~/Downloads |
| `files.pictures.read-only` | Read ~/Pictures |
| `files.pictures.read-write` | Read/write ~/Pictures |
| `files.music.read-only` | Read ~/Music |
| `files.music.read-write` | Read/write ~/Music |
| `files.movies.read-only` | Read ~/Movies |
| `files.movies.read-write` | Read/write ~/Movies |

### Network Entitlements

```xml
<!-- Outgoing connections -->
<key>com.apple.security.network.client</key>
<true/>

<!-- Incoming connections (server) -->
<key>com.apple.security.network.server</key>
<true/>
```

## Security-Scoped Bookmarks

Sandboxed apps can maintain access to user-selected files across launches using security-scoped bookmarks:

```bash
# Security-scoped bookmarks are created programmatically
# They allow persistent access to files outside the container

# View bookmark data (binary format)
$ xxd -l 100 ~/Library/Containers/com.example.app/Data/Library/Application\ Support/bookmarks.data
```

This is primarily an API feature, but understanding it helps debug sandbox behavior.

## Checking Sandbox Status

### For Running Processes

```bash
# Check if a process is sandboxed
$ ps -p <PID> -o flags
# Look for sandbox flag

# More detailed check using asctl (App Sandbox status)
$ asctl sandbox check --pid <PID>

# List sandboxed processes
$ ps aux | while read line; do
    pid=$(echo "$line" | awk '{print $2}')
    if sandbox-check 2>/dev/null; then
        echo "$line"
    fi
done
```

### For Applications

```bash
# Check if an app bundle is sandboxed
$ codesign -d --entitlements - /path/to/app.app 2>&1 | grep -A1 "app-sandbox"

# Quick script to check sandbox status
#!/bin/bash
check_sandbox() {
    local app="$1"
    if codesign -d --entitlements - "$app" 2>&1 | grep -q "com.apple.security.app-sandbox"; then
        echo "Sandboxed: $app"
    else
        echo "Not sandboxed: $app"
    fi
}

check_sandbox /Applications/Safari.app
check_sandbox /Applications/iTerm.app
```

## Sandbox Violations

When a sandboxed app tries to do something it's not allowed to, a violation occurs:

### Viewing Violations

```bash
# View recent sandbox violations
$ log show --predicate 'subsystem == "com.apple.sandbox" AND category == "violation"' --last 1h

# Stream violations in real-time
$ log stream --predicate 'subsystem == "com.apple.sandbox" AND category == "violation"'

# Example violation log:
# Sandbox: MyApp(1234) deny(1) file-read-data /private/etc/passwd
```

### Common Violation Patterns

```bash
# File access violation
# Sandbox: app(pid) deny file-read-data /path/to/file

# Network violation
# Sandbox: app(pid) deny network-outbound

# Mach service violation
# Sandbox: app(pid) deny mach-lookup com.apple.some.service

# Hardware access violation
# Sandbox: app(pid) deny device-camera
```

## Testing Sandbox Restrictions

Create test environments to verify sandbox behavior:

```bash
#!/bin/bash
# test-sandbox.sh - Test sandbox restrictions

# Test no-write sandbox
echo "Testing no-write sandbox..."
sandbox-exec -n no-write bash -c 'echo test > /tmp/test.txt' 2>&1 && echo "FAIL" || echo "PASS: Write blocked"

# Test no-network sandbox
echo "Testing no-network sandbox..."
sandbox-exec -n no-network curl -s https://example.com 2>&1 | grep -q "Could not resolve" && echo "PASS: Network blocked" || echo "FAIL"

# Test custom profile
cat > /tmp/test-sandbox.sb << 'EOF'
(version 1)
(deny default)
(allow file-read* (subpath "/"))
(allow file-write* (subpath "/tmp"))
(deny file-write* (subpath "/tmp/restricted"))
(allow process-exec)
(allow process-fork)
EOF

mkdir -p /tmp/restricted

echo "Testing custom sandbox..."
sandbox-exec -f /tmp/test-sandbox.sb bash -c 'echo test > /tmp/ok.txt' 2>&1 && echo "PASS: /tmp write allowed" || echo "FAIL"
sandbox-exec -f /tmp/test-sandbox.sb bash -c 'echo test > /tmp/restricted/fail.txt' 2>&1 && echo "FAIL" || echo "PASS: /tmp/restricted blocked"

rm /tmp/test-sandbox.sb /tmp/ok.txt 2>/dev/null
rmdir /tmp/restricted 2>/dev/null
```

## Container Management

### Viewing Container Contents

```bash
# List all containers
$ ls ~/Library/Containers/

# Find container for a specific app
$ ls -la ~/Library/Containers/ | grep Safari

# View container size
$ du -sh ~/Library/Containers/com.apple.Safari/

# Find containers using significant space
$ du -sh ~/Library/Containers/* 2>/dev/null | sort -hr | head -10
```

### Resetting Containers

```bash
# Remove an app's container (resets app to fresh state)
# WARNING: Deletes all app data
$ rm -rf ~/Library/Containers/com.example.app/

# The container is recreated when the app launches
```

### Container Symlinks

Sandboxed apps see a different filesystem view:

```bash
# Inside a sandboxed app, ~ points to the container
# ~/Library is actually ~/Library/Containers/com.example.app/Data/Library

# Some directories are symlinked to real locations
$ ls -la ~/Library/Containers/com.apple.Safari/Data/
# Desktop -> /Users/david/Desktop (if allowed by entitlements)
```

## Sandbox Profiles for System Services

Many macOS system services run sandboxed:

```bash
# View sandbox profiles in use
$ sudo find /System -name "*.sb" -type f 2>/dev/null

# System sandbox profiles location
$ ls /System/Library/Sandbox/Profiles/ 2>/dev/null

# Many profiles are now embedded in the Sandbox kernel extension
# and not visible as separate files
```

## Practical Examples

### Restricting a Build Process

```bash
# Run a build with restricted file access
$ cat > build-sandbox.sb << 'EOF'
(version 1)
(deny default)
(allow file-read*)
(allow file-write*
    (subpath "/tmp")
    (subpath (param "BUILD_DIR")))
(allow process-exec)
(allow process-fork)
(allow mach-lookup)
EOF

$ sandbox-exec -f build-sandbox.sb -D BUILD_DIR="$(pwd)/build" make
```

### Running Untrusted Scripts

```bash
# Run an untrusted script with minimal permissions
$ cat > untrusted-sandbox.sb << 'EOF'
(version 1)
(deny default)
(allow file-read*
    (subpath "/usr/lib")
    (subpath "/usr/bin")
    (subpath "/bin")
    (subpath "/System/Library"))
(allow file-read-data (literal "/dev/null"))
(allow file-write-data (literal "/dev/null"))
(allow process-exec)
(allow process-fork)
EOF

$ sandbox-exec -f untrusted-sandbox.sb bash untrusted-script.sh
```

## Summary

Sandboxing provides containment for macOS applications:

| Component | Purpose |
|-----------|---------|
| App Sandbox | Isolates Mac App Store apps |
| Containers | Per-app data directories |
| Entitlements | Declare required capabilities |
| `sandbox-exec` | Run commands with restrictions |
| Sandbox profiles | Define access rules |

Key concepts:

- Sandboxed apps have limited filesystem access
- Access must be declared via entitlements
- User can grant additional access via file dialogs
- Violations are logged and can be monitored
- System services also use sandboxing

```bash
# Check if app is sandboxed
$ codesign -d --entitlements - /path/to/app | grep app-sandbox

# Run command with restrictions
$ sandbox-exec -n no-network /path/to/command

# View sandbox violations
$ log show --predicate 'subsystem == "com.apple.sandbox"' --last 10m
```

Sandboxing is a critical defense-in-depth layer that limits the damage a compromised application can do, even if other security measures fail.
