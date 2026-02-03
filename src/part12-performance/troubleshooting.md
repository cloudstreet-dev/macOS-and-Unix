# Troubleshooting Performance Issues

When macOS feels slow, a systematic diagnostic approach helps identify the root cause. This chapter provides workflows for diagnosing CPU, memory, disk, network, and graphics performance issues, along with common problems and their solutions.

## Diagnostic Framework

### The Performance Triangle

Performance issues typically fall into three categories:

```
                    CPU
                   /   \
                  /     \
                 /       \
                /  System \
               /  Resources \
              /             \
           Memory --------- I/O
              (RAM, Swap)    (Disk, Network)
```

When investigating performance:
1. Check all three resource types
2. Identify which is the bottleneck
3. Investigate the specific resource
4. Address the root cause

### Quick Health Check

Start every investigation with this script:

```bash
#!/bin/bash
# quick-check.sh - Rapid system health assessment

echo "=== System Health Check - $(date) ==="
echo ""

# 1. Load Average
echo "--- Load Average ---"
uptime
NCPU=$(sysctl -n hw.ncpu)
echo "CPU count: $NCPU"
echo ""

# 2. CPU Usage
echo "--- CPU Usage ---"
top -l 1 -n 0 | grep "CPU usage"
echo ""

# 3. Memory Pressure
echo "--- Memory Pressure ---"
memory_pressure 2>/dev/null | head -3
echo ""

# 4. Disk Space
echo "--- Disk Space ---"
df -h / | tail -1
echo ""

# 5. Top Processes
echo "--- Top 5 by CPU ---"
ps aux | sort -nrk 3 | head -6
echo ""

echo "--- Top 5 by Memory ---"
ps aux | sort -nrk 4 | head -6
```

## CPU Troubleshooting

### Symptoms

- Fans running constantly
- System feels sluggish
- Beach ball cursor
- High CPU temperature
- Short battery life

### Diagnostic Steps

```bash
# Step 1: Check overall CPU usage
$ top -l 1 -n 0 | grep "CPU usage"
CPU usage: 85.0% user, 10.0% sys, 5.0% idle

# Step 2: Find CPU-heavy processes
$ top -l 1 -o cpu -n 10 | tail -11

# Step 3: Check if process is stuck (sample it)
$ sample PID 5 -file /tmp/sample.txt
$ head -50 /tmp/sample.txt

# Step 4: Check for runaway threads
$ ps -M -p PID  # Thread list for process

# Step 5: Verify process is responsive
$ kill -0 PID && echo "Process is alive" || echo "Process may be zombie"
```

### Common CPU Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| kernel_task high CPU | Thermal throttling | Cool the Mac, check vents |
| mds_stores high CPU | Spotlight indexing | Wait, or exclude folders |
| WindowServer high CPU | Graphics issue | Reduce transparency, check GPU |
| nsurlsessiond | Background downloads | Check for updates |
| softwareupdated | System updates | Let it complete |
| Single app 100%+ CPU | App issue | Restart app, check for updates |

### Investigating kernel_task

`kernel_task` artificially uses CPU to throttle when the system is too hot:

```bash
# Check thermal state
$ sudo powermetrics --samplers thermal -n 1

# If thermal level > 0, system is throttling
# Solutions:
# - Move to cooler environment
# - Check for blocked vents
# - Use a cooling pad
# - Reduce workload
```

### Investigating Spotlight

```bash
# Check if Spotlight is indexing
$ sudo mdutil -s /
/System/Volumes/Data:
    Indexing enabled.
    Scan in progress.  <-- Active indexing

# Check mds activity
$ sudo fs_usage mds_stores 2>/dev/null | head -20

# If problematic, disable temporarily
$ sudo mdutil -i off /

# Or exclude specific folders
$ sudo mdutil -E /Volumes/ExternalDrive

# Rebuild index (nuclear option)
$ sudo mdutil -E /
```

## Memory Troubleshooting

### Symptoms

- "Your system has run out of application memory"
- Extreme slowness
- Apps crashing
- Beach ball when switching apps
- High swap usage

### Diagnostic Steps

```bash
# Step 1: Check memory pressure
$ memory_pressure
System-wide memory free percentage: 12%
System memory pressure level: 2  # Warning level

# Step 2: Check detailed memory stats
$ vm_stat

# Step 3: Check swap usage
$ sysctl vm.swapusage
vm.swapusage: total = 2048.00M  used = 1500.00M  free = 548.00M

# Step 4: Find memory hogs
$ ps aux --sort=-%mem | head -10

# Step 5: Check specific process memory
$ vmmap --summary $(pgrep Safari) | head -20
```

### Memory Pressure Interpretation

```bash
# Run memory_pressure and interpret
$ memory_pressure
System-wide memory free percentage: 45%
System memory pressure level: 1

# Level meanings:
# 1 (Normal): >25% free, no action needed
# 2 (Warning): 5-25% free, close unused apps
# 4 (Critical): <5% free, urgent action needed
```

### Finding Memory Leaks

```bash
# Monitor memory growth over time
$ while true; do
    RSS=$(ps -o rss= -p PID)
    echo "$(date +%H:%M:%S) RSS: $((RSS/1024)) MB"
    sleep 60
done

# Growing steadily = probable memory leak

# Use leaks tool (if debug symbols available)
$ leaks PID

# Use Instruments for detailed analysis
# (Xcode > Open Developer Tool > Instruments > Leaks)
```

### Addressing Memory Pressure

```bash
# Immediate relief:
# 1. Close unused applications
# 2. Close browser tabs

# Check what's using memory
$ ps aux --sort=-%mem | head -10

# Restart memory-heavy apps
$ killall Safari  # Safari will reopen

# Clear file caches (helps a little)
$ sudo purge

# If chronic issue, consider:
# - More RAM (if upgradeable)
# - Reducing number of concurrent apps
# - Using lighter alternatives (Safari vs Chrome)
```

## Disk I/O Troubleshooting

### Symptoms

- Slow app launches
- Spinning beach ball
- Sluggish file operations
- System hangs during saves

### Diagnostic Steps

```bash
# Step 1: Check disk activity
$ iostat -d -w 2
              disk0
    KB/t  tps  MB/s
   48.00  456  21.3  <-- High activity

# Step 2: Find I/O sources
$ sudo fs_usage -f diskio 2>/dev/null | head -50

# Step 3: Check disk space
$ df -h /
Filesystem     Size   Used  Avail Capacity
/dev/disk3s1  460Gi  450Gi   10Gi    98%  <-- Very full!

# Step 4: Check for Spotlight indexing
$ sudo fs_usage mds_stores 2>/dev/null

# Step 5: Verify disk health
$ diskutil verifyVolume /
```

### Common Disk Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Disk nearly full | Insufficient space | Free up space |
| Constant disk activity | Spotlight/Time Machine | Wait or exclude |
| Slow read/write | Disk failing | Run diagnostics |
| Beach ball on save | Disk I/O blocked | Check for hangs |

### Freeing Disk Space

```bash
# Check what's using space
$ sudo du -sh /* 2>/dev/null | sort -h | tail -20

# Find large files
$ find ~ -type f -size +500M -exec ls -lh {} \; 2>/dev/null

# Clear system caches
$ sudo rm -rf /Library/Caches/*
$ rm -rf ~/Library/Caches/*

# Empty Trash
$ rm -rf ~/.Trash/*

# Remove old iOS backups
$ ls -la ~/Library/Application\ Support/MobileSync/Backup/

# Clear Xcode derived data
$ rm -rf ~/Library/Developer/Xcode/DerivedData/*

# Remove local Time Machine snapshots
$ tmutil listlocalsnapshots /
$ sudo tmutil deletelocalsnapshots YYYY-MM-DD-HHMMSS
```

### Disk Health Check

```bash
# Verify filesystem
$ diskutil verifyVolume /

# First Aid (repairs issues)
$ diskutil repairVolume /

# If issues persist, boot to Recovery Mode (Cmd+R on Intel, hold power on AS)
# and run Disk Utility First Aid on the volume
```

## Network Troubleshooting

### Symptoms

- Slow downloads
- Web pages load slowly
- Network requests hang
- High latency

### Diagnostic Steps

```bash
# Step 1: Check network connectivity
$ ping -c 5 8.8.8.8

# Step 2: Check DNS
$ dig google.com

# Step 3: Check bandwidth
$ networkQuality
==== SUMMARY ====
Uplink: 45.234 Mbps
Downlink: 120.456 Mbps
Responsiveness: 234 RPM

# Step 4: Find network-heavy processes
$ nettop -P -L 1

# Step 5: Check for connection issues
$ netstat -an | grep ESTABLISHED | wc -l
```

### Common Network Issues

```bash
# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder

# Restart network services
$ sudo ifconfig en0 down
$ sudo ifconfig en0 up

# Check Wi-Fi issues
$ /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I

# Renew DHCP lease
$ sudo ipconfig set en0 DHCP
```

## Graphics/UI Troubleshooting

### Symptoms

- Choppy scrolling
- Laggy animations
- Screen tearing
- High WindowServer CPU

### Diagnostic Steps

```bash
# Check WindowServer CPU
$ top -l 1 | grep WindowServer

# Check GPU usage
$ sudo powermetrics --samplers gpu_power -n 1

# On Intel Macs with discrete GPU, check which is active
$ system_profiler SPDisplaysDataType | grep -E "Chipset|VRAM"
```

### Graphics Fixes

```bash
# Reduce visual effects
# System Preferences > Accessibility > Display
# - Reduce motion
# - Reduce transparency

# Via defaults
$ defaults write com.apple.universalaccess reduceMotion -bool true
$ defaults write com.apple.universalaccess reduceTransparency -bool true

# Restart WindowServer (logs you out!)
$ sudo killall -HUP WindowServer
```

## Application-Specific Issues

### App Won't Launch

```bash
# Check app signature
$ codesign -v /Applications/Problem.app

# Check quarantine
$ xattr /Applications/Problem.app

# Check console for errors
$ log show --predicate 'process == "Problem"' --last 5m

# Try launching from Terminal
$ /Applications/Problem.app/Contents/MacOS/Problem
```

### App Crashes Repeatedly

```bash
# Find crash logs
$ ls -lt ~/Library/Logs/DiagnosticReports/*.crash | head -5

# Read recent crash
$ cat ~/Library/Logs/DiagnosticReports/Problem*.crash | head -100

# Clear app preferences
$ rm ~/Library/Preferences/com.example.problem.plist

# Clear app caches
$ rm -rf ~/Library/Caches/com.example.problem
```

### App Using Too Many Resources

```bash
# Identify resource usage
$ ps aux | grep "Problem"
$ vmmap --summary $(pgrep Problem)

# Sample for analysis
$ sample Problem 10 -file /tmp/problem-sample.txt

# Consider alternatives or report to developer
```

## Systematic Troubleshooting Script

```bash
#!/bin/bash
# full-diagnostic.sh - Comprehensive performance diagnostic

OUTPUT_DIR="/tmp/mac-diagnostic-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "Running full diagnostic. Output: $OUTPUT_DIR"
echo ""

# System info
echo "Gathering system info..."
sw_vers > "$OUTPUT_DIR/system-version.txt"
sysctl hw > "$OUTPUT_DIR/hardware.txt"
uname -a >> "$OUTPUT_DIR/system-version.txt"

# CPU
echo "Gathering CPU info..."
top -l 3 -n 10 > "$OUTPUT_DIR/top.txt"
ps aux --sort=-%cpu | head -50 > "$OUTPUT_DIR/top-cpu-processes.txt"

# Memory
echo "Gathering memory info..."
vm_stat > "$OUTPUT_DIR/vm_stat.txt"
memory_pressure > "$OUTPUT_DIR/memory_pressure.txt" 2>&1
sysctl vm.swapusage >> "$OUTPUT_DIR/vm_stat.txt"
ps aux --sort=-%mem | head -50 > "$OUTPUT_DIR/top-mem-processes.txt"

# Disk
echo "Gathering disk info..."
df -h > "$OUTPUT_DIR/disk-space.txt"
diskutil list >> "$OUTPUT_DIR/disk-space.txt"
iostat -d -c 5 > "$OUTPUT_DIR/iostat.txt"

# Network
echo "Gathering network info..."
netstat -an | head -100 > "$OUTPUT_DIR/netstat.txt"
ifconfig > "$OUTPUT_DIR/ifconfig.txt"

# Power
echo "Gathering power info..."
pmset -g > "$OUTPUT_DIR/pmset.txt"
pmset -g assertions >> "$OUTPUT_DIR/pmset.txt"
pmset -g batt >> "$OUTPUT_DIR/pmset.txt"

# Logs
echo "Gathering recent logs..."
log show --predicate 'eventType == logEvent' --last 30m 2>/dev/null | head -1000 > "$OUTPUT_DIR/recent-logs.txt"

# Summary
echo "Creating summary..."
cat > "$OUTPUT_DIR/summary.txt" << EOF
=== Performance Diagnostic Summary ===
Date: $(date)
macOS: $(sw_vers -productVersion)
Model: $(sysctl -n hw.model)
CPU: $(sysctl -n machdep.cpu.brand_string 2>/dev/null || echo "Apple Silicon")
RAM: $(sysctl -n hw.memsize | awk '{print $1/1024/1024/1024 " GB"}')

--- Current State ---
Load Average: $(uptime | awk -F'load averages:' '{print $2}')
Memory Pressure: $(memory_pressure 2>/dev/null | grep "free percentage" | awk '{print $5}')
Disk Usage: $(df -h / | tail -1 | awk '{print $5}')
Battery: $(pmset -g batt | grep -o '[0-9]*%' | head -1)

--- Top CPU Consumers ---
$(ps aux --sort=-%cpu | awk 'NR<=6 {print $3 "% " $11}')

--- Top Memory Consumers ---
$(ps aux --sort=-%mem | awk 'NR<=6 {print $4 "% " $11}')

EOF

echo ""
echo "Diagnostic complete. Output saved to: $OUTPUT_DIR"
echo "Review summary: cat $OUTPUT_DIR/summary.txt"
```

## Common Problems Quick Reference

### System Feels Generally Slow

```bash
# 1. Check load average vs CPU count
$ uptime
$ sysctl -n hw.ncpu

# If load > 2x CPU count, system is overloaded

# 2. Check memory pressure
$ memory_pressure | head -3

# 3. Check disk space (needs >10% free)
$ df -h /

# 4. Reboot if uptime is very long
$ uptime
$ sudo reboot  # Sometimes helps
```

### Slow After macOS Update

```bash
# Spotlight reindexing - wait or:
$ sudo mdutil -s /

# Permissions issues
$ diskutil resetUserPermissions / $(id -u)

# Clear font caches
$ sudo atsutil databases -remove

# Rebuild Launch Services database
$ /System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -kill -r -domain local -domain system -domain user
```

### Slow After Sleep/Wake

```bash
# Check power assertions
$ pmset -g assertions

# Check for hung processes
$ top -o cpu

# Restart problem services
$ sudo killall -9 WindowServer  # Logs you out
```

## When to Seek Further Help

- Hardware diagnostics show errors (Apple Diagnostics: hold D on boot)
- Disk Utility shows uncorrectable errors
- Kernel panics occur regularly
- Issues persist after clean macOS install
- SMC reset doesn't help (Intel Macs)

```bash
# Run Apple Diagnostics
# Intel: Restart, hold D
# Apple Silicon: Restart, hold power button, select Options > Diagnostics

# Generate system report for Apple Support
$ sudo sysdiagnose
# Creates archive in /var/tmp/
```

## Summary

Troubleshooting workflow:

1. **Identify symptoms** - What feels slow?
2. **Check resources** - CPU, memory, disk, network
3. **Find the bottleneck** - Which resource is constrained?
4. **Identify the cause** - Which process or condition?
5. **Apply the fix** - Address root cause
6. **Verify resolution** - Confirm improvement

Essential diagnostic commands:

| Resource | Quick Check | Detailed |
|----------|-------------|----------|
| CPU | `top -l 1 -n 0` | `sample PID` |
| Memory | `memory_pressure` | `vmmap PID` |
| Disk | `iostat -d` | `sudo fs_usage` |
| Network | `nettop -P` | `netstat -an` |
| Power | `pmset -g batt` | `sudo powermetrics` |
| Overall | `uptime` | `sysdiagnose` |

Systematic diagnosis leads to effective solutions, while random fixes often mask the real problem.
