# Working with NFS and SMB

Network file sharing is essential for development teams and heterogeneous environments. macOS supports both NFS (Network File System, common in Unix environments) and SMB (Server Message Block, the Windows/Samba protocol). Understanding how to mount, configure, and troubleshoot these protocols helps you work seamlessly across macOS, Linux, and Windows systems.

## Protocol Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                    Network Filesystem Comparison                 │
├─────────────────────┬──────────────────────┬────────────────────┤
│        NFS          │         SMB          │        AFP         │
├─────────────────────┼──────────────────────┼────────────────────┤
│ Unix-native         │ Windows-native       │ Apple-native       │
│ Fast on Unix        │ Universal support    │ Legacy (deprecated)│
│ UID/GID mapping     │ Username/password    │ Deprecated         │
│ Ports 111, 2049     │ Port 445             │ Port 548           │
│ Stateless (v3)      │ Stateful             │ Stateful           │
│ Best for Unix-Unix  │ Best for mixed env   │ Avoid for new use  │
└─────────────────────┴──────────────────────┴────────────────────┘
```

## SMB: The Universal Protocol

SMB is the most compatible choice for mixed environments. macOS, Windows, and Linux (via Samba) all support it.

### Connecting to SMB Shares

**Using Finder:**

```bash
# Go → Connect to Server (Cmd+K)
# Enter: smb://server/share
# Or: smb://username@server/share

# For Windows servers:
# smb://DOMAIN;username@server/share
```

**Using command line:**

```bash
# Mount SMB share
$ mount_smbfs //user@server/share /Volumes/share

# With domain
$ mount_smbfs //DOMAIN\;user@server/share /Volumes/share

# With specific options
$ mount_smbfs -o nobrowse //user@server/share /Volumes/share

# Create mount point first
$ mkdir -p /Volumes/myshare
$ mount_smbfs //user@server/share /Volumes/myshare
```

### SMB Mount Options

```bash
# Common mount options
$ mount_smbfs -o option1,option2 //user@server/share /mount/point

# Available options:
# -N            Don't prompt for password (use keychain)
# -o nobrowse   Hide from Finder sidebar
# -o soft       Soft mount (operations can fail)
# -o nodev      Don't interpret device files
# -o nosuid     Ignore setuid bits
```

### Storing SMB Credentials

```bash
# Store credentials in Keychain (happens automatically via Finder)

# Or use security command
$ security add-internet-password \
    -a "username" \
    -s "server.example.com" \
    -w "password" \
    -D "SMB" \
    -T /System/Library/Extensions/smbfs.kext/Contents/Resources/NetAuthSysAgent

# Find stored credentials
$ security find-internet-password -s "server.example.com"
```

### Troubleshooting SMB

```bash
# Check SMB version being used
$ smbutil statshares -a

# List shares on server
$ smbutil view //user@server
Share                                           Type
----------------------------------------------
public                                          Disk
admin$                                          Disk
...

# Check connection
$ smbutil lookup server.local

# Force SMB version (if having compatibility issues)
# Create /etc/nsmb.conf
$ sudo tee /etc/nsmb.conf << 'EOF'
[default]
protocol_vers_map=4  # Force SMB2+
signing_required=no
EOF

# Protocol values:
# 7 = SMB1, SMB2, SMB3
# 6 = SMB2, SMB3
# 4 = SMB2 only
# 2 = SMB3 only
```

### SMB Performance Tuning

```bash
# /etc/nsmb.conf for performance
$ sudo tee /etc/nsmb.conf << 'EOF'
[default]
# Disable signing for performance (less secure)
signing_required=no
validate_neg_off=yes

# Connection timeout
conn_timeout=10

# Use SMB2/3 only (more efficient)
protocol_vers_map=6

# Disable notifications (reduces overhead)
notify_off=yes
EOF

# Per-server configuration
$ sudo tee -a /etc/nsmb.conf << 'EOF'
[fast-server]
streams=yes

[legacy-server]
protocol_vers_map=7  # Include SMB1 for old servers
EOF
```

## NFS: Native Unix File Sharing

NFS provides better performance and Unix compatibility than SMB when sharing between Unix-like systems.

### Mounting NFS Shares

```bash
# Basic NFS mount
$ sudo mount -t nfs server:/export/share /Volumes/share

# With options
$ sudo mount -t nfs -o resvport,rw server:/export/share /Volumes/share

# NFSv4 (macOS 10.15+)
$ sudo mount -t nfs -o vers=4 server:/export /Volumes/share

# NFSv3 (more compatible)
$ sudo mount -t nfs -o vers=3,resvport server:/export/share /Volumes/share
```

### NFS Mount Options

```bash
# Common options for macOS NFS client
-o resvport    # Use privileged port (often required)
-o rw          # Read-write (default)
-o ro          # Read-only
-o soft        # Soft mount (operations can fail)
-o hard        # Hard mount (retry forever)
-o intr        # Allow interrupt
-o bg          # Background mount
-o rsize=32768 # Read buffer size
-o wsize=32768 # Write buffer size
-o tcp         # Use TCP (default)
-o udp         # Use UDP
-o vers=3      # NFS version 3
-o vers=4      # NFS version 4
-o nolock      # Disable file locking
-o locallocks  # Use local locking only

# Example with performance options
$ sudo mount -t nfs -o resvport,rw,rsize=65536,wsize=65536,soft,intr \
    server:/export /Volumes/nfsmount
```

### Setting Up NFS Exports on macOS

macOS can also serve NFS shares:

```bash
# Edit /etc/exports
$ sudo vim /etc/exports

# Export home directory to specific subnet
/Users/developer -mapall=developer -network 192.168.1.0 -mask 255.255.255.0

# Export with specific options
/Volumes/Data -ro -alldirs -network 192.168.1.0 -mask 255.255.255.0

# Export format:
# path [options] [-network ip -mask mask | host...]

# Start NFS server
$ sudo nfsd enable
$ sudo nfsd start

# Check NFS server status
$ sudo nfsd status
nfsd service is enabled
nfsd is running (pid 1234, 8 threads)

# Check exports
$ showmount -e
Exports list on localhost:
/Users/developer                        192.168.1.0

# Reload exports after editing
$ sudo nfsd update
```

### NFS Exports Options

| Option | Description |
|--------|-------------|
| `-ro` | Read-only export |
| `-rw` | Read-write (rarely needed, it's default) |
| `-alldirs` | Allow mounting any subdirectory |
| `-maproot=user` | Map root to specified user |
| `-mapall=user` | Map all users to specified user |
| `-network` | Allowed network |
| `-mask` | Network mask |

### Troubleshooting NFS

```bash
# Check if NFS server is reachable
$ showmount -e server
Exports list on server:
/export/share         192.168.1.0/24

# Check RPC services
$ rpcinfo -p server

# Test mount (verbose)
$ sudo mount -t nfs -v server:/share /mnt
mount_nfs: /mnt: mounted over server:/share

# Common issues:

# "Operation not permitted"
# Solution: Add -o resvport
$ sudo mount -t nfs -o resvport server:/share /mnt

# "Access denied"
# Check: Server exports, network restrictions

# "No such file or directory"
# Check: Export path exists on server

# Stale file handle
$ sudo umount -f /mnt  # Force unmount
$ sudo mount ...       # Remount
```

## Autofs: Automatic Mounting

Autofs mounts filesystems on-demand and unmounts them after inactivity.

### How Autofs Works

```
┌─────────────────────────────────────────────────────────────────┐
│                      Autofs Flow                                 │
│                                                                  │
│  1. Access /Network/Servers or configured path                  │
│           ↓                                                      │
│  2. automountd receives trigger                                  │
│           ↓                                                      │
│  3. Looks up mount info from auto_master/auto_* maps            │
│           ↓                                                      │
│  4. Mounts filesystem                                            │
│           ↓                                                      │
│  5. After timeout (default 60s), unmounts if idle               │
└─────────────────────────────────────────────────────────────────┘
```

### Autofs Configuration Files

```bash
# Main configuration
/etc/auto_master     # Master map, defines mount points

# Map files
/etc/auto_home       # Home directory mounts
/etc/auto_nfs        # NFS mounts
/etc/auto_smb        # SMB mounts (custom)

# View current automounts
$ automount -vc
```

### Configuring Auto-mounted NFS Shares

```bash
# /etc/auto_master - add a mount point
$ sudo vim /etc/auto_master

# Add line:
/mnt/nfs    auto_nfs    -nosuid,nodev

# Create /etc/auto_nfs
$ sudo vim /etc/auto_nfs

# Format: key [options] location
projects    -fstype=nfs,resvport,rw    server:/export/projects
data        -fstype=nfs,resvport,ro    server:/export/data
backup      -fstype=nfs,resvport,rw    nas:/backup

# Reload automount
$ sudo automount -vc
automount: /mnt/nfs updated
```

### Configuring Auto-mounted SMB Shares

```bash
# /etc/auto_master
/mnt/smb    auto_smb    -nosuid,nodev

# /etc/auto_smb
$ sudo vim /etc/auto_smb

# SMB automounts
shared      -fstype=smbfs    ://user:password@server/share
documents   -fstype=smbfs    ://DOMAIN\;user@server/docs

# Note: Storing passwords in plain text is insecure
# Better: Use Keychain credentials

# Alternative: Use URL with no password (prompts or uses Keychain)
documents   -fstype=smbfs    ://user@server/docs
```

### Testing Autofs

```bash
# Reload configuration
$ sudo automount -vc

# Trigger mount by accessing
$ ls /mnt/nfs/projects
# Automount happens automatically

# Check what's mounted
$ mount | grep auto
map auto_nfs on /mnt/nfs (autofs, automounted, nobrowse)

# View automount activity
$ sudo automount -fvc
```

### Autofs on Login Items

For mounts that should be available at login:

```bash
# Create a script that triggers autofs mounts
#!/bin/bash
# ~/bin/mount-shares.sh
ls /mnt/nfs/projects > /dev/null 2>&1
ls /mnt/smb/shared > /dev/null 2>&1

# Add to login items or launchd agent
```

## Permissions and User Mapping

### The UID/GID Challenge

Different systems may use different user IDs:

```
┌─────────────────────────────────────────────────────────────────┐
│                    UID/GID Mapping Issues                        │
│                                                                  │
│  macOS:  user "david" has UID 501                               │
│  Linux:  user "david" has UID 1000                              │
│                                                                  │
│  Files created on Linux with UID 1000 appear as                 │
│  "unknown user" on macOS, and vice versa.                       │
└─────────────────────────────────────────────────────────────────┘
```

### NFS User Mapping

```bash
# On macOS NFS server - map all access to specific user
/Users/shared -mapall=501:20 -network 192.168.1.0 -mask 255.255.255.0

# -mapall=UID:GID maps all remote users to specified UID/GID

# On Linux NFS server (/etc/exports):
/export/shared 192.168.1.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)

# all_squash: Maps all users to anonymous
# anonuid/anongid: UID/GID for anonymous
```

### SMB User Mapping

SMB uses username/password authentication, avoiding UID issues:

```bash
# SMB maps users based on credentials
# Files are owned by the authenticated user

# Force file ownership (macOS mount option)
$ mount_smbfs -o uid=501,gid=20 //user@server/share /Volumes/share
```

## Persistent Mounts

### Using /etc/fstab (Limited on macOS)

macOS supports `/etc/fstab` but with limitations:

```bash
# /etc/fstab format
# device    mount_point    type    options    dump    pass

# NFS mount
server:/export/share    /mnt/nfs    nfs    resvport,rw,bg    0    0

# Note: macOS fstab support is limited
# Autofs or login scripts are often better
```

### Using Login Items

For user-specific mounts:

```bash
# Create mount script
$ cat > ~/bin/mount-shares.sh << 'EOF'
#!/bin/bash
# Mount work shares

# NFS
if ! mount | grep -q "/Volumes/work-nfs"; then
    mkdir -p /Volumes/work-nfs
    mount -t nfs -o resvport server:/export/work /Volumes/work-nfs
fi

# SMB (will use Keychain credentials)
if ! mount | grep -q "/Volumes/work-smb"; then
    mkdir -p /Volumes/work-smb
    mount_smbfs //user@server/share /Volumes/work-smb
fi
EOF

$ chmod +x ~/bin/mount-shares.sh

# Add to System Settings → Login Items
# Or create a launchd agent
```

### Using launchd for Mounting

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.mountshares</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/user/bin/mount-shares.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>300</integer>
</dict>
</plist>
```

```bash
# Save to ~/Library/LaunchAgents/com.user.mountshares.plist
$ launchctl load ~/Library/LaunchAgents/com.user.mountshares.plist
```

## Performance Optimization

### NFS Performance

```bash
# Larger read/write buffers
$ sudo mount -t nfs -o resvport,rsize=65536,wsize=65536 server:/share /mnt

# Use TCP (default, but explicit)
$ sudo mount -t nfs -o resvport,tcp server:/share /mnt

# Disable attribute caching for consistency (slower but more accurate)
$ sudo mount -t nfs -o resvport,noac server:/share /mnt

# Enable attribute caching for speed (default)
$ sudo mount -t nfs -o resvport,ac server:/share /mnt
```

### SMB Performance

```bash
# /etc/nsmb.conf optimizations
[default]
# Disable signing (significant performance impact)
signing_required=no

# Disable opportunistic locks if causing issues
dir_cache_max_cnt=0

# Increase timeout for slow networks
conn_timeout=30

# Disable change notifications
notify_off=yes
```

### Testing Transfer Speeds

```bash
# Test write speed
$ dd if=/dev/zero of=/Volumes/share/testfile bs=1m count=1000
1000+0 records in
1000+0 records out
1048576000 bytes transferred in 12.345 secs (84.9 MB/sec)

# Test read speed
$ dd if=/Volumes/share/testfile of=/dev/null bs=1m
1000+0 records in
1000+0 records out
1048576000 bytes transferred in 8.765 secs (119.6 MB/sec)

# Clean up
$ rm /Volumes/share/testfile
```

## Common Issues and Solutions

### "Operation not permitted" (NFS)

```bash
# macOS requires privileged ports for NFS
$ sudo mount -t nfs -o resvport server:/share /mnt
```

### "Permission denied" (SMB)

```bash
# Check credentials in Keychain
$ security find-internet-password -s "server"

# Try explicit username
$ mount_smbfs //DOMAIN\;username@server/share /mnt

# Reset Keychain entry
$ security delete-internet-password -s "server"
# Then remount (will prompt for password)
```

### Slow Performance

```bash
# For NFS
# - Increase buffer sizes: rsize=65536,wsize=65536
# - Check network path: traceroute server
# - Test raw network speed: iperf3

# For SMB
# - Disable signing in /etc/nsmb.conf
# - Use SMB3 if available: protocol_vers_map=2
# - Check for packet loss: ping -c 100 server
```

### Mount Becomes Unresponsive

```bash
# Force unmount
$ sudo umount -f /Volumes/share

# If that fails, kill processes using mount
$ lsof +D /Volumes/share
# Kill listed processes
$ sudo umount -f /Volumes/share

# Last resort: lazy unmount
$ sudo diskutil unmount force /Volumes/share
```

### Files Show Wrong Ownership

```bash
# For NFS, configure user mapping on server
# For SMB, use uid/gid mount options:
$ mount_smbfs -o uid=$(id -u),gid=$(id -g) //user@server/share /mnt
```

## Quick Reference

### Mounting Shares

| Protocol | Command |
|----------|---------|
| SMB | `mount_smbfs //user@server/share /mnt` |
| NFS | `sudo mount -t nfs -o resvport server:/share /mnt` |
| Finder | `open smb://server/share` or `open nfs://server/share` |

### Common Mount Options

| Protocol | Option | Description |
|----------|--------|-------------|
| NFS | `resvport` | Use privileged port (required for most servers) |
| NFS | `vers=3` or `vers=4` | NFS version |
| NFS | `soft` | Allow operations to fail |
| SMB | `nobrowse` | Hide from Finder sidebar |
| Both | `ro` | Read-only |

### Configuration Files

| File | Purpose |
|------|---------|
| `/etc/nsmb.conf` | SMB client configuration |
| `/etc/exports` | NFS server exports |
| `/etc/auto_master` | Autofs master map |
| `/etc/auto_*` | Autofs mount maps |
| `/etc/fstab` | Static mounts (limited use on macOS) |

## Summary

Choosing between NFS and SMB:

| Use Case | Recommended |
|----------|-------------|
| Mac ↔ Mac | SMB (default) or NFS |
| Mac ↔ Linux | NFS (if pure Unix) or SMB (if mixed) |
| Mac ↔ Windows | SMB |
| Mixed environment | SMB (most compatible) |
| Performance critical (Unix) | NFS |
| Simple setup | SMB (Finder support) |

Key practices:
1. Use autofs for on-demand mounting
2. Store SMB credentials in Keychain
3. Use `resvport` option for NFS on macOS
4. Configure user mapping for NFS to avoid ownership issues
5. Tune SMB settings in `/etc/nsmb.conf` for performance
6. Always use force unmount (`umount -f`) if mount becomes stuck
