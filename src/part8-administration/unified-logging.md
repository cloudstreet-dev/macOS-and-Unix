# Unified Logging System

macOS 10.12 (Sierra) introduced Unified Logging, replacing the traditional Unix syslog system. All system and application logs now flow through a single, efficient logging mechanism. Understanding this system is essential for troubleshooting macOS issues.

## Unified Logging vs Traditional Syslog

Traditional Unix logging had limitations:

| Aspect | Traditional Syslog | Unified Logging |
|--------|-------------------|-----------------|
| Storage | Plain text files | Compressed binary database |
| Performance | Disk I/O intensive | Memory-buffered, compressed |
| Query | grep through files | Structured queries |
| Metadata | Limited | Rich (subsystem, category, type) |
| Privacy | None | Built-in privacy controls |
| Location | /var/log/ | /var/db/diagnostics/ |

```bash
# Old way (some logs still here)
$ ls /var/log/
asl/
com.apple.xpc.launchd/
install.log
system.log
wifi.log

# New way
$ ls /var/db/diagnostics/
Persist/
Special/
timesync/
```

## The log Command

The primary tool for unified logging is the `log` command:

```bash
# Basic usage
$ log show                    # Show recent logs
$ log stream                  # Follow logs in real-time
$ log collect                 # Gather logs for analysis
```

### Real-Time Log Streaming

```bash
# Stream all logs (very verbose)
$ log stream
Timestamp                       Thread     Type        Activity             PID    TTL
2024-01-15 10:00:00.123456-0800 0x1234     Default     0x0                  123    0    kernel: (AppleUSBXHCI) ...
2024-01-15 10:00:00.234567-0800 0x2345     Info        0x0                  456    0    mds: (Spotlight) ...
...

# Stream with level filter
$ log stream --level error

# Stream for specific process
$ log stream --process Safari

# Stream for specific subsystem
$ log stream --predicate 'subsystem == "com.apple.network"'

# Stream with debug messages (normally hidden)
$ log stream --level debug

# Stream with info level and above
$ log stream --level info
```

### Viewing Historical Logs

```bash
# Show logs from last hour
$ log show --last 1h

# Show logs from last 30 minutes
$ log show --last 30m

# Show logs from specific time range
$ log show --start "2024-01-15 09:00:00" --end "2024-01-15 10:00:00"

# Show logs from today
$ log show --start "$(date '+%Y-%m-%d') 00:00:00"

# Show logs since boot
$ log show --start "$(sysctl -n kern.boottime | awk '{print $4}' | tr -d ',')"
```

## Filtering with Predicates

Predicates are the key to finding relevant logs. They use NSPredicate syntax:

### Basic Predicate Examples

```bash
# Filter by process name
$ log show --predicate 'process == "Safari"' --last 1h

# Filter by subsystem
$ log show --predicate 'subsystem == "com.apple.wifi"' --last 1h

# Filter by category
$ log show --predicate 'category == "connection"' --last 1h

# Filter by message content
$ log show --predicate 'eventMessage contains "error"' --last 1h

# Case-insensitive search
$ log show --predicate 'eventMessage contains[c] "ERROR"' --last 1h

# Filter by log type/level
$ log show --predicate 'messageType == error' --last 1h
$ log show --predicate 'messageType == fault' --last 1h
```

### Combining Predicates

```bash
# AND condition
$ log show --predicate 'process == "kernel" AND eventMessage contains "USB"' --last 1h

# OR condition
$ log show --predicate 'process == "Safari" OR process == "WebKit"' --last 1h

# Complex combinations
$ log show --predicate '(subsystem == "com.apple.network" OR subsystem == "com.apple.wifi") AND messageType == error' --last 1h

# NOT condition
$ log show --predicate 'NOT process == "kernel"' --last 1h

# Multiple conditions
$ log show --predicate 'subsystem BEGINSWITH "com.apple" AND category == "default" AND eventMessage contains[c] "fail"' --last 1h
```

### Predicate Operators

| Operator | Meaning |
|----------|---------|
| `==` | Equals |
| `!=` | Not equals |
| `<`, `>`, `<=`, `>=` | Comparisons |
| `contains` | String contains |
| `BEGINSWITH` | String starts with |
| `ENDSWITH` | String ends with |
| `MATCHES` | Regular expression |
| `AND`, `OR`, `NOT` | Logical operators |
| `[c]` | Case-insensitive modifier |

### Available Predicate Fields

| Field | Description |
|-------|-------------|
| `process` | Process name |
| `processID` | Process ID (PID) |
| `subsystem` | Logging subsystem |
| `category` | Log category |
| `eventMessage` | The log message |
| `messageType` | Log level (default, info, debug, error, fault) |
| `senderImagePath` | Path to the binary that logged |
| `eventType` | activityCreateEvent, logEvent, etc. |

## Log Levels and Types

Unified logging has five log types:

| Type | Description | When to Use |
|------|-------------|-------------|
| Default | Standard messages | General information |
| Info | Informational | Helpful but verbose |
| Debug | Debugging | Development only |
| Error | Errors | Something went wrong |
| Fault | Critical | System/app failure |

```bash
# Show only errors and faults
$ log show --predicate 'messageType == error OR messageType == fault' --last 1h

# Show debug messages (normally hidden, requires debug profile)
$ log show --level debug --predicate 'process == "myapp"' --last 1h
```

By default, Debug and Info messages may be hidden. Enable them:

```bash
# Enable debug logging for a subsystem (persistent)
$ sudo log config --mode "persist:debug" --subsystem com.example.myapp

# Enable for current boot only
$ sudo log config --mode "level:debug" --subsystem com.example.myapp

# Reset to default
$ sudo log config --mode "persist:default" --subsystem com.example.myapp

# Check current configuration
$ sudo log config --status --subsystem com.example.myapp
```

## Practical Examples

### Troubleshooting Wi-Fi

```bash
# Stream Wi-Fi logs
$ log stream --predicate 'subsystem == "com.apple.wifi"' --level debug

# Show recent Wi-Fi errors
$ log show --predicate 'subsystem == "com.apple.wifi" AND messageType >= error' --last 1h

# Find Wi-Fi disconnection reasons
$ log show --predicate 'subsystem == "com.apple.wifi" AND eventMessage contains "disconnect"' --last 4h
```

### Investigating Crashes

```bash
# Find crash logs
$ log show --predicate 'eventMessage contains "crash" OR process == "ReportCrash"' --last 1h

# Application-specific crashes
$ log show --predicate 'process == "Safari" AND (eventMessage contains "crash" OR messageType == fault)' --last 1h

# Kernel panics
$ log show --predicate 'process == "kernel" AND messageType == fault' --last 24h
```

### Login/Authentication Issues

```bash
# Authentication logs
$ log show --predicate 'subsystem == "com.apple.opendirectoryd" OR subsystem == "com.apple.securityd"' --last 1h

# Login events
$ log show --predicate 'eventMessage contains "login" OR eventMessage contains "authentication"' --last 1h

# SSH connections
$ log show --predicate 'process == "sshd"' --last 1h
```

### Disk and Storage

```bash
# Disk-related messages
$ log show --predicate 'subsystem contains "disk" OR process == "diskutil" OR process == "fsck"' --last 1h

# APFS operations
$ log show --predicate 'subsystem == "com.apple.apfs"' --last 1h

# Time Machine
$ log show --predicate 'subsystem == "com.apple.TimeMachine"' --last 4h
```

### Network Issues

```bash
# Network subsystem
$ log show --predicate 'subsystem == "com.apple.network"' --last 30m

# DNS issues
$ log show --predicate 'eventMessage contains "DNS" OR process == "mDNSResponder"' --last 1h

# VPN connections
$ log show --predicate 'subsystem == "com.apple.networkextension"' --last 1h
```

### Application Debugging

```bash
# Specific application
$ log show --predicate 'process == "MyApp"' --last 1h

# Application launch issues
$ log show --predicate 'eventMessage contains "MyApp" AND (eventMessage contains "launch" OR eventMessage contains "terminate")' --last 1h

# launchd service issues
$ log show --predicate 'subsystem == "com.apple.launchd" AND eventMessage contains "com.example"' --last 1h
```

## Output Formatting

### Format Options

```bash
# Default format
$ log show --last 5m

# JSON format (for scripting)
$ log show --last 5m --style json

# Compact format
$ log show --last 5m --style compact

# Syslog-like format
$ log show --last 5m --style syslog

# Include more information
$ log show --last 5m --info --debug
```

### JSON Output for Scripting

```bash
# Get JSON output
$ log show --predicate 'process == "Safari"' --last 5m --style json

# Parse with jq
$ log show --predicate 'messageType == error' --last 1h --style json | jq '.[].eventMessage'

# Count errors by process
$ log show --predicate 'messageType == error' --last 1h --style json | jq 'group_by(.processImagePath) | map({process: .[0].processImagePath, count: length}) | sort_by(.count) | reverse'
```

## Collecting Logs

For support or analysis, collect logs into a file:

```bash
# Collect logs to archive
$ sudo log collect --last 1d --output ~/Desktop/logs.logarchive

# Collect with specific time range
$ sudo log collect --start "2024-01-15 09:00:00" --end "2024-01-15 12:00:00" --output ~/Desktop/incident.logarchive

# Open in Console.app
$ open ~/Desktop/logs.logarchive
```

The `.logarchive` format can be opened in Console.app or queried with `log show`:

```bash
# Query collected logs
$ log show --archive ~/Desktop/logs.logarchive --predicate 'messageType == error'
```

## Legacy Log Files

Some traditional log files still exist:

```bash
# Install log
$ cat /var/log/install.log

# System log (limited)
$ cat /var/log/system.log

# Wi-Fi log
$ cat /var/log/wifi.log

# Apache (if enabled)
$ cat /var/log/apache2/error_log

# Application-specific
$ cat ~/Library/Logs/DiagnosticReports/*.crash
```

## Console.app

Console.app provides a GUI for unified logging:

```bash
# Open Console
$ open -a Console

# Open with specific log archive
$ open ~/Desktop/logs.logarchive
```

Console.app features:
- Real-time streaming
- Predicate builder
- Log favorites
- Device logs (iOS devices connected)

## Common Subsystems Reference

| Subsystem | Description |
|-----------|-------------|
| `com.apple.wifi` | Wi-Fi |
| `com.apple.network` | Networking |
| `com.apple.networkextension` | VPN, network extensions |
| `com.apple.apfs` | APFS filesystem |
| `com.apple.launchd` | launchd services |
| `com.apple.securityd` | Security daemon |
| `com.apple.opendirectoryd` | Directory services |
| `com.apple.TimeMachine` | Time Machine |
| `com.apple.Spotlight` | Spotlight indexing |
| `com.apple.xpc` | XPC services |
| `com.apple.coredata` | Core Data |
| `com.apple.bluetooth` | Bluetooth |
| `com.apple.audio` | Audio |
| `com.apple.powerd` | Power management |

## Performance Considerations

Unified logging is designed for efficiency, but verbose queries can impact performance:

```bash
# Expensive: search all logs
$ log show --last 24h  # Can be slow

# Better: use predicates to filter
$ log show --predicate 'process == "Safari"' --last 24h

# Best: combine specific predicates
$ log show --predicate 'process == "Safari" AND messageType == error' --last 24h
```

## Summary

Unified logging replaces traditional syslog with a modern, efficient system:

| Task | Command |
|------|---------|
| Stream live logs | `log stream` |
| Show historical logs | `log show --last 1h` |
| Filter by process | `--predicate 'process == "name"'` |
| Filter by subsystem | `--predicate 'subsystem == "com.apple.x"'` |
| Filter by message | `--predicate 'eventMessage contains "text"'` |
| Show only errors | `--predicate 'messageType == error'` |
| Collect logs | `sudo log collect --output file.logarchive` |
| JSON output | `--style json` |

Key predicates to remember:

```bash
# Process-based
process == "ProcessName"

# Subsystem-based
subsystem == "com.apple.something"

# Message content
eventMessage contains "search term"
eventMessage contains[c] "case insensitive"

# Log level
messageType == error
messageType == fault

# Combined
process == "Safari" AND messageType == error
```

Master the `log` command, and you'll be able to diagnose almost any macOS issue from the command line.
