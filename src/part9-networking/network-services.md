# Understanding Network Services

macOS organizes network interfaces into "services"—a layer of abstraction that separates the physical hardware from the logical configuration. This chapter explains how network services work, how to manage them, and how to use advanced features like locations and VLANs.

## Network Services Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Location: "Office"                        │
├─────────────────────────────────────────────────────────────┤
│  Service: "Wi-Fi"    │  Service: "Ethernet"  │  Service:... │
│  - Device: en0       │  - Device: en1        │              │
│  - IP: DHCP          │  - IP: Static         │              │
│  - DNS: 8.8.8.8      │  - DNS: Company       │              │
│  - Proxy: None       │  - Proxy: Yes         │              │
├─────────────────────────────────────────────────────────────┤
│                    Hardware Ports                            │
│  en0 (Wi-Fi)  │  en1 (Ethernet)  │  en2 (USB)  │  ...      │
└─────────────────────────────────────────────────────────────┘
```

A **network service** is a named configuration for a hardware port. Multiple services can exist for the same hardware port (though only one can be active). Services are grouped into **locations**, which are switchable sets of network configurations.

## Listing Network Services

### All Services

```bash
# List all network services
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
USB 10/100/1000 LAN
Thunderbolt Bridge
iPhone USB
*Bluetooth PAN

# List with hardware port mapping
$ networksetup -listallhardwareports

Hardware Port: Wi-Fi
Device: en0
Ethernet Address: aa:bb:cc:dd:ee:ff

Hardware Port: USB 10/100/1000 LAN
Device: en7
Ethernet Address: bb:cc:dd:ee:ff:00

Hardware Port: Thunderbolt Bridge
Device: bridge0
Ethernet Address: cc:dd:ee:ff:00:11
```

### Service Order and Priority

Network services have a priority order. macOS uses the highest-priority active service:

```bash
# View service order
$ networksetup -listnetworkserviceorder
An asterisk (*) denotes that a network service is disabled.
(1) Wi-Fi
(Hardware Port: Wi-Fi, Device: en0)

(2) USB 10/100/1000 LAN
(Hardware Port: USB 10/100/1000 LAN, Device: en7)

(3) Thunderbolt Bridge
(Hardware Port: Thunderbolt Bridge, Device: bridge0)

(4) iPhone USB
(Hardware Port: iPhone USB, Device: en8)

# Change service order
$ sudo networksetup -ordernetworkservices "Ethernet" "Wi-Fi" "Thunderbolt Bridge"

# After reordering
$ networksetup -listnetworkserviceorder
(1) Ethernet
(Hardware Port: USB 10/100/1000 LAN, Device: en7)

(2) Wi-Fi
(Hardware Port: Wi-Fi, Device: en0)
...
```

This is useful when you want Ethernet to take priority over Wi-Fi when both are connected.

## Creating and Managing Services

### Creating a New Service

```bash
# Create a new service on a hardware port
$ sudo networksetup -createnetworkservice "Secondary Wi-Fi" "Wi-Fi"

# Now you have two services on the same hardware:
$ networksetup -listallnetworkservices | grep -i wi-fi
Wi-Fi
Secondary Wi-Fi
```

### Duplicating a Service

```bash
# Duplicate an existing service
$ sudo networksetup -duplicatenetworkservice "Wi-Fi" "Wi-Fi Backup"
```

### Renaming a Service

```bash
# Rename a service
$ sudo networksetup -renamenetworkservice "Wi-Fi" "Primary Wireless"

# Verify
$ networksetup -listallnetworkservices
Primary Wireless
...
```

### Removing a Service

```bash
# Remove a network service
$ sudo networksetup -removenetworkservice "Secondary Wi-Fi"

# Note: You cannot remove the last service on a hardware port
```

### Enabling and Disabling Services

```bash
# Disable a service
$ sudo networksetup -setnetworkserviceenabled "Bluetooth PAN" off

# Enable a service
$ sudo networksetup -setnetworkserviceenabled "Bluetooth PAN" on

# Check if enabled
$ networksetup -listallnetworkservices
# Disabled services are marked with *
*Bluetooth PAN
```

## Network Locations

Locations are named sets of network service configurations. They're useful for switching between different network environments (home, office, coffee shop).

### Listing Locations

```bash
# List all locations
$ networksetup -listlocations
Automatic
Home
Office
Travel

# Get current location
$ networksetup -getcurrentlocation
Automatic
```

### Switching Locations

```bash
# Switch to a different location
$ sudo networksetup -switchtolocation "Office"

# Verify
$ networksetup -getcurrentlocation
Office
```

### Creating and Deleting Locations

```bash
# Create a new location
$ sudo networksetup -createlocation "Coffee Shop"

# You can also create and switch in one command
$ sudo networksetup -createlocation "Hotel" populate
# 'populate' copies services from current location

# Delete a location
$ sudo networksetup -deletelocation "Coffee Shop"

# Note: Cannot delete "Automatic" or the current location
```

### Practical Location Setup

Here's a practical example of setting up locations:

```bash
# Create Office location with specific proxy settings
$ sudo networksetup -createlocation "Office" populate
$ sudo networksetup -switchtolocation "Office"
$ sudo networksetup -setwebproxy "Wi-Fi" proxy.company.com 8080
$ sudo networksetup -setsecurewebproxy "Wi-Fi" proxy.company.com 8080
$ sudo networksetup -setdnsservers "Wi-Fi" 10.0.0.1 10.0.0.2

# Create Home location with different settings
$ sudo networksetup -createlocation "Home" populate
$ sudo networksetup -switchtolocation "Home"
$ sudo networksetup -setwebproxystate "Wi-Fi" off
$ sudo networksetup -setsecurewebproxystate "Wi-Fi" off
$ sudo networksetup -setdnsservers "Wi-Fi" 8.8.8.8 8.8.4.4

# Switch back to Automatic for general use
$ sudo networksetup -switchtolocation "Automatic"
```

### Automating Location Switching

You can create a script to switch locations automatically:

```bash
#!/bin/bash
# switch-network-location.sh

SSID=$(networksetup -getairportnetwork en0 | awk -F': ' '{print $2}')

case "$SSID" in
    "OfficeWiFi")
        networksetup -switchtolocation "Office"
        ;;
    "HomeNetwork")
        networksetup -switchtolocation "Home"
        ;;
    *)
        networksetup -switchtolocation "Automatic"
        ;;
esac
```

## VLAN Configuration

VLANs (Virtual LANs) allow you to create virtual network interfaces tagged with a VLAN ID.

### Creating VLANs

```bash
# List existing VLANs
$ networksetup -listVLANs
There are no VLANs currently configured on this system

# Create a VLAN
$ sudo networksetup -createVLAN "VLAN100" "Wi-Fi" 100
#                                  name     parent tag

# Verify creation
$ networksetup -listVLANs
VLAN User Defined Name: VLAN100
Parent Device: en0
Device (BSD Name): vlan0
Tag: 100

# The VLAN appears as a network service
$ networksetup -listallnetworkservices
Wi-Fi
VLAN100
...
```

### Configuring VLAN IP

```bash
# Set VLAN to DHCP
$ sudo networksetup -setdhcp "VLAN100"

# Or set static IP
$ sudo networksetup -setmanual "VLAN100" 192.168.100.10 255.255.255.0 192.168.100.1
```

### Deleting VLANs

```bash
# Delete a VLAN
$ sudo networksetup -deleteVLAN "VLAN100" "Wi-Fi" 100

# Verify
$ networksetup -listVLANs
There are no VLANs currently configured on this system
```

### VLAN Use Cases

VLANs are useful for:

- **Network segmentation**: Isolate traffic types (management, data, voice)
- **Testing**: Connect to multiple VLANs for testing
- **Security**: Separate sensitive traffic

```bash
# Example: Create management and data VLANs
$ sudo networksetup -createVLAN "Management" "Ethernet" 10
$ sudo networksetup -createVLAN "Data" "Ethernet" 20

# Configure management VLAN
$ sudo networksetup -setmanual "Management" 10.10.10.5 255.255.255.0 10.10.10.1

# Configure data VLAN with DHCP
$ sudo networksetup -setdhcp "Data"
```

## Bond Interfaces (Link Aggregation)

Bond interfaces combine multiple network interfaces for increased bandwidth or redundancy.

### Creating a Bond

```bash
# List available Ethernet interfaces
$ networksetup -listallhardwareports | grep -A2 "Ethernet"
Hardware Port: Ethernet 1
Device: en1

Hardware Port: Ethernet 2
Device: en2

# Create bond interface
$ sudo networksetup -createBond "BondedEthernet" en1 en2

# List bonds
$ networksetup -listBonds
BondedEthernet
Device (BSD Name): bond0
Bonded Ports: en1 en2

# Configure the bond
$ sudo networksetup -setmanual "BondedEthernet" 192.168.1.10 255.255.255.0 192.168.1.1
```

### Bond Modes

macOS supports different bond modes (though configuration is limited from CLI):

- **Round-robin**: Packets transmitted in sequential order
- **Active backup**: One interface active, others standby
- **LACP (802.3ad)**: Link Aggregation Control Protocol

### Deleting a Bond

```bash
$ sudo networksetup -deleteBond "BondedEthernet"
```

## Network Service Details

### Getting Complete Service Information

```bash
# Get all information about a service
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

### MAC Address Configuration

```bash
# Get MAC (hardware) address
$ networksetup -getmacaddress "Wi-Fi"
Ethernet Address: aa:bb:cc:dd:ee:ff (Hardware Port: Wi-Fi)

# Note: macOS doesn't have a built-in way to spoof MAC addresses
# For temporary MAC spoofing (until reboot):
$ sudo ifconfig en0 ether bb:cc:dd:ee:ff:00
```

### Additional Service Settings

```bash
# Get media status
$ networksetup -getMedia "Ethernet"
Current: autoselect
Active: 1000baseT full-duplex

# Set specific media (for Ethernet)
$ sudo networksetup -setMedia "Ethernet" 1000baseT full-duplex

# View hardware port for a service
$ networksetup -listallhardwareports | grep -A1 "Wi-Fi"
Hardware Port: Wi-Fi
Device: en0
```

## Using scutil for Service Information

For lower-level service details, use `scutil`:

```bash
# Interactive mode
$ scutil
> list Setup:/Network/Service/.*
  subKey [0] = Setup:/Network/Service/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  subKey [1] = Setup:/Network/Service/YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY
  ...

> show Setup:/Network/Service/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
<dictionary> {
  IPv4 : <dictionary> {
    ConfigMethod : DHCP
  }
  IPv6 : <dictionary> {
    ConfigMethod : Automatic
  }
  Interface : <dictionary> {
    DeviceName : en0
    Hardware : AirPort
    Type : IEEE80211
    UserDefinedName : Wi-Fi
  }
  Proxies : <dictionary> {
    ExceptionsList : <array> {
      0 : *.local
      1 : 169.254/16
    }
    FTPPassive : 1
  }
}
> quit
```

## Troubleshooting Network Services

### Service Not Working

```bash
# Check if service is enabled
$ networksetup -listallnetworkservices | grep "Wi-Fi"
Wi-Fi  # No asterisk means enabled

# Check interface status
$ ifconfig en0 | grep status
	status: active

# Try removing and re-adding the service
$ sudo networksetup -removenetworkservice "Wi-Fi"
$ sudo networksetup -createnetworkservice "Wi-Fi" "Wi-Fi"
$ sudo networksetup -setdhcp "Wi-Fi"
```

### Reset Network Configuration

```bash
# Delete current location's preferences (will recreate on restart)
$ sudo rm /Library/Preferences/SystemConfiguration/preferences.plist
$ sudo rm /Library/Preferences/SystemConfiguration/NetworkInterfaces.plist

# Reboot to regenerate
$ sudo reboot
```

### Identify Active Service

```bash
# Find which service is providing connectivity
$ scutil --nwi
Network information

IPv4 network interface information
     en0 : flags      : 0x5 (IPv4,DNS)
           address    : 192.168.1.100
           reach      : 0x00000002 (Reachable)

   REACH : flags 0x00000002 (Reachable)

# The en0 interface tells you it's the Wi-Fi service
```

## Summary

| Task | Command |
|------|---------|
| List services | `networksetup -listallnetworkservices` |
| View service order | `networksetup -listnetworkserviceorder` |
| Change service priority | `networksetup -ordernetworkservices` |
| Create service | `networksetup -createnetworkservice` |
| Remove service | `networksetup -removenetworkservice` |
| List locations | `networksetup -listlocations` |
| Switch location | `networksetup -switchtolocation` |
| Create location | `networksetup -createlocation` |
| Create VLAN | `networksetup -createVLAN` |
| Create bond | `networksetup -createBond` |

Key concepts:

- **Services** are logical configurations for hardware ports
- **Locations** are switchable sets of service configurations
- **VLANs** allow traffic separation on the same physical interface
- **Bonds** combine interfaces for bandwidth or redundancy
- **Service order** determines which connection takes priority
