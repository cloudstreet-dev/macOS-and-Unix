# Performance Profiling Tools

macOS provides powerful profiling tools for understanding application performance. This chapter covers both command-line tools and Instruments, Apple's comprehensive profiling application.

## Overview of Profiling Tools

| Tool | Purpose | Best For |
|------|---------|----------|
| `time` | Basic timing | Quick measurements |
| `sample` | CPU sampling | Quick CPU profile |
| `spindump` | Hang analysis | Unresponsive apps |
| `leaks` | Memory leaks | Memory debugging |
| `heap` | Heap analysis | Memory usage |
| `vmmap` | Memory map | Virtual memory |
| `fs_usage` | File system | I/O debugging |
| `dtrace` | Dynamic tracing | Advanced profiling |
| Instruments | GUI profiling | Comprehensive analysis |

## Basic Timing with time

### Built-in time Command

```bash
# Bash/zsh built-in
$ time ./myprogram
real    0m1.234s
user    0m0.456s
sys     0m0.078s
```

- **real**: Wall clock time
- **user**: CPU time in user mode
- **sys**: CPU time in kernel mode

### GNU time (more detailed)

```bash
# Install
$ brew install gnu-time

# Use with full path or alias
$ /opt/homebrew/bin/gtime -v ./myprogram
        Command being timed: "./myprogram"
        User time (seconds): 0.45
        System time (seconds): 0.07
        Percent of CPU this job got: 42%
        Elapsed (wall clock) time: 1.23
        Maximum resident set size (kbytes): 12800
        ...
```

### Format Output

```bash
$ /opt/homebrew/bin/gtime -f "Time: %E\nMemory: %M KB\nCPU: %P" ./myprogram
Time: 0:01.23
Memory: 12800 KB
CPU: 42%
```

## sample - CPU Sampling

`sample` periodically records call stacks to identify where time is spent:

```bash
# Sample for 5 seconds
$ sample myprogram 5

# Sample specific PID
$ sample 12345 5

# Save to file
$ sample myprogram 5 -f output.txt
```

### Sample Output

```
Sampling process 12345 for 5 seconds with 1 millisecond of run time between samples
Sampling completed, processing symbols...

Analysis of sampling myprogram (pid 12345) every 1 millisecond
Process:         myprogram [12345]
Path:            /path/to/myprogram
Load Address:    0x100000000
...

Call graph:
    2500 Thread_1234   DispatchQueue_1: com.apple.main-thread
      2500 start  (in libdyld.dylib)
        2500 main  (in myprogram)
          1800 expensive_function  (in myprogram)
            1800 calculate_something  (in myprogram)
          700 other_function  (in myprogram)

Total number in stack (recursive counted multiple, when):
        2500       main  (in myprogram)
        1800       expensive_function  (in myprogram)
        1800       calculate_something  (in myprogram)
```

### Understanding Sample Output

The numbers represent how many samples included that function. Higher numbers = more time spent.

## spindump - Hang Analysis

`spindump` captures detailed state when an app is unresponsive:

```bash
# Capture hung process
$ sudo spindump myprogram -o spindump.txt

# With duration
$ sudo spindump 12345 5 -o spindump.txt  # 5 second sample

# Sample all processes
$ sudo spindump -notarget -o system_spindump.txt
```

### Triggered Automatically

macOS generates spindumps automatically for hung apps. Find them at:

```bash
$ ls ~/Library/Logs/DiagnosticReports/*spin*
```

## Memory Analysis Tools

### leaks - Memory Leak Detection

```bash
# Check for leaks in running process
$ leaks 12345
Process 12345: 1234 nodes malloced for 567 KB
Process 12345: 2 leaks for 128 bytes

Leak: 0x600000c00100  size=64  zone: DefaultMallocZone_0x108500000
    Call stack: main | allocate_something | malloc

Leak: 0x600000c00140  size=64  zone: DefaultMallocZone_0x108500000
    Call stack: main | another_leak | malloc
```

### Using MallocStackLogging

For detailed leak stacks:

```bash
# Run with malloc logging
$ MallocStackLogging=1 ./myprogram &
[1] 12345

# Check leaks with full stacks
$ leaks 12345

# Or use malloc_history
$ malloc_history 12345 0x600000c00100
```

### heap - Heap Analysis

```bash
# Summary of heap allocations
$ heap 12345
Process 12345: 3 zones
Zone DefaultMallocZone_0x108500000: Overall size: 2.5MB; 12345 nodes malloced

All zones: 12345 nodes malloced - 2.5MB

Zone DefaultMallocZone_0x108500000: 12345 nodes (2.5MB)
    COUNT     BYTES     AVG   CLASS_NAME
    5000    500000   100.0   non-object
    1234    123400   100.0   CFString
     567     56700   100.0   NSArray
```

### vmmap - Virtual Memory Map

```bash
# Memory map overview
$ vmmap 12345

# Summary only
$ vmmap --summary 12345

Process:         myprogram [12345]
Path:            /path/to/myprogram
...

                                VIRTUAL   RESIDENT   DIRTY
REGION TYPE                     SIZE       SIZE     SIZE
===========                   ======     ======   ======
MALLOC                         50.0M     40.0M    35.0M
MALLOC guard page              32K          0K       0K
Stack                          8.0M       200K     200K
__DATA                         2.0M       1.0M     512K
__TEXT                         1.5M       1.5M       0K
...

# Wide output
$ vmmap --wide 12345
```

## File System Tracing

### fs_usage - File System Activity

```bash
# All filesystem activity
$ sudo fs_usage

# Filter by process
$ sudo fs_usage -w -f filesys myprogram

# Filter by operation type
$ sudo fs_usage -w -f diskio

# Output to file
$ sudo fs_usage myprogram > fs_trace.txt 2>&1
```

### fs_usage Output

```
14:23:45  open     /path/to/file          0.000234   myprogram
14:23:45  read     F=3            4096    0.000123   myprogram
14:23:45  close    F=3                    0.000012   myprogram
```

Columns: timestamp, operation, path/details, duration, process

## DTrace (Where Available)

DTrace is a powerful dynamic tracing framework. Note: Full DTrace requires disabling SIP on modern macOS.

### Basic Usage

```bash
# One-liner: trace system calls
$ sudo dtrace -n 'syscall:::entry { @[execname] = count(); }'

# Trace process syscalls
$ sudo dtrace -n 'syscall:::entry /execname == "myprogram"/ { @[probefunc] = count(); }'

# Stop with Ctrl+C to see output
^C
  read                              234
  write                             123
  open                               45
```

### DTrace Scripts

```bash
# syscall_count.d
#!/usr/sbin/dtrace -s

syscall:::entry
/execname == "myprogram"/
{
    @syscalls[probefunc] = count();
}

END
{
    printf("\nSystem call counts:\n");
    printa(@syscalls);
}
```

Run:
```bash
$ sudo dtrace -s syscall_count.d
```

### DTrace Limitations on Modern macOS

With SIP enabled:
- Can only trace user-space
- Many probes unavailable
- Cannot trace system processes

For full DTrace access:
1. Boot to Recovery Mode
2. `csrutil disable` (security risk)
3. Reboot

Not recommended for most users.

## Instruments

Instruments is Apple's comprehensive profiling tool, part of Xcode.

### Launching Instruments

```bash
# From command line
$ open -a Instruments

# With specific template
$ instruments -t "Time Profiler"

# Profile a command
$ instruments -t "Time Profiler" -D output.trace ./myprogram
```

### Common Instruments Templates

| Template | Purpose |
|----------|---------|
| Time Profiler | CPU usage sampling |
| Allocations | Memory allocation tracking |
| Leaks | Memory leak detection |
| System Trace | Comprehensive system events |
| File Activity | File system operations |
| Network | Network connections |
| Core Data | Core Data performance |
| Animation Hitches | UI performance |
| Metal System Trace | GPU profiling |

### Using Time Profiler

1. Open Instruments
2. Choose "Time Profiler"
3. Select target (app or process)
4. Click Record
5. Exercise your code
6. Click Stop
7. Analyze call tree

### Command-Line Profiling

```bash
# Record with instruments
$ xcrun xctrace record --template 'Time Profiler' \
    --launch -- ./myprogram arg1 arg2

# Output creates a .trace file
$ ls *.trace

# Open in Instruments
$ open output.trace
```

### Analyzing Trace Files

```bash
# Export to XML
$ xcrun xctrace export --input recording.trace --output results.xml

# List available schemas
$ xcrun xctrace export --input recording.trace --toc
```

## Additional Tools

### Activity Monitor Command-Line Equivalent

```bash
# top - live process info
$ top

# Sort by CPU
$ top -o cpu

# Sort by memory
$ top -o mem

# Specific process
$ top -pid 12345
```

### System-Wide Memory Pressure

```bash
# Memory pressure statistics
$ memory_pressure
The system has 16384 MB of physical memory.
The system has 8192 MB of free memory.
...

# Simulate pressure
$ memory_pressure -l warn
```

### CPU Usage Statistics

```bash
# powermetrics (detailed power/performance)
$ sudo powermetrics --samplers cpu_power -n 1
...
CPU Average frequency: 3200 MHz
CPU Percent at or above 3000 MHz: 85%
...

# sysctl for CPU info
$ sysctl -a | grep cpu
hw.ncpu: 8
hw.activecpu: 8
...
```

## Profiling Best Practices

### Before Profiling

1. Build with optimizations (to profile realistic code)
2. But keep debug symbols (`-g` flag)
3. Use Release configuration, not Debug
4. Have representative test data

### During Profiling

1. Minimize background activity
2. Close unnecessary applications
3. Run multiple trials
4. Use consistent inputs

### Interpreting Results

```bash
# Get symbol info
$ atos -o ./myprogram -l 0x100000000 0x100003f00
expensive_function (in myprogram) (main.c:42)
```

### Release vs Debug Builds

```bash
# Debug build (slow but informative)
$ clang -g -O0 program.c -o program_debug

# Release build (realistic performance)
$ clang -g -O2 program.c -o program_release

# Profile the release build
$ sample program_release 5
```

## Combining Tools

### Performance Investigation Workflow

1. **Quick timing**: Use `time`
2. **CPU hotspots**: Use `sample`
3. **Memory issues**: Use `leaks`, `heap`, `vmmap`
4. **I/O problems**: Use `fs_usage`
5. **Deep analysis**: Use Instruments

### Example Investigation

```bash
# Step 1: Basic timing
$ time ./myprogram
real    0m5.234s

# Step 2: Where's the time going?
$ sample myprogram 5

# Step 3: Memory usage?
$ vmmap --summary $(pgrep myprogram)

# Step 4: Any leaks?
$ MallocStackLogging=1 ./myprogram &
$ leaks $!

# Step 5: File I/O?
$ sudo fs_usage myprogram
```

## Summary

| Tool | What It Shows | When to Use |
|------|---------------|-------------|
| `time` | Execution time | Quick benchmarks |
| `sample` | CPU call stacks | Finding hot functions |
| `spindump` | Hang state | App freezes |
| `leaks` | Memory leaks | Memory debugging |
| `heap` | Heap contents | Memory optimization |
| `vmmap` | Memory map | Memory layout |
| `fs_usage` | File operations | I/O debugging |
| `dtrace` | Dynamic tracing | Advanced analysis |
| Instruments | Everything | Comprehensive profiling |

macOS provides excellent profiling tools at various levels of detail. Start with simple tools (`time`, `sample`) and move to more complex ones (Instruments, DTrace) as needed.
