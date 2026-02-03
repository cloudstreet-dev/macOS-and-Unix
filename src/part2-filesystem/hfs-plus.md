# HFS+ Legacy and Migration

While APFS is now the default, HFS+ (Mac OS Extended) served macOS for nearly two decades and remains relevant. You'll encounter it on older systems, external drives, Time Machine backups from before 2017, and in specific scenarios where APFS isn't appropriate.

## HFS+ History and Design

HFS+ debuted in 1998 as an evolution of the original Hierarchical File System (HFS) from 1985. Key improvements included:

- **32-bit allocation blocks**: vs HFS's 16-bit, allowing efficient use of large volumes
- **Unicode filenames**: Up to 255 UTF-16 characters
- **Journaling**: Added in Mac OS X 10.2.2 (2002) for crash recovery
- **Case sensitivity option**: Added for Unix compatibility

### HFS+ Variants

HFS+ comes in several flavors, identified in `diskutil`:

| Variant | Description |
|---------|-------------|
| Mac OS Extended | Base HFS+ without journaling |
| Mac OS Extended (Journaled) | Standard format with journaling enabled |
| Mac OS Extended (Case-sensitive) | Case-sensitive variant |
| Mac OS Extended (Case-sensitive, Journaled) | Both features |

```bash
# Identify HFS+ volumes
$ diskutil list
...
   2:                  Apple_HFS Backup                   1.0 TB     disk2s2
...

$ diskutil info disk2s2 | grep "Type"
   Type (Bundle):                  hfs
   File System Personality:        Mac OS Extended (Journaled)
```

## Why HFS+ Still Exists

### Compatibility Scenarios

**Older Mac hardware**: Macs that can't run macOS High Sierra or later require HFS+.

**Bootable external drives for old Macs**: If you need a bootable drive for older hardware.

**Windows cross-compatibility**: Some tools read HFS+ but not APFS. (Note: Neither is truly cross-platform without third-party tools.)

**Specific applications**: Some virtualization or disk imaging tools may expect HFS+.

### Time Machine Archives

Time Machine backups created before APFS store data on HFS+:

```bash
# Check Time Machine drive format
$ diskutil info /Volumes/TimeMachine | grep "File System"
   File System Personality:        Mac OS Extended (Journaled)
```

Modern Time Machine on APFS uses snapshots differently than the hardlink-based approach on HFS+.

## HFS+ Features

### Journaling

The journal records filesystem changes before they occur, enabling recovery after crashes:

```bash
# Check journaling status
$ diskutil info disk2s2 | grep -i journal
   File System Personality:        Mac OS Extended (Journaled)
   Journal:                        Journal size 40960 KB at offset 0x...

# Enable journaling
$ sudo diskutil enableJournal disk2s2

# Disable journaling (not recommended)
$ sudo diskutil disableJournal disk2s2
```

### Transparent Compression

HFS+ supports transparent file compression (introduced in Snow Leopard):

```bash
# Check if a file is compressed
$ ls -lO /bin/ls
-rwxr-xr-x  1 root  wheel  compressed  38704 Jan  1  2020 /bin/ls

# The compressed flag indicates HFS+ compression
# Actual on-disk size may be smaller than reported size

# Get actual disk usage
$ stat -f "%z bytes (logical), %b blocks (physical)" /bin/ls
```

Note: While APFS carries forward compressed files, it doesn't actively compress new files.

### Resource Forks and Extended Attributes

HFS+ natively supports Apple's extended file metadata:

```bash
# View resource fork (if present)
$ ls -l /path/to/file/..namedfork/rsrc

# View extended attributes
$ xattr -l file
```

## Working with HFS+ Volumes

### Creating HFS+ Volumes

```bash
# Format as HFS+ (Journaled)
$ sudo diskutil eraseDisk JHFS+ "BackupDrive" GPT disk2

# Format a single partition
$ sudo diskutil eraseVolume JHFS+ "NewVolume" disk2s2

# Create case-sensitive HFS+
$ sudo diskutil eraseDisk JHFS+X "CaseSensitive" GPT disk2
```

### Verifying and Repairing

```bash
# Verify HFS+ volume
$ sudo diskutil verifyVolume disk2s2
Started file system verification on disk2s2 Backup
Checking Journaled HFS Plus volume
...
The volume Backup appears to be OK
Finished file system verification on disk2s2 Backup

# Repair HFS+ volume
$ sudo diskutil repairVolume disk2s2

# For more serious issues, use fsck_hfs directly
$ sudo fsck_hfs -fy /dev/disk2s2
```

### Command-Line Formatting with newfs_hfs

For more control, use the underlying filesystem tools:

```bash
# Create HFS+ filesystem with specific options
$ sudo newfs_hfs -v "MyVolume" -J /dev/disk2s2

# Options:
#   -v: Volume name
#   -J: Enable journaling
#   -s: Case-sensitive
#   -b: Block size
```

## Converting HFS+ to APFS

### In-Place Conversion

Apple provides non-destructive HFS+ to APFS conversion:

```bash
# Check if conversion is possible
$ diskutil info disk2s2 | grep -i "APFS"

# Convert (non-destructive, but backup first!)
$ sudo diskutil apfs convert disk2s2
Converting HFS volume to APFS...
...
```

Conversion requirements:
- macOS High Sierra or later
- Journaled HFS+ (not plain HFS+)
- No Fusion Drive issues
- Sufficient free space for conversion process

### When Conversion Isn't Possible

If conversion fails or isn't desired:

```bash
# Backup, erase, restore approach
# 1. Backup all data
# 2. Erase as APFS
$ sudo diskutil eraseDisk APFS "NewDrive" GPT disk2
# 3. Restore data
```

## HFS+ Limitations

### No Space Sharing

HFS+ requires fixed partition sizes:

```bash
# Partition must be resized explicitly
$ sudo diskutil resizeVolume disk2s2 500GB
```

### No Native Snapshots

HFS+ lacks APFS's snapshot capability. Time Machine on HFS+ uses hard links to directories, a workaround with its own complexities.

### Performance on SSD

HFS+ wasn't designed for flash storage:
- No TRIM support optimization
- Journaling patterns not ideal for SSD
- File operations less efficient than APFS

### File Size and Volume Limits

| Limit | HFS+ |
|-------|------|
| Maximum volume size | 8 EB (theoretical) |
| Maximum file size | 8 EB (theoretical) |
| Maximum files | ~4.3 billion |
| Maximum filename | 255 characters |

Practical limits are lower due to macOS constraints.

## Interacting with HFS+ from Unix Perspective

### Mount Options

Hbash
# View mount options for HFS+ volume
$ mount | grep hfs
/dev/disk2s2 on /Volumes/Backup (hfs, local, nodev, nosuid, journaled)

# Mount HFS+ volume read-only
$ sudo mount -t hfs -o ro /dev/disk2s2 /Volumes/Backup
```

### File System Attributes

```bash
# Get HFS+ specific information
$ /usr/sbin/fstyp /dev/disk2s2
hfs

# Detailed filesystem info
$ sudo diskutil info -plist disk2s2 | plutil -p -
```

## Common Issues

### Catalog File Corruption

HFS+'s B-tree catalog can become corrupted:

```bash
# Symptoms: Files disappear, disk errors
# Solution: Run Disk Utility or fsck_hfs
$ sudo fsck_hfs -fy /dev/disk2s2
```

### Journal Overflow

On heavily used volumes, the journal can overflow:

```bash
# Symptoms: Slow performance, errors
# Solution: Recreate journal
$ sudo diskutil disableJournal disk2s2
$ sudo diskutil enableJournal disk2s2
```

### Permissions Repair (Deprecated)

Older macOS versions had "Repair Disk Permissions." This is no longer relevant on modern macOS where system files are protected by SIP.

## When to Use HFS+ Today

**Recommended for**:
- External drives shared with older Macs
- Time Machine destinations for older Mac backups
- Virtual machine disk images (check VM software requirements)
- Bootable drives for pre-High Sierra Macs

**Not recommended for**:
- Primary macOS boot drives (use APFS)
- General storage on modern Macs
- SSD drives (APFS is better optimized)

## Summary

HFS+ served macOS well for nearly 20 years, but APFS represents the future. Understanding HFS+ remains valuable for:

- Working with legacy systems and backups
- Troubleshooting older storage media
- Understanding the evolution of macOS storage
- Specific compatibility requirements

For new storage, prefer APFS unless you have a specific reason to use HFS+. The performance, reliability, and features of APFS make it the clear choice for modern Mac use.
