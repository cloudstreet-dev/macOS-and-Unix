# launchctl Command Reference

launchctl is the command-line interface to launchd. Its syntax has evolved over macOS versions, with modern versions using a domain-based approach. This chapter covers both legacy and modern commands.

## Command Overview

### Legacy Commands (Still Supported)

```bash
launchctl load <plist>         # Load a job
launchctl unload <plist>       # Unload a job
launchctl start <label>        # Start a loaded job
launchctl stop <label>         # Stop a running job
launchctl list [label]         # List jobs
launchctl submit               # Submit job without plist
launchctl remove <label>       # Remove a submitted job
```

### Modern Commands (macOS 10.10+)

```bash
launchctl bootstrap <domain> <plist>   # Load job
launchctl bootout <domain-target>      # Unload job
launchctl enable <domain-target>       # Enable job
launchctl disable <domain-target>      # Disable job
launchctl kickstart <domain-target>    # Force start
launchctl kill <signal> <domain-target># Send signal
launchctl print <domain-target>        # Show job info
launchctl print-cache                  # Show cache
launchctl print-disabled <domain>      # Show disabled jobs
```

## Domains and Targets

### Domain Types

| Domain | Syntax | Description |
|--------|--------|-------------|
| system | `system` | System services |
| user | `user/<uid>` | User services |
| gui | `gui/<uid>` | GUI session services |
| login | `login/<uid>` | Login services |
| pid | `pid/<pid>` | Per-process services |

### Target Syntax

```bash
# Domain only
system
user/501
gui/501

# Domain + service
system/com.apple.mds
user/501/com.example.agent
gui/501/com.apple.Dock
```

### Getting Your UID

```bash
$ id -u
501

# Or in commands
$ launchctl print gui/$(id -u)
```

## Job Management

### Loading Jobs

```bash
# Legacy (still works)
$ launchctl load ~/Library/LaunchAgents/com.example.agent.plist
$ sudo launchctl load /Library/LaunchDaemons/com.example.daemon.plist

# Modern
$ launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.agent.plist
$ sudo launchctl bootstrap system /Library/LaunchDaemons/com.example.daemon.plist

# Load and enable
$ launchctl load -w ~/Library/LaunchAgents/com.example.agent.plist
# -w writes enabled status to overrides database
```

### Unloading Jobs

```bash
# Legacy
$ launchctl unload ~/Library/LaunchAgents/com.example.agent.plist

# Modern
$ launchctl bootout gui/$(id -u)/com.example.agent
$ sudo launchctl bootout system/com.example.daemon

# Unload and disable
$ launchctl unload -w ~/Library/LaunchAgents/com.example.agent.plist
```

### Starting and Stopping

```bash
# Start a loaded job
$ launchctl start com.example.agent

# Stop a running job
$ launchctl stop com.example.agent

# Modern: Force start (even if conditions not met)
$ launchctl kickstart gui/$(id -u)/com.example.agent

# Force start and restart if running
$ launchctl kickstart -k gui/$(id -u)/com.example.agent

# Force start and print PID
$ launchctl kickstart -p gui/$(id -u)/com.example.agent
```

### Enabling and Disabling

```bash
# Enable (will load at next boot/login)
$ launchctl enable user/$(id -u)/com.example.agent
$ sudo launchctl enable system/com.example.daemon

# Disable (won't load)
$ launchctl disable user/$(id -u)/com.example.agent
$ sudo launchctl disable system/com.example.daemon

# Check disabled status
$ launchctl print-disabled user/$(id -u)
```

## Listing and Inspecting

### List Jobs

```bash
# List all loaded jobs
$ launchctl list
PID     Status  Label
-       0       com.apple.AMPArtworkAgent
1234    0       com.apple.Dock
...

# Filter by pattern
$ launchctl list | grep com.apple
$ launchctl list | grep -E "^-"  # Not running

# Get specific job
$ launchctl list com.apple.Dock
{
    "LimitLoadToSessionType" = "Aqua";
    "Label" = "com.apple.Dock";
    "OnDemand" = false;
    "LastExitStatus" = 0;
    "PID" = 1234;
    "Program" = "/System/Library/CoreServices/Dock.app/Contents/MacOS/Dock";
};
```

### Detailed Information

```bash
# Print job details (modern)
$ launchctl print gui/$(id -u)/com.apple.Dock
com.apple.Dock = {
    active count = 1
    path = /System/Library/LaunchAgents/com.apple.Dock.plist
    state = running
    program = /System/Library/CoreServices/Dock.app/Contents/MacOS/Dock
    ...
}

# Print entire domain
$ launchctl print gui/$(id -u) | head -50

# Print system domain (requires sudo)
$ sudo launchctl print system | head -50
```

### Process Information

```bash
# Get info about a specific service
$ sudo launchctl procinfo <pid>

# Blame: which job launched a process
$ launchctl blame gui/$(id -u)/com.example.agent
```

## Environment and Configuration

### Setting Environment Variables

```bash
# Set environment variable for GUI session
$ launchctl setenv MY_VAR "my_value"

# Get environment variable
$ launchctl getenv MY_VAR

# Unset environment variable
$ launchctl unsetenv MY_VAR

# Note: These affect new processes, not already running ones
```

### Managing Resources

```bash
# Set resource limits
$ launchctl limit
    cpu         unlimited      unlimited
    filesize    unlimited      unlimited
    data        unlimited      unlimited
    stack       8388608        67104768
    core        0              unlimited
    rss         unlimited      unlimited
    memlock     unlimited      unlimited
    maxproc     2784           4176
    maxfiles    256            unlimited

# Modify limit (for session)
$ launchctl limit maxfiles 65536 200000
```

## Signals and Control

### Sending Signals

```bash
# Kill with signal
$ launchctl kill SIGTERM gui/$(id -u)/com.example.agent
$ launchctl kill SIGKILL gui/$(id -u)/com.example.agent
$ launchctl kill SIGHUP gui/$(id -u)/com.example.agent

# Signal numbers also work
$ launchctl kill 15 gui/$(id -u)/com.example.agent
```

### Service Control

```bash
# Reboot (requires authorization)
$ sudo launchctl reboot

# Shutdown
$ sudo launchctl shutdown

# Single-user mode
$ sudo launchctl reboot -s
```

## Debugging

### Debug Mode

```bash
# Enable debug mode for job
$ launchctl debug gui/$(id -u)/com.example.agent

# Run with additional output
$ launchctl debug gui/$(id -u)/com.example.agent -- /path/to/program
```

### Examining Errors

```bash
# Check why job failed
$ launchctl error <error_number>
# Example:
$ launchctl error 78
78: Function not implemented

# View launchd log
$ log show --predicate 'subsystem == "com.apple.launchd"' --last 1h

# View specific job
$ log show --predicate 'process == "myprocess"' --last 1h
```

### Analyzing

```bash
# Export service to XML
$ launchctl dumpstate > /tmp/launchd-state.txt
$ launchctl dumpjpcategory > /tmp/jpcategory.txt

# Analyze job
$ sudo launchctl plist /Library/LaunchDaemons/com.example.plist
```

## Homebrew Services

Homebrew provides a wrapper for launchctl:

```bash
# Start service (loads and starts)
$ brew services start postgresql@14

# Stop service
$ brew services stop postgresql@14

# Restart
$ brew services restart postgresql@14

# List services
$ brew services list

# Run once (doesn't persist across reboot)
$ brew services run postgresql@14
```

Behind the scenes, `brew services` creates and manages plists in `~/Library/LaunchAgents/`.

## Quick Reference

| Task | Command |
|------|---------|
| Load agent | `launchctl load ~/Library/LaunchAgents/X.plist` |
| Load daemon | `sudo launchctl load /Library/LaunchDaemons/X.plist` |
| Unload | `launchctl unload /path/to/plist` |
| Start | `launchctl start label` |
| Stop | `launchctl stop label` |
| List all | `launchctl list` |
| List one | `launchctl list label` |
| Job info | `launchctl print gui/$(id -u)/label` |
| Enable | `launchctl enable user/$(id -u)/label` |
| Disable | `launchctl disable user/$(id -u)/label` |
| Force start | `launchctl kickstart gui/$(id -u)/label` |
| Set env | `launchctl setenv VAR value` |
| Get env | `launchctl getenv VAR` |

## Summary

launchctl provides comprehensive control over launchd:

- **Legacy commands** still work and are simpler
- **Modern commands** provide more control via domains
- **Domains** organize services by scope (system, user, gui)
- **Enable/disable** controls whether jobs load
- **Kickstart** forces immediate execution
- Use `brew services` for Homebrew packages
