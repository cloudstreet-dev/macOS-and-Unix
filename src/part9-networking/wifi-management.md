# Wi-Fi Management from Terminal

macOS provides several command-line tools for managing Wi-Fi connections, from the well-known `networksetup` to the hidden but powerful `airport` utility. This chapter covers scanning networks, connecting, diagnosing issues, and managing Wi-Fi configurations from Terminal.

## Wi-Fi Tools Overview

| Tool | Purpose | Location |
|------|---------|----------|
| `networksetup` | High-level Wi-Fi configuration | `/usr/sbin/networksetup` |
| `airport` | Low-level Wi-Fi diagnostics | Hidden in framework |
| `wdutil` | Wireless diagnostics | `/usr/bin/wdutil` |
| `system_profiler` | Hardware information | `/usr/sbin/system_profiler` |

## The airport Utility

The `airport` command is hidden within a system framework but provides detailed Wi-Fi information and control.

### Setting Up airport

```bash
# Create an alias for easy access
$ alias airport='/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport'

# Or create a symbolic link
$ sudo ln -s /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport /usr/local/bin/airport

# Add to your shell profile (~/.zshrc) for persistence
echo "alias airport='/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport'" >> ~/.zshrc
```

### Current Connection Information

```bash
# Show current Wi-Fi connection details
$ airport -I
     agrCtlRSSI: -52
     agrExtRSSI: 0
    agrCtlNoise: -88
    agrExtNoise: 0
          state: running
        op mode: station
     lastTxRate: 866
        maxRate: 1200
lastAssocStatus: 0
    802.11 auth: open
      link auth: wpa2-psk
          BSSID: aa:bb:cc:dd:ee:ff
           SSID: MyNetwork
            MCS: 9
  guardInterval: 800
            NSS: 2
        channel: 149,80
```

### Understanding airport Output

| Field | Description |
|-------|-------------|
| `agrCtlRSSI` | Signal strength in dBm (higher is better, -50 is excellent, -80 is poor) |
| `agrCtlNoise` | Noise floor in dBm (lower is better) |
| `state` | Connection state (running, init, etc.) |
| `lastTxRate` | Current transmit rate in Mbps |
| `maxRate` | Maximum possible rate |
| `link auth` | Authentication type (wpa2-psk, wpa3, etc.) |
| `BSSID` | MAC address of the access point |
| `SSID` | Network name |
| `channel` | Channel number (and width for 802.11ac/ax) |
| `NSS` | Number of spatial streams |

### Signal Quality Assessment

```bash
# Quick signal strength check
$ airport -I | grep agrCtlRSSI
     agrCtlRSSI: -52

# Signal-to-Noise Ratio (SNR)
$ airport -I | grep -E "agrCtlRSSI|agrCtlNoise"
     agrCtlRSSI: -52
    agrCtlNoise: -88
# SNR = RSSI - Noise = -52 - (-88) = 36 dB (good)
```

Signal strength guidelines:

| RSSI (dBm) | Quality | Typical Use |
|------------|---------|-------------|
| -30 to -50 | Excellent | Any application |
| -50 to -60 | Good | Reliable for most uses |
| -60 to -70 | Fair | Web browsing, email |
| -70 to -80 | Weak | May have issues |
| -80 to -90 | Very Weak | Likely unreliable |

### Scanning for Networks

```bash
# Scan for available Wi-Fi networks
$ airport -s
                            SSID BSSID             RSSI CHANNEL HT CC SECURITY
                       MyNetwork aa:bb:cc:dd:ee:ff -52  149,+1  Y  US WPA2(PSK/AES/AES)
                   Neighbor_WiFi bb:cc:dd:ee:ff:00 -68  6       Y  US WPA2(PSK/AES/AES)
                     Guest_Wifi  cc:dd:ee:ff:00:11 -75  11      Y  -- WPA2(PSK/AES/AES)
                   HiddenNetwork dd:ee:ff:00:11:22 -80  36,+1   Y  US WPA3(SAE/AES/AES)
                        OpenWiFi ee:ff:00:11:22:33 -62  1       N  -- NONE

# Scan with more details (XML output)
$ airport -s -x
```

### Understanding Scan Output

| Column | Description |
|--------|-------------|
| SSID | Network name |
| BSSID | Access point MAC address |
| RSSI | Signal strength in dBm |
| CHANNEL | Channel (with width indicator for AC/AX) |
| HT | High Throughput (802.11n+) capable |
| CC | Country code |
| SECURITY | Security protocol |

Channel notation:
- `6` - Channel 6, 20MHz width
- `149,+1` - Channel 149, 40MHz bonded with upper channel
- `36,-1` - Channel 36, 40MHz bonded with lower channel
- `149,80` - Channel 149, 80MHz width

### Disconnecting from Wi-Fi

```bash
# Disconnect from current network
$ sudo airport -z

# Verify disconnection
$ airport -I | grep SSID
# (no output means disconnected)
```

### Supported Channels

```bash
# Show supported channels for your Wi-Fi adapter
$ airport -c
Supported channels:
1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 36, 40, 44, 48, 52, 56, 60, 64, 100, 104, 108, 112, 116, 120, 124, 128, 132, 136, 140, 144, 149, 153, 157, 161, 165
```

## networksetup for Wi-Fi

`networksetup` provides high-level Wi-Fi management.

### Wi-Fi Power Control

```bash
# Check Wi-Fi power state
$ networksetup -getairportpower en0
Wi-Fi Power (en0): On

# Turn Wi-Fi off
$ networksetup -setairportpower en0 off

# Turn Wi-Fi on
$ networksetup -setairportpower en0 on
```

### Connecting to Networks

```bash
# Connect to a network (with password)
$ networksetup -setairportnetwork en0 "NetworkName" "password"

# Connect to open network
$ networksetup -setairportnetwork en0 "OpenNetwork"

# Get current network
$ networksetup -getairportnetwork en0
Current Wi-Fi Network: MyNetwork
```

### Managing Preferred Networks

Preferred networks are automatically connected to when in range:

```bash
# List preferred (known) networks
$ networksetup -listpreferredwirelessnetworks en0
Preferred networks on en0:
	MyHomeNetwork
	OfficeNetwork
	CoffeeShopWiFi

# Add a preferred network
$ networksetup -addpreferredwirelessnetworkatindex en0 "NewNetwork" 0 WPA2 "password"
#                                                   interface SSID    index security password

# Remove a preferred network
$ sudo networksetup -removepreferredwirelessnetwork en0 "CoffeeShopWiFi"
Removed CoffeeShopWiFi from the preferred networks list

# Remove all preferred networks
$ sudo networksetup -removeallpreferredwirelessnetworks en0
```

### Preferred Network Order

```bash
# Networks are tried in order; index 0 has highest priority
# Add network at index 0 (highest priority)
$ networksetup -addpreferredwirelessnetworkatindex en0 "PriorityNetwork" 0 WPA2 "password"
```

## wdutil: Wireless Diagnostics

`wdutil` provides additional diagnostic capabilities:

```bash
# Show wireless diagnostics info
$ sudo wdutil info

# Capture wireless diagnostic information
$ sudo wdutil diagnose

# Log wireless events
$ sudo wdutil log

# Note: wdutil requires sudo for most operations
```

The `diagnose` command creates a diagnostic report that can be useful for troubleshooting.

## System Information

### system_profiler

```bash
# Detailed Wi-Fi hardware info
$ system_profiler SPAirPortDataType
Wi-Fi:

      Software Versions:
          CoreWLAN: 16.0 (1657)
          CoreWLANKit: 16.0 (1657)
          Menu Extra: 17.0 (1728)
          IO80211 Family: 12.0 (1200.12.2)
          Diagnostics: 11.0 (1163)
          AirPort Utility: 6.3.9 (639.15)
      Interfaces:
        en0:
          Card Type: Wi-Fi  (0x14E4, 0x4387)
          Firmware Version: wl0: Jul 12 2023 02:42:47 version 20.10.1062.3.8.7.158 FWID 01-...
          Locale: ETSI
          Country Code: US
          Supported PHY Modes: 802.11 a/b/g/n/ac/ax
          Supported Channels: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 36, 40, 44, 48, 52, 56, 60, 64, 100, 104, 108, 112, 116, 120, 124, 128, 132, 136, 140, 144, 149, 153, 157, 161, 165
          Status: Connected
          Current Network Information:
            MyNetwork:
              PHY Mode: 802.11ax
              Channel: 149 (5 GHz, 80 MHz)
              Network Type: Infrastructure
              Security: WPA2 Personal
              Signal / Noise: -52 dBm / -88 dBm
              Transmit Rate: 1201
              MCS Index: 11
```

## Wi-Fi Diagnostics and Troubleshooting

### Check Wi-Fi Status

```bash
#!/bin/bash
# wifi-status.sh - Comprehensive Wi-Fi status

echo "=== Wi-Fi Status ==="

# Check if Wi-Fi is on
power=$(networksetup -getairportpower en0 | awk '{print $4}')
echo "Power: $power"

if [ "$power" = "On" ]; then
    # Get current network
    network=$(networksetup -getairportnetwork en0 | cut -d: -f2 | xargs)
    echo "Network: $network"

    # Get signal info
    airport -I | grep -E "agrCtlRSSI|agrCtlNoise|lastTxRate|channel"

    # Calculate SNR
    rssi=$(airport -I | grep agrCtlRSSI | awk '{print $2}')
    noise=$(airport -I | grep agrCtlNoise | awk '{print $2}')
    snr=$((rssi - noise))
    echo "Signal-to-Noise Ratio: $snr dB"
fi
```

### Monitor Wi-Fi Signal

```bash
# Continuous signal monitoring
$ while true; do
    airport -I | grep agrCtlRSSI
    sleep 1
done

# Or with timestamp
$ while true; do
    echo "$(date +%H:%M:%S) - $(airport -I | grep agrCtlRSSI | awk '{print $2}') dBm"
    sleep 1
done
```

### Find Best Channel

```bash
#!/bin/bash
# best-channel.sh - Find least congested Wi-Fi channels

echo "=== Channel Usage Analysis ==="
echo "Scanning networks..."

# Count networks per channel
airport -s | tail -n +2 | awk '{print $4}' | cut -d, -f1 | sort | uniq -c | sort -rn

echo -e "\n2.4 GHz Channels (1, 6, 11 recommended):"
airport -s | tail -n +2 | awk '$4 <= 14 {print $4}' | sort | uniq -c

echo -e "\n5 GHz Channels:"
airport -s | tail -n +2 | awk '$4 > 14 {print $4}' | cut -d, -f1 | sort | uniq -c
```

### Troubleshooting Connection Issues

```bash
# 1. Check if Wi-Fi hardware is available
$ networksetup -listallhardwareports | grep -A1 "Wi-Fi"
Hardware Port: Wi-Fi
Device: en0

# 2. Check if interface is up
$ ifconfig en0 | grep status
	status: active

# 3. Check signal strength
$ airport -I | grep agrCtlRSSI
     agrCtlRSSI: -52

# 4. Check for IP address
$ ifconfig en0 | grep "inet "
	inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255

# 5. If no IP, try renewing DHCP
$ sudo ipconfig set en0 DHCP

# 6. Check DNS
$ scutil --dns | grep "nameserver"

# 7. Test connectivity
$ ping -c 3 8.8.8.8
```

### Reset Wi-Fi Configuration

When all else fails:

```bash
# Turn Wi-Fi off and on
$ networksetup -setairportpower en0 off
$ sleep 2
$ networksetup -setairportpower en0 on

# Remove and reconnect to network
$ sudo networksetup -removepreferredwirelessnetwork en0 "ProblemNetwork"
$ networksetup -setairportnetwork en0 "ProblemNetwork" "password"

# Full Wi-Fi reset (removes all preferences)
$ sudo rm -f /Library/Preferences/SystemConfiguration/com.apple.airport.preferences.plist
$ sudo rm -f /Library/Preferences/com.apple.wifi.message-tracer.plist
# Then reboot
```

## Captive Portal Networks

Captive portals (hotel, coffee shop Wi-Fi) sometimes cause issues:

```bash
# Force captive portal check
$ /System/Library/CoreServices/Captive\ Network\ Assistant.app/Contents/MacOS/Captive\ Network\ Assistant

# Or manually open the captive portal
$ open http://captive.apple.com
```

## Wi-Fi Logging

### Enable Wi-Fi Debug Logging

```bash
# Enable debug logging
$ sudo defaults write /Library/Preferences/com.apple.wifi.message-tracer.plist LogLevel -int 5

# View Wi-Fi logs
$ log show --predicate 'subsystem == "com.apple.wifi"' --last 5m

# Stream Wi-Fi logs in real-time
$ log stream --predicate 'subsystem == "com.apple.wifi"' --level debug

# Disable debug logging when done
$ sudo defaults delete /Library/Preferences/com.apple.wifi.message-tracer.plist LogLevel
```

### Wireless Diagnostics Report

```bash
# Generate comprehensive wireless diagnostic report
$ sudo wdutil diagnose

# Report is saved to /var/tmp/
# Look for WirelessDiagnostics*.tar.gz
```

## Automation Scripts

### Auto-Connect Script

```bash
#!/bin/bash
# auto-connect.sh - Connect to first available preferred network

INTERFACE="en0"
PREFERRED_NETWORKS=("HomeNetwork" "OfficeNetwork" "BackupNetwork")
PASSWORDS=("homepass" "officepass" "backuppass")

# Check if already connected
current=$(networksetup -getairportnetwork $INTERFACE | cut -d: -f2 | xargs)
if [ -n "$current" ]; then
    echo "Already connected to: $current"
    exit 0
fi

# Scan and connect
echo "Scanning for networks..."
available=$(airport -s | tail -n +2 | awk '{print $1}')

for i in "${!PREFERRED_NETWORKS[@]}"; do
    network="${PREFERRED_NETWORKS[$i]}"
    if echo "$available" | grep -q "^${network}$"; then
        echo "Connecting to $network..."
        networksetup -setairportnetwork $INTERFACE "$network" "${PASSWORDS[$i]}"
        sleep 3
        if networksetup -getairportnetwork $INTERFACE | grep -q "$network"; then
            echo "Connected successfully!"
            exit 0
        fi
    fi
done

echo "No preferred networks available"
exit 1
```

### Signal Monitor Script

```bash
#!/bin/bash
# signal-monitor.sh - Monitor and log Wi-Fi signal over time

LOGFILE="/tmp/wifi-signal.log"
INTERVAL=5

echo "Monitoring Wi-Fi signal (logging to $LOGFILE)..."
echo "Timestamp,SSID,RSSI,Noise,SNR,TxRate,Channel" > $LOGFILE

while true; do
    info=$(airport -I)
    ssid=$(echo "$info" | grep " SSID:" | awk '{print $2}')
    rssi=$(echo "$info" | grep "agrCtlRSSI:" | awk '{print $2}')
    noise=$(echo "$info" | grep "agrCtlNoise:" | awk '{print $2}')
    txrate=$(echo "$info" | grep "lastTxRate:" | awk '{print $2}')
    channel=$(echo "$info" | grep "channel:" | awk '{print $2}')
    snr=$((rssi - noise))

    timestamp=$(date "+%Y-%m-%d %H:%M:%S")
    echo "$timestamp,$ssid,$rssi,$noise,$snr,$txrate,$channel" >> $LOGFILE
    echo "$timestamp - SSID: $ssid, RSSI: $rssi dBm, SNR: $snr dB, Rate: $txrate Mbps"

    sleep $INTERVAL
done
```

### Network Switcher

```bash
#!/bin/bash
# wifi-switch.sh - Quick network switching

case "$1" in
    home)
        networksetup -setairportnetwork en0 "HomeNetwork" "homepassword"
        ;;
    office)
        networksetup -setairportnetwork en0 "OfficeNetwork" "officepassword"
        ;;
    hotspot)
        networksetup -setairportnetwork en0 "iPhone" "hotspotpassword"
        ;;
    off)
        networksetup -setairportpower en0 off
        ;;
    on)
        networksetup -setairportpower en0 on
        ;;
    scan)
        airport -s
        ;;
    status)
        airport -I
        ;;
    *)
        echo "Usage: $0 {home|office|hotspot|off|on|scan|status}"
        exit 1
        ;;
esac
```

## Summary

| Task | Command |
|------|---------|
| Turn Wi-Fi on/off | `networksetup -setairportpower en0 on/off` |
| Current connection info | `airport -I` |
| Scan for networks | `airport -s` |
| Connect to network | `networksetup -setairportnetwork en0 "SSID" "password"` |
| Disconnect | `sudo airport -z` |
| Current network | `networksetup -getairportnetwork en0` |
| List preferred networks | `networksetup -listpreferredwirelessnetworks en0` |
| Remove preferred network | `networksetup -removepreferredwirelessnetwork en0 "SSID"` |
| Check signal strength | `airport -I \| grep agrCtlRSSI` |
| Show supported channels | `airport -c` |
| Wi-Fi hardware info | `system_profiler SPAirPortDataType` |

Key points:

- **`airport`** is the hidden power tool for Wi-Fi diagnostics
- **`networksetup`** handles high-level configuration and connections
- **Signal strength** (RSSI) and **Signal-to-Noise Ratio** (SNR) are key metrics
- **Preferred networks** determine automatic connection priority
- **Debug logging** can be enabled for troubleshooting
- **Channel analysis** helps optimize Wi-Fi performance
