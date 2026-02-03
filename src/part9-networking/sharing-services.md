# Sharing Services via Command Line

macOS includes several built-in sharing services that are typically configured through System Preferences > Sharing. This chapter shows how to enable and configure these services from the command line, which is useful for automation, remote administration, and headless systems.

## Overview of Sharing Services

macOS sharing services and their underlying technologies:

| Service | Protocol | Port | CLI Control |
|---------|----------|------|-------------|
| Remote Login (SSH) | SSH | 22 | `systemsetup`, `launchctl` |
| Screen Sharing | VNC/ARD | 5900 | `kickstart`, `defaults` |
| File Sharing | SMB/AFP | 445/548 | `sharing`, `defaults` |
| Remote Management | ARD | 3283 | `kickstart` |
| Remote Apple Events | AE | 3031 | `systemsetup` |
| Printer Sharing | CUPS | 631 | `cupsctl` |
| Internet Sharing | NAT | various | `defaults` |
| Bluetooth Sharing | Bluetooth | - | `defaults` |
| Content Caching | HTTP | 49152+ | `AssetCacheManagerUtil` |

## Remote Login (SSH)

SSH is the most commonly enabled sharing service for command-line access.

### Checking SSH Status

```bash
# Check if SSH is enabled
$ sudo systemsetup -getremotelogin
Remote Login: Off

# Or check the launchd job
$ sudo launchctl list | grep ssh
-       0       com.openssh.sshd
# "-" in PID column means not running
```

### Enabling SSH

```bash
# Enable SSH (Remote Login)
$ sudo systemsetup -setremotelogin on
setremotelogin: Remote Login: On

# Verify it's running
$ sudo launchctl list | grep ssh
123     0       com.openssh.sshd
# PID 123 means it's running

# Test connection
$ ssh localhost
```

### Disabling SSH

```bash
# Disable SSH
$ sudo systemsetup -setremotelogin off

# Force disable (no confirmation)
$ sudo systemsetup -f -setremotelogin off
```

### Configuring SSH Access

By default, all administrators can SSH. To restrict access:

```bash
# Allow only specific users
$ sudo dseditgroup -o create -q com.apple.access_ssh
$ sudo dseditgroup -o edit -a username -t user com.apple.access_ssh

# Remove a user from SSH access
$ sudo dseditgroup -o edit -d username -t user com.apple.access_ssh

# Allow a group
$ sudo dseditgroup -o edit -a admin -t group com.apple.access_ssh

# Check who has access
$ sudo dseditgroup -o read com.apple.access_ssh
```

### SSH Configuration File

SSH server configuration is at `/etc/ssh/sshd_config`:

```bash
# View current config
$ cat /etc/ssh/sshd_config

# Edit (carefully!)
$ sudo nano /etc/ssh/sshd_config

# After changes, restart SSH
$ sudo launchctl stop com.openssh.sshd
$ sudo launchctl start com.openssh.sshd

# Or kick the service
$ sudo launchctl kickstart -k system/com.openssh.sshd
```

Common SSH configuration options:

```
# /etc/ssh/sshd_config
Port 22
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
AllowUsers admin developer
```

## Screen Sharing (VNC)

Screen Sharing uses VNC protocol with Apple's extensions.

### Enabling Screen Sharing

```bash
# Enable Screen Sharing
$ sudo defaults write /var/db/launchd.db/com.apple.launchd/overrides.plist \
    com.apple.screensharing -dict Disabled -bool false

# Load the service
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.screensharing.plist

# Alternative: Using kickstart (for ARD, also enables Screen Sharing)
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate -configure -access -on -restart -agent -privs -all
```

### Configuring Screen Sharing

```bash
# Set VNC password (for non-Apple VNC clients)
$ sudo defaults write /Library/Preferences/com.apple.VNCSettings.txt VNCPassword -data $(echo -n 'password' | xxd -p)

# Allow VNC viewers to control screen
$ sudo defaults write /Library/Preferences/com.apple.RemoteManagement VNCLegacyConnectionsEnabled -bool true
```

### Restricting Screen Sharing Access

```bash
# Create access group
$ sudo dseditgroup -o create -q com.apple.access_screensharing

# Add user to group
$ sudo dseditgroup -o edit -a username -t user com.apple.access_screensharing

# Add admin group
$ sudo dseditgroup -o edit -a admin -t group com.apple.access_screensharing
```

### Disabling Screen Sharing

```bash
# Disable Screen Sharing
$ sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.screensharing.plist

# Or using kickstart
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -deactivate -configure -access -off
```

## Remote Management (Apple Remote Desktop)

ARD provides more features than basic Screen Sharing.

### Using kickstart

The `kickstart` command is the primary tool for ARD configuration:

```bash
# Full path to kickstart
KICKSTART="/System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart"

# Enable Remote Management for all users
$ sudo $KICKSTART -activate -configure -access -on \
    -configure -allowAccessFor -allUsers \
    -configure -restart -agent -privs -all

# Enable for specific users with full privileges
$ sudo $KICKSTART -activate -configure -access -on \
    -configure -allowAccessFor -specifiedUsers \
    -configure -users admin,developer \
    -configure -restart -agent -privs -all

# Enable with limited privileges
$ sudo $KICKSTART -activate -configure -access -on \
    -configure -allowAccessFor -specifiedUsers \
    -configure -users helpdesk \
    -privs -ChangeSettings -TextMessages -ControlObserve -RestartShutDown
```

### Available Privileges

```bash
# View all privilege options
$ sudo $KICKSTART -help

# Common privileges:
# -all                 All privileges
# -ControlObserve      Control and observe screen
# -TextMessages        Send text messages
# -OpenQuitApps        Open and quit applications
# -ChangeSettings      Change system settings
# -RestartShutDown     Restart or shutdown
# -CopyItems           Copy items
# -DeleteFiles         Delete files
# -GenerateReports     Generate reports
```

### Checking ARD Status

```bash
# Check if ARD is enabled
$ sudo $KICKSTART -agent -print

# Check ARD processes
$ ps aux | grep -i "ARDAgent\|RemoteManagement"
```

### Disabling ARD

```bash
$ sudo $KICKSTART -deactivate -configure -access -off
```

## File Sharing (SMB/AFP)

macOS file sharing supports SMB (Windows compatible) and AFP (Apple legacy).

### Using the sharing Command

```bash
# List all shares
$ sharing -l
List of Share Points
name:           Public
path:           /Users/username/Public
afp:            {
        name:   Public
}
smb:            {
        name:   Public
}

# Create a new share
$ sudo sharing -a /path/to/folder -n "MyShare"

# Create SMB-only share
$ sudo sharing -a /path/to/folder -n "SMBShare" -s 100

# Create AFP-only share
$ sudo sharing -a /path/to/folder -n "AFPShare" -a 100

# Remove a share
$ sudo sharing -r "MyShare"
```

### Enabling File Sharing Service

```bash
# Enable SMB
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.smbd.plist

# Check SMB status
$ sudo launchctl list | grep smbd

# Enable AFP (if needed for legacy clients)
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.AppleFileServer.plist
```

### SMB Configuration

```bash
# View SMB configuration
$ cat /etc/smb.conf

# Or check defaults
$ defaults read /Library/Preferences/SystemConfiguration/com.apple.smb.server

# Set workgroup
$ sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.smb.server Workgroup -string "WORKGROUP"

# Set NetBIOS name
$ sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.smb.server NetBIOSName -string "MYMAC"

# Enable guest access
$ sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.smb.server AllowGuestAccess -bool true
```

### Managing Share Permissions

```bash
# Set share to read-only
$ sudo sharing -e "MyShare" -g 001

# Set share to read/write
$ sudo sharing -e "MyShare" -g 003

# Permission bits:
# 001 = read only
# 003 = read/write
# 000 = no access
```

### Disabling File Sharing

```bash
# Disable SMB
$ sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.smbd.plist

# Disable AFP
$ sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.AppleFileServer.plist
```

## Printer Sharing

Printer sharing uses CUPS (Common Unix Printing System).

### Using cupsctl

```bash
# Enable printer sharing
$ sudo cupsctl --share-printers

# Disable printer sharing
$ sudo cupsctl --no-share-printers

# Check CUPS configuration
$ cupsctl
_debug=0
_remote_admin=0
_remote_any=0
_share_printers=1
_user_cancel_any=0
```

### Managing Shared Printers

```bash
# List printers
$ lpstat -p -d
printer Brother_HL-L2350DW_series is idle.  enabled since Mon Jan 15 10:00:00 2024

# Share a specific printer
$ lpadmin -p "Brother_HL-L2350DW_series" -o printer-is-shared=true

# Unshare a printer
$ lpadmin -p "Brother_HL-L2350DW_series" -o printer-is-shared=false

# Allow remote access to CUPS
$ sudo cupsctl --remote-admin --remote-any
```

## Internet Sharing

Internet Sharing shares your Mac's internet connection with other devices.

### Checking Current Configuration

```bash
# View Internet Sharing preferences
$ defaults read /Library/Preferences/SystemConfiguration/com.apple.nat

# Check NAT status
$ sudo pfctl -s nat
```

### Enabling Internet Sharing

Internet Sharing is complex to enable from CLI; it's easier via System Preferences. However, you can script it:

```bash
# Set up NAT sharing from en0 (Ethernet) to bridge100 (Wi-Fi hotspot)
$ sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.nat NAT -dict-add Enabled -bool true
$ sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.nat NAT -dict-add PrimaryService -string "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
$ sudo defaults write /Library/Preferences/SystemConfiguration/com.apple.nat NAT -dict-add SharingDevices -array "bridge100"

# Load Internet Sharing
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.InternetSharing.plist
```

### Creating a Wi-Fi Hotspot

```bash
# Create a software access point
$ sudo networksetup -createnetworkservice "Shared Wi-Fi" Wi-Fi

# Configure the interface
$ sudo networksetup -setmanual "Shared Wi-Fi" 192.168.2.1 255.255.255.0

# Note: Full Wi-Fi hotspot creation requires additional setup
# Consider using the GUI or third-party tools for this
```

## Content Caching

Content Caching caches Apple software updates and iCloud content for local devices.

### Using AssetCacheManagerUtil

```bash
# Check status
$ sudo AssetCacheManagerUtil status
Activated: true
Active: true
CacheUsed: 12345678
CacheFree: 987654321
Port: 49152
...

# Activate content caching
$ sudo AssetCacheManagerUtil activate

# Deactivate
$ sudo AssetCacheManagerUtil deactivate

# Check what's cached
$ sudo AssetCacheManagerUtil listCaches
```

### Content Caching Settings

```bash
# View settings
$ defaults read /Library/Preferences/com.apple.AssetCache.plist

# Set cache location
$ sudo defaults write /Library/Preferences/com.apple.AssetCache.plist CacheFolder -string "/Volumes/Cache/AssetCache"

# Set cache size limit (in bytes)
$ sudo defaults write /Library/Preferences/com.apple.AssetCache.plist CacheSizeLimit -int 107374182400  # 100GB
```

## Remote Apple Events

Remote Apple Events allows AppleScript automation from other Macs.

```bash
# Enable Remote Apple Events
$ sudo systemsetup -setremoteappleevents on

# Disable
$ sudo systemsetup -setremoteappleevents off

# Check status
$ sudo systemsetup -getremoteappleevents
```

## Checking All Sharing Services

### Quick Status Script

```bash
#!/bin/bash
# sharing-status.sh - Check status of all sharing services

echo "=== Sharing Services Status ==="
echo

echo "Remote Login (SSH):"
sudo systemsetup -getremotelogin

echo -e "\nScreen Sharing:"
if launchctl list | grep -q "com.apple.screensharing"; then
    echo "Enabled"
else
    echo "Disabled"
fi

echo -e "\nFile Sharing (SMB):"
if launchctl list | grep -q "com.apple.smbd"; then
    echo "Enabled"
else
    echo "Disabled"
fi

echo -e "\nPrinter Sharing:"
cupsctl | grep share_printers

echo -e "\nRemote Apple Events:"
sudo systemsetup -getremoteappleevents

echo -e "\nContent Caching:"
if AssetCacheManagerUtil status 2>/dev/null | grep -q "Active: true"; then
    echo "Active"
else
    echo "Inactive"
fi
```

## Security Considerations

### Firewall Configuration

When enabling sharing services, ensure the firewall allows the traffic:

```bash
# Check application firewall status
$ /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# Allow incoming connections for a service
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /usr/sbin/sshd
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp /usr/sbin/sshd
```

### Restricting Access by Network

Use the pf firewall to restrict sharing services to specific networks:

```bash
# Example: Allow SSH only from 192.168.1.0/24
# Add to /etc/pf.conf:
pass in on en0 proto tcp from 192.168.1.0/24 to any port 22
block in on en0 proto tcp from any to any port 22
```

### Audit Logging

Enable logging for sharing service access:

```bash
# View SSH login attempts
$ log show --predicate 'process == "sshd"' --last 1h

# View Screen Sharing connections
$ log show --predicate 'subsystem == "com.apple.screensharing"' --last 1h

# View file sharing access
$ log show --predicate 'process == "smbd"' --last 1h
```

## Summary

| Service | Enable Command | Disable Command |
|---------|----------------|-----------------|
| SSH | `sudo systemsetup -setremotelogin on` | `sudo systemsetup -setremotelogin off` |
| Screen Sharing | `sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.screensharing.plist` | `sudo launchctl unload -w ...` |
| File Sharing | `sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.smbd.plist` | `sudo launchctl unload -w ...` |
| Remote Management | `sudo kickstart -activate ...` | `sudo kickstart -deactivate ...` |
| Printer Sharing | `sudo cupsctl --share-printers` | `sudo cupsctl --no-share-printers` |
| Content Caching | `sudo AssetCacheManagerUtil activate` | `sudo AssetCacheManagerUtil deactivate` |

Key points:

- **SSH** (`Remote Login`) is the most commonly enabled service for remote management
- **Screen Sharing** uses VNC with Apple extensions
- **Remote Management** (ARD) provides more control than basic Screen Sharing
- **File Sharing** supports both SMB (Windows) and AFP (legacy Apple)
- Always **restrict access** using groups and the firewall
- **Monitor logs** for unauthorized access attempts
