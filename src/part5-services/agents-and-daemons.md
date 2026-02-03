# Creating Launch Agents and Daemons

This chapter walks through creating your own launchd jobs step by step, covering both user agents and system daemons with practical examples.

## Agent vs Daemon: Which Do You Need?

| Requirement | Use |
|-------------|-----|
| Run before any user logs in | Daemon |
| Access user's GUI or files | Agent |
| Run as root | Daemon |
| Run as current user | Agent |
| System-wide scope | Daemon |
| Per-user scope | Agent |

## Creating a Launch Agent

### Step 1: Create Your Script

```bash
$ mkdir -p ~/bin
$ cat > ~/bin/backup-documents.sh << 'EOF'
#!/bin/bash
# Simple backup script
BACKUP_DIR="$HOME/Backups/Documents"
mkdir -p "$BACKUP_DIR"
rsync -av --delete "$HOME/Documents/" "$BACKUP_DIR/"
echo "$(date): Backup completed" >> "$HOME/Backups/backup.log"
EOF
$ chmod +x ~/bin/backup-documents.sh
```

### Step 2: Create the Plist

```bash
$ cat > ~/Library/LaunchAgents/com.example.backup.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.backup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/david/bin/backup-documents.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>14</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/backup.stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/backup.stderr.log</string>
</dict>
</plist>
EOF
```

### Step 3: Set Permissions

```bash
$ chmod 644 ~/Library/LaunchAgents/com.example.backup.plist
$ ls -la ~/Library/LaunchAgents/com.example.backup.plist
-rw-r--r--  1 david  staff  ...
```

### Step 4: Load and Test

```bash
# Load the agent
$ launchctl load ~/Library/LaunchAgents/com.example.backup.plist

# Verify it's loaded
$ launchctl list | grep backup
-       0       com.example.backup

# Run immediately for testing
$ launchctl start com.example.backup

# Check output
$ cat /tmp/backup.stdout.log
```

### Step 5: Unload if Needed

```bash
$ launchctl unload ~/Library/LaunchAgents/com.example.backup.plist
```

## Creating a Launch Daemon

Daemons require root privileges and run system-wide.

### Step 1: Create the Script

```bash
$ sudo mkdir -p /usr/local/bin
$ sudo tee /usr/local/bin/cleanup-system.sh << 'EOF'
#!/bin/bash
# System cleanup script
LOG="/var/log/cleanup.log"
echo "$(date): Starting cleanup" >> "$LOG"

# Clean old logs
find /var/log -name "*.log" -mtime +30 -delete 2>/dev/null

# Clean temp files
find /tmp -type f -atime +7 -delete 2>/dev/null

echo "$(date): Cleanup completed" >> "$LOG"
EOF
$ sudo chmod +x /usr/local/bin/cleanup-system.sh
```

### Step 2: Create the Plist

```bash
$ sudo tee /Library/LaunchDaemons/com.example.cleanup.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.cleanup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/cleanup-system.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/var/log/cleanup.stdout.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/cleanup.stderr.log</string>
</dict>
</plist>
EOF
```

### Step 3: Set Ownership and Permissions

```bash
$ sudo chown root:wheel /Library/LaunchDaemons/com.example.cleanup.plist
$ sudo chmod 644 /Library/LaunchDaemons/com.example.cleanup.plist
```

### Step 4: Load

```bash
$ sudo launchctl load /Library/LaunchDaemons/com.example.cleanup.plist
```

## Common Patterns

### Keep-Alive Service

A service that should always be running:

```xml
<key>KeepAlive</key>
<true/>

<key>ThrottleInterval</key>
<integer>30</integer>
```

### Watch for File Changes

Run when a file or directory changes:

```xml
<key>WatchPaths</key>
<array>
    <string>/path/to/watch</string>
</array>
```

### Run at Login

```xml
<key>RunAtLoad</key>
<true/>
```

### Scheduled Intervals

Every hour:
```xml
<key>StartInterval</key>
<integer>3600</integer>
```

Every day at midnight:
```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>0</integer>
</dict>
```

Every Monday at 9 AM:
```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Weekday</key>
    <integer>1</integer>
    <key>Hour</key>
    <integer>9</integer>
    <key>Minute</key>
    <integer>0</integer>
</dict>
```

Multiple times:
```xml
<key>StartCalendarInterval</key>
<array>
    <dict>
        <key>Hour</key>
        <integer>8</integer>
    </dict>
    <dict>
        <key>Hour</key>
        <integer>12</integer>
    </dict>
    <dict>
        <key>Hour</key>
        <integer>18</integer>
    </dict>
</array>
```

### Environment Variables

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    <key>LANG</key>
    <string>en_US.UTF-8</string>
</dict>
```

### Network-Dependent

Wait for network before running:

```xml
<key>KeepAlive</key>
<dict>
    <key>NetworkState</key>
    <true/>
</dict>
```

## Practical Examples

### Sync Script on Network Change

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.networksync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/david/bin/sync.sh</string>
    </array>
    <key>WatchPaths</key>
    <array>
        <string>/Library/Preferences/SystemConfiguration</string>
    </array>
</dict>
</plist>
```

### Process Queue Directory

Run when files appear in a directory:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.processor</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/david/bin/process-files.sh</string>
    </array>
    <key>QueueDirectories</key>
    <array>
        <string>/Users/david/incoming</string>
    </array>
</dict>
</plist>
```

### Web Server Daemon

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.webserver</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/python3</string>
        <string>-m</string>
        <string>http.server</string>
        <string>8080</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/var/www</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

## Debugging

### Check Job Status

```bash
$ launchctl list | grep com.example
-       78      com.example.myjob
# PID "-" = not running, exit code 78 = error
```

### View Logs

```bash
# Job-specific logs
$ cat /tmp/myjob.stderr.log

# System logs
$ log show --predicate 'subsystem == "com.apple.launchd"' --last 10m

# Specific job
$ log show --predicate 'process == "myscript"' --last 1h
```

### Manual Test

```bash
# Run the program directly
$ /path/to/myscript.sh

# Check exit code
$ echo $?
```

### Common Issues

| Problem | Solution |
|---------|----------|
| Job won't load | Check plist syntax with `plutil` |
| Job loads but doesn't run | Check Program path exists |
| Job runs but fails | Check logs, run manually |
| Permission denied | Check script is executable |
| Wrong environment | Add EnvironmentVariables |

## Summary

Creating launchd jobs:

1. **Choose type**: Agent (user) or Daemon (system)
2. **Create script**: Executable, tested manually
3. **Create plist**: Valid XML, required keys
4. **Set permissions**: 644, correct ownership
5. **Load**: `launchctl load`
6. **Test**: `launchctl start`, check logs
7. **Debug**: View output, check status
