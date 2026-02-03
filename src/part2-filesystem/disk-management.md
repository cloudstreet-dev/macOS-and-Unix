# Disk Management from the Command Line

While Disk Utility provides a graphical interface for disk operations, the command line offers more power and precision. macOS provides `diskutil` as the primary disk management tool, along with lower-level utilities for specific tasks.

## The diskutil Command

`diskutil` is macOS's comprehensive disk management utility:

```bash
# Get help
$ diskutil
Disk Utility Tool
Utility to manage local disks and volumes
...

# List all verbs (commands)
$ diskutil listFilesystems
$ diskutil list
$ diskutil info
...
```

### Listing Disks and Volumes

```bash
# List all disks and partitions
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
   1:                APFS Volume Macintosh HD            10.1 GB    disk3s1
   2:              APFS Snapshot com.apple.os.update-... 10.1 GB    disk3s1s1
   3:                APFS Volume Preboot                 5.3 GB     disk3s2
   4:                APFS Volume Recovery                1.6 GB     disk3s3
   5:                APFS Volume Data                    258.2 GB   disk3s5
   6:                APFS Volume VM                      3.2 GB     disk3s6

# List with more detail
$ diskutil list -plist  # Machine-readable
$ diskutil list external # Only external disks
$ diskutil list internal # Only internal disks
```

### Disk and Volume Information

```bash
# Detailed information about a disk
$ diskutil info disk0
   Device Identifier:         disk0
   Device Node:               /dev/disk0
   Whole:                     Yes
   Part of Whole:             disk0
   Device / Media Name:       APPLE SSD AP0512Q

   Volume Name:               Not applicable (no file system)
   Mounted:                   Not applicable (no file system)
   File System:               None

   Content (IOContent):       GUID_partition_scheme
   OS Can Be Installed:       No
   Media Type:                Generic
   Protocol:                  Apple Fabric
   SMART Status:              Verified
   Disk Size:                 500.1 GB (500107862016 Bytes)
   ...

# Information about a volume
$ diskutil info disk3s5
   Device Identifier:         disk3s5
   Device Node:               /dev/disk3s5
   Whole:                     No
   Part of Whole:             disk3

   Volume Name:               Data
   Mounted:                   Yes
   Mount Point:               /System/Volumes/Data

   File System Personality:   APFS
   Type (Bundle):             apfs
   Name (User Visible):       APFS
   Owners:                    Enabled
   ...

# Information about current volume
$ diskutil info /
```

### Mounting and Unmounting

```bash
# Unmount a volume (leaves disk attached)
$ diskutil unmount disk3s5
Volume Data on disk3s5 unmounted

# Mount a volume
$ diskutil mount disk3s5
Volume Data on disk3s5 mounted

# Mount at specific location
$ diskutil mount -mountPoint /Volumes/MyMount disk3s5

# Unmount entire disk (all volumes)
$ diskutil unmountDisk disk2
Unmount of all volumes on disk2 was successful

# Force unmount (use with caution)
$ diskutil unmount force disk3s5
```

### Ejecting Disks

```bash
# Eject (unmount and remove)
$ diskutil eject disk2
Disk disk2 ejected

# This is equivalent to ejecting in Finder
```

## Formatting and Erasing

### Erasing a Volume

```bash
# Erase and reformat a volume
$ sudo diskutil eraseVolume APFS "NewVolume" disk2s2

# Specify filesystem type:
#   APFS - Apple File System
#   JHFS+ - Mac OS Extended (Journaled)
#   ExFAT - Cross-platform
#   MS-DOS FAT32 - Legacy compatible
```

### Erasing an Entire Disk

```bash
# Erase and partition entire disk with APFS
$ sudo diskutil eraseDisk APFS "MyDisk" GPT disk2
Started erase on disk2
Unmounting disk
Creating the partition map
Waiting for partitions to activate
Formatting disk2s2 as APFS with name MyDisk
...
Finished erase on disk2

# Options:
#   GPT - GUID Partition Table (modern, recommended)
#   MBR - Master Boot Record (legacy, limited)
#   APM - Apple Partition Map (PowerPC era)
```

### Available Filesystems

```bash
$ diskutil listFilesystems
Formattable file systems

These file system personalities can be used for erasing and partitioning.
...
-------------------------------------------------------------------------------
PERSONALITY                      USER VISIBLE NAME
-------------------------------------------------------------------------------
ExFAT                            ExFAT
Free Space                       Free Space
(or) free
MS-DOS                           MS-DOS (FAT)
MS-DOS FAT12                     MS-DOS (FAT12)
MS-DOS FAT16                     MS-DOS (FAT16)
MS-DOS FAT32                     MS-DOS (FAT32)
(or) fat32
HFS+                             Mac OS Extended
Case-sensitive HFS+              Mac OS Extended (Case-sensitive)
Case-sensitive Journaled HFS+    Mac OS Extended (Case-sensitive, Journaled)
(or) jhfsx
Journaled HFS+                   Mac OS Extended (Journaled)
(or) jhfs+
APFS                             APFS
(or) apfs
Case-sensitive APFS              APFS (Case-sensitive)
(or) apfsx
```

## APFS-Specific Operations

### Working with Containers

```bash
# List APFS containers
$ diskutil apfs list
APFS Container (1 found)
|
+-- Container disk3 494.4 GB
    ====================================================
    APFS Container Reference:     disk3
    Size (Capacity Ceiling):      494384795648 B (494.4 GB)
    Capacity In Use By Volumes:   277712732160 B (277.7 GB) (56.2%)
    Capacity Not Allocated:       216672063488 B (216.7 GB) (43.8%)
    |
    +-< Physical Store disk0s2 494.4 GB
    |   ---------------------------------------------------
    |   APFS Physical Store Disk:   disk0s2
    |   Size:                       494384795648 B (494.4 GB)
    |
    +-> Volume disk3s1 Macintosh HD
    ...

# Create a new APFS container
$ sudo diskutil apfs createContainer disk2s2

# Resize a container
$ sudo diskutil apfs resizeContainer disk3 400GB
```

### Working with Volumes

```bash
# Add volume to container
$ sudo diskutil apfs addVolume disk3 APFS "NewVolume"

# Add encrypted volume
$ sudo diskutil apfs addVolume disk3 APFS "SecureVolume" -passphrase

# Delete volume
$ sudo diskutil apfs deleteVolume disk3s7

# List volumes in container
$ diskutil apfs listVolumes disk3
```

### Encryption

```bash
# Check encryption status
$ diskutil apfs list | grep -A5 "FileVault"

# Encrypt a volume
$ sudo diskutil apfs encryptVolume disk3s5 -user disk

# Decrypt a volume
$ sudo diskutil apfs decryptVolume disk3s5

# Change encryption password
$ sudo diskutil apfs changePassphrase disk3s5
```

### Snapshots

```bash
# List snapshots
$ diskutil apfs listSnapshots disk3s1
Snapshots for disk3s1 (2 found)
|
+-- 8B6C8F4E-XXXX-XXXX-XXXX-XXXXXXXXXXXX
|   Snapshot Name:        com.apple.os.update-XXXX
|   Snapshot UUID:        8B6C8F4E-XXXX-XXXX-XXXX-XXXXXXXXXXXX
...

# Delete a snapshot
$ sudo diskutil apfs deleteSnapshot disk3s1 -uuid 8B6C8F4E-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```

## Partitioning

### Creating Partitions

```bash
# Partition a disk with multiple partitions
$ sudo diskutil partitionDisk disk2 GPT \
    APFS "Data" 100GB \
    APFS "Backup" 100GB \
    "Free Space" "Available" R

# R = Remainder (use remaining space)
```

### Resizing Partitions

```bash
# Resize a partition (grow or shrink)
$ sudo diskutil resizeVolume disk2s2 50GB

# Resize using limits (fill available space)
$ sudo diskutil resizeVolume disk2s2 R
```

### Adding Partitions

```bash
# Add a partition using free space after existing partition
$ sudo diskutil addPartition disk2s2 APFS "NewPartition" 50GB
```

### Merging Partitions

```bash
# Merge partitions (data on second partition is lost)
$ sudo diskutil mergePartitions JHFS+ "MergedVolume" disk2s2 disk2s3
```

## Disk Images

### Creating Disk Images

```bash
# Create empty disk image
$ hdiutil create -size 100m -fs APFS -volname "TestImage" test.dmg

# Create from folder
$ hdiutil create -srcfolder /path/to/folder -volname "Backup" backup.dmg

# Create encrypted disk image
$ hdiutil create -size 1g -fs APFS -encryption AES-256 -volname "Secure" secure.dmg

# Create sparse image (grows as needed)
$ hdiutil create -size 10g -fs APFS -type SPARSE -volname "Sparse" sparse.sparseimage

# Create sparse bundle (grows as bundle of files)
$ hdiutil create -size 10g -fs APFS -type SPARSEBUNDLE -volname "Bundle" bundle.sparsebundle
```

### Mounting and Unmounting Images

```bash
# Mount disk image
$ hdiutil attach image.dmg
/dev/disk4          GUID_partition_scheme
/dev/disk4s1        Apple_APFS
/dev/disk5s1        Apple_APFS                      /Volumes/ImageVolume

# Mount read-only
$ hdiutil attach -readonly image.dmg

# Mount without Finder window
$ hdiutil attach -nobrowse image.dmg

# Unmount/detach
$ hdiutil detach /dev/disk4
# Or
$ hdiutil detach /Volumes/ImageVolume
```

### Converting Images

```bash
# Convert to read-only compressed DMG
$ hdiutil convert source.dmg -format UDZO -o compressed.dmg

# Formats:
#   UDRO - Read-only
#   UDZO - Compressed (read-only)
#   UDBZ - bzip2 compressed
#   UDTO - DVD/CD master
#   UDSB - Sparsebundle
```

### Resizing Images

```bash
# Resize sparse image
$ hdiutil resize -size 20g sparse.sparseimage

# Resize to minimum
$ hdiutil resize -size min sparse.sparseimage

# Compact sparse image (reclaim space)
$ hdiutil compact sparse.sparseimage
```

## Verification and Repair

### Verifying Filesystems

```bash
# Verify volume
$ diskutil verifyVolume disk3s5
Started file system verification on disk3s5 Data
Verifying storage system
Checking volume
...
Verified storage system
Storage system check exit code is 0
Finished file system verification on disk3s5 Data

# Verify disk
$ diskutil verifyDisk disk3
```

### Repairing Filesystems

```bash
# Repair volume (must be unmounted for non-live repair)
$ sudo diskutil repairVolume disk3s5

# For boot volume, use Recovery Mode
# or First Aid in Disk Utility
```

### Using fsck Directly

```bash
# Check APFS volume
$ sudo fsck_apfs -n /dev/disk3s5
# -n = no changes (dry run)

# Repair APFS
$ sudo fsck_apfs -y /dev/disk3s5

# Check HFS+
$ sudo fsck_hfs -n /dev/disk2s2
```

## Secure Erase

### Secure Erase Options

```bash
# Note: Secure erase is less effective on SSDs due to wear leveling
# For SSDs, use FileVault encryption instead

# Zero out free space on volume
$ diskutil secureErase freespace 0 /Volumes/DiskName
# Levels: 0=zero, 1=random, 2=US DoD 7-pass, 3=Gutmann 35-pass, 4=US DoE 3-pass

# Securely erase entire disk
$ diskutil secureErase 0 disk2
```

### For SSDs

```bash
# Enable FileVault for secure deletion via encryption
# Then simply erase - data is unrecoverable without key

# Check TRIM status
$ system_profiler SPSerialATADataType | grep TRIM
```

## Practical Examples

### Format USB Drive for Cross-Platform Use

```bash
# ExFAT: Works on macOS, Windows, Linux
$ sudo diskutil eraseDisk ExFAT "USB_DRIVE" MBR disk2

# Use MBR for maximum compatibility with older systems
```

### Create Bootable macOS Installer

```bash
# Download macOS from App Store, then:
$ sudo /Applications/Install\ macOS\ Sonoma.app/Contents/Resources/createinstallmedia \
    --volume /Volumes/USB_DRIVE
```

### Clone a Volume

```bash
# Using asr (Apple Software Restore)
$ sudo asr restore --source /Volumes/Source --target /Volumes/Target --erase

# Using dd (byte-for-byte copy - use with extreme caution)
$ sudo dd if=/dev/rdisk2 of=/dev/rdisk3 bs=1m

# Note: Use 'r' prefix (rdisk) for raw device, faster
```

### Check Disk Health

```bash
# SMART status
$ diskutil info disk0 | grep SMART
   SMART Status:              Verified

# More detailed SMART info
$ brew install smartmontools
$ sudo smartctl -a /dev/disk0
```

## Summary

macOS disk management from the command line:

| Task | Command |
|------|---------|
| List disks | `diskutil list` |
| Disk info | `diskutil info disk0` |
| Mount volume | `diskutil mount disk3s5` |
| Unmount volume | `diskutil unmount disk3s5` |
| Eject disk | `diskutil eject disk2` |
| Format disk | `diskutil eraseDisk APFS "Name" GPT disk2` |
| Format volume | `diskutil eraseVolume APFS "Name" disk2s2` |
| Add APFS volume | `diskutil apfs addVolume disk3 APFS "Name"` |
| Create disk image | `hdiutil create -size 1g -fs APFS name.dmg` |
| Mount disk image | `hdiutil attach image.dmg` |
| Verify filesystem | `diskutil verifyVolume disk3s5` |
| Repair filesystem | `diskutil repairVolume disk3s5` |

The `diskutil` command provides comprehensive disk management while `hdiutil` handles disk images. For most operations, these tools replace the need for lower-level utilities while providing macOS-appropriate behavior.
