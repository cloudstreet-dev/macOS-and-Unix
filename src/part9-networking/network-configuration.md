# Network Configuration from CLI

macOS provides several command-line tools for network configuration, each operating at a different level of abstraction. Understanding when to use each tool is key to effective network management.

## The Configuration Tools Hierarchy

```
High-level    networksetup    User-friendly, persistent settings
                  │
                  ▼
Mid-level       ipconfig       DHCP operations, BootP
                  │
                  ▼
Low-level      ifconfig        Direct interface manipulation (BSD)
                  │
                  ▼
System         scutil          System configuration framework access
```

## networksetup: The Primary Configuration Tool

`networksetup` is the command-line equivalent of System Preferences > Network. Changes made with `networksetup` are persistent and integrated with macOS's network management.

### Listing Network Services

```bash
# List all network services
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
USB 10/100/1000 LAN
Thunderbolt Bridge
iPhone USB
*Bluetooth PAN

# List hardware ports with device names
$ networksetup -listallhardwareports

Hardware Port: Wi-Fi
Device: en0
Ethernet Address: aa:bb:cc:dd:ee:ff

Hardware Port: Thunderbolt 1
Device: en1
Ethernet Address: bb:cc:dd:ee:ff:00

Hardware Port: Thunderbolt Bridge
Device: bridge0
Ethernet Address: cc:dd:ee:ff:00:11
```

### Getting Network Information

```bash
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

# Get just the IP address
$ networksetup -getinfo "Wi-Fi" | grep "IP address" | head -1
IP address: 192.168.1.100

# Get computer name
$ networksetup -getcomputername
My-MacBook-Pro
```

### IP Address Configuration

```bash
# Set static IP address
$ sudo networksetup -setmanual "Wi-Fi" 192.168.1.50 255.255.255.0 192.168.1.1
#                                service    IP          netmask       gateway

# Verify the change
$ networksetup -getinfo "Wi-Fi"
Manual Configuration
IP address: 192.168.1.50
Subnet mask: 255.255.255.0
Router: 192.168.1.1

# Switch back to DHCP
$ sudo networksetup -setdhcp "Wi-Fi"

# Set DHCP with manual IP (DHCP for DNS, manual IP)
$ sudo networksetup -setdhcp "Wi-Fi" 192.168.1.50
```

### DNS Configuration

```bash
# View current DNS servers
$ networksetup -getdnsservers "Wi-Fi"
8.8.8.8
8.8.4.4

# Set DNS servers
$ sudo networksetup -setdnsservers "Wi-Fi" 8.8.8.8 8.8.4.4 1.1.1.1

# Clear DNS (use DHCP-provided servers)
$ sudo networksetup -setdnsservers "Wi-Fi" empty

# Verify DNS settings
$ scutil --dns | grep "nameserver\[" | head -5
  nameserver[0] : 8.8.8.8
  nameserver[1] : 8.8.4.4
  nameserver[2] : 1.1.1.1
```

### Search Domains

```bash
# View search domains
$ networksetup -getsearchdomains "Wi-Fi"
example.com
internal.example.com

# Set search domains
$ sudo networksetup -setsearchdomains "Wi-Fi" example.com internal.example.com

# Clear search domains
$ sudo networksetup -setsearchdomains "Wi-Fi" empty
```

### MTU Configuration

```bash
# Get current MTU
$ networksetup -getMTU "Wi-Fi"
Active MTU: 1500 (Current Setting: 1500)

# Set MTU
$ sudo networksetup -setMTU "Wi-Fi" 1400

# Reset to default
$ sudo networksetup -setMTU "Wi-Fi" 1500
```

### Proxy Configuration

```bash
# Get web proxy settings
$ networksetup -getwebproxy "Wi-Fi"
Enabled: No
Server:
Port: 0
Authenticated Proxy Enabled: 0

# Set web proxy (HTTP)
$ sudo networksetup -setwebproxy "Wi-Fi" proxy.example.com 8080

# Set with authentication
$ sudo networksetup -setwebproxy "Wi-Fi" proxy.example.com 8080 on username password

# Enable/disable web proxy
$ sudo networksetup -setwebproxystate "Wi-Fi" on
$ sudo networksetup -setwebproxystate "Wi-Fi" off

# Set secure web proxy (HTTPS)
$ sudo networksetup -setsecurewebproxy "Wi-Fi" proxy.example.com 8080

# Set SOCKS proxy
$ sudo networksetup -setsocksfirewallproxy "Wi-Fi" proxy.example.com 1080

# Set proxy auto-config (PAC) URL
$ sudo networksetup -setautoproxyurl "Wi-Fi" "http://proxy.example.com/proxy.pac"

# Set bypass domains (proxy exceptions)
$ sudo networksetup -setproxybypassdomains "Wi-Fi" "*.local" "169.254/16" "localhost"

# Get all proxy settings
$ networksetup -getproxybypassdomains "Wi-Fi"
*.local
169.254/16
localhost
```

### Service Order

```bash
# View network service order (priority)
$ networksetup -listnetworkserviceorder
An asterisk (*) denotes that a network service is disabled.
(1) Wi-Fi
(Hardware Port: Wi-Fi, Device: en0)

(2) USB 10/100/1000 LAN
(Hardware Port: USB 10/100/1000 LAN, Device: en7)

(3) Thunderbolt Bridge
(Hardware Port: Thunderbolt Bridge, Device: bridge0)

# Set service order (first service has highest priority)
$ sudo networksetup -ordernetworkservices "Ethernet" "Wi-Fi" "Thunderbolt Bridge"
```

## ipconfig: DHCP and BootP Operations

`ipconfig` handles DHCP client operations and provides detailed interface information.

### Getting Interface Information

```bash
# Get all information for an interface
$ ipconfig getifaddr en0
192.168.1.100

# Get subnet mask
$ ipconfig getoption en0 subnet_mask
255.255.255.0

# Get router (gateway)
$ ipconfig getoption en0 router
192.168.1.1

# Get DHCP server address
$ ipconfig getoption en0 server_identifier
192.168.1.1

# Get DNS servers from DHCP
$ ipconfig getoption en0 domain_name_server
192.168.1.1

# Get lease duration
$ ipconfig getoption en0 lease_time
86400

# Get domain name from DHCP
$ ipconfig getoption en0 domain_name
home
```

### DHCP Operations

```bash
# Renew DHCP lease
$ sudo ipconfig set en0 DHCP

# Request specific IP via DHCP (DHCP INFORM)
$ sudo ipconfig set en0 INFORM 192.168.1.50

# Release DHCP lease (sets to none/unconfigured)
$ sudo ipconfig set en0 NONE

# Set manual IP (temporary, until reboot)
$ sudo ipconfig set en0 MANUAL 192.168.1.50 255.255.255.0
```

### Viewing DHCP Packet Information

```bash
# Show DHCP packet details
$ ipconfig getpacket en0
op = BOOTREPLY
htype = 1
flags = 0
hlen = 6
hops = 0
xid = 0x12345678
secs = 0
ciaddr = 0.0.0.0
yiaddr = 192.168.1.100
siaddr = 0.0.0.0
giaddr = 0.0.0.0
chaddr = aa:bb:cc:dd:ee:ff
sname =
file =
options:
Options count is 10
dhcp_message_type (uint8): ACK 0x5
server_identifier (ip): 192.168.1.1
lease_time (uint32): 0x15180 (86400)
subnet_mask (ip): 255.255.255.0
router (ip_mult): 192.168.1.1
domain_name_server (ip_mult): 192.168.1.1
domain_name (string): home
end (none):
```

### IPv6 Configuration

```bash
# Get IPv6 address
$ ipconfig getv6addr en0
fe80::1c1c:abcd:1234:5678%en0

# Set IPv6 to automatic
$ sudo ipconfig set en0 AUTOMATIC-V6

# Set manual IPv6
$ sudo ipconfig set en0 MANUAL-V6 2001:db8::1 64

# Disable IPv6 on interface
$ sudo networksetup -setv6off "Wi-Fi"

# Enable IPv6 automatic
$ sudo networksetup -setv6automatic "Wi-Fi"
```

## ifconfig: BSD Interface Configuration

`ifconfig` provides low-level interface control. Changes are temporary and don't survive reboots. Use `networksetup` for persistent changes.

### Viewing Interface Status

```bash
# Show all interfaces
$ ifconfig -a

# Show specific interface
$ ifconfig en0
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=6463<RXCSUM,TXCSUM,TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
	ether aa:bb:cc:dd:ee:ff
	inet6 fe80::1c1c:abcd:1234:5678%en0 prefixlen 64 secured scopeid 0x8
	inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active

# Show only UP interfaces
$ ifconfig -u

# Show only DOWN interfaces
$ ifconfig -d

# List interface names only
$ ifconfig -l
lo0 gif0 stf0 en0 en1 en2 bridge0 utun0 utun1

# Get just the IP address (parsing)
$ ifconfig en0 | awk '/inet / {print $2}'
192.168.1.100
```

### Understanding Interface Flags

```
UP          - Interface is enabled
BROADCAST   - Supports broadcast
SMART       - Interface manages own state
RUNNING     - Interface is operational
SIMPLEX     - Can't hear own transmissions
MULTICAST   - Supports multicast
PROMISC     - Promiscuous mode (captures all packets)
```

### Enabling and Disabling Interfaces

```bash
# Bring interface down
$ sudo ifconfig en0 down

# Bring interface up
$ sudo ifconfig en0 up

# Enable promiscuous mode
$ sudo ifconfig en0 promisc

# Disable promiscuous mode
$ sudo ifconfig en0 -promisc
```

### Temporary IP Configuration

```bash
# Set IP address (temporary)
$ sudo ifconfig en0 inet 192.168.1.50 netmask 255.255.255.0

# Add an alias IP (second IP on same interface)
$ sudo ifconfig en0 alias 192.168.1.51 netmask 255.255.255.0

# Remove alias
$ sudo ifconfig en0 -alias 192.168.1.51

# Set broadcast address
$ sudo ifconfig en0 broadcast 192.168.1.255
```

### MTU and Media Configuration

```bash
# Change MTU
$ sudo ifconfig en0 mtu 1400

# View media options
$ ifconfig en0 | grep media
	media: autoselect

# For Ethernet, set specific media (example)
$ sudo ifconfig en1 media 1000baseT mediaopt full-duplex
```

## scutil: System Configuration Utility

`scutil` provides access to the System Configuration framework, offering both interactive and command-line modes.

### Hostname Configuration

macOS has three types of hostnames:

```bash
# ComputerName: User-friendly name shown in Finder
$ scutil --get ComputerName
My MacBook Pro

# LocalHostName: Bonjour name (.local)
$ scutil --get LocalHostName
My-MacBook-Pro

# HostName: BSD hostname (may not be set)
$ scutil --get HostName
my-macbook-pro.example.com

# Set hostnames (requires sudo)
$ sudo scutil --set ComputerName "Work MacBook"
$ sudo scutil --set LocalHostName "Work-MacBook"
$ sudo scutil --set HostName "work-macbook.example.com"
```

### DNS Information

```bash
# Show complete DNS configuration
$ scutil --dns
DNS configuration

resolver #1
  search domain[0] : home
  nameserver[0] : 192.168.1.1
  nameserver[1] : 8.8.8.8
  if_index : 8 (en0)
  flags    : Request A records
  reach    : 0x00000002 (Reachable)

resolver #2
  domain   : local
  options  : mdns
  timeout  : 5
  flags    : Request A records
  reach    : 0x00000000 (Not Reachable)
  order    : 300000
...
```

### Network Reachability

```bash
# Check if host is reachable
$ scutil -r google.com
Reachable

$ scutil -r 192.168.1.1
Reachable,Direct

$ scutil -r 10.0.0.1
Not Reachable

# Reachability flags explained:
# Reachable          - Can reach destination
# Direct             - Directly connected (same subnet)
# Transient Current  - Using temporary connection
# Connection Required - Need to initiate connection
# Connection On Demand - Will connect when needed
```

### Proxy Information

```bash
# Show proxy configuration
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

### Interactive Mode

`scutil` has an interactive mode for exploring system configuration:

```bash
$ scutil
> list
  subKey [0] = Plugin:IPConfiguration
  subKey [1] = Plugin:InterfaceNamer
  subKey [2] = Plugin:KernelEventMonitor
  subKey [3] = Setup:
  subKey [4] = Setup:/
  ...

# Show network state
> show State:/Network/Global/IPv4
<dictionary> {
  PrimaryInterface : en0
  PrimaryService : XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  Router : 192.168.1.1
}

# Show interface details
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

# Show DNS state
> show State:/Network/Service/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/DNS
<dictionary> {
  ServerAddresses : <array> {
    0 : 192.168.1.1
    1 : 8.8.8.8
  }
}

# Exit interactive mode
> quit
```

### Useful scutil One-Liners

```bash
# Get primary network interface
$ scutil --nwi | grep -A1 "Network interfaces"
Network interfaces: en0

# Get primary IPv4 address
$ scutil --nwi | grep "address" | head -1
     address : 192.168.1.100

# Check VPN status
$ scutil --nc list
Available network connection services in the current set (*=enabled):
* (Disconnected)  XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX VPN (IPSec)        "My VPN"
```

## Routing Configuration

### Viewing Routes

```bash
# Show routing table
$ netstat -rn
Routing tables

Internet:
Destination        Gateway            Flags        Netif Expire
default            192.168.1.1        UGScg          en0
127.0.0.1          127.0.0.1          UH             lo0
192.168.1          link#8             UCS            en0      !
192.168.1.1        aa:bb:cc:dd:ee:ff  UHLWIir        en0   1198
192.168.1.100/32   link#8             UCS            en0      !

# Understanding flags:
# U - Route is up
# G - Route is to a gateway
# H - Route is to a host
# S - Route was set manually
# c - Route creates new routes on use
# L - Valid protocol to link address translation
# W - Route was generated by wildcard
# I - Route requires revalidation
# r - Route was rejected
```

### Managing Routes

```bash
# Get route to specific destination
$ route -n get google.com
   route to: 142.250.x.x
destination: default
       mask: default
    gateway: 192.168.1.1
  interface: en0
      flags: <UP,GATEWAY,DONE,STATIC,PRCLONING,GLOBAL>

# Add a static route
$ sudo route add -net 10.0.0.0/8 192.168.1.254
add net 10.0.0.0: gateway 192.168.1.254

# Add host route
$ sudo route add -host 10.0.0.1 192.168.1.254

# Delete a route
$ sudo route delete -net 10.0.0.0/8

# Change gateway for existing route
$ sudo route change default 192.168.1.2

# Flush routing table (use with caution)
$ sudo route flush
```

### Persistent Routes

Routes added with `route` command don't survive reboots. For persistent routes, create a launch daemon:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.static-routes</string>
    <key>ProgramArguments</key>
    <array>
        <string>/sbin/route</string>
        <string>add</string>
        <string>-net</string>
        <string>10.0.0.0/8</string>
        <string>192.168.1.254</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>WatchPaths</key>
    <array>
        <string>/Library/Preferences/SystemConfiguration</string>
    </array>
</dict>
</plist>
```

Save to `/Library/LaunchDaemons/com.example.static-routes.plist` and load with:

```bash
$ sudo launchctl load /Library/LaunchDaemons/com.example.static-routes.plist
```

## Network Interface Creation

### Creating a Bridge

```bash
# Create bridge interface
$ sudo ifconfig bridge0 create

# Add interfaces to bridge
$ sudo ifconfig bridge0 addm en0 addm en1

# Bring bridge up
$ sudo ifconfig bridge0 up

# Delete bridge
$ sudo ifconfig bridge0 destroy
```

### Creating a VLAN Interface

```bash
# Create VLAN interface (VLAN 100 on en0)
$ sudo ifconfig vlan0 create
$ sudo ifconfig vlan0 vlan 100 vlandev en0
$ sudo ifconfig vlan0 inet 192.168.100.1 netmask 255.255.255.0
$ sudo ifconfig vlan0 up

# Or use networksetup for persistent VLAN
$ sudo networksetup -createVLAN "VLAN100" "Wi-Fi" 100
```

## Summary

| Task | Recommended Tool | Notes |
|------|------------------|-------|
| View/change service settings | `networksetup` | Persistent, high-level |
| DHCP operations | `ipconfig` | Renew/release lease |
| Quick interface check | `ifconfig` | View status, temp changes |
| Hostname/DNS/proxy info | `scutil` | System config access |
| Routing | `route`, `netstat -rn` | Route table management |

Key points:

- **Use `networksetup` for persistent changes** that should survive reboots
- **Use `ipconfig` for DHCP operations** like renewing leases
- **Use `ifconfig` for quick temporary changes** or diagnostics
- **Use `scutil` for system-level queries** and hostname configuration
- **Changes made with `ifconfig` and `route` are temporary** and won't survive reboots
