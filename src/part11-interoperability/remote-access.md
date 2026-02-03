# Remote Access: VNC and ARD

Beyond SSH for command-line access, developers often need graphical remote access to machines. macOS includes built-in screen sharing capabilities, while accessing Linux systems typically uses VNC. This chapter covers the various remote access options available, focusing on practical setups for development workflows.

## macOS Screen Sharing Technologies

macOS provides multiple technologies for remote desktop access:

```
┌─────────────────────────────────────────────────────────────────┐
│                 macOS Remote Access Options                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Screen Sharing (VNC)     Apple Remote Desktop     SSH + X11    │
│  ├── Built-in service     ├── Commercial product  ├── X11 apps  │
│  ├── VNC compatible       ├── Mass management     ├── Headless  │
│  └── Basic features       └── Enterprise tools    └── Tunneled  │
│                                                                  │
│  Third-party clients                                             │
│  ├── Remote Desktop (Microsoft)                                  │
│  ├── TeamViewer, AnyDesk                                         │
│  └── Chrome Remote Desktop                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Screen Sharing: macOS Built-in VNC

### Enabling Screen Sharing

Through System Settings:
1. System Settings → General → Sharing
2. Enable "Screen Sharing"
3. Configure allowed users

Via command line:

```bash
# Enable Screen Sharing
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.screensharing.plist

# Check status
$ sudo launchctl list | grep screensharing
-       0       com.apple.screensharing

# Disable Screen Sharing
$ sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.screensharing.plist

# Alternative using systemsetup (older method)
$ sudo systemsetup -setremotelogin on  # SSH
# Note: Screen Sharing requires System Settings
```

### Connecting from macOS to macOS

```bash
# Using Finder
# Go → Connect to Server (Cmd+K)
# vnc://hostname.local
# or vnc://192.168.1.100

# Using open command
$ open vnc://hostname.local
$ open vnc://user@hostname.local

# Using Screen Sharing app directly
$ open -a "Screen Sharing" --args vnc://hostname.local

# Via Bonjour (automatic discovery)
# Finder sidebar shows nearby Macs automatically
```

### VNC Port and Firewall

Screen Sharing uses:
- Port 5900: VNC display 0
- Port 3283: Apple Remote Desktop agent

```bash
# Check if Screen Sharing is listening
$ lsof -i :5900
screenshar 1234 root    4u  IPv6 0x123456789      0t0  TCP *:rfb (LISTEN)

# Allow through firewall (if needed)
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /System/Library/CoreServices/Screen\ Sharing.app
```

### Connecting from Terminal (headless)

```bash
# Using open command from SSH session
$ ssh user@mac
user@mac$ open vnc://localhost  # Won't work headless

# For automated connections, use ARD's kickstart
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate -configure -allowAccessFor -allUsers -privs -all
```

## Apple Remote Desktop (ARD)

Apple Remote Desktop is a commercial tool for managing multiple Macs. Even without buying ARD, you can use some of its features.

### Enabling Remote Management

```bash
# Enable Remote Management (more features than Screen Sharing)
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate

# Full configuration
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate \
    -configure -allowAccessFor -specifiedUsers \
    -access -on \
    -privs -all \
    -users admin

# Enable for all users
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate -configure -allowAccessFor -allUsers -privs -all

# Disable Remote Management
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -deactivate -stop

# Check status
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -getaccess
```

### ARD Privileges

Different privilege levels can be assigned:

```bash
# -privs options:
# -all               All privileges
# -none              No privileges
# -TextMessages      Text messages
# -ControlObserve    Control and observe
# -SendFiles         Send files
# -DeleteFiles       Delete files
# -GenerateReports   Generate reports
# -OpenQuitApps      Open and quit apps
# -ChangeSettings    Change settings
# -RestartShutdown   Restart and shutdown
# -ShowObserve       Observe only

# Example: Control and observe only
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -configure -allowAccessFor -specifiedUsers \
    -privs -ControlObserve -users developer
```

## Accessing Mac Remotely Over the Internet

### Using SSH Tunneling

The most secure method for remote VNC access:

```bash
# On remote Mac, ensure SSH and Screen Sharing are enabled

# From your laptop, create SSH tunnel
$ ssh -L 5901:localhost:5900 user@mac-at-home.example.com

# Connect VNC to the tunnel
$ open vnc://localhost:5901

# Or combined in one step
$ ssh -f -N -L 5901:localhost:5900 user@mac.example.com && open vnc://localhost:5901
```

### SSH Config for VNC Tunnel

```bash
# ~/.ssh/config
Host mac-remote
    HostName mac.example.com
    User myuser
    LocalForward 5901 localhost:5900

# Usage:
$ ssh -f -N mac-remote  # Create tunnel in background
$ open vnc://localhost:5901
```

### Using iCloud Back to My Mac (Legacy)

Back to My Mac was discontinued, but similar functionality exists via iCloud:

```bash
# Modern approach: Use iCloud Private Relay + Screen Sharing
# Requires both Macs signed into same iCloud account
# Find in Finder sidebar or use address:
# vnc://[Apple ID email]@[Computer Name].local

# Note: This only works on local network now
# For internet access, use SSH tunnel or third-party solutions
```

### Third-Party Solutions

For easier remote access over the internet:

```bash
# Install TeamViewer
$ brew install --cask teamviewer

# Install AnyDesk
$ brew install --cask anydesk

# Chrome Remote Desktop
# Install Chrome extension and enable remote access

# Tailscale (VPN mesh network)
$ brew install --cask tailscale
# Create overlay network, then use normal VNC
$ open vnc://mac-remote.tailnet-xxxx.ts.net
```

## Connecting to Linux Systems

### VNC to Linux

Linux systems typically use TigerVNC, RealVNC, x11vnc, or similar.

```bash
# Connect to Linux VNC server from macOS
$ open vnc://linux-server:5901

# Or use a VNC client
$ brew install --cask vnc-viewer  # RealVNC viewer
$ brew install --cask tigervnc-viewer

# Command-line VNC client
$ brew install tiger-vnc
$ vncviewer linux-server:1
```

### Setting Up VNC on Linux

```bash
# On Linux (Ubuntu example)

# Install TigerVNC server
$ sudo apt install tigervnc-standalone-server

# Set VNC password
$ vncpasswd

# Start VNC server (display :1 = port 5901)
$ vncserver :1 -geometry 1920x1080 -depth 24

# Configure to start on boot (systemd)
$ sudo vim /etc/systemd/system/vncserver@.service
```

Example systemd service:

```ini
[Unit]
Description=VNC Server at display %i
After=syslog.target network.target

[Service]
Type=forking
User=developer
ExecStart=/usr/bin/vncserver :%i -geometry 1920x1080 -depth 24
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
$ sudo systemctl enable vncserver@1
$ sudo systemctl start vncserver@1
```

### Secure VNC with SSH Tunnel

VNC traffic is unencrypted by default. Always tunnel through SSH:

```bash
# ~/.ssh/config
Host linux-vnc
    HostName linux-server.example.com
    User developer
    LocalForward 5901 localhost:5901

# Connect
$ ssh -f -N linux-vnc
$ open vnc://localhost:5901
```

### x11vnc: Share Existing Display

x11vnc shares the actual display (like Screen Sharing on Mac):

```bash
# On Linux
$ sudo apt install x11vnc

# Share current display
$ x11vnc -display :0 -forever -loop -shared -rfbauth ~/.vnc/passwd

# With more options
$ x11vnc -display :0 \
    -forever \
    -loop \
    -shared \
    -rfbauth ~/.vnc/passwd \
    -ncache 10 \
    -ncache_cr \
    -noxdamage
```

## RDP: Connecting to Windows

### Microsoft Remote Desktop

```bash
# Install Microsoft Remote Desktop
$ brew install --cask microsoft-remote-desktop

# Or from Mac App Store
```

RDP provides better performance than VNC for Windows connections.

### Connecting to Windows from Terminal

```bash
# Using FreeRDP (open source RDP client)
$ brew install freerdp

# Connect to Windows
$ xfreerdp /u:username /p:password /v:windows-pc /size:1920x1080

# With additional options
$ xfreerdp /u:username /v:windows-pc \
    /size:1920x1080 \
    /bpp:32 \
    /audio-mode:0 \
    /clipboard
```

### RDP from macOS to Linux

Some Linux systems support RDP via xrdp:

```bash
# On Linux
$ sudo apt install xrdp
$ sudo systemctl enable xrdp
$ sudo systemctl start xrdp

# From macOS, use Microsoft Remote Desktop
# Connect to linux-server.local:3389
```

## X11 Forwarding

For running individual graphical Linux apps on your Mac:

### Enabling X11 Forwarding

```bash
# Install XQuartz (X11 for macOS)
$ brew install --cask xquartz

# Log out and log back in (or restart)

# Enable X11 forwarding in SSH config
Host linux-x11
    HostName linux-server.example.com
    ForwardX11 yes
    ForwardX11Trusted yes

# Or on command line
$ ssh -X linux-server  # Basic X11 forwarding
$ ssh -Y linux-server  # Trusted X11 forwarding (less secure but more compatible)
```

### Running X11 Apps

```bash
# Connect with X11 forwarding
$ ssh -Y user@linux-server

# Run graphical application
user@linux$ firefox &
user@linux$ gedit myfile.txt &

# The window appears on your Mac
```

### X11 Compression

For slow connections, enable compression:

```bash
$ ssh -Y -C user@linux-server

# In config
Host linux-slow
    HostName linux-server.example.com
    ForwardX11 yes
    ForwardX11Trusted yes
    Compression yes
```

## Remote Access Best Practices

### Security Recommendations

```bash
# 1. Never expose VNC directly to the internet
# Always use SSH tunneling or VPN

# 2. Use strong VNC passwords
$ vncpasswd  # On Linux
# macOS uses your user password

# 3. Limit access to specific users
# System Settings → Sharing → Screen Sharing → Only these users

# 4. Enable firewall
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# 5. Use Tailscale or similar for secure remote access
$ brew install --cask tailscale
# Creates encrypted WireGuard mesh network
```

### Performance Optimization

```bash
# For slow connections:

# 1. Reduce color depth
# In VNC client, use 256 colors or grayscale

# 2. Enable compression
$ ssh -Y -C user@linux-server

# 3. Reduce resolution
# Remote: System Settings → Displays → Scaled → Lower resolution

# 4. Disable animations
# Remote: System Settings → Accessibility → Display → Reduce motion

# 5. Use a dedicated VNC protocol (not VNC over HTTP)
```

### Headless Mac Administration

For Macs without displays (servers, CI machines):

```bash
# Enable SSH
$ sudo systemsetup -setremotelogin on

# Enable Screen Sharing headless
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.screensharing.plist

# Enable Remote Management
$ sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
    -activate -configure -allowAccessFor -allUsers -privs -all

# Create a virtual display (if needed for some apps)
# Some apps require a display to run
# Use headless display adapter or software solution
```

## Automation and Scripting

### Remote Commands via ARD

```bash
# Run command on remote Mac (requires ARD or kickstart configured)
# This is done from ARD client, not command line on remote

# Alternative: SSH with tmux/screen for persistent sessions
$ ssh user@mac -t "tmux attach || tmux new"
```

### Automating Screen Sharing Connections

```bash
#!/bin/bash
# connect-vnc.sh - Connect to VNC with SSH tunnel

HOST=${1:-"mac-remote"}
VNC_PORT=${2:-5901}

# Kill any existing tunnel
pkill -f "ssh.*-L.*:5900"

# Create new tunnel
ssh -f -N -L "$VNC_PORT:localhost:5900" "$HOST"

# Wait for tunnel
sleep 1

# Connect
open "vnc://localhost:$VNC_PORT"
```

### Checking Remote Mac Status

```bash
#!/bin/bash
# check-remote-mac.sh - Check if remote Mac is accessible

HOST=${1:-"mac-remote"}

echo "Checking $HOST..."

# Check SSH
if ssh -o ConnectTimeout=5 "$HOST" "echo 'SSH: OK'" 2>/dev/null; then
    echo "SSH: Accessible"
else
    echo "SSH: Not accessible"
    exit 1
fi

# Check Screen Sharing
if ssh "$HOST" "lsof -i :5900 | grep -q screensharing" 2>/dev/null; then
    echo "Screen Sharing: Running"
else
    echo "Screen Sharing: Not running"
fi

# Check user sessions
echo "Logged in users:"
ssh "$HOST" "who"
```

## Quick Reference

### Enabling Remote Access

| Method | Enable Command |
|--------|---------------|
| SSH | `sudo systemsetup -setremotelogin on` |
| Screen Sharing | System Settings → Sharing → Screen Sharing |
| Remote Management | `kickstart -activate -configure -allowAccessFor -allUsers -privs -all` |

### Connecting

| From | To | Method |
|------|-----|--------|
| macOS | macOS | `open vnc://host.local` |
| macOS | Linux | VNC client + SSH tunnel |
| macOS | Windows | Microsoft Remote Desktop |
| Any | macOS (internet) | SSH tunnel + VNC |

### Ports

| Service | Port |
|---------|------|
| VNC | 5900 (display :0), 5901 (:1), etc. |
| ARD | 3283 |
| RDP | 3389 |
| SSH | 22 |

## Summary

Remote access options depend on your use case:

| Use Case | Recommended Approach |
|----------|---------------------|
| Mac to Mac (local) | Built-in Screen Sharing |
| Mac to Mac (internet) | SSH tunnel + Screen Sharing |
| Mac to Linux | SSH + VNC through tunnel |
| Mac to Windows | Microsoft Remote Desktop |
| Headless Mac admin | SSH + Remote Management |
| Run Linux GUI apps | SSH with X11 forwarding |
| Easy internet access | Tailscale + Screen Sharing |

Key security practices:
1. Never expose VNC directly to the internet
2. Always use SSH tunnels for remote VNC
3. Consider VPN solutions like Tailscale for easier secure access
4. Use strong authentication
5. Limit which users can access screen sharing
