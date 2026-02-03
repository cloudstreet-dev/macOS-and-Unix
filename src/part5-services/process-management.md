# Process Management: CLI vs GUI

macOS offers both command-line tools and graphical applications for managing processes. Understanding both approaches helps you choose the right tool for each situation and troubleshoot process-related issues effectively.

## Command-Line Process Tools

### ps - Process Status

```bash
# BSD syntax (common on macOS)
$ ps aux
USER   PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
root     1   0.0  0.1 410148544  18624   ??  Ss   Mon10AM   0:15.23 /sbin/launchd
david  501   0.0  0.2 419826688  35712   ??  S    Mon10AM   0:05.67 /usr/libexec/...

# POSIX syntax
$ ps -ef
  UID   PID  PPID   C STIME   TTY           TIME CMD
    0     1     0   0 Mon10AM ??         0:15.23 /sbin/launchd

# Current user's processes
$ ps -u $USER

# Process tree (requires pstree)
$ brew install pstree
$ pstree
```

### Key ps Columns

| Column | Meaning |
|--------|---------|
| PID | Process ID |
| PPID | Parent Process ID |
| %CPU | CPU usage percentage |
| %MEM | Memory usage percentage |
| VSZ | Virtual memory size |
| RSS | Resident set size (physical memory) |
| TT | Controlling terminal |
| STAT | Process state |
| TIME | CPU time consumed |

### Process States

| State | Meaning |
|-------|---------|
| R | Running |
| S | Sleeping (interruptible) |
| U | Uninterruptible sleep |
| I | Idle |
| T | Stopped |
| Z | Zombie |

### top - Real-Time Process Monitor

```bash
# Start top
$ top

# Sort by CPU (default)
$ top -o cpu

# Sort by memory
$ top -o mem

# Show specific user
$ top -U david

# Non-interactive (for scripts)
$ top -l 1 -n 10 -stats pid,command,cpu
```

### htop - Enhanced top

```bash
$ brew install htop
$ htop
# Arrow keys to navigate, F9 to kill, q to quit
```

### Signals and kill

```bash
# List signals
$ kill -l

# Send SIGTERM (graceful shutdown)
$ kill <pid>
$ kill -15 <pid>
$ kill -TERM <pid>

# Send SIGKILL (force kill)
$ kill -9 <pid>
$ kill -KILL <pid>

# Send SIGHUP (reload configuration)
$ kill -HUP <pid>

# Kill by name
$ pkill -f "process_name"
$ killall process_name
```

### Finding Processes

```bash
# Find by name
$ pgrep -l Safari
1234 Safari

# Find by name (full command line)
$ pgrep -fl python

# Find what's using a port
$ lsof -i :8080
COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
python  12345 david  4u   IPv4 0x1234567890      0t0  TCP *:http-alt (LISTEN)

# Find what's using a file
$ lsof /path/to/file

# Find open files by process
$ lsof -p <pid>

# Find processes using network
$ lsof -i
```

### Process Priority

```bash
# Run with lower priority (higher nice value)
$ nice -n 10 ./slow_script.sh

# Change running process priority
$ renice 10 -p <pid>

# Run with higher priority (requires root)
$ sudo nice -n -10 ./important_script.sh
```

## Activity Monitor

Activity Monitor is macOS's graphical process manager, found in `/Applications/Utilities/Activity Monitor.app`.

### Views

| Tab | Shows |
|-----|-------|
| CPU | Processor usage |
| Memory | RAM usage |
| Energy | Battery impact |
| Disk | I/O operations |
| Network | Network activity |

### Key Features

**Process List**:
- Sort by any column
- Search/filter processes
- Show all processes vs user processes

**Process Information** (double-click process):
- Open files and ports
- Memory map
- Statistics
- Parent process

**Actions** (select process, use toolbar or View menu):
- Quit: Sends SIGTERM
- Force Quit: Sends SIGKILL
- Sample: Creates stack trace

### Activity Monitor from CLI

```bash
# Open Activity Monitor
$ open -a "Activity Monitor"

# Get same data as Activity Monitor
$ top -l 1 -n 0 -stats pid,command,cpu,mem

# Sample a process (like Activity Monitor's Sample)
$ sample <pid> 5  # 5 second sample
```

## Comparing CLI and GUI

| Task | CLI | GUI |
|------|-----|-----|
| Quick process list | `ps aux` | Activity Monitor |
| Real-time monitoring | `top`, `htop` | Activity Monitor |
| Kill process | `kill <pid>` | Select → Quit |
| Find resource hog | `top -o cpu` | Sort by CPU column |
| Analyze process | `lsof -p <pid>` | Double-click → Open Files |
| Sample/profile | `sample <pid>` | View → Sample |
| System overview | `vm_stat`, `iostat` | Activity Monitor tabs |

## Process Monitoring Tools

### vm_stat - Virtual Memory

```bash
$ vm_stat
Mach Virtual Memory Statistics: (page size of 16384 bytes)
Pages free:                               41203.
Pages active:                            385918.
Pages inactive:                          347108.
Pages speculative:                        11276.
Pages wired down:                         96482.
...

# Continuous monitoring
$ vm_stat 1  # Every 1 second
```

### iostat - I/O Statistics

```bash
$ iostat
              disk0       cpu    load average
    KB/t  tps  MB/s  us sy id   1m   5m   15m
   25.67   12  0.30   3  2 95  2.07 2.15 2.10

# Continuous
$ iostat 1
```

### fs_usage - File System Activity

```bash
# Monitor file system calls (requires root)
$ sudo fs_usage
```

### nettop - Network Top

```bash
# Monitor network connections per process
$ nettop
```

## Practical Examples

### Find and Kill Runaway Process

```bash
# Find high CPU process
$ ps aux --sort=-%cpu | head -5

# Or interactively
$ top
# Press 'q' when found

# Kill it
$ kill <pid>
# If unresponsive
$ kill -9 <pid>
```

### Find Memory Hog

```bash
$ ps aux --sort=-%mem | head -10

# Or in top
$ top -o mem
```

### Monitor Specific Process

```bash
# Watch a process
$ while true; do ps -p <pid> -o %cpu,%mem,rss; sleep 1; done

# Or use watch
$ brew install watch
$ watch -n 1 "ps -p <pid> -o %cpu,%mem,rss"
```

### Debug Hanging Process

```bash
# Sample the process
$ sample <pid> 10 -f /tmp/sample.txt

# View open files
$ lsof -p <pid>

# View system calls (requires SIP adjustment)
$ sudo dtruss -p <pid>
```

## Summary

Process management tools:

| Tool | Purpose |
|------|---------|
| `ps` | Snapshot of processes |
| `top` | Real-time monitoring |
| `htop` | Enhanced top |
| `kill` | Send signals to processes |
| `pgrep`/`pkill` | Find/kill by name |
| `lsof` | Open files and ports |
| `nice`/`renice` | Process priority |
| `sample` | Stack trace sampling |
| Activity Monitor | GUI all-in-one |

Best practices:
- Use `top` or Activity Monitor for interactive monitoring
- Use `ps` for scripting and one-time checks
- Always try `kill` before `kill -9`
- Use `lsof` to find what's using resources
