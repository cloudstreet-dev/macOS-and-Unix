# Bonjour and mDNS

Bonjour is Apple's implementation of zero-configuration networking (zeroconf), enabling automatic discovery of devices and services on a local network without manual configuration. It's based on multicast DNS (mDNS) and DNS Service Discovery (DNS-SD), both open standards.

## How Bonjour Works

```
┌─────────────────────────────────────────────────────────────┐
│                    Your Mac                                  │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐      │
│  │   Safari    │   │   Finder    │   │  AirDrop    │      │
│  │ (Printers)  │   │ (Servers)   │   │             │      │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘      │
│         │                 │                 │              │
│         └────────────────┬┴─────────────────┘              │
│                          │                                  │
│                 ┌────────┴────────┐                        │
│                 │  mDNSResponder  │                        │
│                 │    (daemon)     │                        │
│                 └────────┬────────┘                        │
└──────────────────────────┼──────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            │  Multicast UDP port 5353    │
            │  224.0.0.251 (IPv4)         │
            │  ff02::fb (IPv6)            │
            └──────────────┬──────────────┘
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      │                      │
┌───┴───┐            ┌────┴────┐           ┌────┴────┐
│Printer│            │Apple TV │           │  NAS    │
│       │            │         │           │         │
└───────┘            └─────────┘           └─────────┘
```

### Key Components

**mDNS (Multicast DNS)**: Resolves hostnames to IP addresses on the local network without a DNS server. Hostnames end in `.local`.

**DNS-SD (DNS Service Discovery)**: Discovers services on the network. Services are advertised with a type (like `_http._tcp`) and can include metadata.

**mDNSResponder**: The macOS daemon that handles all mDNS and DNS-SD operations. It also functions as the system's DNS resolver.

## The .local Domain

Any hostname ending in `.local` is resolved via mDNS, not traditional DNS:

```bash
# These use mDNS
$ ping my-macbook.local
$ ssh server.local
$ open http://nas.local

# Your Mac's .local name
$ scutil --get LocalHostName
My-MacBook-Pro
# This makes your Mac reachable as my-macbook-pro.local
```

### Setting Your .local Hostname

```bash
# View current Bonjour hostname
$ scutil --get LocalHostName
My-MacBook-Pro

# Change it
$ sudo scutil --set LocalHostName "work-laptop"
# Now reachable as work-laptop.local

# Verify
$ dns-sd -G v4 work-laptop.local
```

## The dns-sd Command

`dns-sd` is the command-line tool for DNS Service Discovery. It can browse, register, and query services.

### Browsing Services

```bash
# Browse for HTTP servers
$ dns-sd -B _http._tcp local.
Browsing for _http._tcp.local.
DATE: ---Mon 15 Jan 2024---
14:23:45.123  ...STARTING...
14:23:45.234  Add        2   8 local.               _http._tcp.          My Web Server
14:23:45.345  Add        3   8 local.               _http._tcp.          Router Admin

# Browse for SSH servers
$ dns-sd -B _ssh._tcp local.
Browsing for _ssh._tcp.local.
14:24:12.123  Add        2   8 local.               _ssh._tcp.           My-MacBook-Pro
14:24:12.234  Add        3   8 local.               server

# Browse for printers
$ dns-sd -B _ipp._tcp local.
$ dns-sd -B _printer._tcp local.

# Browse for AirPlay devices
$ dns-sd -B _airplay._tcp local.
14:25:00.123  Add        2   8 local.               _airplay._tcp.       Living Room Apple TV
14:25:00.234  Add        3   8 local.               _airplay._tcp.       Bedroom HomePod

# Browse for SMB file shares
$ dns-sd -B _smb._tcp local.

# Browse for AFP file shares (legacy)
$ dns-sd -B _afpovertcp._tcp local.

# The command runs continuously. Press Ctrl+C to stop.
```

### Common Service Types

| Service | Type |
|---------|------|
| Web servers | `_http._tcp` |
| Secure web | `_https._tcp` |
| SSH | `_ssh._tcp` |
| SFTP | `_sftp-ssh._tcp` |
| SMB file sharing | `_smb._tcp` |
| AFP file sharing | `_afpovertcp._tcp` |
| IPP printing | `_ipp._tcp` |
| AirPlay | `_airplay._tcp` |
| AirDrop | `_airdrop._tcp` |
| iTunes sharing | `_daap._tcp` |
| Screen sharing | `_rfb._tcp` |
| Remote management | `_eppc._tcp` |
| Time Machine | `_adisk._tcp` |
| Spotify Connect | `_spotify-connect._tcp` |

### Resolving Services (Getting Details)

Once you find a service, resolve it to get its address and port:

```bash
# Resolve a specific service
$ dns-sd -L "My-MacBook-Pro" _ssh._tcp local.
Lookup My-MacBook-Pro._ssh._tcp.local.
DATE: ---Mon 15 Jan 2024---
14:30:00.123  My-MacBook-Pro._ssh._tcp.local. can be reached at My-MacBook-Pro.local.:22 (interface 8)

# The output shows hostname (My-MacBook-Pro.local.) and port (22)
```

### Looking Up IP Addresses

```bash
# Look up IPv4 address
$ dns-sd -G v4 My-MacBook-Pro.local.
DATE: ---Mon 15 Jan 2024---
14:31:00.123  My-MacBook-Pro.local. 192.168.1.100

# Look up IPv6 address
$ dns-sd -G v6 My-MacBook-Pro.local.

# Look up both
$ dns-sd -G v4v6 My-MacBook-Pro.local.
```

### Querying Service Records

```bash
# Query for any record
$ dns-sd -Q My-MacBook-Pro.local. any

# Query TXT records (service metadata)
$ dns-sd -Q My-MacBook-Pro._ssh._tcp.local. TXT

# Query SRV records (service location)
$ dns-sd -Q _http._tcp.local. SRV
```

## Advertising Services

You can advertise your own services using `dns-sd`:

### Register a Simple Service

```bash
# Advertise an SSH service
$ dns-sd -R "My SSH Server" _ssh._tcp local. 22
Registering Service My SSH Server._ssh._tcp.local. port 22
DATE: ---Mon 15 Jan 2024---
14:35:00.123  Got a reply for service My SSH Server._ssh._tcp.local.: Name now registered and active

# Advertise a web server
$ dns-sd -R "My Web Server" _http._tcp local. 8080

# The service remains advertised until you press Ctrl+C
```

### Register Service with TXT Records

TXT records provide additional metadata about the service:

```bash
# Advertise with TXT records
$ dns-sd -R "My Web App" _http._tcp local. 8080 path=/app version=1.0

# TXT records appear as key=value pairs
# Other clients can read these to learn about the service
```

### Practical Example: Advertise a Development Server

```bash
# Start a Python web server
$ python3 -m http.server 8000 &

# Advertise it via Bonjour
$ dns-sd -R "Dev Server" _http._tcp local. 8000 path=/ environment=development

# Now other Macs can find it in Safari's Bonjour bookmarks
```

## Using Bonjour in Practice

### Finding Printers

```bash
# Browse for all printer types
$ dns-sd -B _ipp._tcp local.
$ dns-sd -B _pdl-datastream._tcp local.
$ dns-sd -B _printer._tcp local.

# Get printer details
$ dns-sd -L "HP LaserJet" _ipp._tcp local.
```

### Finding File Shares

```bash
# Find SMB shares (Windows/Samba/macOS)
$ dns-sd -B _smb._tcp local.
Browsing for _smb._tcp.local.
14:40:00.123  Add        2   8 local.               _smb._tcp.           NAS
14:40:00.234  Add        3   8 local.               _smb._tcp.           Macmini

# Resolve to get address
$ dns-sd -L "NAS" _smb._tcp local.
14:40:30.123  NAS._smb._tcp.local. can be reached at nas.local.:445

# Connect using the address
$ open smb://nas.local
```

### Finding Screen Sharing Servers

```bash
# Browse for VNC/Screen Sharing
$ dns-sd -B _rfb._tcp local.
14:41:00.123  Add        2   8 local.               _rfb._tcp.           Macmini

# Connect
$ open vnc://Macmini.local
```

### Quick Network Discovery Script

```bash
#!/bin/bash
# discover-services.sh - Quick network service discovery

echo "=== SSH Servers ==="
timeout 3 dns-sd -B _ssh._tcp local. 2>/dev/null | grep Add | awk '{print $7}'

echo -e "\n=== Web Servers ==="
timeout 3 dns-sd -B _http._tcp local. 2>/dev/null | grep Add | awk '{print $7}'

echo -e "\n=== File Shares ==="
timeout 3 dns-sd -B _smb._tcp local. 2>/dev/null | grep Add | awk '{print $7}'

echo -e "\n=== AirPlay Devices ==="
timeout 3 dns-sd -B _airplay._tcp local. 2>/dev/null | grep Add | awk '{print $7}'
```

## mDNSResponder

`mDNSResponder` is the daemon that handles all Bonjour operations. It's also macOS's DNS resolver.

### Checking mDNSResponder Status

```bash
# Check if running
$ ps aux | grep mDNSResponder
_mdnsresponder   123   0.0  0.0  4123456   7890   ??  Ss   10:00AM   0:01.23 /usr/sbin/mDNSResponder

# View mDNSResponder logs
$ log show --predicate 'process == "mDNSResponder"' --last 5m

# Real-time mDNSResponder logging
$ log stream --predicate 'process == "mDNSResponder"' --level debug
```

### Restarting mDNSResponder

Sometimes mDNSResponder needs a restart to resolve issues:

```bash
# Restart mDNSResponder (also flushes DNS cache)
$ sudo killall -HUP mDNSResponder

# Full restart
$ sudo launchctl kickstart -k system/com.apple.mDNSResponder
```

### DNS Cache and mDNSResponder

mDNSResponder handles both mDNS and regular DNS resolution:

```bash
# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder

# View current DNS configuration
$ scutil --dns

# Check DNS resolution
$ dscacheutil -q host -a name google.com
```

## Wide-Area Bonjour

Bonjour can work beyond the local network using DNS-based Service Discovery (DNS-SD):

### Browsing Wide-Area Services

```bash
# Browse services in a DNS domain
$ dns-sd -B _http._tcp example.com.

# This requires the domain to have DNS-SD records configured
```

### Setting Up Wide-Area Bonjour

To advertise services beyond the local network, you need:

1. A DNS server that supports dynamic updates or manually added records
2. Service records (SRV, TXT, PTR) in your DNS zone

Example DNS records for a web service:

```
_http._tcp.example.com.    PTR   My\ Web\ Server._http._tcp.example.com.
My\ Web\ Server._http._tcp.example.com.  SRV   0 0 80 webserver.example.com.
My\ Web\ Server._http._tcp.example.com.  TXT   "path=/" "version=2.0"
```

## Debugging Bonjour

### Check Bonjour Registration

```bash
# See what your Mac is advertising
$ dns-sd -B _services._dns-sd._udp local.
# This lists all service types being advertised

# Then browse specific types
$ dns-sd -B _ssh._tcp local.
```

### Network Traffic Analysis

```bash
# Capture mDNS traffic
$ sudo tcpdump -i en0 port 5353

# More detailed
$ sudo tcpdump -i en0 -v port 5353
```

### Common Issues

**Service not appearing:**
```bash
# Check if mDNSResponder is running
$ pgrep mDNSResponder

# Restart if needed
$ sudo killall -HUP mDNSResponder

# Check firewall isn't blocking port 5353
$ sudo pfctl -s rules | grep 5353
```

**Can't resolve .local names:**
```bash
# Test mDNS resolution
$ dns-sd -G v4 hostname.local.

# If failing, check network interface
$ dns-sd -G v4 hostname.local. -i en0

# Check for DNS server misconfiguration
# Some DNS servers incorrectly handle .local
$ scutil --dns | grep -A5 "resolver #1"
```

## Programmatic Access

### Using dnssd from Python

```python
# Install: pip install pybonjour
import pybonjour

def browse_callback(sdRef, flags, interfaceIndex, errorCode, serviceName, regtype, replyDomain):
    print(f"Found: {serviceName}")

browse_sdRef = pybonjour.DNSServiceBrowse(
    regtype='_http._tcp',
    callBack=browse_callback
)

# Process events
while True:
    ready = select.select([browse_sdRef], [], [], timeout)
    if browse_sdRef in ready[0]:
        pybonjour.DNSServiceProcessResult(browse_sdRef)
```

### Using dns-sd in Scripts

```bash
#!/bin/bash
# wait-for-service.sh - Wait for a service to appear

SERVICE_NAME="$1"
SERVICE_TYPE="${2:-_ssh._tcp}"
TIMEOUT="${3:-30}"

echo "Waiting for $SERVICE_NAME ($SERVICE_TYPE)..."

# Run dns-sd in background, capture output
dns-sd -B "$SERVICE_TYPE" local. 2>&1 | while read line; do
    if echo "$line" | grep -q "Add.*$SERVICE_NAME"; then
        echo "Found $SERVICE_NAME"
        pkill -P $$ dns-sd
        exit 0
    fi
done &

# Timeout handler
sleep $TIMEOUT
pkill -P $$ dns-sd
echo "Timeout waiting for $SERVICE_NAME"
exit 1
```

## Summary

| Task | Command |
|------|---------|
| Browse services | `dns-sd -B _type._tcp local.` |
| Resolve service | `dns-sd -L "name" _type._tcp local.` |
| Look up IP | `dns-sd -G v4 hostname.local.` |
| Advertise service | `dns-sd -R "name" _type._tcp local. port` |
| Query records | `dns-sd -Q name type` |
| Restart mDNSResponder | `sudo killall -HUP mDNSResponder` |

Key points:

- **Bonjour** enables automatic service discovery without configuration
- **mDNS** resolves `.local` hostnames on the local network
- **DNS-SD** discovers and advertises services
- **mDNSResponder** handles all Bonjour operations and DNS resolution
- **dns-sd** is the command-line interface to DNS Service Discovery
- Common service types include `_http._tcp`, `_ssh._tcp`, `_smb._tcp`, `_airplay._tcp`
