# Recovery Mode and Troubleshooting

Recovery Mode is macOS's built-in rescue environment. It provides tools to repair disks, reinstall the operating system, restore from backups, and access Terminal for advanced troubleshooting. Every macOS administrator should know how to access and use Recovery Mode effectively.

## Accessing Recovery Mode

### Intel Macs

1. Restart your Mac (or turn it on)
2. Immediately press and hold **Command + R**
3. Keep holding until you see the Apple logo or spinning globe
4. Release when you see the macOS Utilities window

Alternative boot options:

| Keys | Result |
|------|--------|
| **Command + R** | Recovery from local partition |
| **Option + Command + R** | Internet Recovery (latest macOS) |
| **Shift + Option + Command + R** | Internet Recovery (original macOS) |

### Apple Silicon Macs

1. Shut down your Mac completely
2. Press and hold the **Power button**
3. Keep holding until "Loading startup options" appears
4. Click **Options**, then **Continue**
5. Select a user account and enter password if prompted
6. You'll see the Recovery utilities

## Recovery Mode Interface

Recovery Mode presents macOS Utilities:

```
┌─────────────────────────────────────────────────────┐
│                  macOS Utilities                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌───────────────────────┐  ┌───────────────────┐  │
│  │ Restore from Time     │  │ Reinstall macOS   │  │
│  │ Machine               │  │ Sonoma            │  │
│  └───────────────────────┘  └───────────────────┘  │
│                                                     │
│  ┌───────────────────────┐  ┌───────────────────┐  │
│  │ Safari                │  │ Disk Utility      │  │
│  │                       │  │                   │  │
│  └───────────────────────┘  └───────────────────┘  │
│                                                     │
│  Menu bar: Apple | Recovery | File | Edit | Window │
│                    Utilities menu → Terminal        │
└─────────────────────────────────────────────────────┘
```

Access Terminal from: **Utilities > Terminal**

## Using Terminal in Recovery Mode

Recovery Terminal is a full bash shell with root privileges:

```bash
# You're automatically root
$ whoami
root

# Filesystem is mounted read-only by default
$ mount
/dev/disk1s5 on / (apfs, local, read-only, journaled)
...

# The main volume is usually mounted at /Volumes
$ ls /Volumes/
Macintosh HD
Macintosh HD - Data
```

### Mount the Main Filesystem Read-Write

```bash
# Find the main volume
$ diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.1 GB   disk0
   1:             Apple_APFS_ISC                         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk3         494.4 GB   disk0s2
   3:        Apple_APFS_Recovery                         5.4 GB     disk0s3

/dev/disk3 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +494.4 GB   disk3
   1:                APFS Volume Macintosh HD            15.2 GB    disk3s1
   2:              APFS Snapshot com.apple.os.update-... 15.2 GB    disk3s1s1
   3:                APFS Volume Macintosh HD - Data     154.3 GB   disk3s2
   ...

# Mount read-write (if not already)
$ mount -uw /Volumes/Macintosh\ HD
```

## Common Recovery Tasks

### Reset Administrator Password

If you've forgotten your admin password:

```bash
# In Recovery Terminal:

# Method 1: Using resetpassword tool (GUI)
$ resetpassword
# This opens Reset Password utility

# Method 2: Command line (older macOS)
# Find the user
$ ls /Volumes/Macintosh\ HD/Users/
admin
david
Shared

# Reset password using dscl
$ dscl -f "/Volumes/Macintosh HD/var/db/dslocal/nodes/Default" localonly -passwd /Local/Default/Users/david newpassword

# Or using Directory Services
$ passwd -u david
Changing password for david.
New password:
Retype new password:
```

**Note:** On Apple Silicon with Secure Enclave, you may need to authenticate with another admin account or use Apple ID recovery.

### Enable Root User

```bash
# In Recovery Terminal:
$ dscl -f "/Volumes/Macintosh HD/var/db/dslocal/nodes/Default" localonly -passwd /Local/Default/Users/root newpassword
```

### Create New Admin User

```bash
# In Recovery Terminal:

# Change to the mounted volume
$ cd "/Volumes/Macintosh HD"

# Create user with dscl
$ dscl -f var/db/dslocal/nodes/Default localonly -create /Local/Default/Users/rescue
$ dscl -f var/db/dslocal/nodes/Default localonly -create /Local/Default/Users/rescue UserShell /bin/zsh
$ dscl -f var/db/dslocal/nodes/Default localonly -create /Local/Default/Users/rescue RealName "Rescue Admin"
$ dscl -f var/db/dslocal/nodes/Default localonly -create /Local/Default/Users/rescue UniqueID 550
$ dscl -f var/db/dslocal/nodes/Default localonly -create /Local/Default/Users/rescue PrimaryGroupID 20
$ dscl -f var/db/dslocal/nodes/Default localonly -create /Local/Default/Users/rescue NFSHomeDirectory /Users/rescue
$ dscl -f var/db/dslocal/nodes/Default localonly -passwd /Local/Default/Users/rescue rescuepassword

# Add to admin group
$ dscl -f var/db/dslocal/nodes/Default localonly -append /Local/Default/Groups/admin GroupMembership rescue

# Create home directory
$ mkdir -p Users/rescue
$ chown 550:20 Users/rescue
```

### Disable System Integrity Protection (SIP)

```bash
# In Recovery Terminal:

# Check current status
$ csrutil status
System Integrity Protection status: enabled.

# Disable SIP
$ csrutil disable
Successfully disabled System Integrity Protection. Please restart the machine for the changes to take effect.

# Selectively disable
$ csrutil enable --without kext
$ csrutil enable --without fs

# Re-enable SIP
$ csrutil enable
```

### Repair Disk

Using Disk Utility GUI:
1. Open Disk Utility
2. Select the disk/volume
3. Click "First Aid"
4. Click "Run"

Using Terminal:

```bash
# List disks
$ diskutil list

# Repair volume
$ diskutil repairVolume /dev/disk3s1

# Repair disk (entire disk)
$ diskutil repairDisk /dev/disk0

# For HFS+ filesystems
$ /sbin/fsck_hfs -fy /dev/disk3s1

# For APFS
$ /sbin/fsck_apfs -y /dev/disk3s1

# Repair permissions (older macOS)
$ diskutil repairPermissions /
```

### Erase and Reinstall

From Disk Utility:
1. Select the internal drive (container)
2. Click "Erase"
3. Name it, choose APFS format
4. Quit Disk Utility
5. Choose "Reinstall macOS"

From Terminal:

```bash
# WARNING: This erases everything!

# List disks
$ diskutil list

# Erase and format as APFS
$ diskutil eraseDisk APFS "Macintosh HD" disk0

# Or erase just the container
$ diskutil apfs deleteContainer disk3
$ diskutil apfs createContainer disk0s2
$ diskutil apfs addVolume disk3 APFS "Macintosh HD"
```

### Restore from Time Machine

GUI method:
1. Choose "Restore from Time Machine"
2. Select Time Machine disk
3. Choose backup to restore
4. Select destination disk
5. Click "Restore"

Terminal method:

```bash
# List Time Machine backups
$ tmutil listbackups

# Restore specific files
$ tmutil restore /Volumes/Time\ Machine/Backups.backupdb/MacName/Latest/Macintosh\ HD/Users/david/file.txt /Volumes/Macintosh\ HD/Users/david/

# Full restore (use GUI for this)
```

### Copy Files from Broken System

```bash
# Mount an external drive
$ diskutil list    # Find external drive
$ diskutil mount /dev/disk4s1

# Copy files
$ cp -R "/Volumes/Macintosh HD/Users/david/Documents" /Volumes/External/

# Or use ditto (preserves metadata)
$ ditto "/Volumes/Macintosh HD/Users/david/Documents" /Volumes/External/Documents
```

### Network in Recovery Mode

```bash
# Check network
$ ifconfig en0
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255
    ...

# Wi-Fi (if not connected via menu bar)
$ networksetup -setairportnetwork en0 "NetworkName" "password"

# Test connectivity
$ ping -c 3 apple.com

# Download something
$ curl -O https://example.com/file
```

### Reinstall macOS from Terminal

```bash
# Start the installer
$ /Volumes/Macintosh\ HD/Applications/Install\ macOS\ Sonoma.app/Contents/Resources/startosinstall --agreetolicense --volume /Volumes/Macintosh\ HD

# For Internet Recovery, the installer downloads automatically
```

## Apple Silicon Specific

### Startup Security Utility

In Recovery Mode, you can access Startup Security Utility:

1. **Utilities > Startup Security Utility**
2. Options:
   - **Full Security**: Only current, signed macOS
   - **Reduced Security**: Allows older signed macOS
   - **Permissive Security**: Allows unsigned kexts

```bash
# From Terminal, check security policy
$ bputil -d
```

### Change Security Settings

```bash
# Allow kernel extensions
$ spctl kext-consent add TEAMID

# View kext consent list
$ spctl kext-consent list
```

### Share Disk (Target Disk Mode Alternative)

1. In Recovery Mode
2. **Utilities > Share Disk**
3. Select disk to share
4. Connect to another Mac and access via Finder

## Safe Mode

Boot into Safe Mode for basic troubleshooting:

### Intel Macs
Hold **Shift** during startup

### Apple Silicon Macs
1. Hold Power button
2. Select disk, hold **Shift**
3. Click "Continue in Safe Mode"

Safe Mode:
- Disables login items
- Loads only required kernel extensions
- Clears font caches
- Clears kernel cache
- Runs basic disk check

```bash
# Check if in Safe Mode
$ sysctl kern.safeboot
kern.safeboot: 1
```

## Verbose Mode

See boot messages for troubleshooting:

### Intel Macs
Hold **Command + V** during startup

### Apple Silicon Macs
1. Hold Power button
2. Select disk, press **Command + V**

Or set permanently:

```bash
# Enable verbose boot (Intel)
$ sudo nvram boot-args="-v"
```

## Single User Mode (Deprecated)

Single User Mode is no longer available on Apple Silicon and modern Intel Macs with T2 chip. Use Recovery Mode Terminal instead.

On older Intel Macs:

1. Hold **Command + S** during startup
2. Arrives at root shell
3. Run repairs:

```bash
/sbin/fsck -fy
/sbin/mount -uw /
# Make changes
exit
```

## Troubleshooting Boot Issues

### Mac Won't Start

1. **Check power**: Hold power 10 seconds, release, press again
2. **Reset SMC** (Intel):
   - Shut down
   - Hold Shift + Control + Option + Power for 10 seconds
   - Release all keys, press power
3. **Reset NVRAM** (Intel):
   - Restart, hold Command + Option + P + R for 20 seconds
4. **Safe Mode**: Hold Shift (eliminates software issues)
5. **Recovery Mode**: Command + R (repair or reinstall)
6. **Apple Diagnostics**: Hold D (hardware test)

### Mac Stuck at Login

1. Boot to Safe Mode (Shift)
2. Remove login items:

```bash
# In Recovery Terminal
$ rm /Volumes/Macintosh\ HD/Users/username/Library/Preferences/com.apple.loginitems.plist
```

3. Check LaunchAgents:

```bash
$ ls "/Volumes/Macintosh HD/Users/username/Library/LaunchAgents/"
# Move problematic plists
$ mv "/Volumes/Macintosh HD/Users/username/Library/LaunchAgents/suspect.plist" /tmp/
```

### Application Causing Problems

```bash
# In Recovery Terminal

# Remove app's preferences
$ rm "/Volumes/Macintosh HD/Users/username/Library/Preferences/com.problem.app.plist"

# Remove app's caches
$ rm -rf "/Volumes/Macintosh HD/Users/username/Library/Caches/com.problem.app"

# Remove app's application support
$ rm -rf "/Volumes/Macintosh HD/Users/username/Library/Application Support/Problem App"
```

### Kernel Panic on Boot

```bash
# In Recovery Terminal

# Check for bad kexts in third-party location
$ ls "/Volumes/Macintosh HD/Library/Extensions/"

# Move suspect kext
$ mv "/Volumes/Macintosh HD/Library/Extensions/SuspectDriver.kext" /tmp/

# Rebuild kext cache
$ kmutil install --update-all --volume "/Volumes/Macintosh HD"
```

## Internet Recovery

If Recovery partition is damaged:

### Intel Macs
- **Option + Command + R**: Latest compatible macOS
- **Shift + Option + Command + R**: Original macOS that came with Mac

### Apple Silicon Macs
Internet Recovery is automatic if local recovery fails.

Requirements:
- Network connection (Ethernet or Wi-Fi)
- Connection to Apple's servers
- May take 30-60+ minutes depending on connection

## Recovery Logs

```bash
# View recovery session logs
$ log show --predicate 'subsystem == "com.apple.install"' --last 1h

# Check install logs
$ cat /var/log/install.log
```

## Summary

Recovery Mode is your primary tool for macOS troubleshooting:

| Task | Method |
|------|--------|
| Access Recovery | Cmd+R (Intel) / Hold Power (AS) |
| Reset password | Recovery > Utilities > Terminal |
| Disable SIP | Recovery > Terminal > `csrutil disable` |
| Repair disk | Disk Utility > First Aid |
| Reinstall macOS | Recovery > Reinstall macOS |
| Restore backup | Recovery > Restore from Time Machine |
| Safe Mode | Hold Shift |
| Verbose Mode | Cmd+V |

Key commands in Recovery Terminal:

```bash
# Disk operations
diskutil list
diskutil repairVolume /dev/diskXsY
diskutil eraseDisk APFS "Name" diskX

# Password reset
resetpassword  # GUI tool
dscl -f "/path/to/dslocal" localonly -passwd /Local/Default/Users/name password

# SIP management
csrutil status
csrutil disable
csrutil enable

# File recovery
cp -R /Volumes/Source/path /Volumes/Dest/path
ditto /source /dest
```

Remember: Recovery Mode provides root access to your system. Use it carefully, especially when modifying system files or resetting passwords.
