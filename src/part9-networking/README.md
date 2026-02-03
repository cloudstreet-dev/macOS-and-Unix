# Networking on macOS

macOS provides a sophisticated networking stack that blends traditional BSD networking tools with Apple's modern network configuration frameworks. If you're coming from Linux, you'll find familiar concepts but different tools: there's no `ip` command, no NetworkManager, and no systemd-networkd. Instead, macOS uses a combination of BSD utilities and Apple-specific tools like `networksetup`, `scutil`, and the `configd` daemon.

## The macOS Network Architecture

At its core, macOS networking is built on:

```
┌─────────────────────────────────────────────────────────────┐
│                    Applications                              │
├─────────────────────────────────────────────────────────────┤
│                    Network Framework                         │
│              (URLSession, Network.framework)                 │
├─────────────────────────────────────────────────────────────┤
│  SystemConfiguration    │    Bonjour/mDNS    │    VPN       │
│     Framework           │   (mDNSResponder)  │   Framework  │
├─────────────────────────────────────────────────────────────┤
│                    configd daemon                            │
│              (System Configuration Agent)                    │
├─────────────────────────────────────────────────────────────┤
│                    BSD Network Stack                         │
│              (Sockets, TCP/IP, Routing)                     │
├─────────────────────────────────────────────────────────────┤
│                    XNU Kernel                                │
│        (Network interfaces, drivers, packet filtering)      │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

**configd**: The system configuration daemon that manages network state, detects changes, and notifies applications.

**mDNSResponder**: Handles Bonjour/mDNS, multicast DNS, and DNS service discovery. Also serves as the local DNS resolver.

**SystemConfiguration Framework**: Provides APIs for network configuration and state monitoring.

## Command-Line Tools Overview

macOS provides several categories of networking tools:

### Configuration Tools

| Tool | Purpose |
|------|---------|
| `networksetup` | High-level network service configuration |
| `ipconfig` | DHCP and IP configuration |
| `ifconfig` | Interface configuration (BSD) |
| `scutil` | System configuration utility |

### Diagnostic Tools

| Tool | Purpose |
|------|---------|
| `ping` | Test host reachability |
| `traceroute` | Trace packet route |
| `mtr` | Combined ping/traceroute (via Homebrew) |
| `netstat` | Network statistics |
| `nettop` | Real-time network activity |
| `tcpdump` | Packet capture |
| `networkQuality` | Network performance test (macOS 12+) |

### DNS and Discovery

| Tool | Purpose |
|------|---------|
| `dig` | DNS lookup |
| `host` | Simple DNS lookup |
| `nslookup` | DNS lookup (legacy) |
| `dscacheutil` | Directory services cache |
| `dns-sd` | DNS service discovery |

### Wi-Fi Tools

| Tool | Purpose |
|------|---------|
| `airport` | Wi-Fi diagnostics (hidden utility) |
| `networksetup` | Wi-Fi configuration |
| `wdutil` | Wireless diagnostics |

## Quick Network Status Check

Get a quick overview of your network configuration:

```bash
# Show all active interfaces with IPs
$ ifconfig | grep -E "^[a-z]|inet "
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
	inet 127.0.0.1 netmask 0xff000000
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	inet 192.168.1.100 netmask 0xffffff00 broadcast 192.168.1.255

# Show current default gateway
$ route get default | grep gateway
    gateway: 192.168.1.1

# Show DNS servers
$ scutil --dns | grep nameserver | head -5
  nameserver[0] : 192.168.1.1
  nameserver[1] : 8.8.8.8

# Show active network service
$ networksetup -listallnetworkservices | head -3
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
Ethernet
```

## Network Locations

macOS supports "Locations"—saved network configurations you can switch between:

```bash
# List locations
$ networksetup -listlocations
Automatic
Home
Office

# Get current location
$ networksetup -getcurrentlocation
Automatic

# Switch location
$ networksetup -switchtolocation "Office"
```

Locations are useful for laptops that move between different network environments with different proxy settings, DNS servers, or IP configurations.

## What You'll Learn in This Part

**[Network Configuration from CLI](./network-configuration.md)** covers the essential tools for configuring network interfaces, IP addresses, DNS, and routing using `networksetup`, `ipconfig`, `ifconfig`, and `scutil`.

**[Understanding Network Services](./network-services.md)** explains how macOS organizes network interfaces into services, how to manage network locations, and configure advanced features like VLANs.

**[Bonjour and mDNS](./bonjour-mdns.md)** introduces Apple's zero-configuration networking technology, including how to browse and advertise services using the `dns-sd` command.

**[VPN Configuration](./vpn-configuration.md)** shows how to configure and manage VPN connections from the command line, including IKEv2, L2TP/IPSec, and third-party VPN solutions.

**[Sharing Services via Command Line](./sharing-services.md)** explains how to enable and configure macOS sharing services like SSH, Screen Sharing, and File Sharing from Terminal.

**[Network Diagnostics and Troubleshooting](./diagnostics.md)** provides a comprehensive guide to troubleshooting network issues using `ping`, `traceroute`, `mtr`, `nettop`, `tcpdump`, and other diagnostic tools.

**[Wi-Fi Management from Terminal](./wifi-management.md)** covers the hidden `airport` utility and other tools for scanning networks, connecting, and diagnosing Wi-Fi issues.

## Quick Troubleshooting Guide

Common network issues and quick fixes:

### Can't Connect to Network

```bash
# Check if interface is up
$ ifconfig en0 | grep status
	status: active

# Renew DHCP lease
$ sudo ipconfig set en0 DHCP

# Restart network service
$ sudo ifconfig en0 down && sudo ifconfig en0 up
```

### DNS Not Resolving

```bash
# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder

# Check DNS servers
$ scutil --dns | grep nameserver

# Test with specific DNS server
$ dig @8.8.8.8 example.com
```

### Slow Network

```bash
# Test network quality (macOS 12+)
$ networkQuality -s

# Check for packet loss
$ ping -c 100 gateway_ip | grep "packet loss"

# Monitor real-time network activity
$ nettop -m tcp
```

### Check What's Using the Network

```bash
# See network connections by process
$ lsof -i -P | grep ESTABLISHED

# Real-time network usage by process
$ nettop -P

# See what's listening
$ lsof -i -P | grep LISTEN
```

The following chapters dive deep into each of these areas, providing the knowledge you need to configure, secure, and troubleshoot networking on macOS.
