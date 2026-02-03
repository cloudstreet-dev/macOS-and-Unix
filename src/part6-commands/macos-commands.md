# Essential macOS-Specific Commands

macOS includes many commands that don't exist on Linux—tools designed for Apple's ecosystem. These commands interact with macOS services, Spotlight search, the clipboard, and system configuration. Mastering them makes you significantly more productive on macOS.

## Clipboard: pbcopy and pbpaste

The pasteboard (clipboard) commands let you pipe data to and from the system clipboard:

```bash
# Copy command output to clipboard
$ ls -la | pbcopy

# Copy file contents to clipboard
$ pbcopy < ~/.ssh/id_rsa.pub

# Paste clipboard contents
$ pbpaste

# Paste to a file
$ pbpaste > clipped.txt

# Use clipboard contents in a command
$ pbpaste | wc -l

# Transform clipboard contents
$ pbpaste | sort | uniq | pbcopy
```

Real-world examples:

```bash
# Copy current directory path
$ pwd | pbcopy

# Copy a file's contents with line numbers
$ cat -n script.sh | pbcopy

# Copy output of a complex command
$ git diff HEAD~5 | pbcopy

# Convert clipboard to uppercase
$ pbpaste | tr '[:lower:]' '[:upper:]' | pbcopy

# Copy just filenames from ls output
$ ls | pbcopy

# Format JSON from clipboard
$ pbpaste | python -m json.tool | pbcopy
```

Linux equivalents require additional tools:

```bash
# Linux clipboard (X11)
$ command | xclip -selection clipboard
$ xclip -selection clipboard -o

# Linux clipboard (Wayland)
$ command | wl-copy
$ wl-paste
```

## open: Launch Files and Applications

The `open` command launches files with their default application:

```bash
# Open file with default application
$ open document.pdf
$ open image.png
$ open movie.mp4

# Open current directory in Finder
$ open .

# Open URL in default browser
$ open https://apple.com

# Open specific application
$ open -a "Visual Studio Code" myfile.txt
$ open -a Safari https://github.com

# Open with specific application (by bundle ID)
$ open -b com.apple.TextEdit file.txt

# Reveal file in Finder (don't open, just show)
$ open -R myfile.txt

# Open new instance of an application
$ open -n -a "Google Chrome"

# Open and wait until app closes
$ open -W document.pdf    # Script pauses until you close Preview

# Open multiple files
$ open *.txt

# Open in background (don't bring to front)
$ open -g file.pdf
```

Open with text editors:

```bash
# Open in TextEdit
$ open -a TextEdit notes.txt
$ open -e notes.txt    # Shorthand for TextEdit

# Open in VS Code
$ open -a "Visual Studio Code" .
$ code .    # If VS Code shell command is installed
```

Linux equivalent:

```bash
# Linux
$ xdg-open document.pdf
$ xdg-open https://example.com
```

## mdfind: Spotlight Search from Terminal

`mdfind` queries the Spotlight index—much faster than `find` for searching file contents:

```bash
# Search by filename
$ mdfind -name "report.pdf"

# Search by content
$ mdfind "project deadline"

# Search in specific directory
$ mdfind -onlyin ~/Documents "meeting notes"

# Combine filename and content search
$ mdfind -name ".py" "import pandas"

# Search by file type
$ mdfind "kMDItemContentType == 'public.plain-text'"

# Find files modified today
$ mdfind 'kMDItemFSContentChangeDate >= $time.today'

# Find files by author
$ mdfind "kMDItemAuthors == 'John Smith'"

# Find images
$ mdfind "kMDItemContentTypeTree == 'public.image'"

# Find applications
$ mdfind "kMDItemKind == 'Application'"
```

Complex queries:

```bash
# Large files modified this week
$ mdfind 'kMDItemFSSize > 100000000 && kMDItemFSContentChangeDate > $time.this_week'

# PDFs containing "invoice" in Downloads
$ mdfind -onlyin ~/Downloads "kMDItemContentType == 'com.adobe.pdf' && invoice"

# Find presentations
$ mdfind "kMDItemKind == 'Keynote Document' || kMDItemKind == 'PowerPoint Presentation'"

# Count results
$ mdfind -count "kMDItemContentType == 'public.python-script'"
42
```

Useful aliases:

```bash
# Add to ~/.zshrc
alias findf='mdfind -name'
alias findtext='mdfind -onlyin .'
```

## mdls: View Spotlight Metadata

`mdls` shows all metadata Spotlight knows about a file:

```bash
$ mdls document.pdf
kMDItemAuthors                     = (
    "John Smith"
)
kMDItemContentCreationDate         = 2024-01-15 10:00:00 +0000
kMDItemContentModificationDate     = 2024-01-15 14:30:00 +0000
kMDItemContentType                 = "com.adobe.pdf"
kMDItemContentTypeTree             = (
    "com.adobe.pdf",
    "public.data",
    "public.item"
)
kMDItemDisplayName                 = "document.pdf"
kMDItemFSContentChangeDate         = 2024-01-15 14:30:00 +0000
kMDItemFSCreationDate              = 2024-01-15 10:00:00 +0000
kMDItemFSName                      = "document.pdf"
kMDItemFSSize                      = 1048576
kMDItemKind                        = "PDF Document"
kMDItemNumberOfPages               = 10
kMDItemPageHeight                  = 792
kMDItemPageWidth                   = 612
...
```

Query specific attributes:

```bash
# Get just file size
$ mdls -name kMDItemFSSize document.pdf
kMDItemFSSize = 1048576

# Get multiple attributes
$ mdls -name kMDItemContentType -name kMDItemFSSize file.txt

# Raw value (no attribute name)
$ mdls -raw -name kMDItemFSSize document.pdf
1048576

# List all photos with dimensions
$ for f in *.jpg; do
    echo "$f: $(mdls -raw -name kMDItemPixelWidth "$f")x$(mdls -raw -name kMDItemPixelHeight "$f")"
done
```

## say: Text-to-Speech

The `say` command converts text to speech:

```bash
# Basic usage
$ say "Hello, world"

# Read from stdin
$ echo "The process is complete" | say

# Read a file
$ say -f script.txt

# Different voices
$ say -v Alex "Hello"
$ say -v Samantha "Hello"
$ say -v Daniel "Hello"    # British English

# List available voices
$ say -v '?'
Alex                en_US    # Most people recognize me by my voice.
Alice               it_IT    # Salve, mi chiamo Alice e sono una voce italiana.
Alva                sv_SE    # Hej, jag heter Alva. Jag är en svensk röst.
...

# Save to audio file
$ say -o greeting.aiff "Welcome to the application"

# Save as different format
$ say -o greeting.m4a --data-format=aac "Hello"

# Adjust rate (words per minute, default ~175)
$ say -r 100 "Slow speech"
$ say -r 250 "Fast speech"
```

Practical uses:

```bash
# Notification when long command completes
$ make all && say "Build complete" || say "Build failed"

# Read documentation aloud
$ man ls | col -b | say

# Countdown timer
$ for i in 5 4 3 2 1; do say $i; sleep 1; done; say "Time is up"
```

## screencapture: Screenshots from Terminal

Take screenshots without keyboard shortcuts:

```bash
# Capture entire screen
$ screencapture screenshot.png

# Capture specific window (interactive selection)
$ screencapture -W screenshot.png

# Capture selection (interactive)
$ screencapture -s screenshot.png

# Capture to clipboard instead of file
$ screencapture -c

# Capture after delay (seconds)
$ screencapture -T 5 screenshot.png

# Capture window without shadow
$ screencapture -o -W screenshot.png

# Capture specific display (multi-monitor)
$ screencapture -D 1 screenshot.png

# Capture in different formats
$ screencapture -t pdf screenshot.pdf
$ screencapture -t jpg screenshot.jpg

# Capture and open in Preview
$ screencapture -P screenshot.png

# Silent capture (no camera sound)
$ screencapture -x screenshot.png

# Capture touch bar (if available)
$ screencapture -b touchbar.png
```

Automate screenshots:

```bash
# Timed screenshot sequence
$ for i in {1..10}; do
    screencapture -x "screen_$i.png"
    sleep 60
done

# Screenshot with timestamp
$ screencapture "screenshot_$(date +%Y%m%d_%H%M%S).png"
```

## networksetup: Network Configuration

Configure network settings from the terminal:

```bash
# List all network services
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
Bluetooth PAN
Thunderbolt Bridge
iPhone USB

# List hardware ports
$ networksetup -listallhardwareports
Hardware Port: Wi-Fi
Device: en0
Ethernet Address: aa:bb:cc:dd:ee:ff

Hardware Port: Thunderbolt 1
Device: en1
...

# Get current Wi-Fi network
$ networksetup -getairportnetwork en0
Current Wi-Fi Network: MyNetwork

# Get IP address for interface
$ networksetup -getinfo Wi-Fi
DHCP Configuration
IP address: 192.168.1.100
Subnet mask: 255.255.255.0
Router: 192.168.1.1
Client ID:
IPv6: Automatic
IPv6 IP address: none
IPv6 Router: none
Wi-Fi ID: aa:bb:cc:dd:ee:ff

# Set to DHCP
$ sudo networksetup -setdhcp Wi-Fi

# Set static IP
$ sudo networksetup -setmanual Wi-Fi 192.168.1.50 255.255.255.0 192.168.1.1

# Set DNS servers
$ sudo networksetup -setdnsservers Wi-Fi 8.8.8.8 8.8.4.4

# Get DNS servers
$ networksetup -getdnsservers Wi-Fi
8.8.8.8
8.8.4.4

# Clear custom DNS (use DHCP-provided)
$ sudo networksetup -setdnsservers Wi-Fi empty

# Turn Wi-Fi on/off
$ networksetup -setairportpower en0 on
$ networksetup -setairportpower en0 off

# Get Wi-Fi power state
$ networksetup -getairportpower en0
Wi-Fi Power (en0): On

# Join a Wi-Fi network
$ networksetup -setairportnetwork en0 "NetworkName" "password"

# Set proxy
$ sudo networksetup -setwebproxy Wi-Fi proxy.example.com 8080

# Get proxy settings
$ networksetup -getwebproxy Wi-Fi

# Disable proxy
$ sudo networksetup -setwebproxystate Wi-Fi off

# Set network service order (priority)
$ sudo networksetup -ordernetworkservices "Wi-Fi" "Ethernet" "Bluetooth PAN"
```

## scutil: System Configuration Utility

Query and modify system configuration:

```bash
# Get hostname
$ scutil --get HostName
my-mac

$ scutil --get LocalHostName
my-mac

$ scutil --get ComputerName
My Mac

# Set hostname (requires sudo)
$ sudo scutil --set HostName newname
$ sudo scutil --set LocalHostName newname
$ sudo scutil --set ComputerName "New Name"

# Check network reachability
$ scutil -r apple.com
Reachable,Direct

# Interactive mode (explore system configuration)
$ scutil
> list
  subKey [0] = Plugin:IPConfiguration
  subKey [1] = Plugin:InterfaceNamer
  subKey [2] = Setup:
  ...
> show State:/Network/Global/IPv4
<dictionary> {
  PrimaryInterface : en0
  PrimaryService : XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  Router : 192.168.1.1
}
> quit

# Get DNS configuration
$ scutil --dns
DNS configuration

resolver #1
  nameserver[0] : 192.168.1.1
  if_index : 8 (en0)
  ...

# Get proxy configuration
$ scutil --proxy
<dictionary> {
  ExceptionsList : <array> {
    0 : *.local
    1 : 169.254/16
  }
  FTPPassive : 1
  HTTPEnable : 0
  ...
}
```

## defaults: Read and Write System Preferences

The `defaults` command reads and writes macOS preference files (.plist):

```bash
# Read an application's preferences
$ defaults read com.apple.finder

# Read specific key
$ defaults read com.apple.finder ShowExternalHardDrivesOnDesktop
1

# Write a preference
$ defaults write com.apple.finder ShowExternalHardDrivesOnDesktop -bool false

# Delete a preference (revert to default)
$ defaults delete com.apple.finder ShowExternalHardDrivesOnDesktop

# Read global preferences
$ defaults read NSGlobalDomain

# Show hidden files in Finder
$ defaults write com.apple.finder AppleShowAllFiles -bool true
$ killall Finder    # Restart Finder to apply

# Disable auto-correct
$ defaults write NSGlobalDomain NSAutomaticSpellingCorrectionEnabled -bool false

# Set default screenshot location
$ defaults write com.apple.screencapture location ~/Screenshots
$ killall SystemUIServer

# Set default screenshot format
$ defaults write com.apple.screencapture type png    # png, jpg, gif, pdf

# Speed up dock animations
$ defaults write com.apple.dock autohide-time-modifier -float 0.15
$ killall Dock

# List all domains
$ defaults domains
```

Common customizations:

```bash
# Show full path in Finder title
$ defaults write com.apple.finder _FXShowPosixPathInTitle -bool true

# Disable .DS_Store on network volumes
$ defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true

# Enable text selection in Quick Look
$ defaults write com.apple.finder QLEnableTextSelection -bool true

# Expand save panel by default
$ defaults write NSGlobalDomain NSNavPanelExpandedStateForSaveMode -bool true

# Show all filename extensions
$ defaults write NSGlobalDomain AppleShowAllExtensions -bool true

# Disable the warning when changing file extension
$ defaults write com.apple.finder FXEnableExtensionChangeWarning -bool false
```

## airport: Wi-Fi Diagnostics

The airport command is hidden but powerful:

```bash
# Create alias (the path is inconvenient)
$ alias airport='/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport'

# Show current connection info
$ airport -I
     agrCtlRSSI: -52
     agrExtRSSI: 0
    agrCtlNoise: -88
    agrExtNoise: 0
          state: running
        op mode: station
     lastTxRate: 1200
        maxRate: 1200
lastAssocStatus: 0
    802.11 auth: open
      link auth: wpa2-psk
          BSSID: aa:bb:cc:dd:ee:ff
           SSID: MyNetwork
            MCS: 11
  guardInterval: 800
            NSS: 2
        channel: 149,80

# Scan for available networks
$ airport -s
                            SSID BSSID             RSSI CHANNEL HT CC SECURITY
                       MyNetwork aa:bb:cc:dd:ee:ff -52  149,+1  Y  US WPA2(PSK/AES)
                    Neighbor_5GHz bb:cc:dd:ee:ff:00 -65  36      Y  US WPA2(PSK/AES)
                  Office-Guest cc:dd:ee:ff:00:11 -70  6       Y  US WPA2(PSK/AES)

# Disconnect from current network
$ sudo airport -z
```

## Other Useful macOS Commands

### caffeinate: Prevent Sleep

```bash
# Prevent sleep while command runs
$ caffeinate -i long_running_process

# Prevent sleep for specified time (seconds)
$ caffeinate -t 3600    # 1 hour

# Prevent display sleep
$ caffeinate -d

# Prevent disk sleep
$ caffeinate -m

# Create assertion until process exits
$ caffeinate -w PID
```

### diskutil: Disk Management

```bash
# List disks
$ diskutil list

# Get info about a disk
$ diskutil info disk0

# Eject a disk
$ diskutil eject disk2

# Mount/unmount volumes
$ diskutil mount disk2s1
$ diskutil unmount disk2s1

# Format a disk (CAREFUL!)
$ diskutil eraseDisk APFS NewDisk disk2
```

### pmset: Power Management

```bash
# Show current power settings
$ pmset -g

# Show battery info
$ pmset -g batt
Now drawing from 'Battery Power'
 -InternalBattery-0 (id=123456)	78%; discharging; 3:45 remaining

# Show sleep settings
$ pmset -g custom

# Prevent sleep (requires sudo)
$ sudo pmset -a disablesleep 1

# Schedule shutdown
$ sudo pmset schedule shutdown "01/20/2024 23:00:00"
```

### textutil: Document Conversion

```bash
# Convert RTF to plain text
$ textutil -convert txt document.rtf

# Convert Word doc to HTML
$ textutil -convert html document.docx

# Convert multiple files
$ textutil -convert txt *.rtf

# Get document info
$ textutil -info document.docx
```

### sips: Image Processing

```bash
# Get image dimensions
$ sips -g pixelWidth -g pixelHeight image.jpg

# Resize image
$ sips -Z 800 image.jpg    # Max dimension 800, maintain aspect ratio

# Convert format
$ sips -s format png image.jpg --out image.png

# Rotate image
$ sips -r 90 image.jpg

# Batch resize
$ sips -Z 1200 *.jpg
```

## Summary

macOS-specific commands fill gaps that Linux handles differently:

| macOS Command | Purpose | Linux Equivalent |
|---------------|---------|------------------|
| `pbcopy/pbpaste` | Clipboard access | `xclip`, `xsel` |
| `open` | Launch files/apps | `xdg-open` |
| `mdfind` | Search file contents | `locate`, `grep -r` |
| `mdls` | View file metadata | `stat`, `file` |
| `say` | Text-to-speech | `espeak`, `festival` |
| `screencapture` | Screenshots | `gnome-screenshot`, `scrot` |
| `networksetup` | Network config | `nmcli`, `ip` |
| `scutil` | System config | Various tools |
| `defaults` | Preferences | `gsettings`, `dconf` |
| `caffeinate` | Prevent sleep | `systemd-inhibit` |

These commands are part of what makes macOS unique. Learn them, and you'll work more efficiently than users who rely solely on GUI tools.
