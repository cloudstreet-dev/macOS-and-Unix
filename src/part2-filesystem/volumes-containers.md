# Volumes, Containers, and Snapshots

APFS introduces a new storage hierarchy that differs fundamentally from traditional partitioning. Understanding containers, volumes, and snapshots is essential for effective storage management on modern macOS.

## The APFS Storage Hierarchy

```
Physical Disk
    │
    └── Partition (APFS Container)
            │
            ├── Volume A ─── Snapshot 1
            │                Snapshot 2
            ├── Volume B
            └── Volume C
```

Unlike traditional partitioning where each partition has a fixed size, APFS volumes share space within a container dynamically.

## Understanding Containers

### What Is a Container?

An APFS container is:
- Equivalent to a partition in terms of disk allocation
- A pool of storage that multiple volumes share
- The unit of encryption (container-level encryption wraps all volumes)

```bash
# View containers
$ diskutil apfs list
APFS Container (1 found)
|
+-- Container disk3 494.4 GB
    ====================================================
    APFS Container Reference:     disk3
    Size (Capacity Ceiling):      494384795648 B (494.4 GB)
    Capacity In Use By Volumes:   277848256512 B (277.8 GB) (56.2%)
    Capacity Not Allocated:       216536539136 B (216.5 GB) (43.8%)
    |
    +-< Physical Store disk0s2 494.4 GB
```

### Container vs Partition

| Traditional Partition | APFS Container |
|----------------------|----------------|
| Fixed size | Fixed size |
| One filesystem | Multiple volumes |
| Resizing requires unmount | Volumes resize dynamically |
| Independent of other partitions | Volumes share space |

### Multiple Physical Stores

A container can span multiple physical devices (like Fusion Drive):

```bash
# Fusion Drive example
+-< Physical Store disk0s2 (SSD) 128.0 GB
+-< Physical Store disk1s2 (HDD) 1.0 TB
```

This allows APFS to move data between fast and slow storage automatically.

## Working with Volumes

### Volume Roles

APFS assigns roles that define volume behavior:

| Role | Purpose | Characteristics |
|------|---------|-----------------|
| System | Boot OS | Read-only, sealed |
| Data | User data | Read-write |
| VM | Swap space | Not persistent |
| Preboot | Boot helpers | Small, specific files |
| Recovery | Recovery OS | Hidden, bootable |

```bash
# View volume roles
$ diskutil apfs list | grep "Role:"
    APFS Volume Disk (Role):   disk3s1 (System)
    APFS Volume Disk (Role):   disk3s5 (Data)
    APFS Volume Disk (Role):   disk3s6 (VM)
```

### The System-Data Volume Pair

Modern macOS uses a "firmlinked" pair:

```
System Volume (read-only)     Data Volume (read-write)
/System                       /Users
/Library (system)             /Library (user)
/bin, /sbin, /usr            /Applications
```

Firmlinks make these appear as a single filesystem:

```bash
# These paths are on different volumes but appear unified
$ ls /
Applications  Library  System  Users  ...

# Check which volume a path is on
$ df /Users
Filesystem      512-blocks      Used Available Capacity  Mounted on
/dev/disk3s5   966871088 532846072 422952016    56%    /System/Volumes/Data

$ df /System
Filesystem     512-blocks     Used Available Capacity  Mounted on
/dev/disk3s1   966871088 19782344 422952016     5%    /
```

### Creating Volumes

```bash
# Add a standard volume
$ sudo diskutil apfs addVolume disk3 APFS "DataVolume"
Exporting new APFS Volume "DataVolume" from APFS Container Reference disk3
Started APFS operation on disk3
Preparing to add APFS Volume to APFS Container disk3
Creating APFS Volume
...
Mounting APFS Volume
APFS Volume created and mounted at /Volumes/DataVolume
Finished APFS operation on disk3

# Add case-sensitive volume
$ sudo diskutil apfs addVolume disk3 "Case-sensitive APFS" "CaseSensitive"

# Add encrypted volume
$ sudo diskutil apfs addVolume disk3 APFS "Encrypted" -passphrase
Enter new passphrase:
Re-enter new passphrase:
...

# Add volume with quota (space limit)
$ sudo diskutil apfs addVolume disk3 APFS "Limited" -quota 10g

# Add volume with reserve (guaranteed minimum space)
$ sudo diskutil apfs addVolume disk3 APFS "Reserved" -reserve 5g
```

### Volume Quotas and Reserves

```bash
# Set quota on existing volume (maximum space)
$ sudo diskutil apfs setQuota disk3s7 10g

# Set reserve on existing volume (guaranteed space)
$ sudo diskutil apfs setReserve disk3s7 5g

# Remove quota
$ sudo diskutil apfs setQuota disk3s7 0

# View settings
$ diskutil apfs list | grep -A10 "Volume disk3s7"
    +-> Volume disk3s7 DataVolume
        ---------------------------------------------------
        APFS Volume Disk (Role):   disk3s7 (No specific role)
        Quota:                     10.0 GB (10737418240 Bytes)
        Reserve:                   5.0 GB (5368709120 Bytes)
```

### Deleting Volumes

```bash
# Delete a volume (space returns to container pool)
$ sudo diskutil apfs deleteVolume disk3s7
Started APFS operation on disk3s7 DataVolume
Deleting APFS Volume from APFS Container disk3
...
Finished APFS operation on disk3s7 DataVolume
```

**Warning**: This permanently destroys all data on the volume.

### Renaming Volumes

```bash
# Rename a volume
$ sudo diskutil rename disk3s5 "New Name"

# Or
$ sudo diskutil apfs rename disk3s5 "New Name"
```

## Working with Snapshots

### What Are Snapshots?

Snapshots capture the state of a volume at a point in time:
- Read-only view of the filesystem
- Space-efficient (only stores changes)
- Created instantly
- Used by Time Machine and system updates

### Viewing Snapshots

```bash
# List snapshots with tmutil
$ tmutil listlocalsnapshots /
Snapshots for volume group containing disk /:
com.apple.TimeMachine.2024-01-15-091234.local
com.apple.TimeMachine.2024-01-15-101234.local
com.apple.TimeMachine.2024-01-15-111234.local

# List with diskutil
$ diskutil apfs listSnapshots disk3s1
Snapshots for disk3s1 (2 found)
|
+-- 5A7C2E48-XXXX-XXXX-XXXX-XXXXXXXXXXXX
|   Snapshot Name:        com.apple.os.update-XXXX
|   Snapshot UUID:        5A7C2E48-XXXX-XXXX-XXXX-XXXXXXXXXXXX
|   Snapshot Sealed:      Yes
|
+-- 8B6C8F4E-XXXX-XXXX-XXXX-XXXXXXXXXXXX
    Snapshot Name:        com.apple.TimeMachine.2024-01-15-091234.local
    Snapshot UUID:        8B6C8F4E-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```

### Time Machine Local Snapshots

Time Machine creates hourly local snapshots:

```bash
# Create a local snapshot manually
$ sudo tmutil localsnapshot
Created local snapshot with date: 2024-01-15-143045

# View space used by local snapshots
$ tmutil listlocalsnapshots / | wc -l
      24

# Delete specific snapshot
$ sudo tmutil deletelocalsnapshots 2024-01-15-143045
Deleted local snapshot '2024-01-15-143045'

# Delete all local snapshots
$ sudo tmutil deletelocalsnapshots /
Deleted 24 Time Machine local snapshots
```

### System Update Snapshots

macOS creates snapshots before system updates:

```bash
$ diskutil apfs listSnapshots disk3s1 | grep "update"
    Snapshot Name:        com.apple.os.update-XXXX
```

These allow rollback if an update causes problems.

### Mounting Snapshots

```bash
# Mount a snapshot read-only
$ sudo mkdir /tmp/snapshot_mount
$ sudo mount_apfs -s com.apple.TimeMachine.2024-01-15-091234.local /dev/disk3s5 /tmp/snapshot_mount

# Browse the snapshot
$ ls /tmp/snapshot_mount

# Unmount
$ sudo umount /tmp/snapshot_mount
```

### Snapshot Space Management

Snapshots don't immediately use space, but they prevent space reclamation:

```bash
# View purgeable space (includes snapshot data)
$ diskutil apfs list | grep -A5 "Container disk3"
    Size (Capacity Ceiling):      494384795648 B (494.4 GB)
    Capacity In Use By Volumes:   277848256512 B (277.8 GB) (56.2%)
    Capacity Not Allocated:       216536539136 B (216.5 GB) (43.8%)

# When deleting files, space may not free until snapshots are removed
```

### Deleting Snapshots

```bash
# Delete by name
$ sudo diskutil apfs deleteSnapshot disk3s5 -name "com.apple.TimeMachine.2024-01-15-091234.local"

# Delete by UUID
$ sudo diskutil apfs deleteSnapshot disk3s5 -uuid 8B6C8F4E-XXXX-XXXX-XXXX-XXXXXXXXXXXX

# Delete all Time Machine local snapshots (frees space)
$ sudo tmutil deletelocalsnapshots /
```

## Sealed System Volumes

macOS Big Sur+ uses a "sealed" system volume:

```bash
$ diskutil apfs list | grep -A3 "Macintosh HD"
    +-> Volume disk3s1 Macintosh HD
        ---------------------------------------------------
        APFS Volume Disk (Role):   disk3s1 (System)
        Snapshot Sealed:           Yes
```

### What Sealing Means

- Cryptographic verification of system files
- Any modification breaks the seal
- Ensures system integrity
- Snapshot contains known-good state

### Sealed Snapshot Boot

The system actually boots from a sealed snapshot:

```bash
# Current boot is a snapshot
$ mount | grep "disk3s1"
/dev/disk3s1s1 on / (apfs, sealed, local, read-only, journaled)
#            ^^ Note the 's1' suffix indicating snapshot
```

## Container Operations

### Resizing Containers

```bash
# Shrink container to specific size
$ sudo diskutil apfs resizeContainer disk3 400GB

# Expand container to fill available space
$ sudo diskutil apfs resizeContainer disk3 0
```

### Creating New Containers

```bash
# On a new disk or partition
$ sudo diskutil apfs createContainer disk2s2
Started APFS operation on disk2s2
Creating new APFS Container
...
```

### Deleting Containers

```bash
# Delete container (destroys all volumes!)
$ sudo diskutil apfs deleteContainer disk3
```

**Warning**: This destroys all data in the container.

## Practical Scenarios

### Create a Development Volume

```bash
# Case-sensitive volume for cross-platform development
$ sudo diskutil apfs addVolume disk3 "Case-sensitive APFS" "Development" -quota 100g

# Will mount at /Volumes/Development
$ cd /Volumes/Development
$ git clone git@github.com:user/project.git
```

### Create an Encrypted Work Volume

```bash
# Encrypted volume for sensitive data
$ sudo diskutil apfs addVolume disk3 APFS "WorkSecure" -passphrase

# Lock/unlock manually
$ diskutil apfs lockVolume disk3s7
$ diskutil apfs unlockVolume disk3s7
```

### Recover Space from Snapshots

```bash
# Check space
$ df -h /
Filesystem       Size   Used  Avail Capacity  Mounted on
/dev/disk3s1s1  460Gi  256Gi  180Gi    59%    /

# Delete unnecessary snapshots
$ sudo tmutil deletelocalsnapshots /
$ sudo tmutil deletelocalsnapshots /System/Volumes/Data

# Verify space recovered
$ df -h /
```

### Browse Past File Versions

```bash
# Find available snapshots
$ tmutil listlocalsnapshots /System/Volumes/Data

# Mount a snapshot
$ sudo mkdir /tmp/old_data
$ sudo mount_apfs -s com.apple.TimeMachine.2024-01-14-091234.local /dev/disk3s5 /tmp/old_data

# Browse old versions
$ ls /tmp/old_data/Users/david/Documents/

# Copy needed files
$ cp /tmp/old_data/Users/david/Documents/important.doc ~/Desktop/

# Unmount
$ sudo umount /tmp/old_data
```

## Summary

APFS storage hierarchy:

| Concept | Description | Key Commands |
|---------|-------------|--------------|
| Container | Pool of storage | `diskutil apfs list`, `createContainer`, `deleteContainer` |
| Volume | Filesystem within container | `addVolume`, `deleteVolume`, `setQuota` |
| Snapshot | Point-in-time capture | `listSnapshots`, `deleteSnapshot`, `tmutil` |
| Role | Volume purpose (System, Data, VM) | View with `diskutil apfs list` |

Key advantages of APFS:
- **Space sharing**: No need to pre-size volumes
- **Instant snapshots**: Capture state without copying
- **Fast clones**: Duplicate files without using space
- **Encryption**: Built into the filesystem

Understanding this hierarchy helps you manage storage effectively, recover from issues, and take advantage of features that traditional Unix filesystems don't offer.
