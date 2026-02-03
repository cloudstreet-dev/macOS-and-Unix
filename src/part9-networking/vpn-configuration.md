# VPN Configuration

macOS includes built-in support for several VPN protocols and provides command-line tools for configuration and management. This chapter covers configuring VPN connections using `networksetup`, `scutil`, and managing third-party VPN solutions.

## Built-in VPN Support

macOS natively supports:

| Protocol | Description | Use Case |
|----------|-------------|----------|
| IKEv2 | Internet Key Exchange v2 | Modern, secure, recommended |
| L2TP/IPSec | Layer 2 Tunneling Protocol | Legacy, widely supported |
| Cisco IPSec | Cisco proprietary | Enterprise Cisco environments |
| PPTP | Point-to-Point Tunneling | Deprecated, insecure (removed in macOS 10.12+) |

For OpenVPN, WireGuard, and other protocols, you'll need third-party software.

## Listing VPN Connections

### Using scutil

```bash
# List all VPN services
$ scutil --nc list
Available network connection services in the current set (*=enabled):
* (Disconnected)   XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX VPN (IKEv2)      "Work VPN"
* (Disconnected)   YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY VPN (L2TP)       "Home VPN"
  (Disconnected)   ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ VPN (IPSec)      "Legacy VPN"
```

### Using networksetup

```bash
# List network services (VPNs included)
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
Work VPN
Home VPN
```

## Connecting and Disconnecting VPNs

### Using scutil

```bash
# Connect to VPN (by name)
$ scutil --nc start "Work VPN"

# Connect with password from stdin
$ scutil --nc start "Work VPN" --password

# Check connection status
$ scutil --nc status "Work VPN"
Connected
...

# Disconnect
$ scutil --nc stop "Work VPN"

# Show VPN statistics
$ scutil --nc show "Work VPN"
```

### Connection Status Details

```bash
# Detailed status
$ scutil --nc status "Work VPN"
Connected
  Extended Status <dictionary> {
    IPv4 : <dictionary> {
      Addresses : <array> {
        0 : 10.0.0.100
      }
      DestAddresses : <array> {
        0 : 10.0.0.1
      }
    }
    Status : 2
  }
```

### Monitor VPN Connection

```bash
# Watch for connection state changes
$ scutil --nc watch "Work VPN"
# Outputs status changes in real-time
```

## Creating IKEv2 VPN Connections

IKEv2 is the recommended VPN protocol for macOS. Creating IKEv2 connections from the command line requires using `networksetup` and configuring through System Configuration.

### Using networksetup

```bash
# Create IKEv2 VPN service
$ sudo networksetup -createnetworkservice "My IKEv2 VPN" VPN IKEv2

# Note: This creates the service but doesn't configure it
# Full configuration requires using profiles or System Preferences
```

### IKEv2 Configuration via Profile

For complete IKEv2 configuration, use a configuration profile (.mobileconfig):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>IKEv2</key>
            <dict>
                <key>AuthenticationMethod</key>
                <string>Certificate</string>
                <key>ChildSecurityAssociationParameters</key>
                <dict>
                    <key>DiffieHellmanGroup</key>
                    <integer>14</integer>
                    <key>EncryptionAlgorithm</key>
                    <string>AES-256</string>
                    <key>IntegrityAlgorithm</key>
                    <string>SHA2-256</string>
                    <key>LifeTimeInMinutes</key>
                    <integer>1440</integer>
                </dict>
                <key>IKESecurityAssociationParameters</key>
                <dict>
                    <key>DiffieHellmanGroup</key>
                    <integer>14</integer>
                    <key>EncryptionAlgorithm</key>
                    <string>AES-256</string>
                    <key>IntegrityAlgorithm</key>
                    <string>SHA2-256</string>
                    <key>LifeTimeInMinutes</key>
                    <integer>1440</integer>
                </dict>
                <key>LocalIdentifier</key>
                <string>user@example.com</string>
                <key>RemoteAddress</key>
                <string>vpn.example.com</string>
                <key>RemoteIdentifier</key>
                <string>vpn.example.com</string>
            </dict>
            <key>PayloadType</key>
            <string>com.apple.vpn.managed</string>
            <key>PayloadIdentifier</key>
            <string>com.example.vpn.ikev2</string>
            <key>PayloadUUID</key>
            <string>XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>UserDefinedName</key>
            <string>My IKEv2 VPN</string>
            <key>VPNType</key>
            <string>IKEv2</string>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>VPN Configuration</string>
    <key>PayloadIdentifier</key>
    <string>com.example.vpn</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

Install the profile:

```bash
# Install profile
$ open my-vpn-config.mobileconfig

# Or via profiles command (requires user approval)
$ sudo profiles install -path my-vpn-config.mobileconfig

# List installed profiles
$ profiles list

# Remove profile
$ sudo profiles remove -identifier com.example.vpn
```

## Creating L2TP/IPSec VPN Connections

L2TP/IPSec is widely supported but requires shared secret or certificate authentication.

### Configuration via Profile

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>PayloadContent</key>
    <array>
        <dict>
            <key>IPSec</key>
            <dict>
                <key>AuthenticationMethod</key>
                <string>SharedSecret</string>
                <key>LocalIdentifierType</key>
                <string>KeyID</string>
                <key>SharedSecret</key>
                <data>BASE64_ENCODED_SECRET</data>
            </dict>
            <key>PPP</key>
            <dict>
                <key>AuthName</key>
                <string>username</string>
                <key>CommRemoteAddress</key>
                <string>vpn.example.com</string>
            </dict>
            <key>PayloadType</key>
            <string>com.apple.vpn.managed</string>
            <key>PayloadIdentifier</key>
            <string>com.example.vpn.l2tp</string>
            <key>PayloadUUID</key>
            <string>XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX</string>
            <key>PayloadVersion</key>
            <integer>1</integer>
            <key>UserDefinedName</key>
            <string>My L2TP VPN</string>
            <key>VPNType</key>
            <string>L2TP</string>
        </dict>
    </array>
    <key>PayloadDisplayName</key>
    <string>L2TP VPN Configuration</string>
    <key>PayloadIdentifier</key>
    <string>com.example.vpn.l2tp.config</string>
    <key>PayloadType</key>
    <string>Configuration</string>
    <key>PayloadUUID</key>
    <string>YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY</string>
    <key>PayloadVersion</key>
    <integer>1</integer>
</dict>
</plist>
```

## VPN Configuration via scutil

For lower-level VPN management, use `scutil` interactively:

```bash
$ sudo scutil
> list
# Lists all configuration keys

> show VPN/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/IPSec
# Shows IPSec configuration for a VPN

> show VPN/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/IPv4
# Shows IPv4 configuration
```

## OpenVPN Configuration

OpenVPN requires third-party software. The most common options are:

### Tunnelblick (GUI)

```bash
# Install via Homebrew Cask
$ brew install --cask tunnelblick

# Tunnelblick uses .ovpn or .tblk configuration files
# Place configs in ~/Library/Application Support/Tunnelblick/Configurations/
```

### OpenVPN Connect (Official Client)

```bash
# Install via Homebrew Cask
$ brew install --cask openvpn-connect

# Import .ovpn configuration through the GUI
```

### Command-Line OpenVPN

```bash
# Install OpenVPN
$ brew install openvpn

# Connect using config file
$ sudo openvpn --config /path/to/config.ovpn

# Run in daemon mode
$ sudo openvpn --config /path/to/config.ovpn --daemon

# With authentication file
$ sudo openvpn --config /path/to/config.ovpn --auth-user-pass /path/to/auth.txt
```

Example OpenVPN configuration (`config.ovpn`):

```
client
dev tun
proto udp
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
cipher AES-256-CBC
auth SHA256
comp-lzo
verb 3
```

### OpenVPN as a Service

Create a launch daemon for automatic connection:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.openvpn</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/sbin/openvpn</string>
        <string>--config</string>
        <string>/etc/openvpn/client.ovpn</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardErrorPath</key>
    <string>/var/log/openvpn.log</string>
    <key>StandardOutPath</key>
    <string>/var/log/openvpn.log</string>
</dict>
</plist>
```

## WireGuard Configuration

WireGuard is a modern, fast VPN protocol:

```bash
# Install WireGuard tools
$ brew install wireguard-tools

# Install WireGuard GUI (optional)
$ brew install --cask wireguard-tools
```

### Creating WireGuard Configuration

```bash
# Generate key pair
$ wg genkey | tee privatekey | wg pubkey > publickey

# View keys
$ cat privatekey
$ cat publickey
```

Create configuration file (`/usr/local/etc/wireguard/wg0.conf`):

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### Managing WireGuard

```bash
# Bring up VPN
$ sudo wg-quick up wg0

# Check status
$ sudo wg show
interface: wg0
  public key: YOUR_PUBLIC_KEY
  private key: (hidden)
  listening port: 51820

peer: SERVER_PUBLIC_KEY
  endpoint: vpn.example.com:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 1 minute, 23 seconds ago
  transfer: 123.45 MiB received, 67.89 MiB sent

# Bring down VPN
$ sudo wg-quick down wg0
```

### WireGuard as a Service

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.wireguard.wg0</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/wg-quick</string>
        <string>up</string>
        <string>wg0</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
</dict>
</plist>
```

## Split Tunneling

Split tunneling routes only specific traffic through the VPN.

### Checking Current Routes

```bash
# View routing table
$ netstat -rn

# Check if VPN is default route
$ route -n get default
   route to: default
destination: default
       mask: default
    gateway: 10.0.0.1        # VPN gateway if all traffic routed
  interface: utun0           # VPN interface
```

### Configuring Split Tunnel

For built-in VPNs, configure in System Preferences or via profile. For OpenVPN, modify the config:

```
# Route only specific networks through VPN
route 10.0.0.0 255.0.0.0
route 192.168.1.0 255.255.255.0

# Don't set VPN as default gateway
pull-filter ignore redirect-gateway
```

For WireGuard, modify `AllowedIPs`:

```ini
[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
# Only route these networks through VPN
AllowedIPs = 10.0.0.0/8, 192.168.1.0/24
```

## VPN Troubleshooting

### Check VPN Status

```bash
# List VPN interfaces
$ ifconfig | grep -E "^utun"
utun0: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1400
utun1: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1280

# Check VPN connection status
$ scutil --nc status "Work VPN"

# View VPN-related logs
$ log show --predicate 'subsystem == "com.apple.networkextension"' --last 5m
```

### Debug Connection Issues

```bash
# Test connectivity to VPN server
$ ping vpn.example.com
$ nc -zv vpn.example.com 500   # IKE
$ nc -zv vpn.example.com 4500  # NAT-T
$ nc -zv vpn.example.com 1194  # OpenVPN default

# Check if ports are blocked
$ sudo tcpdump -i en0 host vpn.example.com

# Verify DNS resolution
$ dig vpn.example.com
```

### Common Issues

**IKEv2 certificate errors:**
```bash
# View system certificates
$ security find-certificate -a -p /Library/Keychains/System.keychain

# Import CA certificate
$ sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ca.crt
```

**L2TP connection drops:**
```bash
# Check NAT-T (port 4500)
$ nc -zv vpn.example.com 4500

# May need to disable "Send all traffic over VPN" if behind NAT
```

**DNS not working over VPN:**
```bash
# Check DNS configuration
$ scutil --dns

# Flush DNS cache
$ sudo dscacheutil -flushcache
$ sudo killall -HUP mDNSResponder

# Set VPN DNS manually
$ sudo networksetup -setdnsservers "Work VPN" 10.0.0.1
```

## Scripting VPN Connections

### Auto-Connect Script

```bash
#!/bin/bash
# vpn-connect.sh - Connect to VPN with retry

VPN_NAME="Work VPN"
MAX_RETRIES=3
RETRY_DELAY=5

connect_vpn() {
    scutil --nc start "$VPN_NAME"
    sleep 3
    status=$(scutil --nc status "$VPN_NAME" | head -1)
    [ "$status" = "Connected" ]
}

for i in $(seq 1 $MAX_RETRIES); do
    echo "Attempt $i to connect to $VPN_NAME..."
    if connect_vpn; then
        echo "Connected successfully"
        exit 0
    fi
    sleep $RETRY_DELAY
done

echo "Failed to connect after $MAX_RETRIES attempts"
exit 1
```

### Disconnect All VPNs

```bash
#!/bin/bash
# vpn-disconnect-all.sh

scutil --nc list | grep "Connected" | while read line; do
    vpn_name=$(echo "$line" | sed 's/.*"\(.*\)"/\1/')
    echo "Disconnecting $vpn_name..."
    scutil --nc stop "$vpn_name"
done
```

## Summary

| Task | Command |
|------|---------|
| List VPNs | `scutil --nc list` |
| Connect | `scutil --nc start "VPN Name"` |
| Disconnect | `scutil --nc stop "VPN Name"` |
| Check status | `scutil --nc status "VPN Name"` |
| Show details | `scutil --nc show "VPN Name"` |
| Monitor | `scutil --nc watch "VPN Name"` |
| Install profile | `sudo profiles install -path config.mobileconfig` |

Key points:

- **IKEv2** is recommended for new deployments
- **L2TP/IPSec** is widely compatible but older
- **OpenVPN** and **WireGuard** require third-party tools
- **Configuration profiles** (.mobileconfig) are the best way to deploy VPN settings
- **scutil --nc** is the primary CLI tool for VPN management
- **Split tunneling** routes only specific traffic through the VPN
