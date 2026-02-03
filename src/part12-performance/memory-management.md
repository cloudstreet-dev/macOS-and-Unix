# Memory Management Deep Dive

macOS employs sophisticated memory management that goes beyond traditional Unix approaches. Understanding how macOS handles memory, compression, swap, and memory pressure is essential for diagnosing performance issues and optimizing applications.

## Memory Architecture Overview

### Memory Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                    Memory Hierarchy                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  L1 Cache (per core)     ~192 KB     ~1 ns access              │
│           │                                                      │
│           ▼                                                      │
│  L2 Cache (per core)     ~3-12 MB    ~3-4 ns access            │
│           │                                                      │
│           ▼                                                      │
│  System Memory (RAM)     8-128 GB    ~100 ns access            │
│           │                                                      │
│           ▼                                                      │
│  Compressed Memory       Variable    ~500 ns access            │
│           │                                                      │
│           ▼                                                      │
│  Swap (SSD)              Variable    ~50-100 µs access          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Memory Categories in macOS

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
...
```

| Category | Description | Can Be Reclaimed |
|----------|-------------|------------------|
| Free | Immediately available | N/A |
| Active | Recently used by apps | Yes, if needed |
| Inactive | Not recently used | Yes, readily |
| Speculative | Preemptively cached files | Yes, readily |
| Wired | Kernel, drivers, system | No |
| Purgeable | App-marked disposable | Yes, readily |
| Compressed | Inactive, compressed in RAM | Partially |

## Memory Compression

macOS compresses inactive memory pages rather than immediately swapping to disk.

### How Compression Works

```
┌─────────────────────────────────────────────────────────────────┐
│              Memory Compression Process                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Memory Pressure Increases                                       │
│           │                                                      │
│           ▼                                                      │
│  Identify Inactive Pages                                         │
│           │                                                      │
│           ▼                                                      │
│  Compress Pages (WKdm algorithm)                                │
│  ┌─────────────────────────────────────┐                        │
│  │  Original: 16 KB page               │                        │
│  │  Compressed: ~4-6 KB (typical)      │                        │
│  │  Ratio: 2.5-4x compression          │                        │
│  └─────────────────────────────────────┘                        │
│           │                                                      │
│           ▼                                                      │
│  Store in Compressor (VM region)                                │
│           │                                                      │
│           ▼                                                      │
│  Only swap if compressor full                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Viewing Compression Statistics

```bash
$ vm_stat | grep -E "compressor|Compressions|Decompressions"
Pages stored in compressor:              567890.
Pages occupied by compressor:            123456.
Decompressions:                          345678.
Compressions:                            567890.

# Calculate compression ratio
$ vm_stat | awk '
/Pages stored in compressor/ {stored=$6}
/Pages occupied by compressor/ {occupied=$6}
END {
    if (occupied > 0) {
        ratio = stored / occupied
        printf "Compression ratio: %.2f:1\n", ratio
    }
}'
```

### Compressor Memory Script

```bash
#!/bin/bash
# compressor-stats.sh - Show memory compressor statistics

PAGE_SIZE=$(pagesize)

stats=$(vm_stat)

stored=$(echo "$stats" | awk '/stored in compressor/ {print $6}' | tr -d '.')
occupied=$(echo "$stats" | awk '/occupied by compressor/ {print $6}' | tr -d '.')
compressions=$(echo "$stats" | awk '/^Compressions:/ {print $2}' | tr -d '.')
decompressions=$(echo "$stats" | awk '/^Decompressions:/ {print $2}' | tr -d '.')

stored_mb=$((stored * PAGE_SIZE / 1024 / 1024))
occupied_mb=$((occupied * PAGE_SIZE / 1024 / 1024))

if [[ $occupied -gt 0 ]]; then
    ratio=$(echo "scale=2; $stored / $occupied" | bc)
else
    ratio="N/A"
fi

echo "Memory Compressor Statistics"
echo "============================"
echo "Logical compressed:  ${stored_mb} MB (${stored} pages)"
echo "Physical compressor: ${occupied_mb} MB (${occupied} pages)"
echo "Compression ratio:   ${ratio}:1"
echo "Total compressions:  ${compressions}"
echo "Total decompressions: ${decompressions}"
echo ""
echo "Memory saved: $((stored_mb - occupied_mb)) MB"
```

## Memory Pressure

macOS uses a memory pressure system to signal when memory is constrained.

### Checking Memory Pressure

```bash
# Quick check
$ memory_pressure
System-wide memory free percentage: 42%
System memory pressure level: 1

The system memory pressure level is
currently: 1 (Normal)
```

### Pressure Levels

| Level | State | System Behavior |
|-------|-------|-----------------|
| 1 | Normal | No special action |
| 2 | Warning | Compress memory, notify apps |
| 4 | Critical | Aggressive compression, swap, terminate |

### Simulating Memory Pressure

For testing how apps respond to memory pressure:

```bash
# Simulate memory warning (apps receive notification)
$ memory_pressure -S -l warn

# Simulate critical memory
$ memory_pressure -S -l critical

# Release simulated pressure
$ memory_pressure -S -l normal
```

### Monitoring Pressure Over Time

```bash
#!/bin/bash
# pressure-monitor.sh - Log memory pressure changes

echo "Monitoring memory pressure (Ctrl+C to stop)..."
last_level=0

while true; do
    level=$(memory_pressure 2>/dev/null | grep "pressure level:" | head -1 | awk '{print $NF}')

    if [[ "$level" != "$last_level" ]]; then
        case $level in
            1) state="NORMAL" ;;
            2) state="WARNING" ;;
            4) state="CRITICAL" ;;
            *) state="UNKNOWN" ;;
        esac
        echo "$(date): Memory pressure changed to $state (level $level)"
        last_level=$level
    fi

    sleep 5
done
```

## Swap Management

macOS uses encrypted swap files when physical RAM and compressed memory are exhausted.

### Swap Configuration

```bash
# Check swap usage
$ sysctl vm.swapusage
vm.swapusage: total = 2048.00M  used = 256.00M  free = 1792.00M  (encrypted)

# Swap file location
$ ls -la /private/var/vm/
total 524288
drwxr-xr-x  4 root  wheel       128 Jan 15 10:00 .
drwxr-xr-x  31 root  wheel       992 Jan 10 08:00 ..
-rw-------   1 root  wheel  2147483648 Jan 15 10:00 swapfile0
-rw-r--r--   1 root  wheel         0 Jan 15 10:00 swapfile.lock

# Swap statistics in vm_stat
$ vm_stat | grep -E "Swapins|Swapouts|Pageins|Pageouts"
Pageins:                                 123456.
Pageouts:                                 12345.
Swapins:                                    567.
Swapouts:                                   890.
```

### Swap Activity Monitoring

```bash
# Watch for swap activity
$ vm_stat 1 | awk '
NR==1 {next}
NR==2 {header=1; next}
{
    if (header) {
        header=0
        print "Time       SwapIn  SwapOut  PageIn  PageOut"
    }
    # vm_stat continuous output format varies
    print strftime("%H:%M:%S"), $10, $11, $8, $9
}
'
```

### Understanding Swap Impact

When swap is active:

```
Performance impact of memory location:
┌────────────────────────────────────────────────────────────────┐
│ Memory Type      │ Access Time    │ Relative Speed            │
├──────────────────┼────────────────┼───────────────────────────┤
│ RAM              │ ~100 ns        │ 1x (baseline)             │
│ Compressed RAM   │ ~500 ns        │ 5x slower                 │
│ SSD Swap         │ ~50-100 µs     │ 500-1000x slower          │
│ HDD Swap (old)   │ ~10 ms         │ 100,000x slower           │
└────────────────────────────────────────────────────────────────┘
```

## Per-Process Memory Analysis

### Using ps

```bash
# Memory columns: VSZ (virtual), RSS (resident)
$ ps aux | head -1
USER  PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND

# Top memory consumers
$ ps aux --sort=-%mem | head -10

# Specific process memory
$ ps -o pid,rss,vsz,comm -p 12345
```

### Using top

```bash
# Memory-sorted view
$ top -o mem

# Show specific columns
$ top -stats pid,command,mem,rprvt,purg,cmprs,vprvt
```

### Using vmmap

`vmmap` provides detailed memory mapping for a process:

```bash
# Full memory map
$ vmmap 12345

# Summary only
$ vmmap --summary 12345

# Example output:
$ vmmap --summary $(pgrep Safari)
Process:         Safari [12345]
Path:            /Applications/Safari.app/Contents/MacOS/Safari
...
ReadOnly portion of Libraries: Total=456.7M resident=234.5M(51%) swapped_out_or_unallocated=222.2M(49%)
Writable regions: Total=1.2G written=567.8M(47%) resident=890.1M(74%) swapped_out=0K(0%) unallocated=333.3M(28%)

VIRTUAL   RESIDENT    DIRTY  SWAPPED VOLATILE   NONVOL    EMPTY   REGION
SIZE      SIZE        SIZE   OUT SIZE            PURGEABLE PURGEABLE  DETAIL
========  ========  ========  ========  ========  ========  ======== ========
1.2G      890.1M    567.8M     0K       45.6M     12.3M     0K       TOTAL
```

### Understanding vmmap Output

| Category | Description |
|----------|-------------|
| VIRTUAL SIZE | Address space allocated |
| RESIDENT SIZE | Actually in RAM |
| DIRTY SIZE | Modified, must be saved |
| SWAPPED OUT | Paged to disk |
| PURGEABLE | Can be discarded |

### Memory Regions Script

```bash
#!/bin/bash
# process-memory.sh - Analyze process memory usage

if [[ -z "$1" ]]; then
    echo "Usage: $0 <process-name-or-pid>"
    exit 1
fi

if [[ "$1" =~ ^[0-9]+$ ]]; then
    PID=$1
else
    PID=$(pgrep -x "$1" | head -1)
    if [[ -z "$PID" ]]; then
        echo "Process not found: $1"
        exit 1
    fi
fi

PROCESS=$(ps -p $PID -o comm=)

echo "Memory Analysis for $PROCESS (PID: $PID)"
echo "=========================================="
echo ""

# Basic stats from ps
echo "From ps:"
ps -o rss=,vsz= -p $PID | awk '{
    printf "  Resident (RSS): %.1f MB\n", $1/1024
    printf "  Virtual (VSZ):  %.1f MB\n", $2/1024
}'

echo ""
echo "From vmmap summary:"
vmmap --summary $PID 2>/dev/null | grep -E "^(TOTAL|.*MALLOC|.*Writable)" | head -10
```

## Diagnosing Memory Issues

### High Memory Usage

```bash
# Find memory hogs
$ ps aux --sort=-%mem | head -10

# Detailed view of largest process
$ PID=$(ps aux --sort=-%mem | awk 'NR==2 {print $2}')
$ vmmap --summary $PID
```

### Memory Leaks

```bash
# Check for growing memory over time
$ while true; do
    ps -o rss= -p $PID
    sleep 10
done

# Use leaks tool (requires debug symbols for best results)
$ leaks $PID

# Example output:
Process 12345: 234 nodes malloced for 567 KB
Process 12345: 5 leaks for 128 bytes
```

### Memory Pressure Issues

```bash
#!/bin/bash
# diagnose-memory.sh - Memory diagnostics

echo "=== System Memory Overview ==="
vm_stat | head -20

echo ""
echo "=== Memory Pressure ==="
memory_pressure 2>/dev/null | head -5

echo ""
echo "=== Swap Usage ==="
sysctl vm.swapusage

echo ""
echo "=== Top Memory Consumers ==="
ps aux --sort=-%mem | head -6

echo ""
echo "=== Compressed Memory ==="
vm_stat | grep -E "compressor|Compression"

echo ""
echo "=== Recommendations ==="
FREE_PCT=$(memory_pressure 2>/dev/null | grep "free percentage" | awk '{print $5}' | tr -d '%')
if [[ -n "$FREE_PCT" ]] && [[ $FREE_PCT -lt 20 ]]; then
    echo "! Memory free is low ($FREE_PCT%). Consider closing applications."
fi

SWAPUSED=$(sysctl vm.swapusage | awk '{print $7}' | tr -d 'M')
if [[ "${SWAPUSED%.*}" -gt 0 ]]; then
    echo "! Swap is being used (${SWAPUSED}M). System may be slow."
fi
```

## Memory Optimization

### Purging Memory

The `purge` command forces disk cache to be purged:

```bash
# Clear disk caches (requires sudo)
$ sudo purge

# Note: This doesn't free app memory, only file caches
# Use for benchmarking with cold caches
```

### Application-Level Optimization

Apps can respond to memory warnings:

```bash
# Check if app responds to memory warnings
$ log show --predicate 'eventMessage contains "memory"' --last 1h | grep -i warning

# Memory warning notifications
# Apps receive: NSProcessInfoPowerStateDidChange
# Or: UIApplicationDidReceiveMemoryWarningNotification (iOS ported apps)
```

### Managing Wired Memory

Wired memory cannot be reclaimed. High wired memory indicates:
- Many kernel extensions
- Large file buffers
- GPU memory allocations

```bash
# Check wired memory
$ vm_stat | grep "wired"
Pages wired down:                        456789.

# Convert to MB
$ vm_stat | awk -v ps=$(pagesize) '/wired/ {printf "Wired: %.0f MB\n", $4 * ps / 1024 / 1024}'

# List kernel extensions (can contribute to wired)
$ kextstat | wc -l
```

## Unified Memory on Apple Silicon

Apple Silicon uses unified memory shared between CPU and GPU.

### GPU Memory Allocation

```bash
# Check GPU memory usage
$ sudo powermetrics --samplers gpu_power -n 1 | grep -i memory

# In Activity Monitor:
# Enable GPU Memory column (View > Columns)
```

### Implications

| Aspect | Traditional | Unified Memory |
|--------|-------------|----------------|
| GPU allocation | Dedicated VRAM | Shared RAM pool |
| Data transfer | Copy between RAM/VRAM | Zero-copy |
| Total available | RAM + VRAM | Single RAM pool |
| Memory pressure | Separate | Combined |

## Advanced Memory Diagnostics

### Using Instruments

For deep memory analysis, use Instruments (part of Xcode):

```bash
# Record memory allocations
$ xcrun xctrace record --template 'Allocations' --attach $PID --time-limit 30s

# Analyze leaks
$ xcrun xctrace record --template 'Leaks' --attach $PID --time-limit 30s
```

### Memory Footprint Tool

```bash
# footprint command (requires Xcode command line tools)
$ footprint $PID

# Shows detailed memory attribution:
# - Dirty memory
# - Swapped memory
# - Compressed memory
# - IOKit mappings
```

### System Memory Snapshot

```bash
#!/bin/bash
# memory-snapshot.sh - Comprehensive memory snapshot

echo "Memory Snapshot - $(date)"
echo "========================="
echo ""

# Hardware
echo "=== Hardware ==="
echo "Total RAM: $(sysctl -n hw.memsize | awk '{print $1/1024/1024/1024 " GB"}')"
echo "Page size: $(pagesize) bytes"
echo ""

# vm_stat summary
echo "=== VM Statistics ==="
vm_stat | head -15
echo ""

# Memory pressure
echo "=== Pressure ==="
memory_pressure 2>/dev/null | head -5
echo ""

# Swap
echo "=== Swap ==="
sysctl vm.swapusage
echo ""

# Top processes
echo "=== Top 10 by Memory ==="
printf "%-8s %-6s %12s %12s %s\n" "PID" "%MEM" "RSS(MB)" "VSZ(MB)" "COMMAND"
ps aux --sort=-%mem | awk 'NR>1 && NR<=11 {
    printf "%-8s %-6s %12.1f %12.1f %s\n", $2, $4, $6/1024, $5/1024, $11
}'
```

## Summary

Key memory management concepts:

| Concept | Description | Command |
|---------|-------------|---------|
| Memory Pressure | System-wide memory demand | `memory_pressure` |
| Compression | In-memory page compression | `vm_stat \| grep compressor` |
| Wired Memory | Non-pageable kernel memory | `vm_stat \| grep wired` |
| Swap | Disk-backed virtual memory | `sysctl vm.swapusage` |
| RSS | Resident Set Size (actual RAM) | `ps -o rss` |
| VSZ | Virtual Size (address space) | `ps -o vsz` |

Critical commands for memory diagnosis:

```bash
# Quick health check
$ memory_pressure

# Detailed statistics
$ vm_stat

# Per-process analysis
$ vmmap --summary $PID

# Memory consumers
$ ps aux --sort=-%mem | head -10

# Swap status
$ sysctl vm.swapusage

# Check for leaks
$ leaks $PID
```

Understanding memory management helps diagnose slowdowns, optimize applications, and make informed decisions about system resources.
