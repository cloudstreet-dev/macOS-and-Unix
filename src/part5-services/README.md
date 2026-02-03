# System Services on macOS

If you're coming from Linux, you'll search for `systemctl` and find nothing. macOS uses launchd, a fundamentally different approach to service management that Apple introduced in 2005. Understanding launchd is essential for managing background processes, scheduling tasks, and running services on macOS.

## What Is launchd?

launchd is macOS's unified service management framework. It replaces:
- `init` (process 1)
- `cron` (scheduled tasks)
- `inetd` (network services)
- `xinetd` (extended inetd)
- `rc.d` scripts (startup scripts)

launchd is always PID 1:

```bash
$ ps aux | head -2
USER   PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
root     1   0.1  0.1 410148544  18624   ??  Ss   10:00AM   0:05.23 /sbin/launchd
```

## Key Concepts

### Jobs vs Services

In launchd terminology:
- **Job**: Any task managed by launchd (one-time or recurring)
- **Daemon**: A background process running as root
- **Agent**: A background process running as a user

### Property Lists (plists)

launchd jobs are configured via property list files:

```bash
# System-wide daemons
/Library/LaunchDaemons/

# System-wide agents
/Library/LaunchAgents/

# Per-user agents
~/Library/LaunchAgents/

# Apple's daemons (don't modify)
/System/Library/LaunchDaemons/

# Apple's agents (don't modify)
/System/Library/LaunchAgents/
```

### launchctl

`launchctl` is the command-line interface to launchd:

```bash
# List all services
$ launchctl list

# Load/start a service
$ launchctl load /Library/LaunchDaemons/com.example.daemon.plist

# Unload/stop a service
$ launchctl unload /Library/LaunchDaemons/com.example.daemon.plist
```

## What You'll Learn in This Part

**[launchd: The Heart of macOS](./launchd-overview.md)** provides a comprehensive overview of launchd architecture, how it boots the system, and its role in process management.

**[Understanding Property Lists (plists)](./property-lists.md)** explains the XML and binary formats used for launchd configuration and how to create and edit them.

**[Creating Launch Agents and Daemons](./agents-and-daemons.md)** walks through creating your own background services step by step.

**[launchctl Command Reference](./launchctl.md)** is a comprehensive guide to the launchctl command and all its subcommands.

**[Process Management: CLI vs GUI](./process-management.md)** covers managing processes from both Terminal and Activity Monitor.

**[Background Task Scheduling](./task-scheduling.md)** explains how to schedule recurring tasks using launchd (the macOS way) and cron (the traditional Unix way).

**[XPC Services and IPC](./xpc-services.md)** introduces Apple's modern inter-process communication framework.

**[Comparing launchd to systemd and init](./launchd-vs-others.md)** helps Linux users understand how macOS's approach differs from what they know.

## Quick Example

Here's a simple launch agent that runs a script every hour:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.hourly</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myscript.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>3600</integer>
</dict>
</plist>
```

Save as `~/Library/LaunchAgents/com.example.hourly.plist` and load:

```bash
$ launchctl load ~/Library/LaunchAgents/com.example.hourly.plist
```

The following chapters explain this and much more in detail.
