# Background Task Scheduling

Scheduling tasks on macOS can be done through launchd (the modern macOS way) or cron (the traditional Unix way). Both have their place, but launchd is preferred for most use cases on macOS.

## launchd Scheduling

### Interval-Based Scheduling

Run every N seconds:

```xml
<key>StartInterval</key>
<integer>300</integer>  <!-- Every 5 minutes -->
```

```xml
<key>StartInterval</key>
<integer>3600</integer>  <!-- Every hour -->
```

### Calendar-Based Scheduling

Using StartCalendarInterval for specific times:

```xml
<!-- Every day at 3:30 AM -->
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>3</integer>
    <key>Minute</key>
    <integer>30</integer>
</dict>
```

```xml
<!-- Every Monday at 9 AM -->
<key>StartCalendarInterval</key>
<dict>
    <key>Weekday</key>
    <integer>1</integer>
    <key>Hour</key>
    <integer>9</integer>
</dict>
```

```xml
<!-- First of every month at midnight -->
<key>StartCalendarInterval</key>
<dict>
    <key>Day</key>
    <integer>1</integer>
    <key>Hour</key>
    <integer>0</integer>
</dict>
```

### Calendar Interval Keys

| Key | Range | Description |
|-----|-------|-------------|
| Month | 1-12 | Month of year |
| Day | 1-31 | Day of month |
| Weekday | 0-7 | Day of week (0 and 7 = Sunday) |
| Hour | 0-23 | Hour of day |
| Minute | 0-59 | Minute of hour |

### Multiple Schedules

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

### Complete Scheduled Agent Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.scheduled-backup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/backup.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>2</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/var/log/backup.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/backup.err</string>
</dict>
</plist>
```

## cron (Traditional Unix)

cron still works on macOS but is somewhat deprecated in favor of launchd.

### cron Basics

```bash
# Edit crontab
$ crontab -e

# List crontab
$ crontab -l

# Remove crontab
$ crontab -r
```

### cron Syntax

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, Sunday=0 or 7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

### cron Examples

```bash
# Every minute
* * * * * /path/to/script.sh

# Every hour
0 * * * * /path/to/script.sh

# Every day at 2 AM
0 2 * * * /path/to/script.sh

# Every Monday at 9 AM
0 9 * * 1 /path/to/script.sh

# Every 15 minutes
*/15 * * * * /path/to/script.sh

# First of month at midnight
0 0 1 * * /path/to/script.sh

# Weekdays at 6 PM
0 18 * * 1-5 /path/to/script.sh
```

### Full Crontab Example

```bash
$ crontab -e
# Add:
SHELL=/bin/zsh
PATH=/opt/homebrew/bin:/usr/bin:/bin

# Backup at 2 AM daily
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Cleanup every Sunday at 3 AM
0 3 * * 0 /usr/local/bin/cleanup.sh

# Check disk space every hour
0 * * * * df -h | mail -s "Disk Report" admin@example.com
```

### cron Limitations on macOS

1. **No on-demand loading**: Unlike launchd, cron always runs
2. **No power management**: Doesn't integrate with sleep/wake
3. **No GUI access**: Can't interact with user session
4. **Limited logging**: Must redirect output manually

## at Command

For one-time scheduled tasks:

```bash
# Schedule command for specific time
$ at 2:30 PM
at> /usr/local/bin/myscript.sh
at> <Ctrl+D>
job 1 at Tue Jan 16 14:30:00 2024

# Schedule for specific date
$ at 2:30 PM Jan 20
at> /usr/local/bin/myscript.sh
at> <Ctrl+D>

# Schedule relative time
$ at now + 1 hour
$ at now + 30 minutes
$ at midnight
$ at noon tomorrow

# List pending jobs
$ atq

# Remove job
$ atrm <job_number>
```

### Enabling atrun

atrun must be enabled for `at` to work:

```bash
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.atrun.plist
```

## launchd vs cron Comparison

| Feature | launchd | cron |
|---------|---------|------|
| On-demand loading | Yes | No |
| Power management | Yes | No |
| Resource limits | Yes | No |
| GUI session access | Yes (agents) | No |
| Path watching | Yes | No |
| Network conditions | Yes | No |
| Logging | Unified log | Manual |
| Complex schedules | StartCalendarInterval | More flexible syntax |

## When to Use Which

**Use launchd when**:
- Building macOS-native services
- Need on-demand or event-based execution
- Want integration with power management
- Need to access GUI session

**Use cron when**:
- Porting scripts from Linux
- Simple time-based scheduling
- More familiar with cron syntax
- Writing cross-platform scripts

## Practical Example: Migration from cron to launchd

cron entry:
```bash
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

Equivalent launchd plist:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.backup</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/backup.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>2</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/var/log/backup.log</string>
    <key>StandardErrorPath</key>
    <string>/var/log/backup.log</string>
</dict>
</plist>
```

## Summary

Task scheduling options:

| Method | Best For |
|--------|----------|
| launchd | Most macOS scheduling needs |
| cron | Simple, cross-platform scripts |
| at | One-time future execution |

launchd advantages:
- Better integration with macOS
- On-demand and event-based triggers
- Power management awareness
- Unified logging

For new projects on macOS, prefer launchd. Use cron for compatibility or when its syntax is more convenient.
