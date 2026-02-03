# XPC Services and IPC

XPC (Cross-Process Communication) is Apple's modern IPC framework, built on top of Mach ports. While application developers use XPC extensively, understanding it helps explain how macOS services communicate and how to troubleshoot issues.

## IPC Mechanisms on macOS

| Mechanism | Use Case | API Level |
|-----------|----------|-----------|
| XPC | Modern Apple IPC | High-level |
| Mach ports | Kernel-level IPC | Low-level |
| Unix sockets | Network-style IPC | POSIX |
| Pipes | Parent-child communication | POSIX |
| Shared memory | High-performance data sharing | POSIX |

## Understanding XPC

### What XPC Provides

XPC offers:
- **Type-safe message passing**: Dictionaries, arrays, strings, data
- **Crash isolation**: Service crashes don't bring down client
- **Resource management**: Automatic cleanup
- **Security**: Entitlements and sandboxing integration
- **Launchd integration**: On-demand service activation

### XPC Service Types

| Type | Description |
|------|-------------|
| XPC Service | Bundled in application, private |
| Launch Daemon | System-wide, runs as root |
| Launch Agent | Per-user, runs as user |
| Mach Service | Registered with launchd |

## XPC from Command Line

### Viewing XPC Services

```bash
# List XPC services in an application
$ ls /Applications/Safari.app/Contents/XPCServices/
com.apple.Safari.SearchHelper.xpc
com.apple.Safari.SandboxBroker.xpc
...

# View XPC service info
$ plutil -p /Applications/Safari.app/Contents/XPCServices/com.apple.Safari.SearchHelper.xpc/Contents/Info.plist
```

### XPC and launchd

```bash
# List Mach services (registered with launchd)
$ launchctl print system | grep "mach services"

# Check specific service
$ launchctl print system/com.apple.metadata.mds
```

### Debugging XPC

```bash
# View XPC activity in logs
$ log show --predicate 'subsystem == "com.apple.xpc"' --last 10m

# Activity related to specific service
$ log show --predicate 'process == "com.apple.xpc.serviceproxy"' --last 1h
```

## Mach Ports

Mach ports underlie all macOS IPC:

```bash
# List Mach ports for a process
$ sudo lsmp -p <pid>
Process (12345) : Safari
  name      ipc-object    rights     flags   boost  reqs...
---------   ----------    ------     -----   -----  ----
...

# View Mach port rights
$ sudo lsmp -a | head -50
```

## Unix IPC Tools

### Named Pipes (FIFOs)

```bash
# Create named pipe
$ mkfifo /tmp/mypipe

# Terminal 1: Read from pipe
$ cat /tmp/mypipe

# Terminal 2: Write to pipe
$ echo "Hello" > /tmp/mypipe

# Cleanup
$ rm /tmp/mypipe
```

### Unix Domain Sockets

```bash
# List Unix sockets
$ netstat -f unix | head -20

# Find socket files
$ find /var/run -name "*.sock" 2>/dev/null
```

### Shared Memory

```bash
# List shared memory segments
$ ipcs -m

# Detailed info
$ ipcs -a
```

## Practical Applications

### Checking Service Communication

```bash
# See what an app is connecting to
$ lsof -c Safari | grep -E "sock|unix"

# Network connections by process
$ lsof -i -P | grep Safari
```

### Diagnosing XPC Issues

Common XPC problems:

1. **Service not launching**
   ```bash
   $ log show --predicate 'subsystem == "com.apple.xpc" AND eventMessage CONTAINS "error"' --last 1h
   ```

2. **Entitlement issues**
   ```bash
   $ codesign -d --entitlements :- /path/to/app
   ```

3. **Sandbox violations**
   ```bash
   $ log show --predicate 'eventMessage CONTAINS "sandbox"' --last 1h
   ```

### Creating Simple IPC

For shell scripts, use simple mechanisms:

```bash
# Using a socket with nc
# Server:
$ nc -l 8080

# Client:
$ echo "message" | nc localhost 8080

# Using files (simple but not ideal)
$ echo "message" > /tmp/ipc_file
$ inotifywait -e modify /tmp/ipc_file  # Linux
# macOS alternative: use fswatch
$ fswatch /tmp/ipc_file
```

## Security Considerations

### XPC Security Features

1. **Code signing validation**: XPC verifies code signatures
2. **Entitlements**: Services can require specific entitlements
3. **Sandboxing**: XPC respects sandbox boundaries
4. **Audit tokens**: Services can verify client identity

### Checking Security

```bash
# Verify code signature
$ codesign -vvv /path/to/service

# Check entitlements
$ codesign -d --entitlements :- /path/to/service

# Check sandbox profile
$ sandbox-exec -n /path/to/profile /bin/ls  # Test sandbox
```

## For Developers

XPC services are created in Xcode. Key components:

1. **XPC Service Target**: Bundle containing the service
2. **Info.plist**: Service configuration
3. **NSXPCConnection**: Client-side API
4. **Protocol**: Defines service interface

Example service registration in launchd plist:
```xml
<key>MachServices</key>
<dict>
    <key>com.example.myservice</key>
    <true/>
</dict>
```

## Summary

IPC on macOS:

| Method | Complexity | Use Case |
|--------|------------|----------|
| XPC | High | Modern Apple development |
| Mach ports | Very high | Kernel/system services |
| Unix sockets | Medium | Network-style IPC |
| Named pipes | Low | Simple shell scripts |
| Files/fswatch | Low | Quick prototypes |

Key points:
- XPC is Apple's preferred modern IPC
- Mach ports are the underlying mechanism
- Unix IPC still works for traditional needs
- Use `log show` and `lsof` for debugging
- Security is built into XPC

For most users, understanding XPC helps debug issues and understand how macOS services interact. Actual XPC development requires Xcode and deeper study.
