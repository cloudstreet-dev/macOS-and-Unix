# Command-Line Performance Tools

The command line provides powerful performance monitoring tools that work over SSH, can be scripted, and often provide more detail than GUI alternatives. This chapter covers essential CLI tools for monitoring CPU, memory, disk, network, and power performance on macOS.

## top - Process Monitor

`top` is the classic Unix process monitor, available on every macOS system.

### Basic Usage

```bash
# Start top
$ top

# Non-interactive mode (for scripts)
$ top -l 1

# Show only 10 processes
$ top -l 1 -n 10

# Update every 2 seconds
$ top -s 2
```

### Interactive Commands

While `top` is running:

| Key | Action |
|-----|--------|
| `q` | Quit |
| `o` | Change sort order |
| `O` | Secondary sort |
| `s` | Change update interval |
| `U` | Filter by user |
| `S` | Toggle cumulative mode |
| `p` | Toggle process ID |
| `e` | Toggle task info |
| `?` | Help |

### Sorting Options

```bash
# Sort by CPU (default)
$ top -o cpu

# Sort by memory
$ top -o mem

# Sort by process ID
$ top -o pid

# Sort by time
$ top -o time

# Sort by threads
$ top -o th

# Available sort keys
$ top -O
```

### Output Interpretation

```
Processes: 456 total, 3 running, 453 sleeping, 1234 threads
Load Avg: 1.25, 2.30, 2.15
CPU usage: 12.25% user, 5.50% sys, 82.25% idle
SharedLibs: 150M resident, 45M data, 10M linkedit
MemRegions: 78901 total, 3456M resident, 123M private, 890M shared
PhysMem: 14G used (3456M wired, 5678M compressor), 2000M unused
VM: 2345G vsize, 1234M framework vsize, 12345(0) swapins, 23456(0) swapouts
Networks: packets: 1234567/890M in, 456789/123M out
Disks: 123456/4567M read, 78901/2345M written

PID    COMMAND      %CPU TIME     #TH   #WQ  #PORT MEM    PURG   CMPRS  PGRP
12345  Safari       25.0 02:30.45 48    12   456   1234M  45M    567M   12345
```

**Header sections:**

| Section | Meaning |
|---------|---------|
| Load Avg | 1, 5, 15-minute CPU load averages |
| CPU usage | User/system/idle breakdown |
| PhysMem | Physical memory: used, wired, compressed, unused |
| VM | Virtual memory and swap activity |
| Networks | Network I/O summary |
| Disks | Disk I/O summary |

**Process columns:**

| Column | Meaning |
|--------|---------|
| %CPU | CPU utilization percentage |
| TIME | Total CPU time consumed |
| #TH | Thread count |
| #WQ | Work queue threads |
| #PORT | Mach ports |
| MEM | Memory footprint |
| PURG | Purgeable memory |
| CMPRS | Compressed memory |
| PGRP | Process group |

### Scripting with top

```bash
# Get CPU usage summary
$ top -l 1 -n 0 | grep "CPU usage"
CPU usage: 12.25% user, 5.50% sys, 82.25% idle

# Get top 5 CPU consumers
$ top -l 1 -n 5 -o cpu -stats pid,command,cpu | tail -6

# Monitor specific process
$ top -pid 12345

# CSV-friendly output
$ top -l 1 -n 10 -stats pid,cpu,mem,command | tail -11
```

## htop - Enhanced Process Monitor

`htop` provides a more user-friendly interface than `top`, with color-coded displays and mouse support.

### Installation

```bash
# Install via Homebrew
$ brew install htop
```

### Features Over top

| Feature | top | htop |
|---------|-----|------|
| Colors | No | Yes |
| Mouse support | No | Yes |
| Scroll process list | Limited | Yes |
| Tree view | No | Yes |
| Kill with key | No | Yes |
| Search | No | Yes |
| Filter | Limited | Yes |
| Setup menu | No | Yes |

### Interactive Commands

| Key | Action |
|-----|--------|
| `F1` / `h` | Help |
| `F2` / `S` | Setup menu |
| `F3` / `/` | Search |
| `F4` / `\` | Filter |
| `F5` / `t` | Tree view |
| `F6` / `>` | Sort by column |
| `F9` / `k` | Kill process |
| `F10` / `q` | Quit |
| `Space` | Tag process |
| `u` | Filter by user |
| `p` | Sort by CPU |
| `M` | Sort by memory |
| `T` | Sort by time |

### Display Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ CPU[||||||||      25.0%]  Tasks: 456, 123 thr; 3 running        │
│ CPU[|||           12.5%]  Load average: 1.25 2.30 2.15          │
│ CPU[||||||        18.0%]  Uptime: 5 days, 03:45:30              │
│ Mem[|||||||||||4.5G/16G]                                        │
│ Swp[              0K/0K]                                        │
├─────────────────────────────────────────────────────────────────┤
│   PID USER      PRI  NI  VIRT   RES   SHR S  CPU%  MEM%   TIME+ │
│ 12345 david      20   0 5432M  890M  123M S  25.0  5.6  2:30.45 │
│ 23456 david      20   0 3210M  567M   89M S  12.5  3.5  1:15.22 │
└─────────────────────────────────────────────────────────────────┘
```

### htop on macOS Notes

```bash
# Run with sudo for full information
$ sudo htop

# Some features require root:
# - Viewing all process details
# - Killing other users' processes
# - Seeing kernel threads
```

## vm_stat - Virtual Memory Statistics

`vm_stat` displays Mach virtual memory statistics.

### Basic Usage

```bash
# One-time snapshot
$ vm_stat

# Continuous monitoring (every 1 second)
$ vm_stat 1
```

### Output Interpretation

```bash
$ vm_stat
Mach Virtual Memory Statistics: (page size of 16384 bytes)
Pages free:                               45231.
Pages active:                            892341.
Pages inactive:                          234521.
Pages speculative:                        12345.
Pages throttled:                              0.
Pages wired down:                        456789.
Pages purgeable:                          23456.
"Translation faults":                1234567890.
Pages copy-on-write:                   12345678.
Pages zero filled:                    234567890.
Pages reactivated:                      1234567.
Pages purged:                            234567.
File-backed pages:                       345678.
Anonymous pages:                         567890.
Pages stored in compressor:              890123.
Pages occupied by compressor:            123456.
Decompressions:                          345678.
Compressions:                            567890.
Pageins:                                 123456.
Pageouts:                                     0.
Swapins:                                      0.
Swapouts:                                     0.
```

**Key metrics:**

| Metric | Meaning |
|--------|---------|
| Pages free | Available memory pages |
| Pages active | Recently used pages |
| Pages inactive | Not recently used, can be reclaimed |
| Pages wired down | Kernel memory, cannot be paged out |
| Pages stored in compressor | Compressed memory |
| Pageins | Pages read from disk |
| Pageouts | Pages written to disk (swap) |
| Swapins/Swapouts | Swap file activity |

### Converting to Bytes

Page size varies by system. Convert with:

```bash
# Get page size
$ pagesize
16384

# Calculate memory values
$ vm_stat | awk -v pagesize=$(pagesize) '
/Pages free/ {printf "Free: %.2f GB\n", $3 * pagesize / 1024/1024/1024}
/Pages active/ {printf "Active: %.2f GB\n", $3 * pagesize / 1024/1024/1024}
/Pages wired/ {printf "Wired: %.2f GB\n", $4 * pagesize / 1024/1024/1024}
'
```

### Monitoring Script

```bash
#!/bin/bash
# vm-monitor.sh - Monitor memory stats over time

echo "Time,Free(GB),Active(GB),Wired(GB),Compressed(GB),Swapouts"
while true; do
    stats=$(vm_stat)
    pagesize=$(pagesize)

    free=$(echo "$stats" | awk '/Pages free/ {print $3}' | tr -d '.')
    active=$(echo "$stats" | awk '/Pages active/ {print $3}' | tr -d '.')
    wired=$(echo "$stats" | awk '/Pages wired/ {print $4}' | tr -d '.')
    compressed=$(echo "$stats" | awk '/Pages stored in compressor/ {print $6}' | tr -d '.')
    swapouts=$(echo "$stats" | awk '/Swapouts/ {print $2}' | tr -d '.')

    echo "$(date +%H:%M:%S),$(echo "scale=2; $free * $pagesize / 1073741824" | bc),$(echo "scale=2; $active * $pagesize / 1073741824" | bc),$(echo "scale=2; $wired * $pagesize / 1073741824" | bc),$(echo "scale=2; $compressed * $pagesize / 1073741824" | bc),$swapouts"

    sleep 5
done
```

## memory_pressure - Memory Pressure Assessment

`memory_pressure` provides a quick memory status check.

```bash
$ memory_pressure
System-wide memory free percentage: 42%
System memory pressure level: 1

The system memory pressure level is
currently: 1 (Normal)

Mach zone information: 11345 total zones.
...
```

**Pressure levels:**

| Level | Meaning |
|-------|---------|
| 1 (Normal) | Plenty of memory available |
| 2 (Warn) | Memory becoming constrained |
| 4 (Critical) | Memory severely limited |

```bash
# Just get the pressure level
$ memory_pressure | grep "pressure level" | head -1
System memory pressure level: 1

# Monitor for pressure changes
$ while true; do
    level=$(memory_pressure 2>/dev/null | grep "System memory pressure" | awk '{print $NF}')
    echo "$(date): Pressure level $level"
    sleep 10
done
```

## iostat - I/O Statistics

`iostat` displays disk and CPU I/O statistics.

### Basic Usage

```bash
# Default output
$ iostat
              disk0
    KB/t  tps  MB/s
   24.00   45  1.05

# Update every 2 seconds, 5 times
$ iostat 2 5

# Show CPU statistics too
$ iostat -C

# Extended statistics
$ iostat -d -K -w 2
```

### Output Interpretation

```bash
$ iostat -C -w 2
          cpu     load average        disk0
    us sy id 1m    5m   15m   KB/t  tps  MB/s
    12  5 83 1.25  2.30 2.15  24.0   45  1.05
```

| Column | Meaning |
|--------|---------|
| us | User CPU % |
| sy | System CPU % |
| id | Idle CPU % |
| KB/t | Kilobytes per transfer |
| tps | Transfers per second |
| MB/s | Megabytes per second |

### Per-Disk Statistics

```bash
# List all disks
$ iostat -d
              disk0               disk1
    KB/t  tps  MB/s      KB/t  tps  MB/s
   24.00   45  1.05     32.00   12  0.38

# Specific disk
$ iostat disk0
```

## fs_usage - File System Usage

`fs_usage` traces file system activity in real-time.

### Requirements

- Requires root privileges
- Terminal needs Full Disk Access for complete information

### Basic Usage

```bash
# All filesystem activity
$ sudo fs_usage

# Filter by process name
$ sudo fs_usage -w Safari

# Filter by process ID
$ sudo fs_usage -p 12345

# Only show file activity (not network)
$ sudo fs_usage -f filesys

# Only show network activity
$ sudo fs_usage -f network

# Only show disk I/O
$ sudo fs_usage -f diskio
```

### Output Interpretation

```
14:30:45.123  stat64        /usr/lib/libc.dylib      0.000023  Safari
14:30:45.124  open          /Users/david/file.txt    0.000045  Safari
14:30:45.125  read          F=5                      0.000012  Safari
14:30:45.126  close         F=5                      0.000003  Safari
```

| Column | Meaning |
|--------|---------|
| Timestamp | When the call occurred |
| System call | Type of operation |
| Path/Details | File path or file descriptor |
| Duration | Time for the operation |
| Process | Process name |

### Filtering Examples

```bash
# Watch a specific directory
$ sudo fs_usage -w -f filesys | grep "/Users/david/project"

# Find what's writing to disk
$ sudo fs_usage -f diskio -w | grep "WrData"

# Watch for file deletions
$ sudo fs_usage -f filesys | grep -E "unlink|rmdir"

# Monitor Time Machine
$ sudo fs_usage -w backupd
```

### Performance Analysis Script

```bash
#!/bin/bash
# io-summary.sh - Summarize I/O activity for a process

if [[ -z "$1" ]]; then
    echo "Usage: $0 <process-name> <duration-seconds>"
    exit 1
fi

PROCESS=$1
DURATION=${2:-10}

echo "Monitoring $PROCESS for $DURATION seconds..."
sudo timeout $DURATION fs_usage -w -f filesys "$PROCESS" 2>/dev/null | \
    awk '{print $2}' | sort | uniq -c | sort -rn | head -20
```

## powermetrics - Power and Performance Metrics

`powermetrics` provides detailed power consumption and performance data, especially useful on laptops.

### Requirements

- Requires root privileges
- More detailed on Apple Silicon

### Basic Usage

```bash
# All metrics
$ sudo powermetrics

# Specific samplers
$ sudo powermetrics --samplers cpu_power,gpu_power,battery

# Sample once
$ sudo powermetrics -n 1

# Sample every 5 seconds
$ sudo powermetrics -i 5000
```

### Available Samplers

```bash
# List available samplers
$ sudo powermetrics --samplers help

# Common samplers:
# cpu_power      - CPU power consumption
# gpu_power      - GPU power consumption
# battery        - Battery stats
# thermal        - Thermal state
# tasks          - Per-process power
# network        - Network power
# disk           - Disk power
```

### CPU Power Output

```bash
$ sudo powermetrics --samplers cpu_power -n 1
Machine model: MacBookPro18,3
OS version: 14.0

*** Processor Info ***
CPU: Apple M1 Pro
CPU Complex Energy: 145 mJ

CPU Power: 2850 mW
E-Cluster Power: 450 mW
P-Cluster Power: 2400 mW

E-Cluster HW active frequency: 1200 MHz
E-Cluster HW active residency:  25.00%

P-Cluster HW active frequency: 3200 MHz
P-Cluster HW active residency:  45.00%
```

### Battery Information

```bash
$ sudo powermetrics --samplers battery -n 1
*** Battery Info ***
Current Capacity: 85%
Design Capacity: 5103 mAh
Cycle Count: 234
Temperature: 32.5 C
Voltage: 12.45 V
Amperage: -1234 mA
Instant power: -15.37 W
```

### Per-Process Power

```bash
$ sudo powermetrics --samplers tasks -n 1

*** Running tasks ***
Name                    PID     CPU_Time(ns)  CPU_Pct  Idle_Wake
Safari                  12345   234567890     12.5     234
Chrome                  23456   123456789     8.3      567
WindowServer            234     98765432      5.2      123
```

### Thermal Information

```bash
$ sudo powermetrics --samplers thermal -n 1
*** Thermal State ***
System Thermal Level: 0 (nominal)
CPU Thermal Level: 0 (nominal)
GPU Thermal Level: 0 (nominal)
```

### Monitoring Script

```bash
#!/bin/bash
# power-monitor.sh - Track power usage over time

echo "Timestamp,CPU_Power(W),GPU_Power(W),Battery(%),Amperage(mA)"
while true; do
    output=$(sudo powermetrics --samplers cpu_power,gpu_power,battery -n 1 2>/dev/null)

    cpu_power=$(echo "$output" | grep "CPU Power:" | awk '{print $3}')
    gpu_power=$(echo "$output" | grep "GPU Power:" | awk '{print $3}')
    battery=$(echo "$output" | grep "Current Capacity:" | awk '{print $3}' | tr -d '%')
    amperage=$(echo "$output" | grep "Amperage:" | awk '{print $2}')

    echo "$(date +%H:%M:%S),$cpu_power,$gpu_power,$battery,$amperage"
    sleep 30
done
```

## sample - Process Sampling

`sample` captures a time-profile of a process, showing where it spends CPU time.

### Basic Usage

```bash
# Sample for 5 seconds
$ sample Safari 5

# Save to file
$ sample Safari 5 -file /tmp/safari-sample.txt

# Sample by PID
$ sample 12345 5
```

### Output Interpretation

```
Sampling process 12345 for 5 seconds with 1 millisecond of run time between samples
Sampling completed, processing symbols...
Analysis of sampling Safari (pid 12345) every 1 millisecond
Process: Safari [12345]
Path:    /Applications/Safari.app/Contents/MacOS/Safari

Call graph:
    2500 Thread_12345678   DispatchQueue_1: com.apple.main-thread  (serial)
      2500 start  (in dyld) + 1234  [0x12345678]
        2500 main  (in Safari) + 567  [0x23456789]
          1500 -[BrowserController loadURL:]  (in Safari) + 234
            1200 WebCore::FrameLoader::load()  (in WebCore) + 456
          1000 -[NSApplication run]  (in AppKit) + 789
```

**Reading the call graph:**
- Numbers indicate sample counts (more = more time spent)
- Indentation shows call hierarchy
- Most time-consuming functions appear with highest counts

### Finding Performance Issues

```bash
# Sample a slow process
$ sample SlowApp 10 -file /tmp/slowapp.txt

# Find heavy functions
$ grep -E "^[[:space:]]*[0-9]{3,}" /tmp/slowapp.txt | head -20

# Look for lock contention
$ grep -i "pthread_mutex\|semaphore\|lock" /tmp/slowapp.txt
```

## spindump - System-Wide Sampling

`spindump` captures system-wide process sampling, especially useful for hangs.

```bash
# Basic spindump
$ sudo spindump

# Specific process
$ sudo spindump -pid 12345

# Duration and output
$ sudo spindump 5 1 -file /tmp/spindump.txt
```

## Additional Useful Tools

### iotop - I/O by Process

Not installed by default, but available via Homebrew:

```bash
$ brew install iotop

# Monitor I/O (requires root)
$ sudo iotop
```

### nettop - Network by Process

Built into macOS:

```bash
# Interactive mode
$ nettop

# Process view
$ nettop -P

# One sample, machine-readable
$ nettop -P -L 1
```

### sysctl - System Parameters

Query system information:

```bash
# CPU info
$ sysctl -n hw.ncpu
$ sysctl -n machdep.cpu.brand_string

# Memory info
$ sysctl -n hw.memsize

# All hardware info
$ sysctl hw

# Kernel stats
$ sysctl kern | head -20
```

## Tool Comparison Summary

| Tool | Purpose | Root Required | Continuous |
|------|---------|---------------|------------|
| top | Process monitor | No | Yes |
| htop | Enhanced process monitor | Optional | Yes |
| vm_stat | Memory statistics | No | Yes |
| memory_pressure | Memory pressure | No | No |
| iostat | I/O statistics | No | Yes |
| fs_usage | File system trace | Yes | Yes |
| powermetrics | Power analysis | Yes | Yes |
| sample | Process profiling | No | One-shot |
| spindump | System sampling | Yes | One-shot |
| nettop | Network by process | No | Yes |

## Quick Reference Commands

```bash
# CPU hogs
$ top -l 1 -o cpu -n 5 | tail -6

# Memory hogs
$ top -l 1 -o mem -n 5 | tail -6

# Memory pressure
$ memory_pressure | head -3

# Disk I/O rate
$ iostat -d -w 1 -c 3

# What's writing to disk
$ sudo fs_usage -f diskio -w | head -50

# Power consumption
$ sudo powermetrics --samplers cpu_power -n 1 | grep "CPU Power"

# Sample slow process
$ sample ProcessName 10 -file /tmp/sample.txt

# System-wide snapshot
$ sudo spindump 5 1 -file /tmp/spindump.txt
```

These command-line tools form the foundation of performance analysis on macOS, enabling detailed investigation of system behavior beyond what GUI tools provide.
