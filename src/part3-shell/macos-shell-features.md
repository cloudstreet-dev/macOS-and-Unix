# macOS-Specific Shell Features

macOS provides unique commands and capabilities that integrate the command line with the graphical system. These tools let you leverage macOS features—clipboard, Spotlight, text-to-speech, notifications—from your shell scripts and daily workflow.

## Clipboard Integration

### pbcopy and pbpaste

The pasteboard (clipboard) is accessible from the command line:

```bash
# Copy to clipboard
$ echo "Hello" | pbcopy

# Paste from clipboard
$ pbpaste
Hello

# Copy file contents
$ cat file.txt | pbcopy
# or
$ pbcopy < file.txt

# Paste to file
$ pbpaste > output.txt
```

### Practical Uses

```bash
# Copy current directory path
$ pwd | pbcopy

# Copy file contents to clipboard
$ cat ~/.ssh/id_rsa.pub | pbcopy
echo "SSH key copied to clipboard"

# Generate and copy a password
$ openssl rand -base64 12 | pbcopy

# Copy command output
$ ls -la | pbcopy

# Pipe clipboard through a command
$ pbpaste | sort | uniq | pbcopy

# Convert clipboard contents
$ pbpaste | tr '[:lower:]' '[:upper:]' | pbcopy
```

### Multiple Pasteboards

macOS has multiple pasteboards:

```bash
# General pasteboard (default)
$ pbcopy
$ pbpaste

# Find pasteboard (Cmd+E in many apps)
$ pbcopy -pboard find
$ pbpaste -pboard find

# Ruler pasteboard
$ pbcopy -pboard ruler
```

## The open Command

`open` is one of the most versatile macOS commands:

### Basic Usage

```bash
# Open file with default application
$ open document.pdf      # Opens in Preview
$ open image.png         # Opens in Preview
$ open file.html         # Opens in Safari

# Open directory in Finder
$ open .                 # Current directory
$ open ~/Documents       # Specific directory

# Open URL
$ open https://apple.com
```

### Specifying Applications

```bash
# Open with specific application
$ open -a Safari https://google.com
$ open -a "Visual Studio Code" file.txt
$ open -a Preview *.png

# Open with application bundle identifier
$ open -b com.apple.Safari https://apple.com
```

### Additional Options

```bash
# Open new instance of application
$ open -n -a Safari

# Reveal file in Finder (don't open it)
$ open -R file.txt

# Open in background
$ open -g document.pdf

# Wait for application to close before returning
$ open -W document.pdf

# Edit with TextEdit
$ open -e file.txt

# Read from stdin
$ ls -la | open -f    # Opens output in TextEdit
```

### Opening System Preferences

```bash
# Open System Preferences
$ open "x-apple.systempreferences:"

# Open specific preference pane
$ open "x-apple.systempreferences:com.apple.preference.security"
$ open "x-apple.systempreferences:com.apple.preference.network"
$ open "x-apple.systempreferences:com.apple.preferences.Bluetooth"

# Apple Silicon: System Settings uses different IDs
$ open "x-apple.systempreferences:com.apple.settings.PrivacySecurity.extension"
```

## Spotlight from Command Line

### mdfind

Search using Spotlight:

```bash
# Basic search
$ mdfind "search term"

# Search by filename
$ mdfind -name "document.pdf"

# Search in specific directory
$ mdfind -onlyin ~/Documents "budget"

# Search by file type
$ mdfind "kMDItemContentType == 'public.jpeg'"

# Search by date
$ mdfind "kMDItemLastUsedDate > $time.today(-7)"

# Live query (continues watching)
$ mdfind -live "kMDItemContentType == 'public.jpeg'"
```

### Common Spotlight Attributes

```bash
# Find all PDFs
$ mdfind "kMDItemContentType == 'com.adobe.pdf'"

# Find images
$ mdfind "kMDItemContentType == 'public.image'"

# Find by author
$ mdfind "kMDItemAuthors == 'John Smith'"

# Find large files
$ mdfind "kMDItemFSSize > 100000000"  # > 100MB

# Find files modified today
$ mdfind "kMDItemFSContentChangeDate > $time.today"

# Find apps
$ mdfind "kMDItemContentType == 'com.apple.application-bundle'"
```

### mdls - Spotlight Metadata

View metadata for a file:

```bash
$ mdls document.pdf
kMDItemContentType             = "com.adobe.pdf"
kMDItemContentTypeTree         = (
    "com.adobe.pdf",
    "public.data",
    "public.item",
    "public.content"
)
kMDItemDisplayName             = "document.pdf"
kMDItemFSContentChangeDate     = 2024-01-15 10:30:00 +0000
kMDItemFSCreationDate          = 2024-01-10 08:00:00 +0000
kMDItemFSName                  = "document.pdf"
kMDItemFSSize                  = 1048576
...

# Get specific attribute
$ mdls -name kMDItemFSSize document.pdf
kMDItemFSSize = 1048576
```

### mdutil - Spotlight Indexing

Control Spotlight indexing:

```bash
# Check indexing status
$ mdutil -s /
/:
    Indexing enabled.

# Disable indexing for volume
$ sudo mdutil -i off /Volumes/External

# Erase and rebuild index
$ sudo mdutil -E /

# Enable indexing
$ sudo mdutil -i on /
```

## Text-to-Speech

### say Command

```bash
# Basic speech
$ say "Hello, world"

# Specify voice
$ say -v Samantha "Hello"
$ say -v Daniel "Hello"  # British
$ say -v Kyoko "こんにちは"  # Japanese

# List available voices
$ say -v '?'

# Speak file contents
$ say -f document.txt

# Save to audio file
$ say "Hello" -o hello.aiff
$ say "Hello" -o hello.m4a --data-format=aac

# Speak at different rate (words per minute)
$ say -r 200 "Fast speech"
$ say -r 100 "Slow speech"
```

### Practical Uses

```bash
# Notify when long command finishes
$ make build; say "Build complete"

# Read log file
$ tail -f /var/log/system.log | while read line; do say "$line"; done

# Speak clipboard
$ pbpaste | say

# Alarm
$ sleep 300; say "Time's up"
```

## Screen and Window Management

### screencapture

Take screenshots from command line:

```bash
# Capture entire screen
$ screencapture screenshot.png

# Capture selection (interactive)
$ screencapture -i screenshot.png

# Capture specific window (click to select)
$ screencapture -W screenshot.png

# Capture with delay (seconds)
$ screencapture -T 5 screenshot.png

# Capture to clipboard
$ screencapture -c

# Capture without shadow
$ screencapture -o screenshot.png

# Capture in different format
$ screencapture -t jpg screenshot.jpg
$ screencapture -t pdf screenshot.pdf
```

### screen Recording

```bash
# Basic screen recording (stop with Ctrl+C)
$ screencapture -v recording.mov

# Record specific display
$ screencapture -D 1 -v recording.mov

# Record specific area
$ screencapture -R 0,0,800,600 -v recording.mov
```

## Notifications

### osascript for Notifications

```bash
# Display notification
$ osascript -e 'display notification "Hello" with title "My Script"'

# With subtitle and sound
$ osascript -e 'display notification "Task complete" with title "Build" subtitle "Project X" sound name "Glass"'
```

### terminal-notifier (Third-party)

```bash
# Install
$ brew install terminal-notifier

# Use
$ terminal-notifier -message "Hello" -title "My Script"

# With action
$ terminal-notifier -message "Click to open" -title "Alert" -open "https://apple.com"

# With sound
$ terminal-notifier -message "Done" -title "Build" -sound default
```

## System Information

### sw_vers

macOS version information:

```bash
$ sw_vers
ProductName:		macOS
ProductVersion:		14.0
BuildVersion:		23A344
```

### system_profiler

Detailed system information:

```bash
# Full report (very verbose)
$ system_profiler

# Specific data types
$ system_profiler SPHardwareDataType
$ system_profiler SPSoftwareDataType
$ system_profiler SPNetworkDataType
$ system_profiler SPStorageDataType

# List available data types
$ system_profiler -listDataTypes

# Machine-readable output
$ system_profiler -json SPHardwareDataType
```

### sysctl

Kernel parameters:

```bash
# CPU info
$ sysctl -n machdep.cpu.brand_string
Apple M1 Pro

# Memory size
$ sysctl -n hw.memsize
17179869184

# Number of CPUs
$ sysctl -n hw.ncpu
10

# Kernel version
$ sysctl -n kern.osrelease
23.0.0
```

## Power Management

### pmset

Power management settings:

```bash
# View current settings
$ pmset -g
System-wide power settings:
Currently in use:
 standby              1
 Sleep On Power Button 1
 hibernatefile        /var/vm/sleepimage
...

# Prevent sleep for a command
$ caffeinate -i long_running_command

# Prevent sleep for duration (seconds)
$ caffeinate -t 3600

# Prevent sleep until process exits
$ caffeinate -w $(pgrep -f "my_process")

# Schedule sleep
$ sudo pmset schedule sleep "01/20/24 22:00:00"

# Schedule wake
$ sudo pmset schedule wake "01/20/24 06:00:00"
```

## User Management

### dscl (Directory Service command line)

```bash
# List users
$ dscl . -list /Users

# User info
$ dscl . -read /Users/david

# Check group membership
$ dscl . -read /Groups/admin GroupMembership

# List groups
$ dscl . -list /Groups
```

### id

```bash
$ id
uid=501(david) gid=20(staff) groups=20(staff),12(everyone),...

$ id david
$ groups david
```

## Networking Commands

### networksetup

```bash
# List network services
$ networksetup -listallnetworkservices
An asterisk (*) denotes that a network service is disabled.
Wi-Fi
Thunderbolt Ethernet

# Get current Wi-Fi network
$ networksetup -getairportnetwork en0
Current Wi-Fi Network: MyNetwork

# Get IP address
$ networksetup -getinfo "Wi-Fi"
```

### airport

```bash
# Scan for networks
$ /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -s

# Current connection info
$ /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I

# Create alias for convenience
alias airport='/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport'
```

## Disk Commands

```bash
# List disks
$ diskutil list

# Disk info
$ diskutil info disk0

# Eject disk
$ diskutil eject /Volumes/External

# Mounting/unmounting
$ diskutil mount disk2s1
$ diskutil unmount /Volumes/External
```

## Summary

macOS-specific shell commands:

| Command | Purpose |
|---------|---------|
| `pbcopy`/`pbpaste` | Clipboard access |
| `open` | Open files/URLs/apps |
| `mdfind` | Spotlight search |
| `mdls` | View file metadata |
| `say` | Text-to-speech |
| `screencapture` | Screenshots/recordings |
| `osascript` | Run AppleScript |
| `sw_vers` | macOS version |
| `system_profiler` | System information |
| `pmset` | Power management |
| `networksetup` | Network configuration |
| `diskutil` | Disk management |

These commands bridge the gap between command-line efficiency and macOS's graphical features, enabling powerful workflows that combine the best of both worlds.
