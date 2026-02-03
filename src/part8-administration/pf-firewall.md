# Configuring the pf Firewall

macOS uses pf (packet filter), the powerful firewall from OpenBSD. Unlike Linux's iptables/nftables, pf uses a clean, readable configuration syntax. This chapter covers configuring pf for network security on macOS.

## pf Overview

pf has been macOS's built-in firewall since OS X 10.7 (Lion). It operates at the kernel level, filtering packets before they reach applications.

```bash
# Check if pf is enabled
$ sudo pfctl -s info
Status: Disabled                      # or Enabled
...

# Enable pf
$ sudo pfctl -e
pf enabled

# Disable pf
$ sudo pfctl -d
pf disabled
```

## pf vs iptables

Coming from Linux, here's how pf compares to iptables:

| Feature | pf (macOS) | iptables (Linux) |
|---------|------------|------------------|
| Syntax | English-like | Command flags |
| Config file | /etc/pf.conf | /etc/iptables/rules.v4 |
| Rule order | Last match wins | First match wins |
| Tables | Built-in support | ipset (separate) |
| NAT | Integrated | Separate chain |
| Logging | Built-in | -j LOG target |

Syntax comparison:

```bash
# iptables: Block incoming SSH
iptables -A INPUT -p tcp --dport 22 -j DROP

# pf: Block incoming SSH
block in proto tcp from any to any port 22
```

## Configuration File: /etc/pf.conf

The main configuration file is `/etc/pf.conf`:

```bash
$ sudo cat /etc/pf.conf
#
# Default PF configuration file.
#
# This file contains the main ruleset, which gets automatically loaded
# at startup.  PF will not be automatically enabled, however.  Instead,
# each component which utilizes PF is responsible for enabling and disabling
# PF via -E and -X as documented in pfctl(8).  That will ensure that PF
# is disabled only when the last enable reference is released.
#
# Care must be taken to ensure that the main ruleset does not get flushed,
# as the nested anchors rely on the anchor point defined here. In
# particular, establish this anchor point first before flushing the main
# ruleset.
#

# Options
set block-policy drop
set fingerprints "/etc/pf.os"
set ruleset-optimization basic
set skip on lo0

# Normalization
scrub in all no-df

# Anchors
anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
```

### Configuration Sections

A typical pf.conf has these sections:

1. **Macros** (variables)
2. **Tables** (IP address lists)
3. **Options** (global settings)
4. **Scrub** (packet normalization)
5. **NAT/Redirect** (if needed)
6. **Filter rules** (the actual firewall rules)

## Basic Rule Syntax

### Rule Structure

```
action [direction] [log] [quick] [on interface] [proto protocol]
       [from source] [to destination] [flags] [state]
```

Components:
- **action**: `block` or `pass`
- **direction**: `in` or `out`
- **log**: Log matching packets
- **quick**: Stop processing on match (first-match wins)
- **on interface**: `on en0`, `on lo0`
- **proto**: `tcp`, `udp`, `icmp`
- **from/to**: Source and destination
- **port**: Port number or name

### Simple Examples

```bash
# Block all incoming traffic
block in all

# Pass all outgoing traffic
pass out all

# Block incoming SSH
block in proto tcp from any to any port 22

# Allow incoming HTTP and HTTPS
pass in proto tcp from any to any port { 80, 443 }

# Block specific IP
block in from 192.168.1.100

# Allow from specific network
pass in from 192.168.1.0/24
```

## Creating a Custom Firewall

Let's create a practical firewall configuration:

### Step 1: Create Custom Config

```bash
$ sudo nano /etc/pf.custom.conf
```

```bash
# /etc/pf.custom.conf - Custom firewall rules

# === MACROS ===
ext_if = "en0"                          # External interface (Wi-Fi or Ethernet)
tcp_services = "{ 22, 80, 443 }"        # Allowed inbound TCP ports
udp_services = "{ 53, 123 }"            # Allowed inbound UDP ports

# Trusted networks
trusted_nets = "{ 192.168.1.0/24, 10.0.0.0/8 }"

# === TABLES ===
# Blocked hosts (can be modified at runtime)
table <blocklist> persist

# === OPTIONS ===
set block-policy drop                    # Silently drop blocked packets
set skip on lo0                          # Don't filter loopback

# === SCRUB ===
scrub in all                             # Normalize incoming packets

# === FILTER RULES ===

# Default deny incoming, allow outgoing
block in all
pass out all keep state

# Allow all traffic on loopback
pass quick on lo0 all

# Block hosts in blocklist
block in quick from <blocklist>

# Allow ICMP (ping)
pass in inet proto icmp all icmp-type { echoreq, unreach }

# Allow incoming SSH from trusted networks only
pass in on $ext_if proto tcp from $trusted_nets to any port 22

# Allow incoming web traffic
pass in on $ext_if proto tcp from any to any port { 80, 443 }

# Allow established connections
pass in on $ext_if proto tcp from any to any flags S/SA keep state

# Block and log everything else
block in log all
```

### Step 2: Test the Configuration

```bash
# Check syntax without loading
$ sudo pfctl -n -f /etc/pf.custom.conf
# No output means no errors

# Verbose check
$ sudo pfctl -n -v -f /etc/pf.custom.conf
# Shows parsed rules
```

### Step 3: Load the Configuration

```bash
# Load rules
$ sudo pfctl -f /etc/pf.custom.conf

# Enable pf
$ sudo pfctl -e
pf enabled

# Verify rules are loaded
$ sudo pfctl -s rules
block drop in all
pass out all flags S/SA keep state
pass quick on lo0 all flags S/SA keep state
block drop in quick from <blocklist> to any
pass in inet proto icmp all icmp-type echoreq keep state
pass in inet proto icmp all icmp-type unreach keep state
pass in on en0 proto tcp from 192.168.1.0/24 to any port = ssh flags S/SA keep state
pass in on en0 proto tcp from 10.0.0.0/8 to any port = ssh flags S/SA keep state
pass in on en0 proto tcp from any to any port = http flags S/SA keep state
pass in on en0 proto tcp from any to any port = https flags S/SA keep state
block drop in log all
```

## Managing Tables

Tables are dynamic lists of IP addresses, perfect for blocklists:

```bash
# View table contents
$ sudo pfctl -t blocklist -T show
# (empty if no IPs added)

# Add IP to blocklist
$ sudo pfctl -t blocklist -T add 192.168.1.100
1/1 addresses added.

# Add multiple IPs
$ sudo pfctl -t blocklist -T add 10.0.0.1 10.0.0.2 10.0.0.3

# Add network
$ sudo pfctl -t blocklist -T add 172.16.0.0/16

# Remove IP from blocklist
$ sudo pfctl -t blocklist -T delete 192.168.1.100
1/1 addresses deleted.

# Flush entire table
$ sudo pfctl -t blocklist -T flush
0 addresses deleted.

# Load table from file
$ cat /etc/pf.blocklist.txt
# Bad actors
192.168.1.100
10.0.0.50
172.16.0.0/16

$ sudo pfctl -t blocklist -T replace -f /etc/pf.blocklist.txt

# Show table statistics
$ sudo pfctl -t blocklist -T show -v
   192.168.1.100
        Cleared:     Wed Jan 15 10:00:00 2024
        In/Block:    [ Packets: 0         Bytes: 0         ]
        In/Pass:     [ Packets: 0         Bytes: 0         ]
        Out/Block:   [ Packets: 0         Bytes: 0         ]
        Out/Pass:    [ Packets: 0         Bytes: 0         ]
```

## Viewing Firewall Status

```bash
# General info
$ sudo pfctl -s info
Status: Enabled for 0 days 05:30:22    Debug: Urgent

State Table                          Total             Rate
  current entries                       45
  searches                          123456           6.2/s
  inserts                            12345           0.6/s
  removals                           12300           0.6/s
Counters
  match                              67890           3.4/s
  bad-offset                             0           0.0/s
  fragment                               0           0.0/s
  ...

# View all rules
$ sudo pfctl -s rules

# View rules with stats
$ sudo pfctl -s rules -v
@0 block drop in all
  [ Evaluations: 50000     Packets: 1234      Bytes: 98765     States: 0     ]
@1 pass out all flags S/SA keep state
  [ Evaluations: 45000     Packets: 40000     Bytes: 5000000   States: 45    ]
...

# View state table (active connections)
$ sudo pfctl -s state
all tcp 192.168.1.100:52345 -> 93.184.216.34:443       ESTABLISHED:ESTABLISHED
all tcp 192.168.1.100:52346 -> 93.184.216.34:443       ESTABLISHED:ESTABLISHED
...

# View NAT rules
$ sudo pfctl -s nat

# View all (rules, nat, tables, etc.)
$ sudo pfctl -s all
```

## Logging

pf logging uses pflog:

```bash
# Enable logging interface
$ sudo ifconfig pflog0 create
$ sudo ifconfig pflog0 up

# View logged packets in real-time
$ sudo tcpdump -n -e -ttt -i pflog0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on pflog0, link-type PFLOG (OpenBSD pflog file), capture size 262144 bytes
 00:00:00.000000 rule 10/0(match): block in on en0: 192.168.1.50.54321 > 192.168.1.100.22: Flags [S], seq 123456789, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 123456 ecr 0,sackOK,eol], length 0

# Log to file
$ sudo tcpdump -n -e -ttt -i pflog0 -w /var/log/pflog.pcap
```

Add logging to rules:

```bash
# Log blocked packets
block in log all

# Log specific rule
pass in log on en0 proto tcp from any to any port 22

# Log all with extra detail
block in log (all) from any to any
```

## Making Rules Persistent

### Method 1: Modify /etc/pf.conf

Edit the default file (be careful with Apple's anchors):

```bash
$ sudo cp /etc/pf.conf /etc/pf.conf.backup
$ sudo nano /etc/pf.conf

# Add your rules at the end
# ... (keep existing Apple anchors)

# Custom rules
block in proto tcp from any to any port 23    # Block telnet
pass in proto tcp from any to any port { 80, 443 }
```

### Method 2: Use an Anchor (Recommended)

Create a custom anchor:

```bash
# Create custom rules file
$ sudo nano /etc/pf.anchors/custom

# Contents of /etc/pf.anchors/custom:
# Custom firewall rules
table <blocklist> persist file "/etc/pf.blocklist.txt"
block in quick from <blocklist>
pass in proto tcp from any to any port { 80, 443 }
```

Add anchor to main config:

```bash
$ sudo nano /etc/pf.conf

# Add after existing anchors:
anchor "custom"
load anchor "custom" from "/etc/pf.anchors/custom"
```

### Method 3: Launch Daemon

Create a launch daemon to load pf at boot:

```bash
$ sudo nano /Library/LaunchDaemons/com.custom.pfctl.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.custom.pfctl</string>
    <key>Program</key>
    <string>/sbin/pfctl</string>
    <key>ProgramArguments</key>
    <array>
        <string>/sbin/pfctl</string>
        <string>-e</string>
        <string>-f</string>
        <string>/etc/pf.custom.conf</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

Load the daemon:

```bash
$ sudo launchctl load /Library/LaunchDaemons/com.custom.pfctl.plist
```

## Application Firewall vs pf

macOS has two firewalls:

1. **Application Firewall** (socketfilterfw) - GUI-configurable, app-based
2. **pf** - Network-level, rule-based

```bash
# Application Firewall status
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
Firewall is enabled. (State = 1)

# Enable Application Firewall
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# Block all incoming (stealth mode)
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockall on

# Allow specific app
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /Applications/MyApp.app

# List rules
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --listapps
```

Both firewalls can be used together. pf operates at a lower level.

## Common Configurations

### Web Server

```bash
# /etc/pf.anchors/webserver
set skip on lo0

# Default deny
block in all
pass out all keep state

# Allow web traffic
pass in proto tcp from any to any port { 80, 443 } keep state

# Allow SSH from admin network
pass in proto tcp from 10.0.0.0/8 to any port 22 keep state

# Allow ICMP
pass inet proto icmp icmp-type echoreq keep state
```

### Development Machine

```bash
# /etc/pf.anchors/devmachine
set skip on lo0

# Default deny
block in all
pass out all keep state

# Allow local network
pass in from 192.168.1.0/24 keep state

# Allow common dev ports
pass in proto tcp from any to any port { 3000, 4000, 5000, 8000, 8080 } keep state

# Allow AirDrop/Bonjour
pass in proto udp from any to any port { 5353 } keep state
```

### Strict Workstation

```bash
# /etc/pf.anchors/strict
set skip on lo0

table <blocklist> persist file "/etc/pf.blocklist.txt"

# Block known bad actors
block in quick from <blocklist>
block out quick to <blocklist>

# Default policies
block in all
pass out all keep state

# No incoming connections at all
# All outgoing traffic allowed
```

## Troubleshooting

### Rules Not Working

```bash
# Verify pf is enabled
$ sudo pfctl -s info | grep Status
Status: Enabled

# Check rule order (last match wins!)
$ sudo pfctl -s rules

# Test specific rule
$ sudo pfctl -s rules -v | grep -A1 "port = ssh"

# Flush and reload
$ sudo pfctl -F all
$ sudo pfctl -f /etc/pf.conf
```

### Cannot Connect After Rule Change

```bash
# Disable pf temporarily
$ sudo pfctl -d

# Or flush rules
$ sudo pfctl -F rules

# Fix your configuration
$ sudo nano /etc/pf.conf

# Reload
$ sudo pfctl -f /etc/pf.conf
$ sudo pfctl -e
```

### Viewing Dropped Packets

```bash
# Enable logging on drop rule
# In pf.conf: block in log all

# Watch pflog
$ sudo tcpdump -i pflog0 -n

# Or check statistics
$ sudo pfctl -s info | grep -A5 Counters
```

## Summary

pf is a powerful, BSD-style firewall available on macOS:

| Task | Command |
|------|---------|
| Enable | `sudo pfctl -e` |
| Disable | `sudo pfctl -d` |
| Load rules | `sudo pfctl -f /etc/pf.conf` |
| Show rules | `sudo pfctl -s rules` |
| Show state | `sudo pfctl -s state` |
| Add to table | `sudo pfctl -t name -T add IP` |
| Flush rules | `sudo pfctl -F rules` |
| Test config | `sudo pfctl -n -f /etc/pf.conf` |

Key concepts:
- **Last match wins** (unlike iptables)
- **quick** keyword makes it first-match
- **Tables** for dynamic IP lists
- **Anchors** for modular configuration
- **State tracking** with `keep state`

pf provides enterprise-grade firewall capabilities with a clean, readable syntax. Master it, and you'll have complete control over your Mac's network security.
