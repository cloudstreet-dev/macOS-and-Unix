# Comparing launchd to systemd and init

If you're coming from Linux, you're probably familiar with systemd or traditional init scripts. This chapter maps those concepts to launchd, helping you translate your knowledge to macOS.

## Architecture Comparison

| Feature | launchd | systemd | SysVinit |
|---------|---------|---------|----------|
| PID 1 | Yes | Yes | Yes |
| On-demand activation | Yes | Yes | No |
| Socket activation | Yes | Yes | Via inetd |
| Timer units | Via plist | Timer units | Via cron |
| Configuration | XML plists | INI-like units | Shell scripts |
| Dependency management | Implicit | Explicit | Manual |
| Cgroup integration | No | Yes | No |
| Container support | No | Yes | No |

## Concept Mapping

### Service Files

**systemd** (`.service` file):
```ini
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myservice
Restart=always

[Install]
WantedBy=multi-user.target
```

**launchd** (`.plist` file):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.myservice</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myservice</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

### Command Translation

| Task | systemd | launchd |
|------|---------|---------|
| Start service | `systemctl start foo` | `launchctl start foo` |
| Stop service | `systemctl stop foo` | `launchctl stop foo` |
| Enable at boot | `systemctl enable foo` | `launchctl load -w` |
| Disable | `systemctl disable foo` | `launchctl unload -w` |
| Check status | `systemctl status foo` | `launchctl list foo` |
| List services | `systemctl list-units` | `launchctl list` |
| View logs | `journalctl -u foo` | `log show --predicate 'process=="foo"'` |
| Reload config | `systemctl daemon-reload` | Automatic |

### Service Locations

| Type | systemd | launchd |
|------|---------|---------|
| System services | `/etc/systemd/system/` | `/Library/LaunchDaemons/` |
| User services | `~/.config/systemd/user/` | `~/Library/LaunchAgents/` |
| Vendor services | `/usr/lib/systemd/system/` | `/System/Library/LaunchDaemons/` |

## Key Differences

### Dependencies

**systemd**: Explicit dependencies with `After=`, `Requires=`, `Wants=`
```ini
[Unit]
After=network.target postgresql.service
Requires=postgresql.service
```

**launchd**: Implicit dependencies via on-demand activation
```xml
<!-- launchd handles dependencies via on-demand loading -->
<!-- No explicit dependency declaration -->
```

### Service Types

**systemd** has multiple service types:
- `simple`: Default, main process
- `forking`: Forks and exits
- `oneshot`: Runs once
- `notify`: Uses sd_notify
- `dbus`: D-Bus activated

**launchd** determines type implicitly:
- `RunAtLoad`: Starts immediately
- `KeepAlive`: Respawns if exits
- `StartInterval`: Periodic
- `WatchPaths`: Event-triggered

### Restart Policies

**systemd**:
```ini
Restart=always
RestartSec=10
StartLimitBurst=5
```

**launchd**:
```xml
<key>KeepAlive</key>
<true/>
<key>ThrottleInterval</key>
<integer>10</integer>
```

### Environment Variables

**systemd**:
```ini
Environment="VAR1=value1" "VAR2=value2"
EnvironmentFile=/etc/default/myservice
```

**launchd**:
```xml
<key>EnvironmentVariables</key>
<dict>
    <key>VAR1</key>
    <string>value1</string>
    <key>VAR2</key>
    <string>value2</string>
</dict>
```

### Logging

**systemd**: Integrated with journald
```bash
$ journalctl -u myservice -f
```

**launchd**: Uses unified logging or file redirection
```bash
$ log show --predicate 'process == "myservice"' --last 1h
# Or configure StandardOutPath/StandardErrorPath
```

## Migration Guide

### From systemd to launchd

1. **Service name**: Use reverse-domain notation (com.example.service)
2. **ExecStart**: Use `ProgramArguments` array
3. **Restart=always**: Use `KeepAlive`
4. **After=network.target**: Remove (launchd handles implicitly)
5. **Environment**: Use `EnvironmentVariables` dict
6. **Install section**: Use `RunAtLoad` or explicit loading

### Common Patterns

**Simple daemon**:
```xml
<key>RunAtLoad</key>
<true/>
<key>KeepAlive</key>
<true/>
```

**Periodic task** (like systemd timer):
```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>3</integer>
</dict>
```

**Socket activated**:
```xml
<key>Sockets</key>
<dict>
    <key>Listeners</key>
    <dict>
        <key>SockServiceName</key>
        <string>8080</string>
    </dict>
</dict>
```

## Traditional init Comparison

For users familiar with SysVinit:

| SysVinit | launchd Equivalent |
|----------|-------------------|
| `/etc/init.d/foo start` | `launchctl start foo` |
| `/etc/init.d/foo stop` | `launchctl stop foo` |
| `chkconfig foo on` | `launchctl load -w` |
| `update-rc.d foo defaults` | Place plist in LaunchDaemons |
| `/etc/rc.d/rc.local` | Use Launch Agent/Daemon |

## What launchd Doesn't Have

Features in systemd without direct launchd equivalents:

- **Cgroups**: No container-style resource isolation
- **Slice/scope units**: No hierarchical resource management
- **Socket/path/timer as separate units**: All in one plist
- **Templates**: No parameterized service files
- **Drop-in directories**: No .d override directories
- **Portable services**: No standardized service images

## What launchd Has That Others Don't

- **Tight GUI integration**: Agents can interact with user session
- **Power management awareness**: Respects sleep/wake
- **Application bundle integration**: Services in app bundles
- **On-demand loading**: More aggressive than systemd
- **XPC integration**: Modern IPC framework

## Summary

Translation guide:

| Concept | systemd | launchd |
|---------|---------|---------|
| Service definition | .service | .plist |
| Service manager | systemctl | launchctl |
| Logging | journald | Unified log |
| Timers | .timer units | StartCalendarInterval |
| Sockets | .socket units | Sockets dict in plist |
| User services | user units | LaunchAgents |
| System services | system units | LaunchDaemons |

Key mindset shifts:
- launchd prefers on-demand over explicit dependencies
- Configuration is XML instead of INI
- Domains (system/user/gui) replace targets
- Unified logging replaces journald
- Less explicit control, more automatic management

Both are capable service managers; the differences are largely philosophical and syntactic.
