# launchd: The Heart of macOS

launchd is the first process started by the macOS kernel and the ancestor of all other processes. Understanding launchd is fundamental to understanding how macOS manages services, schedules tasks, and controls the lifecycle of processes.

## launchd's Role

launchd serves multiple functions that are separate on other Unix systems:

| Function | Traditional Unix | macOS |
|----------|-----------------|-------|
| Init system | init, systemd | launchd |
| Service manager | systemd, rc.d | launchd |
| Task scheduler | cron | launchd |
| Socket activation | inetd, xinetd | launchd |
| Session management | PAM, login | launchd |

## The Boot Process

When macOS boots:

1. **Firmware** (iBoot on Apple Silicon) loads the kernel
2. **Kernel** starts and launches `/sbin/launchd` as PID 1
3. **launchd** reads system configuration from:
   - `/System/Library/LaunchDaemons/` (Apple services)
   - `/Library/LaunchDaemons/` (third-party system services)
4. **launchd** starts essential system services
5. **loginwindow** appears (GUI login)
6. **User session** launches per-user launchd
7. **Per-user launchd** loads:
   - `/System/Library/LaunchAgents/` (Apple user agents)
   - `/Library/LaunchAgents/` (system-wide user agents)
   - `~/Library/LaunchAgents/` (user agents)

```
Kernel
    │
    └── launchd (PID 1, root)
            │
            ├── System daemons
            │   ├── mds (Spotlight)
            │   ├── configd (network)
            │   ├── bluetoothd
            │   └── ...
            │
            ├── loginwindow
            │   │
            │   └── User session launchd (per user)
            │           │
            │           ├── User agents
            │           │   ├── Dock
            │           │   ├── Finder
            │           │   └── ...
            │           │
            │           └── User applications
            │
            └── Background daemons
```

## Daemons vs Agents

### Daemons

- Run as root (or specified user)
- Start at boot, before user login
- System-wide scope
- Located in `LaunchDaemons` directories
- Can't access GUI or user session

```bash
# System daemons location
$ ls /Library/LaunchDaemons/
com.apple.installer.osmessagetracing.plist
homebrew.mxcl.postgresql@14.plist
...
```

### Agents

- Run as the logged-in user
- Start at user login
- Per-user scope
- Located in `LaunchAgents` directories
- Can access GUI, user session

```bash
# User agents location
$ ls ~/Library/LaunchAgents/
com.example.myagent.plist
...
```

### Which to Use?

| Need | Use |
|------|-----|
| Run before user login | Daemon |
| Access user session | Agent |
| Run as root | Daemon |
| Run as current user | Agent |
| System-wide service | Daemon |
| Per-user service | Agent |

## Job States

launchd jobs have several states:

```bash
# View job status
$ launchctl list | grep com.example
-       0       com.example.myjob
# Columns: PID, Exit Status, Label

# PID "-" means not running
# Exit Status "0" means last run succeeded
```

States:
- **Loaded**: Job definition is registered
- **Running**: Process is executing
- **Waiting**: Waiting for trigger (time, socket, path)
- **Stopped**: Not running, can be started

## On-Demand Loading

launchd's key innovation is on-demand loading. Services can be:

### Always Running (KeepAlive)

```xml
<key>KeepAlive</key>
<true/>
```

launchd restarts the job if it exits.

### Socket Activated

```xml
<key>Sockets</key>
<dict>
    <key>Listeners</key>
    <dict>
        <key>SockServiceName</key>
        <string>http</string>
    </dict>
</dict>
```

launchd listens on the socket; starts service when connection arrives.

### Time-Based

```xml
<key>StartInterval</key>
<integer>3600</integer>
```

Runs every 3600 seconds.

### Path-Based

```xml
<key>WatchPaths</key>
<array>
    <string>/path/to/watch</string>
</array>
```

Runs when file/directory changes.

## launchd Domains

Modern launchctl organizes jobs into domains:

| Domain | Description | Example |
|--------|-------------|---------|
| system | System-wide services | system/com.apple.mds |
| user | Per-user services | user/501/com.example.agent |
| gui | GUI session services | gui/501/com.apple.Dock |
| login | Login session | login/501 |
| pid | Per-process | pid/12345 |

```bash
# List jobs in user domain
$ launchctl print user/$(id -u)

# List jobs in GUI domain
$ launchctl print gui/$(id -u)
```

## Working with launchd

### Basic Commands

```bash
# List all loaded jobs
$ launchctl list

# Load a job
$ launchctl load /path/to/job.plist

# Unload a job
$ launchctl unload /path/to/job.plist

# Start a loaded job
$ launchctl start com.example.myjob

# Stop a running job
$ launchctl stop com.example.myjob
```

### Modern Commands (macOS 10.10+)

```bash
# Bootstrap (load and enable)
$ launchctl bootstrap gui/$(id -u) /path/to/job.plist

# Bootout (unload)
$ launchctl bootout gui/$(id -u)/com.example.myjob

# Enable job to run at load
$ launchctl enable user/$(id -u)/com.example.myjob

# Disable job
$ launchctl disable user/$(id -u)/com.example.myjob

# Kick (force immediate run)
$ launchctl kickstart gui/$(id -u)/com.example.myjob

# Kill and restart
$ launchctl kickstart -k gui/$(id -u)/com.example.myjob
```

### Debugging

```bash
# Print detailed job info
$ launchctl print gui/$(id -u)/com.example.myjob

# Print domain info
$ launchctl print gui/$(id -u)

# Check why job isn't loading
$ launchctl print-disabled user/$(id -u)

# View launchd logs
$ log show --predicate 'subsystem == "com.apple.launchd"' --last 1h
```

## Common launchd Keys

| Key | Purpose |
|-----|---------|
| Label | Unique identifier (required) |
| ProgramArguments | Command to run |
| Program | Simple program path |
| RunAtLoad | Start when loaded |
| KeepAlive | Restart if exits |
| StartInterval | Run every N seconds |
| StartCalendarInterval | Run at specific times |
| WatchPaths | Run when paths change |
| QueueDirectories | Run when directories have files |
| StandardOutPath | Redirect stdout |
| StandardErrorPath | Redirect stderr |
| WorkingDirectory | Set working directory |
| EnvironmentVariables | Set environment |
| UserName | Run as user (daemons) |
| GroupName | Run as group (daemons) |

## Log Output

By default, launchd job output goes to the unified log. To capture output:

```xml
<key>StandardOutPath</key>
<string>/tmp/myjob.stdout.log</string>
<key>StandardErrorPath</key>
<string>/tmp/myjob.stderr.log</string>
```

Or view in Console.app / unified log:

```bash
$ log show --predicate 'process == "myjob"' --last 1h
```

## Security Considerations

### Disabled Jobs

macOS can disable jobs for security:

```bash
# Check disabled status
$ launchctl print-disabled user/$(id -u)
"com.example.suspicious" => disabled
```

### Code Signing

System integrity checks may prevent unsigned jobs from loading.

### Sandbox

launchd jobs can be sandboxed:

```xml
<key>EnableSandboxing</key>
<true/>
```

## Summary

launchd key concepts:

| Concept | Description |
|---------|-------------|
| PID 1 | First process, parent of all |
| Daemon | System-wide background service |
| Agent | Per-user background service |
| On-demand | Services start when needed |
| Domain | Organizational scope |
| plist | XML/binary configuration |
| launchctl | Command-line interface |

launchd is more than an init system—it's a comprehensive process management framework that enables macOS's responsive behavior by starting services only when needed.
