# Disk I/O Optimization

Storage performance significantly impacts overall system responsiveness. This chapter covers measuring disk performance, understanding APFS optimizations, SSD health monitoring, and identifying I/O bottlenecks on macOS.

## Storage Architecture on macOS

### Modern Mac Storage Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                     Applications                                 │
├─────────────────────────────────────────────────────────────────┤
│                     VFS (Virtual File System)                   │
├─────────────────────────────────────────────────────────────────┤
│                     APFS (File System)                          │
│   ├── Snapshots    ├── Clones    ├── Space Sharing             │
│   └── Encryption   └── Compression └── Sparse Files            │
├─────────────────────────────────────────────────────────────────┤
│                     Core Storage (optional legacy)              │
├─────────────────────────────────────────────────────────────────┤
│                     IOKit Storage Stack                         │
│   ├── Block Device Driver                                       │
│   └── NVMe Controller                                           │
├─────────────────────────────────────────────────────────────────┤
│                     Hardware                                     │
│   └── NVMe SSD (Apple Silicon) or SATA/NVMe SSD (Intel)        │
└─────────────────────────────────────────────────────────────────┘
```

### Disk Information

```bash
# List all disks
$ diskutil list
/dev/disk0 (internal):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.1 GB   disk0
   1:             Apple_APFS_ISC                         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk3         494.4 GB   disk0s2
   3:        Apple_APFS_Recovery                         5.4 GB     disk0s3

# APFS container details
$ diskutil apfs list
APFS Container Reference:     disk3
Size (Capacity Ceiling):      494384795648 B (494.4 GB)
Capacity In Use:              234567890123 B (234.6 GB)
Capacity Not Allocated:       259816905525 B (259.8 GB)

# Disk information
$ diskutil info disk0
   Device Identifier:         disk0
   Device Node:               /dev/disk0
   Whole:                     Yes
   Part of Whole:             disk0
   Device / Media Name:       APPLE SSD AP0512Q
   ...
   Solid State:               Yes
   Virtual:                   No
   ...
```

## Measuring Disk Performance

### Using iostat

```bash
# Basic disk statistics
$ iostat -d
              disk0
    KB/t  tps  MB/s
   24.00   45  1.05

# With CPU stats, every 2 seconds
$ iostat -C 2
          cpu     load average        disk0
    us sy id 1m    5m   15m   KB/t  tps  MB/s
    12  5 83 1.25  2.30 2.15  24.0   45  1.05

# Detailed disk stats
$ iostat -d -w 1 -c 5
              disk0
    KB/t  tps  MB/s
   32.00   89  2.78
   16.00  234  3.66
   48.00  156  7.31
```

**Interpreting iostat:**

| Metric | Description | Healthy Range |
|--------|-------------|---------------|
| KB/t | Kilobytes per transfer | Higher = more efficient |
| tps | Transfers per second | Depends on workload |
| MB/s | Throughput | SSD: 500-3000+ MB/s |

### Using fs_usage

Real-time file system tracing:

```bash
# All file system activity
$ sudo fs_usage -f filesys

# Filter by process
$ sudo fs_usage -w Safari

# Only disk I/O (not cache hits)
$ sudo fs_usage -f diskio

# Specific file operations
$ sudo fs_usage -f filesys | grep -E "open|read|write|close"
```

### Using iotop

```bash
# Install iotop
$ brew install iotop

# Monitor I/O by process
$ sudo iotop

# Example output:
# Total DISK READ:  15.2 MB/s | Total DISK WRITE:  5.4 MB/s
# PID   PRIO USER    DISK READ DISK WRITE  COMMAND
# 12345 BE   david   10.2 MB/s  2.3 MB/s   Safari
# 23456 BE   root     4.5 MB/s  3.1 MB/s   mds_stores
```

### Benchmarking with dd

Simple sequential I/O test:

```bash
# Write test (sequential)
$ dd if=/dev/zero of=/tmp/testfile bs=1m count=1024 2>&1 | tail -1
1073741824 bytes transferred in 0.456789 secs (2350 MB/sec)

# Read test (clear cache first for accurate results)
$ sudo purge
$ dd if=/tmp/testfile of=/dev/null bs=1m 2>&1 | tail -1
1073741824 bytes transferred in 0.234567 secs (4577 MB/sec)

# Clean up
$ rm /tmp/testfile
```

### More Accurate Benchmarking

```bash
# Install fio for comprehensive benchmarking
$ brew install fio

# Sequential read
$ fio --name=seqread --rw=read --bs=1m --size=1g --numjobs=1 --runtime=30 --filename=/tmp/fiotest

# Sequential write
$ fio --name=seqwrite --rw=write --bs=1m --size=1g --numjobs=1 --runtime=30 --filename=/tmp/fiotest

# Random read (4K blocks, queue depth 32)
$ fio --name=randread --rw=randread --bs=4k --size=1g --numjobs=1 --iodepth=32 --runtime=30 --filename=/tmp/fiotest

# Random write
$ fio --name=randwrite --rw=randwrite --bs=4k --size=1g --numjobs=1 --iodepth=32 --runtime=30 --filename=/tmp/fiotest

# Clean up
$ rm /tmp/fiotest
```

## APFS Optimizations

### Space Sharing

APFS volumes share space within a container:

```bash
# View space sharing
$ diskutil apfs list
+-- Container disk3 (494.4 GB capacity)
    +-- Volume disk3s1 (Macintosh HD - Data) - 200.5 GB used
    +-- Volume disk3s2 (Macintosh HD) - 15.2 GB used
    +-- Volume disk3s3 (Preboot) - 234 MB used
    +-- Volume disk3s5 (VM) - 2.1 GB used
    Unallocated: 276.4 GB

# All volumes can use the unallocated space
```

### Snapshots

APFS snapshots are instant, space-efficient point-in-time copies:

```bash
# List snapshots
$ tmutil listlocalsnapsshots /
com.apple.TimeMachine.2024-01-15-120000.local

# Snapshot space usage
$ diskutil apfs listSnapshots disk3s1
Snapshots for disk3s1 (4 found):
+-- 12345678-1234-1234-1234-123456789012
|   Name:        com.apple.TimeMachine.2024-01-15-120000.local
|   XID:         12345
|   Created:     2024-01-15 12:00:00
|   Purgeable:   Yes

# Delete a snapshot (frees space)
$ sudo tmutil deletelocalsnapshots 2024-01-15-120000

# Snapshot disk usage
$ diskutil apfs listSnapshots disk3s1 | grep -A2 "Purgeable Space"
```

### Clones

APFS clones share data blocks until modified:

```bash
# Clone a file (instant, shares blocks)
$ cp -c largefile.dmg largefile-clone.dmg

# Check if files share blocks (no direct command, but)
# Both files show full size but disk usage is shared

# Verify with disk usage
$ du -sh largefile.dmg largefile-clone.dmg
10G    largefile.dmg
10G    largefile-clone.dmg

$ df -h .  # Only uses ~10G, not 20G
```

### Sparse Files

APFS handles sparse files efficiently:

```bash
# Create a sparse file
$ dd if=/dev/zero of=sparsefile bs=1 count=0 seek=10g

# Check apparent vs actual size
$ ls -lh sparsefile
-rw-r--r--  1 david  staff    10G Jan 15 12:00 sparsefile

$ du -sh sparsefile
  0B    sparsefile
```

## SSD Considerations

### TRIM Support

TRIM helps SSDs maintain performance by informing them of deleted blocks:

```bash
# Check TRIM status
$ system_profiler SPSerialATADataType | grep -i trim
TRIM Support: Yes

# For NVMe (Apple Silicon, newer Intel Macs)
$ system_profiler SPNVMeDataType | grep -i trim
```

macOS enables TRIM automatically for Apple SSDs. Third-party SSDs may require:

```bash
# Enable TRIM for third-party SSDs (Intel Macs, requires reboot)
$ sudo trimforce enable
```

### SSD Health Monitoring

```bash
# Using smartctl (install smartmontools)
$ brew install smartmontools

# Check SSD health
$ sudo smartctl -a /dev/disk0

# Key metrics to watch:
# - Percentage Used (wear indicator)
# - Available Spare
# - Media and Data Integrity Errors

# Quick health check
$ sudo smartctl -H /dev/disk0
SMART overall-health self-assessment test result: PASSED

# View wear information
$ sudo smartctl -a /dev/disk0 | grep -E "Percentage Used|Available Spare|Data Units"
```

### Wear Leveling

SSDs have limited write cycles. Monitor write volume:

```bash
# Total bytes written (smartctl)
$ sudo smartctl -a /dev/disk0 | grep "Data Units Written"
Data Units Written: 12,345,678 [6.32 TB]

# Daily write rate estimation
# If SSD is 6 months old with 6.32 TB written:
# 6.32 TB / 180 days = ~35 GB/day
```

## Identifying I/O Bottlenecks

### Signs of I/O Bottleneck

| Symptom | Possible Cause |
|---------|---------------|
| Beach ball cursor | App waiting on I/O |
| Slow app launch | Disk reading app files |
| System unresponsive | Heavy disk activity |
| High CPU wait | I/O blocking CPU work |

### Diagnosing with fs_usage

```bash
#!/bin/bash
# io-bottleneck.sh - Find I/O bottlenecks

echo "Monitoring disk I/O for 30 seconds..."
echo "Top 20 files being accessed:"
echo ""

sudo timeout 30 fs_usage -f diskio 2>/dev/null | \
    awk '{print $NF}' | \
    grep -v "^$" | \
    sort | uniq -c | sort -rn | head -20
```

### Finding Heavy I/O Processes

```bash
# Using iostat to see overall disk load
$ iostat -C 1

# Then use fs_usage to find which process
$ sudo fs_usage -f diskio 2>/dev/null | head -100

# Or use Activity Monitor > Disk tab
# Sort by "Bytes Written" or "Bytes Read"
```

### Common I/O Hogs

| Process | Activity | Solution |
|---------|----------|----------|
| mds, mds_stores | Spotlight indexing | Wait, or exclude folders |
| backupd | Time Machine | Schedule backups |
| photolibraryd | Photos library | Wait for completion |
| bird | iCloud sync | Check iCloud status |
| nsurlsessiond | Background downloads | Check for updates |

### Excluding Folders from Spotlight

```bash
# Add exclusion via mdutil
$ sudo mdutil -i off /Volumes/ExternalDrive

# Or for specific folders, use Spotlight preferences
# Or via command:
$ sudo defaults write /.Spotlight-V100/VolumeConfiguration Exclusions -array-add "/path/to/exclude"
$ sudo mdutil -E /
```

## I/O Optimization Strategies

### For General Use

```bash
# 1. Monitor current I/O
$ iostat -C 2

# 2. Identify heavy writers
$ sudo fs_usage -f diskio 2>/dev/null | awk '{print $NF}' | sort | uniq -c | sort -rn | head -10

# 3. Check for Spotlight indexing
$ sudo fs_usage mds_stores

# 4. Verify APFS is healthy
$ diskutil verifyVolume /
```

### For Development

```bash
# Exclude build directories from Spotlight
$ touch ~/Projects/.metadata_never_index

# Use RAM disk for temp files (speeds up builds)
$ diskutil erasevolume HFS+ "RAMDisk" $(hdiutil attach -nomount ram://4194304)
# Creates 2GB RAM disk

# Place build output on RAM disk
$ ln -s /Volumes/RAMDisk/build ~/Projects/myproject/build
```

### For Large File Operations

```bash
# Use cp -c for APFS clones (instant copy)
$ cp -c largefile.dmg copy.dmg

# For move operations, same volume is instant (no copy)
$ mv largefile.dmg /same/volume/newlocation/

# For cross-volume, use rsync with progress
$ rsync -ah --progress source/ destination/
```

## Monitoring Script

```bash
#!/bin/bash
# disk-monitor.sh - Comprehensive disk monitoring

echo "=== Disk Health Check ==="
echo "Date: $(date)"
echo ""

echo "=== Storage Overview ==="
df -h / /System/Volumes/Data 2>/dev/null | awk 'NR==1 || /disk/'
echo ""

echo "=== APFS Container ==="
diskutil apfs list 2>/dev/null | grep -E "Container|Capacity|Volume" | head -20
echo ""

echo "=== Recent I/O Statistics ==="
iostat -d -c 3 -w 1
echo ""

echo "=== Top I/O Processes (5 second sample) ==="
sudo timeout 5 fs_usage -f diskio 2>/dev/null | \
    awk '{print $NF}' | grep -v "^$" | \
    sort | uniq -c | sort -rn | head -10

echo ""
echo "=== Snapshot Usage ==="
SNAPSHOTS=$(tmutil listlocalsnapshots / 2>/dev/null | wc -l)
echo "Local snapshots: $SNAPSHOTS"

echo ""
echo "=== Disk Write Statistics ==="
if command -v smartctl &>/dev/null; then
    sudo smartctl -a /dev/disk0 2>/dev/null | grep -E "Data Units Written|Percentage Used" || echo "SMART data not available"
else
    echo "Install smartmontools for SSD health: brew install smartmontools"
fi
```

## APFS Performance Tuning

### Defragmentation

APFS does not require or support traditional defragmentation. However:

```bash
# APFS automatically:
# - Optimizes file placement
# - Uses copy-on-write to reduce fragmentation
# - Maintains metadata efficiently

# If you suspect fragmentation issues:
# 1. Check available space (low space = more fragmentation)
$ df -h /

# 2. Backup and restore is the nuclear option
# (Recreates file layout optimally)
```

### Volume Group Configuration

```bash
# View volume group (System + Data)
$ diskutil apfs listVolumeGroups

# Modern macOS uses:
# - Macintosh HD (system, read-only)
# - Macintosh HD - Data (user data)
# This improves security and update reliability
```

## Summary

Key disk I/O concepts:

| Tool | Purpose | Usage |
|------|---------|-------|
| iostat | I/O statistics | `iostat -d -w 2` |
| fs_usage | Real-time I/O tracing | `sudo fs_usage -f diskio` |
| diskutil | Disk management | `diskutil apfs list` |
| smartctl | SSD health | `sudo smartctl -a /dev/disk0` |
| tmutil | Snapshot management | `tmutil listlocalsnapshots /` |

APFS features for performance:

| Feature | Benefit |
|---------|---------|
| Space sharing | Efficient multi-volume usage |
| Clones | Instant file copies |
| Snapshots | Fast point-in-time copies |
| Sparse files | Efficient large file handling |
| Copy-on-write | Reduced writes, data integrity |

Quick diagnostic commands:

```bash
# I/O overview
$ iostat -C 2

# Find I/O sources
$ sudo fs_usage -f diskio | head -50

# Check disk space
$ df -h /

# List snapshots
$ tmutil listlocalsnapshots /

# SSD health
$ sudo smartctl -H /dev/disk0

# APFS status
$ diskutil apfs list
```

Understanding disk I/O patterns and APFS features helps maintain optimal storage performance on macOS.
