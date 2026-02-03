# Startup and Boot Process

Understanding the macOS boot process helps you troubleshoot startup issues, optimize boot time, and properly configure services. The boot sequence differs significantly between Intel and Apple Silicon Macs, but both ultimately arrive at launchd managing the running system.

## Boot Process Overview

### Intel Macs

```
┌─────────────────────────────────────────────────────┐
│ 1. Power On                                         │
│    ↓                                                │
│ 2. EFI/UEFI Firmware                               │
│    • POST (Power-On Self Test)                     │
│    • Initialize hardware                           │
│    • Find boot volume                              │
│    ↓                                                │
│ 3. boot.efi (macOS Boot Loader)                    │
│    • Load kernel and kernel extensions             │
│    ↓                                                │
│ 4. XNU Kernel                                      │
│    • Initialize kernel subsystems                  │
│    • Mount root filesystem                         │
│    • Start launchd (PID 1)                        │
│    ↓                                                │
│ 5. launchd (System)                                │
│    • Load LaunchDaemons                            │
│    • Start system services                         │
│    ↓                                                │
│ 6. loginwindow                                     │
│    • Display login screen                          │
│    ↓                                                │
│ 7. User Session launchd                            │
│    • Load LaunchAgents                             │
│    • Start Dock, Finder, user apps                │
└─────────────────────────────────────────────────────┘
```

### Apple Silicon Macs

```
┌─────────────────────────────────────────────────────┐
│ 1. Power On                                         │
│    ↓                                                │
│ 2. Boot ROM / iBoot (Secure Enclave)               │
│    • Hardware initialization                       │
│    • Secure boot chain verification               │
│    • Load LLB (Low-Level Bootloader)              │
│    ↓                                                │
│ 3. iBoot Stage 2                                   │
│    • Verify and load kernel collection            │
│    • Authenticate boot assets                      │
│    ↓                                                │
│ 4. XNU Kernel                                      │
│    • Initialize kernel subsystems                  │
│    • Mount signed system volume                   │
│    • Start launchd (PID 1)                        │
│    ↓                                                │
│ 5-7. Same as Intel...                              │
└─────────────────────────────────────────────────────┘
```

## Firmware Stage

### Intel: EFI/UEFI

Intel Macs use UEFI firmware:

```bash
# View firmware version
$ system_profiler SPHardwareDataType | grep "System Firmware"
      System Firmware Version: 1916.80.2.0.0

# Access EFI variables (limited without SIP disabled)
$ nvram -p
SystemAudioVolume    %80
boot-args
fmm-computer-name    My-Mac

# Set boot arguments (requires SIP disabled)
$ sudo nvram boot-args="-v"  # Verbose boot
```

### Apple Silicon: iBoot

Apple Silicon uses the iBoot chain:

```bash
# View boot loader version
$ system_profiler SPHardwareDataType | grep "OS Loader"
      OS Loader Version: 10151.61.4

# Boot ROM version (in Secure Enclave)
$ system_profiler SPHardwareDataType | grep "Boot ROM"
      Boot ROM Version: 10151.61.4
```

## Startup Modes

### Intel Macs

Hold keys during startup (after power on, before Apple logo):

| Keys | Mode |
|------|------|
| **Command + R** | Recovery Mode |
| **Option** | Startup Manager (choose boot disk) |
| **Shift** | Safe Mode |
| **Command + V** | Verbose Mode |
| **Command + S** | Single User Mode (older macOS) |
| **D** | Apple Diagnostics |
| **N** | NetBoot |
| **T** | Target Disk Mode |
| **Option + Command + R** | Internet Recovery |
| **Command + Option + P + R** | Reset NVRAM |

### Apple Silicon Macs

Different process:

| Action | How |
|--------|-----|
| Recovery Mode | Hold **Power** until "Loading startup options" |
| Startup Manager | Hold **Power**, then select disk |
| Safe Mode | Hold **Shift** during "Loading startup options" |
| Diagnostics | Hold **Power**, then **Command + D** |
| DFU Mode | Special button sequence (for restore) |
| Share Disk | In Recovery, Utilities > Share Disk |

## Verbose Mode

See what happens during boot:

```bash
# Enable verbose boot (Intel)
$ sudo nvram boot-args="-v"

# Apple Silicon: Hold Command+V after selecting boot disk

# Disable verbose boot
$ sudo nvram boot-args=""
```

Verbose boot shows:

```
Darwin Kernel Version 23.2.0: ...
AMFI: allowing...
...
IOKit asserts: ...
...
[PID 1] bootstrap: ...
```

## The Kernel: XNU

XNU (X is Not Unix) is a hybrid kernel:

```bash
# View kernel version
$ uname -a
Darwin hostname 23.2.0 Darwin Kernel Version 23.2.0: Wed Nov 15 21:53:18 PST 2023; root:xnu-10002.61.3~2/RELEASE_ARM64_T6000 arm64

# Kernel location
$ ls -la /System/Library/Kernels/
-rwxr-xr-x  1 root  wheel  kernel
-rwxr-xr-x  1 root  wheel  kernel.release.t8112
...

# View kernel extensions (kexts)
$ kextstat | head -10
Index Refs Address            Size       Wired      Name (Version)
    1  148 0                  0          0          com.apple.kpi.bsd (23.2.0)
    2   18 0                  0          0          com.apple.kpi.dsep (23.2.0)
    3  181 0                  0          0          com.apple.kpi.iokit (23.2.0)
...

# Kernel cache (pre-linked kernel + kexts)
$ ls -la /System/Library/KernelCollections/
```

## launchd: The Heart of macOS

launchd is always PID 1:

```bash
$ ps aux | head -2
USER   PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
root     1   0.1  0.1 410148544  18624   ??  Ss   10:00AM   0:05.23 /sbin/launchd
```

launchd manages all services through these directories:

```bash
# System daemons (boot-time, root)
/System/Library/LaunchDaemons/   # Apple's
/Library/LaunchDaemons/          # Third-party

# User agents (login-time, user)
/System/Library/LaunchAgents/    # Apple's
/Library/LaunchAgents/           # Third-party, all users
~/Library/LaunchAgents/          # Current user only
```

### Boot-Time Services

```bash
# List loaded daemons
$ sudo launchctl list | head -20
PID	Status	Label
-	0	com.apple.airportd
1234	0	com.apple.mds
-	0	com.apple.metadata.mds.index
...

# See what loads at boot
$ ls /Library/LaunchDaemons/
com.docker.vmnetd.plist
com.github.facebook.watchman.plist
homebrew.mxcl.postgresql@14.plist
...
```

### loginwindow and User Sessions

After system services start, loginwindow appears:

```bash
# loginwindow process
$ ps aux | grep loginwindow
root 123 ... /System/Library/CoreServices/loginwindow.app/Contents/MacOS/loginwindow console

# Login hook (deprecated but still works)
$ sudo defaults read com.apple.loginwindow LoginHook
# Not set by default
```

After login, per-user launchd starts:

```bash
# User's launchd session
$ launchctl print user/$(id -u)
com.apple.xpc.launchd.domain.user.501 = {
    type = user
    handle = 501
    ...
    services = {
        com.apple.Dock.agent = { ... }
        com.apple.Finder = { ... }
        ...
    }
}
```

## Login Items

Login items run when a user logs in:

### Modern Login Items (macOS 13+)

```bash
# View login items
$ sfltool dumpbtm
```

### Service Management Framework

```bash
# View login items for current user
$ osascript -e 'tell application "System Events" to get the name of every login item'
```

### Adding Login Items

```bash
# Add login item via AppleScript
$ osascript -e 'tell application "System Events" to make login item at end with properties {name:"MyApp", path:"/Applications/MyApp.app", hidden:false}'

# Remove login item
$ osascript -e 'tell application "System Events" to delete login item "MyApp"'
```

### Launch Agents (Preferred)

Better than login items for services:

```bash
# Create user launch agent
$ cat > ~/Library/LaunchAgents/com.example.myagent.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.myagent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/mycommand</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
EOF

# Load the agent
$ launchctl load ~/Library/LaunchAgents/com.example.myagent.plist
```

## Startup Items (Deprecated)

Old-style startup items still exist but are deprecated:

```bash
# Legacy locations (avoid using)
/Library/StartupItems/
/System/Library/StartupItems/
```

## Controlling Boot Behavior

### systemsetup

```bash
# View startup disk
$ sudo systemsetup -getstartupdisk
Startup Disk: Macintosh HD

# Set startup disk
$ sudo systemsetup -setstartupdisk "Macintosh HD"

# Enable/disable startup from network
$ sudo systemsetup -setnetworkstartup on

# Set startup after power failure
$ sudo systemsetup -setrestartpowerfailure on

# Set startup after freeze
$ sudo systemsetup -setrestartfreeze on

# Wait for network at boot
$ sudo systemsetup -setwaitforstartupafterpowerfailure 30
```

### bless

```bash
# View blessed (bootable) systems
$ sudo bless --info /
finderinfo[0]:     64 => Blessed System Folder is /System/Library/CoreServices
finderinfo[1]:      0 => No Blessed System File
finderinfo[2]:      0 => Open-folder linked list empty
finderinfo[3]:      0 => No OS 9 + X blessed 9 folder
finderinfo[4]:      0 => Unused field unset
finderinfo[5]:     64 => OS X blessed folder is /System/Library/CoreServices
64-bit VSDB volume id:  0xXXXXXXXXXXXXXXXX

# Set boot volume
$ sudo bless --mount /Volumes/OtherDisk --setBoot
```

### nvram

```bash
# View all NVRAM variables
$ nvram -xp

# Common variables
$ nvram boot-args        # Kernel boot arguments
$ nvram SystemAudioVolume  # Startup sound

# Set boot arguments (Intel, SIP must be disabled)
$ sudo nvram boot-args="-v"           # Verbose
$ sudo nvram boot-args="debug=0x100"  # Wait for debugger
$ sudo nvram boot-args="-x"           # Safe mode

# Reset NVRAM from command line
$ sudo nvram -c
```

## Boot Time Analysis

```bash
# Time since boot
$ uptime
10:30  up 5 days, 12:45, 3 users, load averages: 1.75 2.20 2.10

# Exact boot time
$ sysctl kern.boottime
kern.boottime: { sec = 1705234567, usec = 0 } Mon Jan 15 10:00:00 2024

# Last boot time
$ last reboot
reboot    ~                         Mon Jan 15 10:00

# Boot performance (what took time)
$ log show --predicate 'process == "launchd"' --start "$(sysctl -n kern.boottime | awk '{print $4}' | tr -d ',')" --last 2m | head -50
```

## Troubleshooting Boot Issues

### Safe Mode

Safe Mode disables third-party extensions and performs basic checks:

```bash
# Intel: Hold Shift during boot
# Apple Silicon: Hold Shift after selecting disk

# Check if in Safe Mode
$ sysctl -n kern.safeboot
1    # Safe mode
0    # Normal mode

# Or via System Information
$ system_profiler SPSoftwareDataType | grep "Boot Mode"
      Boot Mode: Safe
```

### Verbose Boot

See what's happening:

```bash
# Enable for Intel
$ sudo nvram boot-args="-v"

# Reboot
$ sudo reboot

# Disable after troubleshooting
$ sudo nvram boot-args=""
```

### Single User Mode (Intel, Older macOS)

```bash
# Boot with Command+S
# At prompt:
/sbin/fsck -fy          # Check filesystem
/sbin/mount -uw /       # Mount filesystem read-write
# Make changes...
exit                    # Continue boot
```

### Reset NVRAM/PRAM

```bash
# Intel: Reboot, hold Command+Option+P+R for 20 seconds

# Or from command line
$ sudo nvram -c
$ sudo reboot
```

### Rebuild Kext Cache

```bash
# Rebuild kernel cache
$ sudo kextcache --clear-staging
$ sudo kextcache -u /

# On Apple Silicon with SSV
$ sudo kmutil install --update-all
```

## Automatic Login

```bash
# Check if auto-login is enabled
$ sudo defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser
davidsmith

# Enable auto-login (stored in protected location)
# Better to use System Settings > Users & Groups > Login Options
```

## Power Events

```bash
# View wake/sleep events
$ pmset -g log | tail -20

# View scheduled events
$ pmset -g sched

# Schedule wake
$ sudo pmset schedule wake "01/20/2024 08:00:00"

# Schedule sleep
$ sudo pmset schedule sleep "01/20/2024 23:00:00"

# Cancel scheduled event
$ sudo pmset schedule cancel wake
```

## Summary

The macOS boot process:

| Stage | Component | Purpose |
|-------|-----------|---------|
| 1 | Firmware (EFI/iBoot) | Hardware init, secure boot |
| 2 | Boot loader | Load kernel |
| 3 | XNU Kernel | Core OS, mount root |
| 4 | launchd (system) | Start system services |
| 5 | loginwindow | User authentication |
| 6 | launchd (user) | Start user services |

Key startup modes:

| Mode | Purpose | How |
|------|---------|-----|
| Recovery | Repair, reinstall | Cmd+R / Hold Power |
| Safe | Disable extensions | Shift |
| Verbose | See boot messages | Cmd+V |
| Target Disk | Share disk | T (Intel) |

Essential commands:

```bash
# Boot info
$ uptime
$ sysctl kern.boottime
$ nvram -p

# Service management
$ launchctl list
$ sudo launchctl load /path/to/plist

# Boot configuration
$ sudo systemsetup -getstartupdisk
$ sudo bless --info /

# Troubleshooting
$ log show --predicate 'process == "launchd"' --last 5m
$ kextstat
```

Understanding the boot process helps you configure services correctly, troubleshoot startup problems, and maintain system reliability.
