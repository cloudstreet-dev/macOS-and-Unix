# Network Diagnostics and Troubleshooting

Effective network troubleshooting requires a methodical approach and the right tools. macOS provides both traditional Unix diagnostic utilities and Apple-specific tools. This chapter covers the essential diagnostic commands and common troubleshooting scenarios.

## Diagnostic Tools Overview

```
Layer 7 (Application)    curl, openssl s_client, dns-sd
Layer 4 (Transport)      netstat, lsof, nc
Layer 3 (Network)        ping, traceroute, mtr, route
Layer 2 (Data Link)      arp, ndp
Layer 1 (Physical)       ifconfig, networkQuality

Cross-layer tools:       tcpdump, nettop, Wireshark
```

## ping: Basic Connectivity Test

The most fundamental network diagnostic tool.

### Basic Usage

```bash
# Ping a host (runs continuously, Ctrl+C to stop)
$ ping google.com
PING google.com (142.250.x.x): 56 data bytes
64 bytes from 142.250.x.x: icmp_seq=0 ttl=117 time=12.345 ms
64 bytes from 142.250.x.x: icmp_seq=1 ttl=117 time=11.234 ms
^C
--- google.com ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 11.234/11.789/12.345/0.556 ms

# Ping with count limit
$ ping -c 5 google.com

# Ping with specific interval (seconds)
$ ping -i 2 google.com

# Ping with timeout (seconds)
$ ping -t 5 google.com

# Ping with specific packet size
$ ping -s 1000 google.com
```

### Advanced ping Options

```bash
# Flood ping (requires root, for testing)
$ sudo ping -f google.com

# Quiet output (only summary)
$ ping -q -c 10 google.com

# Numeric output only (no DNS resolution)
$ ping -n google.com

# Ping with record route option
$ ping -R google.com

# Specify source interface
$ ping -S 192.168.1.100 google.com

# IPv6 ping
$ ping6 ipv6.google.com
```

### Interpreting ping Output

```
64 bytes from 142.250.x.x: icmp_seq=0 ttl=117 time=12.345 ms
│                          │          │        │
│                          │          │        └── Round-trip time
│                          │          └── Time-to-live (hops remaining)
│                          └── Sequence number
└── Response size
```

**Common issues indicated by ping:**
- **Request timeout**: Host unreachable or filtering ICMP
- **High latency**: Network congestion or routing issues
- **Variable latency**: Network instability
- **Packet loss**: Network problems or congestion

## traceroute: Path Discovery

Traces the route packets take to reach a destination.

### Basic Usage

```bash
# Trace route to host
$ traceroute google.com
traceroute to google.com (142.250.x.x), 64 hops max, 52 byte packets
 1  192.168.1.1 (192.168.1.1)  1.234 ms  1.123 ms  1.012 ms
 2  isp-gateway (10.0.0.1)  5.678 ms  5.567 ms  5.456 ms
 3  core-router.isp.net (203.0.113.1)  10.123 ms  10.012 ms  9.901 ms
 4  * * *
 5  google-peering (74.125.x.x)  12.345 ms  12.234 ms  12.123 ms
 6  142.250.x.x (142.250.x.x)  12.456 ms  12.345 ms  12.234 ms
```

### traceroute Options

```bash
# Use ICMP instead of UDP (often more reliable)
$ sudo traceroute -I google.com

# Use TCP SYN packets (useful when ICMP blocked)
$ sudo traceroute -T -p 443 google.com

# Set maximum hops
$ traceroute -m 20 google.com

# Don't resolve hostnames (faster)
$ traceroute -n google.com

# Set packet size
$ traceroute -q 3 google.com

# Wait time for response (seconds)
$ traceroute -w 3 google.com

# Specify source interface
$ traceroute -s 192.168.1.100 google.com
```

### Interpreting traceroute Output

```
 3  core-router.isp.net (203.0.113.1)  10.123 ms  10.012 ms  9.901 ms
 │  │                     │             │          │          │
 │  │                     │             └──────────┴──────────┴── Three probe times
 │  │                     └── IP address
 │  └── Hostname (if resolvable)
 └── Hop number
```

- **`* * *`**: No response (could be filtering or timeout)
- **Increasing latency**: Normal as distance increases
- **Sudden latency jump**: Possible congestion point
- **Asymmetric routes**: Different paths for request and response

## mtr: Combined ping/traceroute

`mtr` provides real-time statistics combining ping and traceroute. Install via Homebrew:

```bash
$ brew install mtr
```

### Using mtr

```bash
# Basic mtr (requires sudo for raw sockets)
$ sudo mtr google.com

# Report mode (non-interactive)
$ mtr -r -c 10 google.com
Start: 2024-01-15T10:00:00+0000
HOST: my-mac                      Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.1.1               0.0%    10    1.2   1.3   1.0   1.8   0.2
  2.|-- isp-gateway               0.0%    10    5.6   5.7   5.4   6.2   0.3
  3.|-- core-router.isp.net       0.0%    10   10.1  10.2   9.8  10.8   0.3
  4.|-- google-peering            0.0%    10   12.3  12.4  12.1  12.9   0.2
  5.|-- 142.250.x.x               0.0%    10   12.4  12.5  12.2  13.0   0.2

# Wide report (shows both hostnames and IPs)
$ mtr -rw -c 10 google.com

# Use TCP instead of ICMP
$ sudo mtr -T -P 443 google.com

# Use UDP
$ sudo mtr -u google.com

# No DNS resolution
$ mtr -n google.com
```

### mtr Statistics Explained

| Column | Meaning |
|--------|---------|
| Loss% | Packet loss percentage |
| Snt | Packets sent |
| Last | Last probe time |
| Avg | Average latency |
| Best | Best (lowest) latency |
| Wrst | Worst (highest) latency |
| StDev | Standard deviation |

## nettop: Real-time Network Activity

`nettop` shows real-time network usage by process.

### Basic Usage

```bash
# Show all network activity
$ nettop

# Show TCP connections only
$ nettop -m tcp

# Show UDP connections only
$ nettop -m udp

# Show specific process
$ nettop -p Safari

# Non-interactive mode with updates
$ nettop -P -L 1
```

### nettop Interface

```
                       bytes      bytes     packets    packets
process               in         out       in          out
Safari                12.3 MB    456 KB    8765        2345
Google Chrome         8.7 MB     234 KB    5678        1234
Spotify               5.4 MB     45 KB     3456        456
mDNSResponder         123 KB     23 KB     234         89
```

### Filtering nettop Output

```bash
# Show only established connections
$ nettop -m tcp -t state=ESTABLISHED

# Show only external connections
$ nettop -m tcp -t wifi

# Show connections to specific port
$ nettop -m tcp -j port=443
```

## tcpdump: Packet Capture

`tcpdump` captures and analyzes network traffic. Requires root privileges.

### Basic Capture

```bash
# Capture on interface en0
$ sudo tcpdump -i en0

# Capture with more detail
$ sudo tcpdump -i en0 -v

# Even more detail
$ sudo tcpdump -i en0 -vv

# Capture specific count
$ sudo tcpdump -i en0 -c 100

# Capture all interfaces
$ sudo tcpdump -i any
```

### Filtering Traffic

```bash
# Capture traffic to/from host
$ sudo tcpdump -i en0 host 192.168.1.50

# Capture traffic on specific port
$ sudo tcpdump -i en0 port 80
$ sudo tcpdump -i en0 port 443

# Capture TCP only
$ sudo tcpdump -i en0 tcp

# Capture UDP only
$ sudo tcpdump -i en0 udp

# Capture ICMP (ping)
$ sudo tcpdump -i en0 icmp

# Complex filters
$ sudo tcpdump -i en0 'tcp port 80 and host 192.168.1.50'
$ sudo tcpdump -i en0 'src 192.168.1.50 and dst port 443'
$ sudo tcpdump -i en0 'tcp[tcpflags] & (tcp-syn) != 0'
```

### Saving and Reading Captures

```bash
# Save to file
$ sudo tcpdump -i en0 -w capture.pcap

# Read from file
$ tcpdump -r capture.pcap

# Read with filter
$ tcpdump -r capture.pcap port 443

# Capture with rotation (10 files, 100MB each)
$ sudo tcpdump -i en0 -w capture.pcap -C 100 -W 10
```

### Displaying Packet Contents

```bash
# Show packet contents in hex and ASCII
$ sudo tcpdump -i en0 -X

# Show packet contents in hex only
$ sudo tcpdump -i en0 -x

# Show link-layer header
$ sudo tcpdump -i en0 -e

# Show absolute timestamps
$ sudo tcpdump -i en0 -tt
```

### HTTP Traffic Analysis

```bash
# Capture HTTP requests
$ sudo tcpdump -i en0 -A 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# Capture only HTTP GET requests
$ sudo tcpdump -i en0 -A 'tcp port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

## netstat: Network Statistics

`netstat` shows network connections, routing tables, and statistics.

### Connection Information

```bash
# Show all connections
$ netstat -an

# Show TCP connections
$ netstat -an -p tcp

# Show UDP connections
$ netstat -an -p udp

# Show listening ports
$ netstat -an | grep LISTEN

# Show established connections
$ netstat -an | grep ESTABLISHED
```

### Routing Information

```bash
# Show routing table
$ netstat -rn
Routing tables

Internet:
Destination        Gateway            Flags        Netif Expire
default            192.168.1.1        UGScg          en0
127.0.0.1          127.0.0.1          UH             lo0
192.168.1          link#8             UCS            en0      !
192.168.1.1/32     link#8             UCS            en0      !
```

### Statistics

```bash
# Show all protocol statistics
$ netstat -s

# Show TCP statistics
$ netstat -s -p tcp

# Show interface statistics
$ netstat -i
Name  Mtu   Network       Address            Ipkts Ierrs    Opkts Oerrs  Coll
lo0   16384 <Link#1>                        123456     0   123456     0     0
en0   1500  <Link#8>    aa:bb:cc:dd:ee:ff  987654     0   654321     0     0
```

## lsof: List Open Files (Network)

`lsof` can show network connections by process.

```bash
# Show all network connections
$ lsof -i

# Show listening ports
$ lsof -i -P | grep LISTEN

# Show connections on specific port
$ lsof -i :443

# Show connections by process
$ lsof -i -c Safari

# Show connections for specific protocol
$ lsof -i TCP
$ lsof -i UDP

# Show connections to specific host
$ lsof -i @google.com

# Find process using a port
$ lsof -i :8080 -P
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
node    1234 user   23u  IPv4 0x1234      0t0  TCP *:8080 (LISTEN)
```

## Network Quality Testing

### networkQuality (macOS 12+)

```bash
# Run network quality test
$ networkQuality
==== SUMMARY ====
Upload capacity: 50.123 Mbps
Download capacity: 200.456 Mbps
Upload flows: 12
Download flows: 16
Responsiveness: High (1234 RPM)

# Sequential test (more accurate)
$ networkQuality -s

# Verbose output
$ networkQuality -v

# JSON output
$ networkQuality -c -f json
```

### Manual Speed Test

```bash
# Download speed test
$ curl -o /dev/null -w "Speed: %{speed_download} bytes/sec\n" https://speed.cloudflare.com/__down?bytes=100000000

# Upload speed test
$ curl -X POST -o /dev/null -w "Speed: %{speed_upload} bytes/sec\n" \
    -d @/dev/zero --data-binary @<(dd if=/dev/zero bs=1M count=10 2>/dev/null) \
    https://speed.cloudflare.com/__up
```

## DNS Diagnostics

### dig

```bash
# Basic lookup
$ dig google.com

# Query specific record type
$ dig MX google.com
$ dig NS google.com
$ dig AAAA google.com    # IPv6

# Query specific DNS server
$ dig @8.8.8.8 google.com

# Trace DNS resolution
$ dig +trace google.com

# Short output
$ dig +short google.com

# Reverse lookup
$ dig -x 8.8.8.8
```

### DNS Cache

```bash
# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder

# Query local cache
$ dscacheutil -q host -a name google.com

# View DNS configuration
$ scutil --dns
```

## ARP and Neighbor Discovery

### arp (IPv4)

```bash
# Show ARP table
$ arp -a
? (192.168.1.1) at aa:bb:cc:dd:ee:ff on en0 ifscope [ethernet]
? (192.168.1.100) at 11:22:33:44:55:66 on en0 ifscope permanent [ethernet]

# Delete ARP entry
$ sudo arp -d 192.168.1.50

# Add static ARP entry
$ sudo arp -s 192.168.1.50 aa:bb:cc:dd:ee:ff
```

### ndp (IPv6)

```bash
# Show NDP table
$ ndp -an

# Delete NDP entry
$ sudo ndp -d fe80::1

# Show router advertisements
$ ndp -r
```

## Common Troubleshooting Scenarios

### "No Internet Connection"

```bash
# 1. Check physical/Wi-Fi connection
$ ifconfig en0 | grep status
	status: active

# 2. Check IP address
$ ifconfig en0 | grep "inet "
	inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255

# 3. Check gateway
$ route -n get default | grep gateway
    gateway: 192.168.1.1

# 4. Ping gateway
$ ping -c 3 192.168.1.1

# 5. Ping external IP (bypasses DNS)
$ ping -c 3 8.8.8.8

# 6. Test DNS
$ dig google.com
$ nslookup google.com

# 7. If DNS fails, flush cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder
```

### "Slow Network"

```bash
# 1. Test network quality
$ networkQuality -s

# 2. Check for packet loss
$ ping -c 100 google.com | grep "packet loss"

# 3. Check what's using bandwidth
$ nettop -P

# 4. Check for network congestion (increasing latency at specific hop)
$ mtr -r -c 20 google.com

# 5. Check interface errors
$ netstat -i | grep en0
```

### "Can't Connect to Specific Service"

```bash
# 1. Test DNS resolution
$ dig service.example.com

# 2. Test port connectivity
$ nc -zv service.example.com 443
Connection to service.example.com port 443 [tcp/https] succeeded!

# 3. Test with curl (HTTP/HTTPS)
$ curl -v https://service.example.com

# 4. Check if local firewall is blocking
$ sudo pfctl -s rules | grep 443

# 5. Trace route to identify where traffic stops
$ traceroute service.example.com
```

### "Intermittent Connection Drops"

```bash
# 1. Monitor connection over time
$ ping -c 1000 gateway_ip > ping_log.txt 2>&1

# 2. Check Wi-Fi signal strength
$ /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I | grep agrCtlRSSI
     agrCtlRSSI: -52

# 3. Monitor for channel interference
$ sudo tcpdump -i en0 -c 1000

# 4. Check system logs for network issues
$ log show --predicate 'subsystem == "com.apple.network"' --last 1h | grep -i error
```

## Diagnostic Script

A comprehensive diagnostic script:

```bash
#!/bin/bash
# network-diagnostics.sh

echo "=== Network Diagnostics Report ==="
echo "Date: $(date)"
echo

echo "=== Interface Status ==="
ifconfig en0 | grep -E "status|inet "

echo -e "\n=== Gateway ==="
route -n get default | grep gateway

echo -e "\n=== DNS Servers ==="
scutil --dns | grep nameserver | head -5

echo -e "\n=== Gateway Ping ==="
ping -c 3 $(route -n get default | grep gateway | awk '{print $2}')

echo -e "\n=== External Ping (8.8.8.8) ==="
ping -c 3 8.8.8.8

echo -e "\n=== DNS Test ==="
dig +short google.com

echo -e "\n=== Traceroute (first 10 hops) ==="
traceroute -m 10 8.8.8.8

echo -e "\n=== Active Connections ==="
netstat -an | grep ESTABLISHED | head -10

echo -e "\n=== Listening Ports ==="
lsof -i -P | grep LISTEN | head -10

echo -e "\n=== Network Quality (if available) ==="
networkQuality -s 2>/dev/null || echo "networkQuality not available"
```

## Summary

| Tool | Purpose | Common Usage |
|------|---------|--------------|
| `ping` | Test connectivity | `ping -c 5 host` |
| `traceroute` | Trace packet path | `traceroute host` |
| `mtr` | Real-time trace | `mtr -r host` |
| `nettop` | Live network monitor | `nettop -m tcp` |
| `tcpdump` | Packet capture | `sudo tcpdump -i en0` |
| `netstat` | Connection stats | `netstat -an` |
| `lsof -i` | Network by process | `lsof -i :port` |
| `networkQuality` | Speed test | `networkQuality -s` |
| `dig` | DNS lookup | `dig domain` |
| `nc` | Test port | `nc -zv host port` |

Key troubleshooting steps:

1. **Verify physical/Wi-Fi connection** (`ifconfig`)
2. **Check IP configuration** (`ipconfig`, `ifconfig`)
3. **Test gateway connectivity** (`ping gateway`)
4. **Test external connectivity** (`ping 8.8.8.8`)
5. **Test DNS resolution** (`dig`, `nslookup`)
6. **Trace the path** (`traceroute`, `mtr`)
7. **Capture packets** if needed (`tcpdump`)
