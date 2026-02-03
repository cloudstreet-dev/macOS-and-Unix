# Networking Commands

Network diagnostics on macOS use a mix of traditional BSD tools and Apple-specific utilities. If you're coming from Linux, you'll find that `ip` doesn't exist, but many BSD alternatives are available. This chapter covers network commands for troubleshooting and configuration.

## Interface Configuration

### ifconfig vs ip

Linux uses `ip`, but macOS uses `ifconfig`:

```bash
# Linux
$ ip addr show
$ ip link show
$ ip route show

# macOS - ifconfig is the primary tool
$ ifconfig
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=6463<RXCSUM,TXCSUM,TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
	ether aa:bb:cc:dd:ee:ff
	inet6 fe80::1c1c:abcd:1234:5678%en0 prefixlen 64 secured scopeid 0x8
	inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
```

Common ifconfig operations:

```bash
# List all interfaces
$ ifconfig -a

# Show specific interface
$ ifconfig en0

# Get just the IP address (parsing)
$ ifconfig en0 | grep "inet " | awk '{print $2}'
192.168.1.100

# Show only active interfaces
$ ifconfig | grep -E "^[a-z]" | cut -d: -f1
lo0
en0
en1

# Enable/disable interface (requires sudo)
$ sudo ifconfig en0 down
$ sudo ifconfig en0 up

# Set IP address manually (temporary)
$ sudo ifconfig en0 inet 192.168.1.50 netmask 255.255.255.0

# Add an alias IP
$ sudo ifconfig en0 alias 192.168.1.51 netmask 255.255.255.0
```

### Understanding macOS Interface Names

```
lo0     - Loopback (127.0.0.1)
en0     - Primary Ethernet or Wi-Fi (depends on Mac model)
en1     - Secondary network interface
en2-enX - Additional interfaces (USB Ethernet, etc.)
bridge0 - Network bridge
awdl0   - Apple Wireless Direct Link (AirDrop, etc.)
llw0    - Low-latency Wi-Fi (various Apple services)
utun0-X - VPN tunnel interfaces
gif0    - Generic tunnel interface
stf0    - 6to4 tunnel interface
```

Find your primary interface:

```bash
# Get active interface
$ route get default | grep interface
    interface: en0

# Or using networksetup
$ networksetup -listallhardwareports | grep -A1 "Wi-Fi"
Hardware Port: Wi-Fi
Device: en0
```

## networksetup: macOS Network Configuration

`networksetup` is the command-line equivalent of System Preferences > Network:

```bash
# List all network services
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
USB 10/100/1000 LAN
Thunderbolt Bridge
iPhone USB

# List hardware ports and devices
$ networksetup -listallhardwareports
Hardware Port: Wi-Fi
Device: en0
Ethernet Address: aa:bb:cc:dd:ee:ff

Hardware Port: Thunderbolt 1
Device: en1
...

# Get detailed info for a service
$ networksetup -getinfo "Wi-Fi"
DHCP Configuration
IP address: 192.168.1.100
Subnet mask: 255.255.255.0
Router: 192.168.1.1
Client ID:
IPv6: Automatic
IPv6 IP address: none
IPv6 Router: none
Wi-Fi ID: aa:bb:cc:dd:ee:ff
```

### IP Configuration

```bash
# Set to DHCP
$ sudo networksetup -setdhcp "Wi-Fi"

# Set static IP
$ sudo networksetup -setmanual "Wi-Fi" 192.168.1.50 255.255.255.0 192.168.1.1
#                               service    IP            netmask       gateway

# Get current IP configuration
$ networksetup -getinfo "Wi-Fi"
```

### DNS Configuration

```bash
# View current DNS servers
$ networksetup -getdnsservers "Wi-Fi"
8.8.8.8
8.8.4.4

# Set DNS servers
$ sudo networksetup -setdnsservers "Wi-Fi" 8.8.8.8 8.8.4.4

# Clear DNS (use DHCP-provided)
$ sudo networksetup -setdnsservers "Wi-Fi" empty

# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder
```

### Wi-Fi Management

```bash
# Get Wi-Fi power state
$ networksetup -getairportpower en0
Wi-Fi Power (en0): On

# Turn Wi-Fi on/off
$ networksetup -setairportpower en0 off
$ networksetup -setairportpower en0 on

# Get current Wi-Fi network
$ networksetup -getairportnetwork en0
Current Wi-Fi Network: MyNetwork

# Connect to a Wi-Fi network
$ networksetup -setairportnetwork en0 "NetworkName" "password"

# List preferred (known) networks
$ networksetup -listpreferredwirelessnetworks en0
Preferred networks on en0:
	MyHomeNetwork
	OfficeNetwork
	CoffeeShopWiFi

# Remove a preferred network
$ sudo networksetup -removepreferredwirelessnetwork en0 "CoffeeShopWiFi"
```

### Proxy Configuration

```bash
# Get proxy settings
$ networksetup -getwebproxy "Wi-Fi"
Enabled: No
Server:
Port: 0
Authenticated Proxy Enabled: 0

# Set web proxy
$ sudo networksetup -setwebproxy "Wi-Fi" proxy.example.com 8080

# Set with authentication
$ sudo networksetup -setwebproxy "Wi-Fi" proxy.example.com 8080 on user password

# Disable proxy
$ sudo networksetup -setwebproxystate "Wi-Fi" off

# Set proxy auto-config (PAC) URL
$ sudo networksetup -setautoproxyurl "Wi-Fi" "http://example.com/proxy.pac"
```

## scutil: System Configuration

`scutil` provides low-level access to system configuration, including network settings:

```bash
# Get DNS configuration
$ scutil --dns
DNS configuration

resolver #1
  search domain[0] : home
  nameserver[0] : 192.168.1.1
  nameserver[1] : 8.8.8.8
  if_index : 8 (en0)
  flags    : Request A records
  reach    : 0x00000002 (Reachable)
...

# Check host reachability
$ scutil -r google.com
Reachable

$ scutil -r 192.168.1.1
Reachable,Direct

# Get/set hostname
$ scutil --get HostName
my-macbook

$ scutil --get LocalHostName
my-macbook

$ scutil --get ComputerName
My MacBook Pro

# Set hostname (requires sudo)
$ sudo scutil --set HostName newhostname
$ sudo scutil --set LocalHostName newhostname
$ sudo scutil --set ComputerName "New MacBook Pro"

# Get proxy configuration
$ scutil --proxy
<dictionary> {
  ExceptionsList : <array> {
    0 : *.local
    1 : 169.254/16
  }
  FTPPassive : 1
  HTTPEnable : 0
  HTTPSEnable : 0
}
```

Interactive mode for exploring configuration:

```bash
$ scutil
> list
  subKey [0] = Plugin:IPConfiguration
  subKey [1] = Plugin:InterfaceNamer
  ...
> show State:/Network/Global/IPv4
<dictionary> {
  PrimaryInterface : en0
  PrimaryService : XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  Router : 192.168.1.1
}
> show State:/Network/Interface/en0/IPv4
<dictionary> {
  Addresses : <array> {
    0 : 192.168.1.100
  }
  BroadcastAddresses : <array> {
    0 : 192.168.1.255
  }
  SubnetMasks : <array> {
    0 : 255.255.255.0
  }
}
> quit
```

## airport: Wi-Fi Diagnostics

The airport utility is hidden but powerful:

```bash
# Create an alias (the path is long)
$ alias airport='/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport'

# Show current connection info
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

# Scan for available networks
$ airport -s
                            SSID BSSID             RSSI CHANNEL HT CC SECURITY
                       MyNetwork aa:bb:cc:dd:ee:ff -52  149,+1  Y  US WPA2(PSK/AES/AES)
                   Neighbor_WiFi bb:cc:dd:ee:ff:00 -68  6       Y  US WPA2(PSK/AES/AES)
                   Guest_Network cc:dd:ee:ff:00:11 -75  11      Y  -- WPA2(PSK/AES/AES)

# Disconnect from current network
$ sudo airport -z

# Show supported channels
$ airport -c
Supported channels:
1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 36, 40, 44, 48, 52, 56, 60, 64, 100, ...
```

## netstat: Network Statistics

```bash
# Show all connections
$ netstat -an

# Show listening ports
$ netstat -an | grep LISTEN

# Show routing table
$ netstat -rn
Routing tables

Internet:
Destination        Gateway            Flags        Netif Expire
default            192.168.1.1        UGScg          en0
127.0.0.1          127.0.0.1          UH             lo0
192.168.1          link#8             UCS            en0      !
192.168.1.1        aa:bb:cc:dd:ee:ff  UHLWIir        en0   1198
192.168.1.100      127.0.0.1          UHS            lo0

# Show per-protocol statistics
$ netstat -s

# Show statistics for specific protocol
$ netstat -s -p tcp
$ netstat -s -p udp

# Show interface statistics
$ netstat -i
Name  Mtu   Network       Address            Ipkts Ierrs    Opkts Oerrs  Coll
lo0   16384 <Link#1>                        123456     0   123456     0     0
lo0   16384 127           localhost         123456     -   123456     -     -
en0   1500  <Link#8>    aa:bb:cc:dd:ee:ff  987654     0   654321     0     0
en0   1500  192.168.1     192.168.1.100    987654     -   654321     -     -
```

Linux netstat equivalents:

```bash
# Linux: ss -tuln
# macOS: netstat -an (no ss command)

# Linux: ss -tunp
# macOS: lsof -i (shows processes)

# Show processes using network
$ lsof -i -P | head -20
COMMAND     PID   USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
loginwindow 123   user    6u  IPv4   0x1234      0t0  UDP *:*
Google      456   user   23u  IPv4   0x5678      0t0  TCP 192.168.1.100:51234->142.250.x.x:443 (ESTABLISHED)
```

## lsof: List Open Files (Including Network)

```bash
# Show all network connections
$ lsof -i

# Show listening ports
$ lsof -i -P | grep LISTEN
rapportd    123 user    4u  IPv4 0x1234    0t0  TCP *:49152 (LISTEN)
rapportd    123 user    5u  IPv6 0x5678    0t0  TCP *:49152 (LISTEN)

# Show connections on specific port
$ lsof -i :80
$ lsof -i :443

# Show connections to specific host
$ lsof -i @google.com

# Show TCP connections only
$ lsof -i TCP

# Show UDP connections only
$ lsof -i UDP

# Show process using a port
$ lsof -i :8080 -P
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
node    1234 user   23u  IPv4 0x1234      0t0  TCP *:8080 (LISTEN)

# Kill process on a port
$ lsof -ti :8080 | xargs kill
```

## Routing

```bash
# Show routing table
$ netstat -rn

# Or using route
$ route -n get default
   route to: default
destination: default
       mask: default
    gateway: 192.168.1.1
  interface: en0
      flags: <UP,GATEWAY,DONE,STATIC,PRCLONING,GLOBAL>

# Add a route
$ sudo route add -net 10.0.0.0/8 192.168.1.254

# Delete a route
$ sudo route delete -net 10.0.0.0/8

# Flush routing table (careful!)
$ sudo route flush
```

## ping and traceroute

```bash
# Ping (compatible with Linux)
$ ping -c 5 google.com
PING google.com (142.250.x.x): 56 data bytes
64 bytes from 142.250.x.x: icmp_seq=0 ttl=117 time=12.345 ms
...

# Flood ping (requires sudo)
$ sudo ping -f google.com

# Traceroute
$ traceroute google.com
traceroute to google.com (142.250.x.x), 64 hops max, 52 byte packets
 1  192.168.1.1 (192.168.1.1)  1.234 ms  0.987 ms  0.876 ms
 2  isp-gateway (10.0.0.1)  5.432 ms  4.321 ms  4.567 ms
...

# Use ICMP instead of UDP (requires sudo)
$ sudo traceroute -I google.com
```

## DNS Tools

```bash
# DNS lookup
$ host google.com
google.com has address 142.250.x.x
google.com has IPv6 address 2607:f8b0:...
google.com mail is handled by 10 smtp.google.com.

# More detailed DNS lookup
$ dig google.com
; <<>> DiG 9.10.6 <<>> google.com
;; ANSWER SECTION:
google.com.		123	IN	A	142.250.x.x

# Query specific record types
$ dig MX google.com
$ dig NS google.com
$ dig TXT google.com
$ dig AAAA google.com    # IPv6

# Query specific DNS server
$ dig @8.8.8.8 google.com

# Reverse DNS lookup
$ dig -x 8.8.8.8

# nslookup (also available)
$ nslookup google.com
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	google.com
Address: 142.250.x.x

# dscacheutil for local DNS cache
$ dscacheutil -q host -a name google.com
name: google.com
ip_address: 142.250.x.x

# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder
```

## curl and wget

Both are available (wget via Homebrew):

```bash
# curl is pre-installed
$ curl -I https://apple.com
HTTP/2 200
content-type: text/html; charset=utf-8
...

# Download file
$ curl -O https://example.com/file.zip
$ curl -o localname.zip https://example.com/file.zip

# Follow redirects
$ curl -L https://example.com/redirect

# Show headers and body
$ curl -i https://example.com

# POST request
$ curl -X POST -d "key=value" https://api.example.com

# JSON POST
$ curl -X POST -H "Content-Type: application/json" -d '{"key":"value"}' https://api.example.com

# wget (install via Homebrew)
$ brew install wget
$ wget https://example.com/file.zip
$ wget -c https://example.com/largefile.zip    # Resume download
$ wget -r -l 2 https://example.com             # Recursive, 2 levels deep
```

## nc (netcat): Network Swiss Army Knife

```bash
# Test if port is open
$ nc -zv google.com 443
Connection to google.com port 443 [tcp/https] succeeded!

# Port scanning
$ nc -zv 192.168.1.1 20-25
Connection to 192.168.1.1 port 22 [tcp/ssh] succeeded!

# Create simple TCP server
$ nc -l 8080

# Connect to server
$ nc localhost 8080

# Transfer file
# On receiver:
$ nc -l 8080 > received_file.txt
# On sender:
$ nc destination 8080 < file_to_send.txt

# Chat between two machines
# Machine 1:
$ nc -l 8080
# Machine 2:
$ nc machine1 8080

# HTTP request
$ echo -e "GET / HTTP/1.1\nHost: example.com\n\n" | nc example.com 80
```

## arp: Address Resolution Protocol

```bash
# Show ARP cache
$ arp -a
? (192.168.1.1) at aa:bb:cc:dd:ee:ff on en0 ifscope [ethernet]
? (192.168.1.100) at 11:22:33:44:55:66 on en0 ifscope permanent [ethernet]

# Delete an entry
$ sudo arp -d 192.168.1.50

# Add a static entry
$ sudo arp -s 192.168.1.50 aa:bb:cc:dd:ee:ff
```

## Packet Capture: tcpdump

```bash
# Capture on interface (requires sudo)
$ sudo tcpdump -i en0

# Capture specific host
$ sudo tcpdump -i en0 host 192.168.1.50

# Capture specific port
$ sudo tcpdump -i en0 port 80
$ sudo tcpdump -i en0 port 443

# Capture and save to file
$ sudo tcpdump -i en0 -w capture.pcap

# Read capture file
$ tcpdump -r capture.pcap

# Show packet contents
$ sudo tcpdump -i en0 -X

# Capture HTTP traffic
$ sudo tcpdump -i en0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'
```

## Network Diagnostics Tool

macOS has a built-in network diagnostics:

```bash
# Run network diagnostics (creates report)
$ /System/Library/CoreServices/Applications/Network\ Utility.app/Contents/MacOS/Network\ Utility

# Or use networkQuality (macOS 12+)
$ networkQuality
==== SUMMARY ====
Upload capacity: 50.123 Mbps
Download capacity: 200.456 Mbps
Upload flows: 12
Download flows: 16
Responsiveness: High (1234 RPM)

# Run in sequential mode (more accurate)
$ networkQuality -s
```

## Summary: Linux to macOS Mapping

| Linux Command | macOS Equivalent | Notes |
|---------------|------------------|-------|
| `ip addr` | `ifconfig` | ifconfig is deprecated on Linux |
| `ip link` | `ifconfig` | Same |
| `ip route` | `netstat -rn` or `route` | |
| `ss` | `netstat` or `lsof -i` | |
| `systemctl restart network` | `networksetup` | No systemd on macOS |
| `nmcli` | `networksetup` | macOS equivalent |
| `iwconfig`/`iwlist` | `airport` | Wi-Fi utility |
| `hostnamectl` | `scutil --get/set HostName` | |
| `resolvectl` | `scutil --dns` | |

Key macOS-specific tools:
- `networksetup`: High-level network configuration
- `scutil`: System configuration access
- `airport`: Wi-Fi diagnostics
- `dscacheutil`: Directory services cache
- `networkQuality`: Network speed test (macOS 12+)
