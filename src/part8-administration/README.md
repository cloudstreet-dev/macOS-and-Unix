# Administering macOS

System administration on macOS blends Unix tradition with Apple's unique approaches. If you're coming from Linux, many concepts will feel familiar, but the tools and methods differ significantly. This part covers what you need to know to effectively manage macOS systems from the command line.

## macOS Administration Philosophy

macOS administration differs from traditional Unix in several key ways:

| Aspect | Traditional Unix | macOS |
|--------|-----------------|-------|
| User management | `/etc/passwd`, `useradd` | Directory Services, `dscl`, `sysadminctl` |
| System permissions | Standard Unix permissions | Unix permissions + ACLs + SIP |
| Firewall | iptables, nftables | pf (from OpenBSD) |
| Logging | syslog, journald | Unified Logging (`log` command) |
| Configuration | Text config files | Property lists (plists), `defaults` |
| Startup | systemd, rc.d | launchd (covered in Part V) |
| Recovery | Single-user mode | Recovery Mode partition |

## The Command Line Administrator

macOS is fully administrable from the terminal. While System Settings provides a GUI, every configuration can be modified from the command line:

```bash
# User management
$ sudo sysadminctl -addUser newuser -fullName "New User" -password -

# Firewall control
$ sudo pfctl -e    # Enable firewall

# View system logs
$ log show --predicate 'eventMessage contains "error"' --last 1h

# Read/write preferences
$ defaults read com.apple.finder
$ defaults write com.apple.dock autohide -bool true

# Check SIP status
$ csrutil status
System Integrity Protection status: enabled.
```

## Privilege Escalation

macOS uses the standard Unix `sudo` mechanism, but with Apple's authentication framework:

```bash
# Standard sudo
$ sudo cat /etc/sudoers
Password: ********

# Check sudo privileges
$ sudo -l
User admin may run the following commands on hostname:
    (ALL) ALL

# Run as different user
$ sudo -u www whoami
www

# Edit with sudo
$ sudo visudo
```

The sudo timeout is typically 5 minutes. You can adjust it in `/etc/sudoers`:

```bash
# Extend timeout to 15 minutes
Defaults timestamp_timeout=15

# Require password every time
Defaults timestamp_timeout=0
```

## System Information Quick Reference

Essential commands for understanding your system:

```bash
# macOS version
$ sw_vers
ProductName:		macOS
ProductVersion:		14.2
BuildVersion:		23C64

# Hardware overview
$ system_profiler SPHardwareDataType
Hardware:

    Hardware Overview:

      Model Name: MacBook Pro
      Model Identifier: Mac14,5
      Chip: Apple M2 Pro
      Total Number of Cores: 12 (8 performance and 4 efficiency)
      Memory: 32 GB
      System Firmware Version: 10151.61.4
      OS Loader Version: 10151.61.4
      Serial Number (system): XXXXXXXXXXXX
      Hardware UUID: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

# CPU information
$ sysctl -n machdep.cpu.brand_string
Apple M2 Pro

# Memory
$ sysctl hw.memsize
hw.memsize: 34359738368

# Disk usage
$ df -h
Filesystem       Size   Used  Avail Capacity iused ifree %iused  Mounted on
/dev/disk3s1s1  926Gi   14Gi  756Gi     2%    404k  7.9G    0%   /
/dev/disk3s6    926Gi  3.0Gi  756Gi     1%       3  7.9G    0%   /System/Volumes/VM
/dev/disk3s2    926Gi  9.5Gi  756Gi     2%    1.4k  7.9G    0%   /System/Volumes/Preboot
/dev/disk3s4    926Gi  143Gi  756Gi    16%    1.9M  7.9G    0%   /System/Volumes/Data

# Uptime
$ uptime
10:30  up 5 days, 12:45, 3 users, load averages: 1.75 2.20 2.10
```

## What You'll Learn in This Part

**[User and Group Management](./users-groups.md)** covers how macOS handles users and groups through Directory Services, including `dscl`, `sysadminctl`, and the relationship between local accounts and directory services.

**[The Permissions Model: Unix Meets ACLs](./permissions-acls.md)** explains how macOS combines traditional Unix permissions with Access Control Lists (ACLs) for fine-grained access control.

**[System Integrity Protection (SIP)](./sip.md)** describes Apple's security feature that protects system files and processes, and when you might need to (carefully) disable it.

**[Configuring the pf Firewall](./pf-firewall.md)** shows how to use the powerful pf firewall inherited from OpenBSD, including rules, tables, and persistent configuration.

**[Unified Logging System](./unified-logging.md)** teaches you to use the `log` command to query macOS's modern logging system, which replaced traditional syslog.

**[System Configuration via defaults](./defaults-command.md)** demonstrates how to read and write macOS preferences from the command line, enabling powerful system customization.

**[Startup and Boot Process](./startup-boot.md)** walks through the macOS boot sequence from firmware through launchd, helping you understand and troubleshoot startup issues.

**[Recovery Mode and Troubleshooting](./recovery-mode.md)** covers accessing Recovery Mode, using its Terminal, reinstalling macOS, and resetting passwords when needed.

## Administrator's Toolkit

A quick reference of essential administration commands:

```bash
# System Management
sudo shutdown -h now          # Shutdown immediately
sudo shutdown -r now          # Restart immediately
sudo shutdown -h +60          # Shutdown in 60 minutes
sudo reboot                   # Restart
sudo systemsetup -setcomputersleep 60    # Sleep after 60 minutes

# Service Management
sudo launchctl list           # List all services
sudo launchctl load /path     # Load service
sudo launchctl unload /path   # Unload service

# User Management
dscl . list /Users            # List users
dscl . read /Users/username   # Read user details
sudo sysadminctl -addUser     # Add user

# Disk Management
diskutil list                 # List disks
diskutil info disk0           # Disk information
sudo diskutil repairDisk disk0  # Repair disk

# Security
csrutil status                # SIP status
spctl --status                # Gatekeeper status
sudo pfctl -s info            # Firewall info
```

## Remote Administration

macOS supports remote administration via SSH and Apple Remote Desktop (ARD):

```bash
# Enable SSH (Remote Login)
$ sudo systemsetup -setremotelogin on
$ sudo systemsetup -getremotelogin
Remote Login: On

# Enable ARD for specific users
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate -configure -access -on -users admin -privs -all -restart -agent

# Check ARD status
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -agent -print

# Disable ARD
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -deactivate -stop
```

The following chapters dive deep into each aspect of macOS system administration, giving you the knowledge to manage macOS systems as effectively from the terminal as any Unix administrator manages their Linux servers.
