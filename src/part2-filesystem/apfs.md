# APFS: Apple's Modern Filesystem

Apple File System (APFS) replaced HFS+ as macOS's default filesystem in 2017. Designed for flash storage and modern security requirements, APFS brings capabilities that Unix veterans may find familiar from ZFS or Btrfs—but with Apple's distinctive implementation choices.

## Design Goals

APFS was designed to address HFS+'s limitations:

- **Flash optimization**: HFS+ was designed for spinning disks; APFS is optimized for SSDs
- **Crash protection**: Copy-on-write metadata ensures consistency without full-volume journaling
- **Encryption**: Native encryption integrated at the filesystem level
- **Space efficiency**: Space sharing eliminates wasted space from traditional partitioning
- **Snapshots**: Point-in-time filesystem states for backups and Time Machine
- **Clones**: Instant file copies that share underlying data

## Core Concepts

### Containers and Volumes

APFS introduces a two-level hierarchy that differs from traditional partitioning:

```
┌─────────────────────────────────────────────────┐
│                Physical Disk                     │
├─────────────────────────────────────────────────┤
│              APFS Container                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ Volume  │ │ Volume  │ │ Volume  │           │
│  │ (macOS) │ │ (Data)  │ │ (Preboot│           │
│  │         │ │         │ │         │           │
│  └─────────┘ └─────────┘ └─────────┘           │
│                                                  │
│       All volumes share container space          │
└─────────────────────────────────────────────────┘
```

**Container**: The APFS equivalent of a partition. A container occupies a contiguous region of the disk and can hold multiple volumes.

**Volume**: A logical filesystem within a container. Multiple volumes share the container's space dynamically—no need to pre-allocate sizes.

View your system's structure:

```bash
$ diskutil list
/dev/disk0 (internal):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.1 GB   disk0
   1:             Apple_APFS_ISC Container disk1         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk3         494.4 GB   disk0s2
   3:        Apple_APFS_Recovery Container disk2         5.4 GB     disk0s3

/dev/disk3 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +494.4 GB   disk3
   1:                APFS Volume Macintosh HD            9.8 GB     disk3s1
   2:              APFS Snapshot com.apple.os.update-... 9.8 GB     disk3s1s1
   3:                APFS Volume Preboot                 5.3 GB     disk3s2
   4:                APFS Volume Recovery                1.6 GB     disk3s3
   5:                APFS Volume Data                    256.3 GB   disk3s5
   6:                APFS Volume VM                      3.2 GB     disk3s6
```

### Space Sharing

Unlike traditional partitions with fixed sizes, APFS volumes share space:

```bash
# View container space usage
$ diskutil apfs list
...
+-- Container disk3 494.4 GB
    ====================================================
    APFS Container Reference:     disk3
    Size (Capacity Ceiling):      494384795648 B (494.4 GB)
    Capacity In Use By Volumes:   275847008256 B (275.8 GB) (55.8%)
    Capacity Not Allocated:       218537787392 B (218.5 GB) (44.2%)
    |
    +-< Physical Store disk0s2 494.4 GB
    |
    +-> Volume disk3s1 Macintosh HD
    |   ---------------------------------------------------
    |   APFS Volume Disk (Role):   disk3s1 (System)
    |   Name:                      Macintosh HD
    |   Mount Point:               /
    |   Capacity Consumed:         9.8 GB
    ...
```

The "Capacity Not Allocated" is available to any volume in the container—no repartitioning needed.

### Volume Roles

APFS assigns roles to volumes that affect their behavior:

| Role | Purpose | Mount Point |
|------|---------|-------------|
| System | Boot volume, read-only | `/` |
| Data | User data | `/System/Volumes/Data` (firmlinked to appear at `/Users`, `/Library`, etc.) |
| VM | Swap and sleep image | `/System/Volumes/VM` |
| Preboot | Boot helper data | `/System/Volumes/Preboot` |
| Recovery | Recovery OS | Not normally mounted |

```bash
# View volume roles
$ diskutil apfs listVolumes disk3
...
Role:              Data
Role:              System
Role:              Preboot
Role:              Recovery
Role:              VM
```

## Key Features

### Copy-on-Write

APFS uses copy-on-write (CoW) for metadata and optionally for data:

- **Metadata**: Always CoW; changes write to new locations, then atomically update references
- **Data**: CoW when cloning files; regular writes may overwrite in place

This provides crash protection without the overhead of journaling every write.

### Clones: Instant File Copies

When you copy a file on APFS, the filesystem can create a clone:

```bash
# Create a clone (instant, regardless of file size)
$ cp -c largefile.iso largefile_copy.iso

# Both files share the same data blocks
$ ls -ls largefile.iso largefile_copy.iso
0 -rw-r--r--  1 user  staff  4700000000 Jan 15 10:30 largefile.iso
0 -rw-r--r--  1 user  staff  4700000000 Jan 15 10:30 largefile_copy.iso
```

Notice the `0` in the first column—the copy uses zero additional disk blocks. Modifications to either file write only the changed blocks.

**Note**: The `-c` flag requests cloning, but `cp` on macOS uses clones automatically when possible. The flag ensures failure if cloning isn't possible (e.g., cross-volume copies).

### Snapshots

Snapshots capture the state of a volume at a point in time:

```bash
# List snapshots
$ tmutil listlocalsnapshots /
Snapshots for volume group containing disk /:
com.apple.TimeMachine.2024-01-15-091234.local
com.apple.TimeMachine.2024-01-14-091234.local
...

# Or using diskutil
$ diskutil apfs listSnapshots disk3s1
Snapshots for disk3s1 (1 found)
|
+-- XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
    Snapshot Name:        com.apple.os.update-XXXXXXX
    Snapshot UUID:        XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
    Snapshot Sealed:      Yes
```

Time Machine creates local snapshots hourly. The system creates snapshots before updates for rollback capability.

Creating and managing snapshots:

```bash
# Create a snapshot (requires admin privileges)
$ sudo tmutil localsnapshot /
Created local snapshot with date: 2024-01-15-103045

# Delete a specific snapshot
$ sudo tmutil deletelocalsnapshots 2024-01-15-103045

# Delete all local snapshots
$ sudo tmutil deletelocalsnapshots /
```

### Native Encryption

APFS supports encryption at multiple levels:

**No encryption**: Volume is unencrypted

**Single-key encryption**: Entire volume encrypted with one key

**Multi-key encryption**: Metadata encrypted with one key, file contents with per-file or per-extent keys

```bash
# Check encryption status
$ diskutil apfs list | grep -A2 "FileVault:"
    FileVault:                 Yes (Unlocked)

# Encrypt a volume
$ diskutil apfs encryptVolume disk3s5 -user disk
```

FileVault 2 on APFS uses native APFS encryption rather than the Core Storage layer used with HFS+.

### Space Efficiency Features

**Sparse files**: Files with holes don't consume space for zero-filled regions:

```bash
# Create a sparse file (1GB virtual, ~0 actual)
$ dd if=/dev/zero of=sparse.img bs=1 count=0 seek=1G 2>/dev/null
$ ls -ls sparse.img
0 -rw-r--r--  1 user  staff  1073741824 Jan 15 10:45 sparse.img
```

**Fast directory sizing**: APFS tracks directory sizes efficiently:

```bash
# This is fast on APFS
$ diskutil info disk3s5 | grep "Used"
   Container Total Space:    494.4 GB (494384795648 Bytes)...
   Volume Used Space:        256.3 GB (256300000000 Bytes)...
```

### Atomic Safe-Save

APFS supports atomic rename operations that enable safe document saving:

1. Write new content to a temporary file
2. Atomically rename temporary file to replace original
3. Original is safely replaced without corruption risk

macOS applications use this pattern via `NSDocument`'s save methods.

## APFS Limitations

### No Built-in Compression

Unlike HFS+ (which supported transparent compression) or ZFS/Btrfs (which offer compression), APFS doesn't compress data:

```bash
# Check if a file is compressed (HFS+ compression, may still exist)
$ ls -lO file.txt
-rw-r--r--  1 user  staff  compressed  1000 Jan 15 10:50 file.txt

# APFS doesn't add new compression; existing compressed files remain compressed
```

### No Built-in Checksumming

APFS checksums metadata but not user data. Unlike ZFS, APFS cannot detect silent data corruption (bit rot) in file contents.

### Case Sensitivity Trade-offs

APFS supports case-sensitive volumes, but the default (case-insensitive) maintains compatibility with existing macOS conventions and applications.

### Fusion Drive Complexity

APFS with Fusion Drives (SSD + HDD combinations) works but was added later and lacks some optimizations present on pure SSD.

## Working with APFS

### Creating Volumes

```bash
# Add a volume to an existing container
$ sudo diskutil apfs addVolume disk3 APFS "MyVolume"
# or with case-sensitivity
$ sudo diskutil apfs addVolume disk3 "Case-sensitive APFS" "MyVolume"

# Add an encrypted volume
$ sudo diskutil apfs addVolume disk3 APFS "SecureVolume" -passphrase
```

### Deleting Volumes

```bash
# Delete a volume (space returns to container)
$ sudo diskutil apfs deleteVolume disk3s7
```

### Resizing Containers

```bash
# Resize a container
$ sudo diskutil apfs resizeContainer disk3 400GB

# Resize to fill available space
$ sudo diskutil apfs resizeContainer disk3 0
```

### Converting from HFS+

```bash
# Non-destructive conversion (if supported)
$ sudo diskutil apfs convert disk2s1

# Check conversion eligibility
$ diskutil info disk2s1 | grep "APFS"
```

## Checking Filesystem Health

```bash
# Verify an APFS volume
$ diskutil verifyVolume disk3s1
Started file system verification on disk3s1 Macintosh HD
Verifying storage system
...
Verified storage system
Storage system check exit code is 0
Finished file system verification on disk3s1 Macintosh HD

# Repair (requires booting to Recovery for system volume)
$ sudo diskutil repairVolume disk3s5
```

## APFS vs Other Filesystems

| Feature | APFS | HFS+ | ext4 | ZFS | Btrfs |
|---------|------|------|------|-----|-------|
| Copy-on-Write | Partial | No | No | Yes | Yes |
| Snapshots | Yes | No | No | Yes | Yes |
| Clones | Yes | No | No | Yes | Yes |
| Space Sharing | Yes | No | No | Yes | Yes |
| Data Checksums | Metadata only | No | Metadata | Yes | Yes |
| Compression | No | Limited | No | Yes | Yes |
| Encryption | Native | Via Core Storage | Via LUKS | Yes | Via dm-crypt |
| Flash Optimized | Yes | No | Partial | Yes | Yes |

## Summary

APFS represents a significant advancement over HFS+, bringing macOS filesystems into the modern era with:

- Efficient space sharing through containers and volumes
- Instant file clones and volume snapshots
- Native encryption without external layers
- Crash protection through copy-on-write

For Unix users accustomed to ZFS or Btrfs, APFS will feel familiar in concept but different in implementation. Understanding these differences—and accepting that APFS is optimized for different priorities (consumer devices, flash storage, tight hardware integration)—helps you work effectively with macOS storage.
