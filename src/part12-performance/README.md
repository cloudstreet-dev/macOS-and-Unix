# Performance and Optimization

Understanding and optimizing macOS performance requires knowledge of both traditional Unix performance concepts and Apple-specific technologies. macOS combines the robust process and memory management of BSD with Apple's innovations in power efficiency, storage optimization, and graphics performance. This part covers the tools and techniques for monitoring, analyzing, and improving system performance.

## The Performance Landscape

macOS performance optimization spans multiple layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Application Layer                             │
│          (CPU usage, memory footprint, I/O patterns)            │
├─────────────────────────────────────────────────────────────────┤
│                    Framework Layer                               │
│        (Grand Central Dispatch, Metal, Core Animation)          │
├─────────────────────────────────────────────────────────────────┤
│                    System Services                               │
│         (WindowServer, launchd, mds, kernel_task)               │
├─────────────────────────────────────────────────────────────────┤
│                    Kernel / XNU                                  │
│      (Memory management, scheduler, I/O subsystem)              │
├─────────────────────────────────────────────────────────────────┤
│                    Hardware                                      │
│     (CPU cores, unified memory, SSD, Neural Engine)             │
└─────────────────────────────────────────────────────────────────┘
```

Each layer has its own characteristics and optimization strategies.

## Key Performance Metrics

### CPU Performance

macOS uses a sophisticated scheduler that manages:

| Concept | Description |
|---------|-------------|
| Efficiency Cores (E-cores) | Lower power, background tasks (Apple Silicon) |
| Performance Cores (P-cores) | Maximum throughput (Apple Silicon) |
| Quality of Service (QoS) | Thread priority classification |
| CPU Throttling | Thermal and power management |

```bash
# Quick CPU overview
$ sysctl -n hw.ncpu
8

# Apple Silicon core types
$ sysctl hw.perflevel0.physicalcpu hw.perflevel1.physicalcpu
hw.perflevel0.physicalcpu: 4   # Performance cores
hw.perflevel1.physicalcpu: 4   # Efficiency cores

# Current CPU usage
$ top -l 1 -n 0 | grep "CPU usage"
CPU usage: 5.26% user, 3.50% sys, 91.23% idle
```

### Memory Performance

macOS memory management includes unique features:

| Feature | Purpose |
|---------|---------|
| Unified Memory | Shared CPU/GPU memory (Apple Silicon) |
| Memory Compression | Compress inactive pages instead of swapping |
| App Nap | Reduce memory/CPU for background apps |
| Memory Pressure | System-wide memory demand indicator |

```bash
# Memory overview
$ vm_stat
Pages free:                               45231.
Pages active:                            892341.
Pages inactive:                          234521.
Pages speculative:                        12345.
Pages wired down:                        456789.
Pages compressed:                        567890.

# Pressure level
$ memory_pressure
System-wide memory free percentage: 42%
```

### Storage Performance

APFS and modern SSDs require different optimization approaches:

| Factor | Impact |
|--------|--------|
| SSD Trim | Maintains write performance |
| APFS Snapshots | Can consume space |
| Spotlight Indexing | Background I/O during indexing |
| Time Machine | Periodic backup I/O |

```bash
# Disk I/O statistics
$ iostat -d 1 3
              disk0
    KB/t  tps  MB/s
   24.00   45  1.05

# APFS container space
$ diskutil apfs list
```

### Power Performance

Especially important on laptops:

| Aspect | Consideration |
|--------|---------------|
| CPU Frequency | Dynamic scaling based on demand |
| Display Brightness | Major power consumer |
| Discrete GPU | Significant power draw when active |
| Background Activity | Apps preventing sleep |

```bash
# Power state
$ pmset -g
System-wide power settings:
 SleepDisabled          0
Currently in use:
 hibernatemode        3
 powernap             1
 sleep                1
```

## Quick System Health Check

A fast assessment of overall system performance:

```bash
#!/bin/bash
# Quick system health check

echo "=== CPU ==="
top -l 1 -n 0 | grep "CPU usage"

echo -e "\n=== Memory ==="
memory_pressure 2>/dev/null || vm_stat | head -10

echo -e "\n=== Disk ==="
df -h / | tail -1

echo -e "\n=== Load Average ==="
uptime

echo -e "\n=== Top Processes by CPU ==="
ps aux | sort -nrk 3 | head -6

echo -e "\n=== Top Processes by Memory ==="
ps aux | sort -nrk 4 | head -6
```

## GUI vs CLI Performance Tools

macOS provides both graphical and command-line performance tools:

| Task | GUI Tool | CLI Tool |
|------|----------|----------|
| Process monitoring | Activity Monitor | `top`, `htop`, `ps` |
| Memory analysis | Activity Monitor | `vm_stat`, `memory_pressure` |
| Disk I/O | Activity Monitor | `iostat`, `fs_usage` |
| Network | Activity Monitor | `nettop`, `netstat` |
| CPU profiling | Instruments | `sample`, `spindump` |
| Energy | Activity Monitor | `powermetrics` |
| System trace | Instruments | `fs_usage`, `sc_usage` |

## Performance on Apple Silicon vs Intel

Apple Silicon Macs have fundamentally different performance characteristics:

| Aspect | Intel Mac | Apple Silicon Mac |
|--------|-----------|-------------------|
| Memory | Dedicated RAM | Unified Memory (shared CPU/GPU) |
| Cores | Symmetric | Asymmetric (P-cores + E-cores) |
| Power States | Intel SpeedStep | Apple custom (more granular) |
| Thermal Design | Often throttles under load | More consistent performance |
| GPU | Discrete or integrated | Integrated, shares memory |
| Rosetta 2 | N/A | ~80-90% native performance |

```bash
# Check architecture
$ uname -m
arm64  # Apple Silicon
x86_64 # Intel

# Check if running under Rosetta
$ sysctl sysctl.proc_translated
sysctl.proc_translated: 0  # Native
sysctl.proc_translated: 1  # Rosetta
```

## What You'll Learn in This Part

**[Activity Monitor: Beyond the GUI](./activity-monitor.md)** explores Activity Monitor's tabs in depth, hidden columns, what each metric means, and how to export data for analysis.

**[Command-Line Performance Tools](./cli-performance-tools.md)** covers essential CLI tools: `top`, `htop`, `vm_stat`, `iostat`, `fs_usage`, `powermetrics`, and `sample`, with practical examples and interpretation guides.

**[Intel vs Apple Silicon Considerations](./architecture-considerations.md)** explains performance differences between architectures, Rosetta 2 overhead, universal binaries, and optimizing for each platform.

**[Memory Management Deep Dive](./memory-management.md)** examines how macOS manages memory, including compression, swap, memory pressure, diagnosing leaks, and optimizing memory usage.

**[Disk I/O Optimization](./disk-io.md)** covers measuring and improving storage performance, APFS optimizations, SSD health, and identifying I/O bottlenecks.

**[Power Management and Battery](./power-management.md)** explains `pmset`, `caffeinate`, power assertions, App Nap, and strategies for maximizing battery life.

**[Troubleshooting Performance Issues](./troubleshooting.md)** provides a systematic approach to diagnosing slowdowns, including common causes, diagnostic workflows, and remediation strategies.

## Common Performance Tasks

### Find What's Using CPU

```bash
# Interactive view
$ top -o cpu

# Snapshot of top CPU consumers
$ ps aux | sort -nrk 3 | head -10

# Sample a process for analysis
$ sudo sample PID 5 -file /tmp/sample.txt
```

### Find What's Using Memory

```bash
# Sort by memory in top
$ top -o mem

# Detailed memory stats
$ vm_stat

# Memory pressure
$ memory_pressure
```

### Find What's Using Disk

```bash
# Real-time I/O monitoring (requires Full Disk Access)
$ sudo fs_usage -f filesys

# I/O statistics
$ iostat -d 2
```

### Check System Responsiveness

```bash
# Load average
$ uptime
10:30  up 5 days,  3:45, 3 users, load averages: 1.25 2.30 2.15

# Interpretation:
# 1-minute: 1.25 (recent load)
# 5-minute: 2.30 (short-term trend)
# 15-minute: 2.15 (longer-term trend)

# Compare to CPU count for utilization
$ sysctl -n hw.ncpu
8
# Load of 8.0 = 100% utilization on 8 cores
```

### Monitor Power Usage

```bash
# Detailed power metrics (requires root)
$ sudo powermetrics --samplers cpu_power -n 1

# Battery status
$ pmset -g batt
Now drawing from 'Battery Power'
 -InternalBattery-0 (id=1234567) 85%; discharging; 4:30 remaining
```

## Performance Best Practices

### General Guidelines

1. **Monitor before optimizing**: Establish baseline measurements
2. **Focus on bottlenecks**: Optimize the limiting factor first
3. **Consider power impact**: Performance gains may cost battery life
4. **Test on target hardware**: Performance varies by Mac model
5. **Use native builds**: Avoid Rosetta 2 overhead when possible

### Quick Wins

```bash
# Free up disk space (clears caches)
$ sudo purge

# Rebuild Spotlight index if mds is consuming resources
$ sudo mdutil -E /

# Clear DNS cache if network is slow
$ sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# Disable Spotlight for a volume (development drives)
$ sudo mdutil -i off /Volumes/ExternalDrive

# Check for runaway processes
$ top -l 1 -o cpu -n 5
```

### When to Investigate

Investigate performance when you observe:

- **High CPU**: Fan running, system warm, UI lag
- **High Memory**: Memory pressure warnings, swap usage
- **High Disk I/O**: Spinning cursor, slow app launches
- **High Energy**: Battery draining faster than expected
- **High Network**: Unexpected data usage, slow downloads

The following chapters provide detailed coverage of each performance domain, with practical diagnostic commands and optimization strategies.
