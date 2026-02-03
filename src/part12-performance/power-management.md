# Power Management and Battery

Effective power management extends battery life on laptops and reduces energy consumption on desktops. macOS provides sophisticated power management through `pmset`, `caffeinate`, power assertions, and App Nap. Understanding these systems enables optimization for both performance and efficiency.

## Power Management Architecture

### Power Management Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                    Applications                                  │
│          (Power assertions, App Nap responses)                  │
├─────────────────────────────────────────────────────────────────┤
│                    WindowServer                                  │
│          (Display sleep, screen brightness)                     │
├─────────────────────────────────────────────────────────────────┤
│                    IOKit Power Management                       │
│          (Device power states, sleep/wake)                      │
├─────────────────────────────────────────────────────────────────┤
│                    Kernel                                        │
│          (CPU power states, thermal management)                 │
├─────────────────────────────────────────────────────────────────┤
│                    SMC (System Management Controller)           │
│          (Battery, fans, sensors - Intel Macs)                  │
│                    or                                            │
│                    Apple Silicon Power Management               │
│          (Integrated power controller)                          │
└─────────────────────────────────────────────────────────────────┘
```

## pmset - Power Management Settings

`pmset` is the primary tool for configuring power settings.

### Viewing Current Settings

```bash
# All current settings
$ pmset -g
System-wide power settings:
Currently in use:
 standby              1
 Sleep On Power Button 1
 hibernatefile        /var/vm/sleepimage
 powernap             1
 networkoversleep     0
 disksleep            10
 sleep                1 (sleep prevented by screensharingd)
 hibernatemode        3
 ttyskeepawake        1
 displaysleep         10
 tcpkeepalive         1
 lowpowermode         0
 womp                 0

# Battery status
$ pmset -g batt
Now drawing from 'Battery Power'
 -InternalBattery-0 (id=12345678)    85%; discharging; 4:30 remaining present: true

# Power assertions (what's preventing sleep)
$ pmset -g assertions
2024-01-15 12:00:00 -0800
Assertion status system-wide:
   BackgroundTask                 1
   ApplePushServiceTask           0
   UserIsActive                   1
   PreventUserIdleDisplaySleep    1
   PreventSystemSleep             0
   ExternalMedia                  0
   PreventUserIdleSystemSleep     1
   NetworkClientActive            0
Listed by owning process:
   pid 12345(Safari): [0x0000123400000abc] 01:23:45 PreventUserIdleDisplaySleep named: "Playing Audio"
```

### Power Source Settings

Different settings for battery vs AC power:

```bash
# View settings by power source
$ pmset -g custom
Battery Power:
 displaysleep         2
 disksleep            10
 sleep                10
 womp                 0
 powernap             0
AC Power:
 displaysleep         10
 disksleep            10
 sleep                0
 womp                 1
 powernap             1
```

### Configuring Power Settings

```bash
# Set display sleep timeout (minutes)
$ sudo pmset -b displaysleep 5    # Battery: 5 minutes
$ sudo pmset -c displaysleep 15   # AC: 15 minutes
$ sudo pmset -a displaysleep 10   # All: 10 minutes

# Set system sleep timeout
$ sudo pmset -a sleep 30

# Disable sleep entirely
$ sudo pmset -a sleep 0
$ sudo pmset -a disablesleep 1

# Enable/disable wake on network (Wake on LAN)
$ sudo pmset -a womp 1  # Enable
$ sudo pmset -a womp 0  # Disable

# Power Nap settings
$ sudo pmset -a powernap 0  # Disable Power Nap
$ sudo pmset -a powernap 1  # Enable Power Nap
```

### Important pmset Keys

| Key | Description | Values |
|-----|-------------|--------|
| displaysleep | Display sleep timeout (min) | 0 = never, N = minutes |
| disksleep | Disk sleep timeout (min) | 0 = never, N = minutes |
| sleep | System sleep timeout (min) | 0 = never, N = minutes |
| womp | Wake on Magic Packet (LAN) | 0/1 |
| powernap | Power Nap enabled | 0/1 |
| hibernatemode | Hibernate mode | 0, 3, or 25 |
| standby | Standby mode | 0/1 |
| standbydelay | Delay before standby (sec) | seconds |
| autopoweroff | Auto power off | 0/1 |
| lowpowermode | Low Power Mode | 0/1 |
| tcpkeepalive | TCP keepalive during sleep | 0/1 |

### Hibernate Modes

```bash
# Check current hibernate mode
$ pmset -g | grep hibernatemode

# Hibernate modes:
# 0 = RAM stays powered, no hibernation (desktop default)
# 3 = RAM powered + hibernation file (laptop default, safe sleep)
# 25 = Pure hibernation, RAM powered off (maximum battery preservation)

# Set hibernate mode
$ sudo pmset -a hibernatemode 3
```

**Mode comparison:**

| Mode | RAM Power | Hibernate File | Wake Time | Battery Use |
|------|-----------|----------------|-----------|-------------|
| 0 | Always on | No | Instant | Higher |
| 3 | On until standby | Yes | Instant/Slow | Medium |
| 25 | Off | Yes | Slow | Minimal |

### Low Power Mode

Available on MacBooks with Apple Silicon and recent Intel:

```bash
# Check Low Power Mode
$ pmset -g | grep lowpowermode

# Enable Low Power Mode
$ sudo pmset -a lowpowermode 1

# Disable
$ sudo pmset -a lowpowermode 0
```

Low Power Mode effects:
- Reduces CPU performance
- Dims display
- Reduces background activity
- Extends battery life significantly

## caffeinate - Prevent Sleep

`caffeinate` prevents the system from sleeping.

### Basic Usage

```bash
# Prevent sleep while command runs
$ caffeinate -s make all

# Prevent sleep for duration (seconds)
$ caffeinate -t 3600  # 1 hour

# Prevent display sleep
$ caffeinate -d

# Prevent idle sleep
$ caffeinate -i

# Prevent disk sleep
$ caffeinate -m

# Prevent system sleep
$ caffeinate -s

# All of the above
$ caffeinate -dims
```

### caffeinate Options

| Option | Prevents | Use Case |
|--------|----------|----------|
| -d | Display sleep | Presentations |
| -i | Idle sleep | Long-running tasks |
| -m | Disk sleep | Large file operations |
| -s | System sleep | Downloads, backups |
| -u | Declare user active | Simulate user activity |
| -t N | All for N seconds | Time-limited prevention |
| -w PID | While PID runs | Follow a process |

### Practical Examples

```bash
# Keep awake during long compile
$ caffeinate -i make -j8 all

# Keep awake while download completes
$ caffeinate -s curl -O https://example.com/large-file.zip

# Keep awake while backup runs
$ caffeinate -s rsync -av /source /destination

# Keep display on during presentation (until Ctrl+C)
$ caffeinate -d

# Keep awake as long as another process runs
$ long_running_process &
$ caffeinate -w $!

# Script that manages its own caffeinate
#!/bin/bash
caffeinate -i -w $$ &
# ... long running work ...
# caffeinate automatically exits when script finishes
```

### Scripted Power Control

```bash
#!/bin/bash
# smart-caffeinate.sh - Caffeinate with status

DURATION=${1:-3600}

echo "Preventing sleep for $((DURATION / 60)) minutes"
echo "Press Ctrl+C to cancel"

caffeinate -dims -t $DURATION &
CAFF_PID=$!

trap "kill $CAFF_PID 2>/dev/null; echo 'Sleep prevention cancelled'" EXIT

while kill -0 $CAFF_PID 2>/dev/null; do
    REMAINING=$((DURATION - SECONDS))
    if [[ $REMAINING -lt 0 ]]; then
        break
    fi
    printf "\rRemaining: %02d:%02d" $((REMAINING / 60)) $((REMAINING % 60))
    sleep 1
done

echo -e "\nSleep prevention ended"
```

## Power Assertions

Power assertions are how applications communicate their power needs to the system.

### Viewing Assertions

```bash
# List all active assertions
$ pmset -g assertions

# Detailed assertion info
$ pmset -g assertionslog

# Who's preventing sleep?
$ pmset -g assertions | grep -A2 "PreventSystemSleep\|PreventUserIdleSystemSleep"
```

### Common Assertion Types

| Assertion | Effect |
|-----------|--------|
| PreventUserIdleSystemSleep | System won't idle sleep |
| PreventUserIdleDisplaySleep | Display won't idle sleep |
| PreventSystemSleep | System cannot sleep at all |
| NoIdleSleepAssertion | Legacy, same as PreventUserIdleSystemSleep |
| NoDisplaySleepAssertion | Legacy, same as PreventUserIdleDisplaySleep |

### Creating Assertions from Command Line

```bash
# caffeinate creates assertions internally

# To create specific assertions programmatically:
# Use IOKit APIs (requires code)
# Or use caffeinate with appropriate flags
```

### Finding Assertion Offenders

```bash
#!/bin/bash
# find-assertion-owners.sh - Find processes preventing sleep

echo "Processes with active power assertions:"
echo "======================================="

pmset -g assertions | grep -E "^\s+pid" | while read line; do
    pid=$(echo "$line" | awk -F'[()]' '{print $2}')
    name=$(echo "$line" | awk -F'[()]' '{print $1}' | awk '{print $2}')
    assertion=$(echo "$line" | awk -F'"' '{print $2}')

    echo "Process: $name (PID $pid)"
    echo "  Assertion: $assertion"
    echo ""
done
```

## App Nap

App Nap reduces resource usage for applications not actively being used.

### How App Nap Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    App Nap Conditions                            │
├─────────────────────────────────────────────────────────────────┤
│ App enters App Nap when:                                        │
│ 1. Window is not visible (minimized, behind other windows)     │
│ 2. Not playing audio                                            │
│ 3. No active power assertions                                   │
│ 4. Not explicitly disabled by app                               │
├─────────────────────────────────────────────────────────────────┤
│ App Nap effects:                                                │
│ - Timer coalescing (batched background work)                    │
│ - Reduced I/O priority                                          │
│ - Reduced CPU priority                                          │
│ - Paused or throttled network activity                          │
└─────────────────────────────────────────────────────────────────┘
```

### Checking App Nap Status

In Activity Monitor:
1. View > Columns > App Nap
2. "Yes" = App Nap active, "No" = App running normally

```bash
# Check if an app supports App Nap
$ defaults read /Applications/Safari.app/Contents/Info.plist NSAppSleepDisabled 2>/dev/null
# (no output or error = App Nap enabled)
# 1 = App Nap disabled
```

### Disabling App Nap for an App

```bash
# Disable App Nap for specific app
$ defaults write com.apple.Safari NSAppSleepDisabled -bool YES

# Via Finder:
# 1. Right-click app in Applications
# 2. Get Info
# 3. Check "Prevent App Nap"
```

## Power Monitoring

### Battery Information

```bash
# Battery status
$ pmset -g batt
Now drawing from 'Battery Power'
 -InternalBattery-0 (id=12345678)    85%; discharging; 4:30 remaining

# Detailed battery info
$ system_profiler SPPowerDataType

# Battery health (cycles, condition)
$ system_profiler SPPowerDataType | grep -E "Cycle Count|Condition|Maximum Capacity"
      Cycle Count: 234
      Condition: Normal
      Maximum Capacity: 92%

# ioreg for detailed battery data
$ ioreg -rn AppleSmartBattery | grep -E "Cycle|Capacity|Temperature"
```

### Power Consumption Monitoring

```bash
# Using powermetrics (requires sudo)
$ sudo powermetrics --samplers cpu_power,battery -n 1

# CPU power consumption
$ sudo powermetrics --samplers cpu_power -n 1 | grep "CPU Power"
CPU Power: 2850 mW

# Battery drain rate
$ sudo powermetrics --samplers battery -n 1 | grep "Amperage"
Amperage: -1234 mA

# Continuous monitoring
$ sudo powermetrics --samplers cpu_power,battery -i 5000 | grep -E "CPU Power|Battery Level"
```

### Energy Impact by Process

```bash
# Using top
$ top -o power

# Using powermetrics for detailed per-process
$ sudo powermetrics --samplers tasks -n 1 | head -40
```

### Power Monitoring Script

```bash
#!/bin/bash
# power-monitor.sh - Monitor power consumption

echo "Power Monitor - Press Ctrl+C to stop"
echo "Time,Battery%,Amperage(mA),CPU_Power(mW),Drawing_From"

while true; do
    # Get battery info
    batt_info=$(pmset -g batt)
    battery_pct=$(echo "$batt_info" | grep -o '[0-9]*%' | head -1 | tr -d '%')
    power_source=$(echo "$batt_info" | grep "drawing from" | awk -F"'" '{print $2}')

    # Get power metrics (needs sudo)
    power_data=$(sudo powermetrics --samplers cpu_power,battery -n 1 2>/dev/null)
    cpu_power=$(echo "$power_data" | grep "CPU Power:" | awk '{print $3}' | tr -d 'mW')
    amperage=$(echo "$power_data" | grep "Amperage:" | awk '{print $2}' | tr -d 'mA')

    echo "$(date +%H:%M:%S),$battery_pct,$amperage,$cpu_power,$power_source"

    sleep 30
done
```

## Optimizing Battery Life

### System Settings

```bash
# Maximize battery life settings
$ sudo pmset -b displaysleep 2      # Display off after 2 min on battery
$ sudo pmset -b sleep 5              # System sleep after 5 min
$ sudo pmset -b powernap 0           # Disable Power Nap on battery
$ sudo pmset -b lowpowermode 1       # Enable Low Power Mode
$ sudo pmset -b lessbright 1         # Slightly dim display on battery

# Aggressive standby
$ sudo pmset -b standby 1
$ sudo pmset -b standbydelay 60      # Enter standby after 1 min of sleep
$ sudo pmset -b hibernatemode 25     # Pure hibernation (slowest wake, best battery)
```

### Finding Battery Drains

```bash
#!/bin/bash
# find-battery-drains.sh - Identify battery draining activities

echo "=== Power Assertions ==="
pmset -g assertions | grep -E "pid|named"

echo ""
echo "=== High Energy Impact Processes ==="
echo "(Check Activity Monitor > Energy tab for more detail)"
ps aux | sort -nrk 3 | head -6

echo ""
echo "=== Apps Preventing Sleep ==="
pmset -g assertions | grep "PreventUserIdleSystemSleep\|PreventSystemSleep" -A2

echo ""
echo "=== Bluetooth Devices ==="
system_profiler SPBluetoothDataType 2>/dev/null | grep -E "Connected:|Name:" | head -10

echo ""
echo "=== Active Network Connections ==="
lsof -i | grep -E "ESTABLISHED|LISTEN" | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

### Quick Battery Tips

1. **Reduce screen brightness** - Major power consumer
2. **Disable Bluetooth/Wi-Fi** when not needed
3. **Close unused tabs** - Browser tabs use memory and CPU
4. **Enable Low Power Mode** - Significant battery extension
5. **Check Activity Monitor Energy tab** - Find specific drains
6. **Disable Power Nap on battery** - Prevents background wake
7. **Use native apps** - Rosetta 2 uses more power

## Thermal Management

macOS manages thermals automatically, but you can monitor:

```bash
# Thermal state
$ sudo powermetrics --samplers thermal -n 1
*** Thermal State ***
System Thermal Level: 0 (nominal)
CPU Thermal Level: 0 (nominal)
GPU Thermal Level: 0 (nominal)

# Fan speed (Intel Macs with SMC)
$ sudo powermetrics --samplers smc -n 1 | grep -i fan

# Or use third-party tools
$ brew install osx-cpu-temp
$ osx-cpu-temp  # Shows CPU temperature
```

### Thermal States

| Level | State | Behavior |
|-------|-------|----------|
| 0 | Nominal | Normal operation |
| 1 | Moderate | Slight throttling |
| 2-9 | Elevated | Increasing throttling |
| 10+ | Critical | Aggressive throttling, possible shutdown |

## Summary

Key power management commands:

| Command | Purpose | Example |
|---------|---------|---------|
| pmset -g | View settings | `pmset -g batt` |
| pmset -a | Set for all sources | `sudo pmset -a sleep 30` |
| pmset -b | Set for battery | `sudo pmset -b lowpowermode 1` |
| pmset -c | Set for AC power | `sudo pmset -c sleep 0` |
| caffeinate | Prevent sleep | `caffeinate -s make all` |
| powermetrics | Power analysis | `sudo powermetrics --samplers cpu_power` |

Power optimization checklist:

```bash
# Check current state
$ pmset -g batt              # Battery status
$ pmset -g assertions        # What's preventing sleep
$ pmset -g                   # Current power settings

# Common optimizations
$ sudo pmset -b lowpowermode 1      # Enable Low Power Mode
$ sudo pmset -b displaysleep 2      # Quick display off
$ sudo pmset -b powernap 0          # Disable Power Nap on battery
$ caffeinate -i long_task           # Prevent sleep during task

# Monitor power use
$ sudo powermetrics --samplers cpu_power,battery -n 1
```

Effective power management balances performance needs with battery life and energy efficiency.
