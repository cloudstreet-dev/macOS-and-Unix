# System Information Commands

macOS provides several ways to query system information from the command line. Some commands are BSD standards, others are macOS-specific. This chapter covers the essential tools for gathering system details.

## sw_vers: macOS Version

The simplest way to get macOS version information:

```bash
$ sw_vers
ProductName:		macOS
ProductVersion:		14.2.1
BuildVersion:		23C71

# Individual values
$ sw_vers -productName
macOS

$ sw_vers -productVersion
14.2.1

$ sw_vers -buildVersion
23C71
```

Script usage:

```bash
# Check minimum version
MACOS_VERSION=$(sw_vers -productVersion)
MAJOR_VERSION=$(echo "$MACOS_VERSION" | cut -d. -f1)

if [ "$MAJOR_VERSION" -lt 13 ]; then
    echo "This script requires macOS 13 or later"
    exit 1
fi

# Version comparison function
version_gte() {
    [ "$(printf '%s\n' "$1" "$2" | sort -V | head -n1)" = "$2" ]
}

if version_gte "$MACOS_VERSION" "14.0"; then
    echo "Running macOS Sonoma or later"
fi
```

## uname: System Name and Architecture

Standard Unix command available on all Unix-like systems:

```bash
$ uname -a
Darwin MacBook-Pro.local 23.2.0 Darwin Kernel Version 23.2.0: Wed Nov 15 21:53:18 PST 2023; root:xnu-10002.61.3~2/RELEASE_ARM64_T6000 arm64

# Individual options
$ uname -s    # Kernel name
Darwin

$ uname -n    # Network hostname
MacBook-Pro.local

$ uname -r    # Kernel release
23.2.0

$ uname -v    # Kernel version (verbose)
Darwin Kernel Version 23.2.0: Wed Nov 15 21:53:18 PST 2023; root:xnu-10002.61.3~2/RELEASE_ARM64_T6000

$ uname -m    # Machine hardware
arm64

$ uname -p    # Processor type
arm
```

Detect Apple Silicon vs Intel:

```bash
# Check architecture
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
    echo "Apple Silicon"
elif [ "$ARCH" = "x86_64" ]; then
    echo "Intel"
fi

# Check if running under Rosetta 2
if [ "$(sysctl -n sysctl.proc_translated 2>/dev/null)" = "1" ]; then
    echo "Running under Rosetta 2 (Intel emulation)"
fi
```

## hostinfo: Detailed Host Information

macOS-specific command for host details:

```bash
$ hostinfo
Mach kernel version:
	 Darwin Kernel Version 23.2.0: Wed Nov 15 21:53:18 PST 2023; root:xnu-10002.61.3~2/RELEASE_ARM64_T6000
Kernel configured for up to 10 processors.
10 processors are physically available.
10 processors are logically available.
Processor type: arm64e (ARM64E)
Processors active: 0 1 2 3 4 5 6 7 8 9
Primary memory available: 32.00 gigabytes
Default processor set: 436 tasks, 2847 threads, 10 processors
Load average: 2.15, Mach factor: 7.84
```

## sysctl: Kernel Parameters

Query (and set) kernel parameters:

```bash
# Show all parameters
$ sysctl -a

# Common queries
$ sysctl hw.model
hw.model: Mac14,6

$ sysctl hw.ncpu
hw.ncpu: 10

$ sysctl hw.memsize
hw.memsize: 34359738368

$ sysctl hw.physicalcpu
hw.physicalcpu: 10

$ sysctl hw.logicalcpu
hw.logicalcpu: 10

$ sysctl machdep.cpu.brand_string
machdep.cpu.brand_string: Apple M2 Pro
```

### Hardware Information

```bash
# Memory
$ sysctl hw.memsize                    # Total memory in bytes
hw.memsize: 34359738368

$ echo "$(( $(sysctl -n hw.memsize) / 1073741824 )) GB"
32 GB

# CPU
$ sysctl -n hw.ncpu                    # Number of CPUs
10

$ sysctl -n hw.physicalcpu             # Physical CPU cores
10

$ sysctl -n machdep.cpu.brand_string   # CPU model (Intel only)
# On Apple Silicon, this returns empty or fails

# Machine model
$ sysctl -n hw.model
Mac14,6

# L1/L2/L3 Cache
$ sysctl -n hw.l1icachesize            # L1 instruction cache
131072

$ sysctl -n hw.l1dcachesize            # L1 data cache
65536

$ sysctl -n hw.l2cachesize             # L2 cache
4194304
```

### Kernel and System

```bash
# Kernel version
$ sysctl kern.version
kern.version: Darwin Kernel Version 23.2.0...

$ sysctl kern.ostype
kern.ostype: Darwin

$ sysctl kern.osrelease
kern.osrelease: 23.2.0

# Hostname
$ sysctl kern.hostname
kern.hostname: MacBook-Pro.local

# Boot time
$ sysctl kern.boottime
kern.boottime: { sec = 1705123456, usec = 0 } Sat Jan 13 10:00:00 2024

# Uptime
$ sysctl -n kern.boottime | awk '{print $4}' | cut -d, -f1
# Then calculate difference from current time
```

### Network Parameters

```bash
# Maximum socket buffer
$ sysctl net.inet.tcp.sendspace
net.inet.tcp.sendspace: 131072

$ sysctl net.inet.tcp.recvspace
net.inet.tcp.recvspace: 131072

# View all network parameters
$ sysctl net.inet.tcp
```

### Setting Parameters

```bash
# Set parameter (requires sudo, may require SIP disabled)
$ sudo sysctl -w kern.maxfiles=65536
$ sudo sysctl -w kern.maxfilesperproc=65536

# Persist changes (create file in /etc/sysctl.conf)
$ echo "kern.maxfiles=65536" | sudo tee -a /etc/sysctl.conf
```

## system_profiler: Comprehensive System Information

The command-line equivalent of "About This Mac" and System Information app:

```bash
# All information (very verbose, takes time)
$ system_profiler

# List available data types
$ system_profiler -listDataTypes
SPParallelATADataType
SPUniversalAccessDataType
SPSecureElementDataType
SPApplicationsDataType
SPAudioDataType
SPBluetoothDataType
...

# Specific data type
$ system_profiler SPHardwareDataType
Hardware:

    Hardware Overview:

      Model Name: MacBook Pro
      Model Identifier: Mac14,6
      Model Number: MNWA3LL/A
      Chip: Apple M2 Pro
      Total Number of Cores: 10 (6 performance and 4 efficiency)
      Memory: 32 GB
      System Firmware Version: 10151.61.4
      OS Loader Version: 10151.61.4
      Serial Number (system): XXXXXXXXXXXX
      Hardware UUID: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
      Provisioning UDID: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
      Activation Lock Status: Enabled
```

### Common Data Types

```bash
# Hardware overview
$ system_profiler SPHardwareDataType

# Software overview
$ system_profiler SPSoftwareDataType
Software:

    System Software Overview:

      System Version: macOS 14.2.1 (23C71)
      Kernel Version: Darwin 23.2.0
      Boot Volume: Macintosh HD
      Boot Mode: Normal
      Computer Name: MacBook Pro
      User Name: John Doe (john)
      Secure Virtual Memory: Enabled
      System Integrity Protection: Enabled
      Time since boot: 3 days, 2:15

# Memory slots
$ system_profiler SPMemoryDataType

# Storage
$ system_profiler SPStorageDataType
Storage:

    Macintosh HD:

      Available: 234.5 GB (234,500,000,000 bytes)
      Capacity: 494.38 GB (494,380,000,000 bytes)
      File System: APFS
      Writable: Yes
      BSD Name: disk3s5

# Network interfaces
$ system_profiler SPNetworkDataType

# USB devices
$ system_profiler SPUSBDataType

# Displays
$ system_profiler SPDisplaysDataType

# Audio
$ system_profiler SPAudioDataType

# Bluetooth
$ system_profiler SPBluetoothDataType

# Power/Battery
$ system_profiler SPPowerDataType
Power:

    Battery Information:

      Model Information:
          Serial Number: XXXXXXXXXXXX
          Manufacturer: Apple
          Device Name: bq20z451
          ...
      Charge Information:
          Charge Remaining (mAh): 4500
          State of Charge (%): 78
          ...
      Health Information:
          Cycle Count: 125
          Condition: Normal
          Maximum Capacity: 95%
```

### Output Formats

```bash
# XML output (for parsing)
$ system_profiler SPHardwareDataType -xml

# JSON output (macOS 10.15+)
$ system_profiler SPHardwareDataType -json

# Parse JSON with jq
$ system_profiler SPHardwareDataType -json | jq '.SPHardwareDataType[0].chip_type'
"Apple M2 Pro"
```

### Useful Queries

```bash
# Get serial number
$ system_profiler SPHardwareDataType | grep "Serial Number (system)"
      Serial Number (system): XXXXXXXXXXXX

# Or using ioreg
$ ioreg -l | grep IOPlatformSerialNumber | awk -F'"' '{print $4}'
XXXXXXXXXXXX

# Get model identifier
$ system_profiler SPHardwareDataType | grep "Model Identifier"
      Model Identifier: Mac14,6

# Get macOS version
$ system_profiler SPSoftwareDataType | grep "System Version"
      System Version: macOS 14.2.1 (23C71)

# Get battery cycle count
$ system_profiler SPPowerDataType | grep "Cycle Count"
          Cycle Count: 125

# List installed applications
$ system_profiler SPApplicationsDataType

# Get display resolution
$ system_profiler SPDisplaysDataType | grep Resolution
          Resolution: 3456 x 2234 Retina
```

## ioreg: I/O Registry

The I/O Registry contains hardware and driver information:

```bash
# List all devices
$ ioreg -l

# Find specific device
$ ioreg -l | grep -i bluetooth

# Get battery info
$ ioreg -l -n AppleSmartBattery | grep -E "Capacity|CycleCount|Temperature"
    "AppleRawMaxCapacity" = 4500
    "AppleRawCurrentCapacity" = 3510
    "CycleCount" = 125
    "Temperature" = 2987

# Get serial number
$ ioreg -l | grep IOPlatformSerialNumber

# Power adapter info
$ ioreg -l -n AppleSmartBattery | grep -E "ExternalConnected|Charging"
    "ExternalConnected" = Yes
    "IsCharging" = No

# Display information
$ ioreg -l -w 0 | grep -i "IODisplayPrefs" -A 20
```

## df and du: Disk Space

```bash
# Disk free space
$ df -h
Filesystem       Size   Used  Avail Capacity iused ifree %iused  Mounted on
/dev/disk3s1s1  466Gi   15Gi  234Gi     6%  500000  9999999999    0%   /
/dev/disk3s5    466Gi  200Gi  234Gi    46% 1234567  9999999999    0%   /System/Volumes/Data

# Specific filesystem
$ df -h /

# Disk usage of directory
$ du -sh ~/Documents
15G	/Users/john/Documents

# Sort by size
$ du -sh * | sort -hr | head -10

# Include hidden files
$ du -sh .[^.]* * 2>/dev/null | sort -hr | head -10
```

## vm_stat: Virtual Memory Statistics

```bash
$ vm_stat
Mach Virtual Memory Statistics: (page size of 16384 bytes)
Pages free:                               12345.
Pages active:                            234567.
Pages inactive:                          123456.
Pages speculative:                        12345.
Pages throttled:                              0.
Pages wired down:                        234567.
Pages purgeable:                          12345.
"Translation faults":                 123456789.
Pages copy-on-write:                   12345678.
Pages zero filled:                     12345678.
Pages reactivated:                       123456.
Pages purged:                            123456.
File-backed pages:                       234567.
Anonymous pages:                         234567.
Pages stored in compressor:              123456.
Pages occupied by compressor:             12345.
Decompressions:                          123456.
Compressions:                            234567.
Pageins:                                 123456.
Pageouts:                                  1234.
Swapins:                                      0.
Swapouts:                                     0.
```

Convert to human-readable:

```bash
# Function to show memory in GB
meminfo() {
    local pagesize=$(vm_stat | head -1 | grep -oE '[0-9]+')
    local free=$(vm_stat | grep "Pages free" | awk '{print $3}' | tr -d '.')
    local active=$(vm_stat | grep "Pages active" | awk '{print $3}' | tr -d '.')
    local inactive=$(vm_stat | grep "Pages inactive" | awk '{print $3}' | tr -d '.')
    local wired=$(vm_stat | grep "Pages wired" | awk '{print $4}' | tr -d '.')

    echo "Free:     $(echo "scale=2; $free * $pagesize / 1073741824" | bc) GB"
    echo "Active:   $(echo "scale=2; $active * $pagesize / 1073741824" | bc) GB"
    echo "Inactive: $(echo "scale=2; $inactive * $pagesize / 1073741824" | bc) GB"
    echo "Wired:    $(echo "scale=2; $wired * $pagesize / 1073741824" | bc) GB"
}
```

## top and htop: Process Information

```bash
# top (pre-installed)
$ top
# Press 'q' to quit

# Sort by CPU
$ top -o cpu

# Sort by memory
$ top -o rsize

# Show specific user
$ top -U username

# Non-interactive, show once
$ top -l 1

# Show process statistics
$ top -l 1 -n 0 | head -12
Processes: 456 total, 3 running, 453 sleeping, 2345 threads
Load Avg: 2.15, 2.34, 2.45
CPU usage: 5.0% user, 3.0% sys, 92.0% idle
SharedLibs: 300M resident, 50M data, 20M linkedit.
MemRegions: 123456 total, 5G resident, 200M private, 2G shared.
PhysMem: 20G used (2G wired, 10G compressor), 12G unused.
VM: 5T vsize, 3G swapins, 0B swapouts.
Networks: packets: 1234567/1G in, 123456/500M out.
Disks: 1234567/30G read, 234567/20G written.
```

For a better experience:

```bash
# Install htop
$ brew install htop
$ htop
```

## ps: Process Status

```bash
# All processes (BSD style)
$ ps aux

# All processes (System V style)
$ ps -ef

# Process tree
$ ps -ejH

# Find specific process
$ ps aux | grep [n]ginx

# Show processes by memory usage
$ ps aux --sort=-%mem | head -10

# Show processes by CPU usage
$ ps aux --sort=-%cpu | head -10

# Show specific columns
$ ps -eo pid,ppid,user,%cpu,%mem,comm | head -10
```

## Other Useful Commands

### Machine Info

```bash
# Machine serial number
$ system_profiler SPHardwareDataType | awk '/Serial/ {print $4}'

# Or using ioreg
$ ioreg -rd1 -c IOPlatformExpertDevice | awk -F'"' '/IOPlatformSerialNumber/{print $4}'

# Model identifier
$ sysctl -n hw.model

# Check warranty status (use serial with Apple's checker)
```

### last: Login History

```bash
# Show login history
$ last
john     ttys001  192.168.1.100    Mon Jan 15 10:00 - 10:30  (00:30)
john     console                   Mon Jan 15 08:00   still logged in
reboot   ~                         Mon Jan 15 07:59

# Show reboots
$ last reboot

# Show shutdowns
$ last shutdown
```

### uptime: System Uptime

```bash
$ uptime
10:00  up 3 days,  2:15, 2 users, load averages: 2.15 2.34 2.45
```

### w: Who is Logged In

```bash
$ w
10:00  up 3 days,  2:15, 2 users, load averages: 2.15 2.34 2.45
USER     TTY      FROM              LOGIN@  IDLE WHAT
john     console  -                Mon08   3days -
john     s001     192.168.1.100    10:00       - w
```

## Scripting Examples

### Gather System Report

```bash
#!/bin/bash
# system_report.sh - Generate system information report

echo "=== System Report $(date) ==="
echo ""

echo "--- macOS Version ---"
sw_vers

echo ""
echo "--- Hardware ---"
system_profiler SPHardwareDataType | grep -E "Model Name|Chip|Memory|Serial"

echo ""
echo "--- Disk Usage ---"
df -h /

echo ""
echo "--- Memory ---"
vm_stat | head -10

echo ""
echo "--- Uptime ---"
uptime

echo ""
echo "--- Network ---"
ifconfig en0 | grep "inet "
```

### Check System Requirements

```bash
#!/bin/bash
# check_requirements.sh - Verify system meets requirements

MIN_MACOS="13.0"
MIN_RAM_GB=8

# Check macOS version
MACOS_VERSION=$(sw_vers -productVersion)
if [ "$(printf '%s\n' "$MIN_MACOS" "$MACOS_VERSION" | sort -V | head -n1)" != "$MIN_MACOS" ]; then
    echo "Error: Requires macOS $MIN_MACOS or later (found $MACOS_VERSION)"
    exit 1
fi

# Check RAM
RAM_BYTES=$(sysctl -n hw.memsize)
RAM_GB=$((RAM_BYTES / 1073741824))
if [ "$RAM_GB" -lt "$MIN_RAM_GB" ]; then
    echo "Error: Requires ${MIN_RAM_GB}GB RAM (found ${RAM_GB}GB)"
    exit 1
fi

# Check architecture
ARCH=$(uname -m)
echo "System check passed: macOS $MACOS_VERSION, ${RAM_GB}GB RAM, $ARCH"
```

## Summary: Command Reference

| Information | Command |
|-------------|---------|
| macOS version | `sw_vers -productVersion` |
| Kernel version | `uname -r` |
| Architecture | `uname -m` |
| CPU info | `sysctl -n machdep.cpu.brand_string` |
| CPU cores | `sysctl -n hw.ncpu` |
| Memory | `sysctl -n hw.memsize` |
| Model | `sysctl -n hw.model` |
| Serial number | `ioreg -rd1 -c IOPlatformExpertDevice \| awk -F'"' '/Serial/{print $4}'` |
| Hardware details | `system_profiler SPHardwareDataType` |
| Disk space | `df -h` |
| Memory stats | `vm_stat` |
| Battery | `system_profiler SPPowerDataType` |
| Uptime | `uptime` |
| Processes | `ps aux` or `top` |
| Network config | `ifconfig` or `networksetup -getinfo "Wi-Fi"` |
