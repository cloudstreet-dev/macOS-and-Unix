# Activity Monitor: Beyond the GUI

Activity Monitor is macOS's built-in system monitor, but most users only scratch the surface of its capabilities. This chapter explores every tab, hidden columns, data interpretation, and techniques for extracting actionable performance insights.

## Launching Activity Monitor

```bash
# Open from Terminal
$ open -a "Activity Monitor"

# Or via Spotlight (Cmd+Space, type "Activity Monitor")
```

Activity Monitor can also be controlled via AppleScript for automation:

```bash
# Get process list via AppleScript
$ osascript -e 'tell application "System Events" to get name of every process'
```

## The Five Tabs

Activity Monitor organizes system information into five tabs:

```
┌─────────────────────────────────────────────────────────────────┐
│  CPU  │  Memory  │  Energy  │  Disk  │  Network  │             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    Process List                                 │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    Bottom Panel                                 │
│              (Tab-specific graphs/stats)                        │
└─────────────────────────────────────────────────────────────────┘
```

Each tab shows different columns and bottom panel information.

## CPU Tab

### Default Columns

| Column | Description |
|--------|-------------|
| Process Name | Application or process name |
| % CPU | Percentage of CPU time used |
| CPU Time | Total CPU time consumed |
| Threads | Number of active threads |
| Idle Wake Ups | Times process woke from idle |
| PID | Process identifier |
| User | Username owning the process |

### Hidden CPU Columns

Right-click the column header to add these valuable hidden columns:

| Column | Description | When Useful |
|--------|-------------|-------------|
| % GPU | GPU utilization | Graphics/video work |
| GPU Time | Total GPU time | Identifying GPU hogs |
| Architecture | arm64/x86_64 | Finding Rosetta processes |
| Sandbox | Sandboxed status | Security analysis |
| Restricted | Hardened runtime | Security analysis |
| App Nap | App Nap status | Energy debugging |
| Sudden Termination | Can be killed safely | Shutdown debugging |
| Preventing Sleep | Blocking system sleep | Battery debugging |

### Bottom Panel: CPU Usage

```
┌─────────────────────────────────────────────────────────────────┐
│ System: 5.50%  ████▌                                            │
│ User:   12.25% ████████████▎                                    │
│ Idle:   82.25% ██████████████████████████████████████████████  │
├─────────────────────────────────────────────────────────────────┤
│ Threads: 1,234    Processes: 456                                │
└─────────────────────────────────────────────────────────────────┘
```

**Understanding CPU percentages:**

- **System**: Kernel and system service work
- **User**: User application work
- **Idle**: Available CPU capacity

On multi-core systems, a single process can exceed 100% (e.g., 400% = 4 cores fully utilized).

### CPU Interpretation

```bash
# Equivalent CLI information
$ top -l 1 -n 0 | grep -E "CPU|Processes|Threads"
Processes: 456 total, 3 running, 453 sleeping, 1234 threads
CPU usage: 12.25% user, 5.50% sys, 82.25% idle
```

**Warning signs:**
- System % consistently > 20%: Possible driver or kernel issue
- User % near 100%: CPU-bound application
- High idle wake ups: Power efficiency problem

## Memory Tab

### Default Columns

| Column | Description |
|--------|-------------|
| Memory | Current memory footprint |
| Real Memory | Physical RAM used |
| Virtual Memory | Address space size |
| Shared Memory | Memory shared with other processes |
| Real Private Memory | Non-shared physical RAM |
| Compressed Memory | Compressed pages in RAM |

### Hidden Memory Columns

| Column | Description | When Useful |
|--------|-------------|-------------|
| Purgeable Memory | Memory that can be reclaimed | Memory optimization |
| Real Shared Memory | Actually shared RAM | Library sharing analysis |
| Dirty Memory | Modified pages | Swap prediction |
| Swapped Memory | Memory paged to disk | Performance issues |

### Bottom Panel: Memory Pressure

```
┌─────────────────────────────────────────────────────────────────┐
│ Memory Pressure:                                                │
│ [████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]            │
│                    Green (Normal)                               │
├─────────────────────────────────────────────────────────────────┤
│ Physical Memory:     16.00 GB                                   │
│ Memory Used:         12.45 GB                                   │
│   App Memory:        8.23 GB                                    │
│   Wired Memory:      2.12 GB                                    │
│   Compressed:        2.10 GB                                    │
│ Cached Files:        2.34 GB                                    │
│ Swap Used:           0 bytes                                    │
└─────────────────────────────────────────────────────────────────┘
```

**Memory Pressure Colors:**

| Color | Meaning | Action |
|-------|---------|--------|
| Green | Memory available | Normal operation |
| Yellow | Memory becoming limited | Consider closing apps |
| Red | Memory critically low | Close apps, investigate |

**Memory Categories:**

| Category | Description |
|----------|-------------|
| App Memory | Memory actively used by applications |
| Wired Memory | Kernel memory, cannot be compressed/swapped |
| Compressed | Inactive pages compressed in RAM |
| Cached Files | File data cached for faster access |

### Memory Interpretation

```bash
# Equivalent CLI information
$ vm_stat
Mach Virtual Memory Statistics: (page size of 16384 bytes)
Pages free:                               45231.
Pages active:                            892341.
Pages inactive:                          234521.
Pages wired down:                        456789.
Pages compressed:                        567890.
...

# Memory pressure
$ memory_pressure
System-wide memory free percentage: 42%
System memory pressure level: 1 (normal)
```

**Warning signs:**
- Memory pressure yellow/red: System is memory constrained
- Swap used > 0: RAM exhausted, performance will degrade
- Compressed memory very high: Near memory limits

## Energy Tab

### Default Columns

| Column | Description |
|--------|-------------|
| Energy Impact | Relative power consumption |
| Avg Energy Impact | Average over last 8 hours |
| App Nap | Is App Nap active |
| Preventing Sleep | Blocking system sleep |
| Graphics Card | Which GPU in use |

### Hidden Energy Columns

| Column | Description | When Useful |
|--------|-------------|-------------|
| Power | Instantaneous power draw | Battery debugging |
| Requires High Perf GPU | Needs discrete GPU | GPU switching |
| GPU Activity | GPU busy percentage | Graphics work |

### Bottom Panel: Energy Impact

```
┌─────────────────────────────────────────────────────────────────┐
│ Energy Impact:                                                  │
│ [History graph showing energy over time]                        │
├─────────────────────────────────────────────────────────────────┤
│ Battery Level: 85%                                              │
│ Time on Battery: 2:30                                           │
│ Time Remaining: 4:00                                            │
│ Graphics Card: Integrated (Intel/Apple)                         │
│ Battery (Last 12 hours): Apps Using Significant Energy:         │
│   Chrome, Slack, Xcode                                          │
└─────────────────────────────────────────────────────────────────┘
```

### Energy Impact Interpretation

Energy Impact is a relative score, not watts:

| Score | Impact | Example |
|-------|--------|---------|
| 0-4 | Low | Text editors, system utilities |
| 4-12 | Medium | Browsers (idle), communication apps |
| 12-30 | High | Video playback, compilation |
| 30+ | Very High | Gaming, video rendering |

```bash
# CLI energy information
$ pmset -g batt
Now drawing from 'Battery Power'
 -InternalBattery-0 (id=1234567) 85%; discharging; 4:00 remaining

# Detailed power metrics
$ sudo powermetrics --samplers cpu_power,tasks -n 1
```

**Warning signs:**
- "Preventing Sleep" apps: May drain battery unexpectedly
- High "Avg Energy Impact" apps: Consistently power-hungry
- Discrete GPU active: Significant power draw

## Disk Tab

### Default Columns

| Column | Description |
|--------|-------------|
| Bytes Read | Total bytes read from disk |
| Bytes Written | Total bytes written to disk |
| Reads In | Number of read operations |
| Writes Out | Number of write operations |

### Hidden Disk Columns

| Column | Description | When Useful |
|--------|-------------|-------------|
| Bytes Read/sec | Read throughput | I/O bottleneck detection |
| Bytes Written/sec | Write throughput | I/O bottleneck detection |
| Read Delta | Recent reads | Active I/O identification |
| Write Delta | Recent writes | Active I/O identification |

### Bottom Panel: Disk Activity

```
┌─────────────────────────────────────────────────────────────────┐
│ Disk Activity:                                                  │
│ [Read/Write graph over time]                                    │
├─────────────────────────────────────────────────────────────────┤
│ Data read:     1.25 GB        Data read/sec:     15 MB/s       │
│ Data written:  856 MB         Data written/sec:  5 MB/s        │
│ Reads in:      45,678         Writes out:        23,456        │
└─────────────────────────────────────────────────────────────────┘
```

### Disk Interpretation

```bash
# CLI disk I/O
$ iostat -d 2
              disk0
    KB/t  tps  MB/s
   24.00   45  1.05

# Per-process I/O (requires Full Disk Access)
$ sudo iotop
```

**Warning signs:**
- Constant high writes: May indicate logging issue or memory pressure
- Spiky reads: Spotlight indexing or Time Machine
- High I/O with low CPU: I/O bound workload

## Network Tab

### Default Columns

| Column | Description |
|--------|-------------|
| Sent Bytes | Total bytes transmitted |
| Received Bytes | Total bytes received |
| Sent Packets | Number of packets sent |
| Received Packets | Number of packets received |

### Hidden Network Columns

| Column | Description | When Useful |
|--------|-------------|-------------|
| Sent Bytes/sec | Upload rate | Bandwidth monitoring |
| Received Bytes/sec | Download rate | Bandwidth monitoring |
| Sent Packets/sec | Packet rate | Network debugging |
| Received Packets/sec | Packet rate | Network debugging |

### Bottom Panel: Network Activity

```
┌─────────────────────────────────────────────────────────────────┐
│ Network Activity:                                               │
│ [Send/Receive graph over time]                                  │
├─────────────────────────────────────────────────────────────────┤
│ Data received:  2.5 GB       Data received/sec:  1.2 MB/s      │
│ Data sent:      450 MB       Data sent/sec:      250 KB/s      │
│ Packets in:     1,234,567    Packets out:        456,789       │
└─────────────────────────────────────────────────────────────────┘
```

### Network Interpretation

```bash
# CLI network statistics
$ nettop -P -L 1

# Network connections per process
$ lsof -i -P | head -20
```

**Warning signs:**
- Unexpected high bandwidth: Possible malware or sync issues
- Unknown processes with network activity: Security concern
- High packet rate with low data: Possible DoS or scan

## Advanced Features

### View All Processes

By default, Activity Monitor shows only your processes:

1. View menu > All Processes
2. Or View > All Processes, Hierarchically (shows parent-child)

```bash
# CLI equivalent
$ ps aux | wc -l      # All processes
$ ps -u $(whoami)     # Only your processes
```

### Process Hierarchy View

View > All Processes, Hierarchically shows:

```
▼ launchd (1)
   ▼ UserEventAgent (234)
   ▼ Dock (456)
   ▼ Finder (789)
   ▼ loginwindow (123)
      ▼ Terminal (345)
         ▼ zsh (567)
            ▼ top (890)
```

### Inspect Process

Double-click any process to see detailed information:

**Open Files and Ports tab:**
- File descriptors
- Network connections
- Shows what resources the process uses

```bash
# CLI equivalent
$ lsof -p PID
```

**Memory tab:**
- Memory regions
- Memory map
- Detailed memory breakdown

```bash
# CLI equivalent
$ vmmap PID
```

**Statistics tab:**
- CPU usage history
- Context switches
- Page faults

**Sampling tab:**
- Takes a performance sample
- Shows where CPU time is spent

```bash
# CLI equivalent
$ sample PID 5 -f /tmp/sample.txt
```

### Diagnostic Reports

Activity Monitor can create system reports:

View menu > System Diagnostic

This runs:
- `sysdiagnose` in the background
- Creates a comprehensive system report
- Saves to `~/Desktop` or specified location

```bash
# CLI equivalent
$ sudo sysdiagnose
```

### Spindump

When an app is unresponsive:

1. Select the process
2. View > Sample Process or View > Spindump

```bash
# CLI equivalent
$ sudo spindump PID 5 -file /tmp/spindump.txt
```

## Exporting Data

### Copy Process Information

1. Select process(es)
2. Edit > Copy (Cmd+C)

Copies tab-separated data for spreadsheets.

### Sample Data Script

Export Activity Monitor-like data programmatically:

```bash
#!/bin/bash
# activity-export.sh - Export process data

echo "Timestamp,PID,Process,CPU%,Memory(MB),Threads"
while IFS= read -r line; do
    pid=$(echo "$line" | awk '{print $1}')
    cpu=$(echo "$line" | awk '{print $2}')
    mem=$(echo "$line" | awk '{print $3}')
    name=$(echo "$line" | awk '{for(i=4;i<=NF;i++) printf $i" "; print ""}')
    threads=$(ps -p "$pid" -o nlwp= 2>/dev/null || echo "0")
    echo "$(date +%Y-%m-%d_%H:%M:%S),$pid,$name,$cpu,$mem,$threads"
done < <(ps -eo pid,%cpu,rss,comm | tail -n +2 | sort -k2 -rn | head -20)
```

## Hidden Preferences

Activity Monitor stores preferences in:

```bash
$ defaults read com.apple.ActivityMonitor

# Useful settings:
# Show all processes by default
$ defaults write com.apple.ActivityMonitor ShowCategory -int 100

# Update frequency (1=very often, 5=rarely)
$ defaults write com.apple.ActivityMonitor UpdatePeriod -int 2

# Icon type in Dock (0=app icon, 2=CPU history, 3=network, 5=disk, 6=CPU)
$ defaults write com.apple.ActivityMonitor IconType -int 6
```

## Dock Icon Monitoring

Activity Monitor can show live stats in the Dock:

View menu > Dock Icon:
- Application Icon (default)
- CPU Usage
- CPU History
- Network Usage
- Disk Activity

This provides at-a-glance system monitoring.

## Comparison: Activity Monitor vs CLI Tools

| Feature | Activity Monitor | CLI Tools |
|---------|------------------|-----------|
| Visual graphs | Yes | No (unless using htop) |
| Process hierarchy | Yes | `pstree`, `ps -ef` |
| Real-time updates | Yes | `top`, `htop` |
| Sample process | Yes | `sample` command |
| Export data | Copy only | Redirect to file |
| Scriptable | Limited | Fully scriptable |
| Remote access | No | Via SSH |
| Resource usage | Higher | Lower |

## Best Practices

### For Troubleshooting

1. **Start with the right tab**: CPU for slow system, Memory for app crashes
2. **Enable hidden columns**: Architecture, Preventing Sleep are invaluable
3. **Use hierarchical view**: Find parent processes causing issues
4. **Sample unresponsive apps**: Gather data before force quitting

### For Monitoring

1. **Set Dock icon to CPU**: Quick visual indicator
2. **Keep Activity Monitor running**: Catch intermittent issues
3. **Check Memory Pressure regularly**: Early warning of problems
4. **Review Energy tab on battery**: Identify power hogs

### For Analysis

1. **Sort by relevant metric**: % CPU for performance, Energy for battery
2. **Watch over time**: Patterns reveal issues better than snapshots
3. **Cross-reference tabs**: High memory often correlates with disk I/O
4. **Use Inspect window**: Deep dive into suspicious processes

## Summary

Activity Monitor provides comprehensive system monitoring:

| Tab | Key Metrics | Watch For |
|-----|-------------|-----------|
| CPU | % CPU, System/User split | Runaway processes, high system % |
| Memory | Memory Pressure, Compressed | Yellow/red pressure, swap usage |
| Energy | Energy Impact, Preventing Sleep | Battery drainers, sleep blockers |
| Disk | Read/Write rates | Excessive I/O, constant writes |
| Network | Send/Receive rates | Unexpected traffic, high bandwidth |

Key hidden columns to enable:
- **Architecture**: Identify Rosetta processes
- **Preventing Sleep**: Find battery drainers
- **App Nap**: Verify power optimization
- **GPU**: Track graphics usage

Activity Monitor is excellent for visual monitoring and quick investigations. For scripting, automation, and remote access, the command-line tools covered in the next chapter are essential.
