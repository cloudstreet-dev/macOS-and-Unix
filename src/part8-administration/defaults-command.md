# System Configuration via defaults

The `defaults` command is macOS's tool for reading and writing user preferences. Every application preference, system setting, and hidden configuration option is stored in property list (plist) files, and `defaults` gives you direct access to them. Mastering this command unlocks powerful system customization.

## How Preferences Work

macOS stores preferences in plist files:

```bash
# User preferences
~/Library/Preferences/

# System-wide preferences (require sudo)
/Library/Preferences/

# Current host preferences (hardware-specific)
~/Library/Preferences/ByHost/
```

Each application has a domain (usually its bundle identifier):

```bash
# Finder preferences
~/Library/Preferences/com.apple.finder.plist

# Safari preferences
~/Library/Preferences/com.apple.Safari.plist

# Global preferences
~/Library/Preferences/.GlobalPreferences.plist
# Also accessible as NSGlobalDomain
```

## Basic Usage

### Reading Preferences

```bash
# Read all preferences for an app
$ defaults read com.apple.finder
{
    AppleShowAllExtensions = 1;
    AppleShowAllFiles = 1;
    CreateDesktop = 1;
    ...
}

# Read specific key
$ defaults read com.apple.finder AppleShowAllFiles
1

# Read from global domain
$ defaults read NSGlobalDomain AppleShowAllExtensions
1

# Read a specific plist file
$ defaults read ~/Library/Preferences/com.apple.finder.plist

# List all domains
$ defaults domains
com.apple.AMPDevicesAgent, com.apple.AMPLibraryAgent, com.apple.AddressBook, ...

# Find domains containing "apple"
$ defaults domains | tr ',' '\n' | grep -i apple
```

### Writing Preferences

```bash
# Write a boolean value
$ defaults write com.apple.finder AppleShowAllFiles -bool true

# Write an integer
$ defaults write com.apple.dock autohide-delay -int 0

# Write a float
$ defaults write com.apple.dock autohide-time-modifier -float 0.5

# Write a string
$ defaults write com.apple.finder NewWindowTarget -string "PfLo"

# Write an array
$ defaults write com.apple.dock persistent-apps -array

# Write to global domain
$ defaults write NSGlobalDomain AppleShowAllExtensions -bool true

# Write to current host (hardware-specific)
$ defaults -currentHost write com.apple.screensaver idleTime -int 300
```

### Deleting Preferences

```bash
# Delete a specific key (revert to default)
$ defaults delete com.apple.finder AppleShowAllFiles

# Delete entire domain (dangerous!)
$ defaults delete com.apple.finder

# Be careful - there's no undo!
```

### Data Types

The `defaults` command supports these data types:

| Type | Flag | Example |
|------|------|---------|
| Boolean | `-bool` | `true`, `false`, `yes`, `no`, `1`, `0` |
| Integer | `-int` | `42`, `0`, `-1` |
| Float | `-float` | `0.5`, `1.0`, `3.14` |
| String | `-string` | `"Hello"` |
| Array | `-array` | Multiple values |
| Dict | `-dict` | Key-value pairs |
| Data | `-data` | Hex-encoded data |
| Date | `-date` | ISO 8601 format |

## Essential Finder Customizations

```bash
# Show hidden files
$ defaults write com.apple.finder AppleShowAllFiles -bool true
$ killall Finder

# Show all file extensions
$ defaults write NSGlobalDomain AppleShowAllExtensions -bool true
$ killall Finder

# Show path bar
$ defaults write com.apple.finder ShowPathbar -bool true

# Show status bar
$ defaults write com.apple.finder ShowStatusBar -bool true

# Show full POSIX path in title bar
$ defaults write com.apple.finder _FXShowPosixPathInTitle -bool true

# Keep folders on top when sorting by name
$ defaults write com.apple.finder _FXSortFoldersFirst -bool true

# Default to list view
$ defaults write com.apple.finder FXPreferredViewStyle -string "Nlsv"
# Nlsv = List, icnv = Icon, clmv = Column, glyv = Gallery

# Disable warning when changing file extension
$ defaults write com.apple.finder FXEnableExtensionChangeWarning -bool false

# Disable warning when emptying Trash
$ defaults write com.apple.finder WarnOnEmptyTrash -bool false

# Don't create .DS_Store on network volumes
$ defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true

# Don't create .DS_Store on USB volumes
$ defaults write com.apple.desktopservices DSDontWriteUSBStores -bool true

# Show icons on desktop
$ defaults write com.apple.finder ShowExternalHardDrivesOnDesktop -bool true
$ defaults write com.apple.finder ShowHardDrivesOnDesktop -bool false
$ defaults write com.apple.finder ShowMountedServersOnDesktop -bool true
$ defaults write com.apple.finder ShowRemovableMediaOnDesktop -bool true

# New Finder window shows home folder
$ defaults write com.apple.finder NewWindowTarget -string "PfHm"
$ defaults write com.apple.finder NewWindowTargetPath -string "file://${HOME}/"
# PfHm = Home, PfLo = Computer, PfDe = Desktop, PfDo = Documents

# Restart Finder to apply changes
$ killall Finder
```

## Dock Customizations

```bash
# Auto-hide the Dock
$ defaults write com.apple.dock autohide -bool true

# Remove auto-hide delay
$ defaults write com.apple.dock autohide-delay -float 0

# Speed up auto-hide animation
$ defaults write com.apple.dock autohide-time-modifier -float 0.3

# Change Dock icon size
$ defaults write com.apple.dock tilesize -int 48

# Enable magnification
$ defaults write com.apple.dock magnification -bool true
$ defaults write com.apple.dock largesize -int 64

# Position: left, bottom, right
$ defaults write com.apple.dock orientation -string "left"

# Minimize windows into application icon
$ defaults write com.apple.dock minimize-to-application -bool true

# Minimize effect: genie, scale, suck
$ defaults write com.apple.dock mineffect -string "scale"

# Don't show recent applications
$ defaults write com.apple.dock show-recents -bool false

# Clear persistent apps (empty Dock)
$ defaults write com.apple.dock persistent-apps -array

# Add spacer tiles
$ defaults write com.apple.dock persistent-apps -array-add '{"tile-type"="spacer-tile";}'

# Restart Dock to apply changes
$ killall Dock
```

## Screenshot Customizations

```bash
# Change screenshot location
$ defaults write com.apple.screencapture location -string "${HOME}/Screenshots"

# Change screenshot format (png, jpg, gif, pdf, tiff)
$ defaults write com.apple.screencapture type -string "png"

# Disable shadow in screenshots
$ defaults write com.apple.screencapture disable-shadow -bool true

# Include date in screenshot name
$ defaults write com.apple.screencapture include-date -bool true

# Change screenshot name prefix
$ defaults write com.apple.screencapture name -string "screenshot"

# Apply changes
$ killall SystemUIServer
```

## Safari Customizations

```bash
# Enable Develop menu
$ defaults write com.apple.Safari IncludeDevelopMenu -bool true
$ defaults write com.apple.Safari WebKitDeveloperExtrasEnabledPreferenceKey -bool true

# Show full URL in address bar
$ defaults write com.apple.Safari ShowFullURLInSmartSearchField -bool true

# Disable auto-fill
$ defaults write com.apple.Safari AutoFillCreditCardData -bool false
$ defaults write com.apple.Safari AutoFillPasswords -bool false

# Enable "Do Not Track"
$ defaults write com.apple.Safari SendDoNotTrackHTTPHeader -bool true

# Don't open "safe" files after downloading
$ defaults write com.apple.Safari AutoOpenSafeDownloads -bool false

# Show favorites bar
$ defaults write com.apple.Safari ShowFavoritesBar -bool true

# Enable keyboard navigation
$ defaults write com.apple.Safari WebKitTabToLinksPreferenceKey -bool true

# Restart Safari to apply
$ killall Safari
```

## System UI Customizations

```bash
# Expand save dialog by default
$ defaults write NSGlobalDomain NSNavPanelExpandedStateForSaveMode -bool true
$ defaults write NSGlobalDomain NSNavPanelExpandedStateForSaveMode2 -bool true

# Expand print dialog by default
$ defaults write NSGlobalDomain PMPrintingExpandedStateForPrint -bool true
$ defaults write NSGlobalDomain PMPrintingExpandedStateForPrint2 -bool true

# Disable auto-correct
$ defaults write NSGlobalDomain NSAutomaticSpellingCorrectionEnabled -bool false

# Disable auto-capitalization
$ defaults write NSGlobalDomain NSAutomaticCapitalizationEnabled -bool false

# Disable smart quotes
$ defaults write NSGlobalDomain NSAutomaticQuoteSubstitutionEnabled -bool false

# Disable smart dashes
$ defaults write NSGlobalDomain NSAutomaticDashSubstitutionEnabled -bool false

# Disable "natural" scrolling
$ defaults write NSGlobalDomain com.apple.swipescrolldirection -bool false

# Set key repeat rate (lower = faster)
$ defaults write NSGlobalDomain KeyRepeat -int 2

# Set delay until repeat (lower = shorter)
$ defaults write NSGlobalDomain InitialKeyRepeat -int 15

# Enable full keyboard access (Tab through all controls)
$ defaults write NSGlobalDomain AppleKeyboardUIMode -int 3

# Use dark mode
$ defaults write NSGlobalDomain AppleInterfaceStyle -string "Dark"

# Use light mode (delete dark mode setting)
$ defaults delete NSGlobalDomain AppleInterfaceStyle

# Set accent color (Graphite = -1, Red = 0, Orange = 1, etc.)
$ defaults write NSGlobalDomain AppleAccentColor -int 4
```

## Menu Bar and Control Center

```bash
# Show battery percentage
$ defaults write com.apple.menuextra.battery ShowPercent -string "YES"

# Show Bluetooth in menu bar
$ defaults write com.apple.controlcenter "NSStatusItem Visible Bluetooth" -bool true

# Show Sound in menu bar
$ defaults write com.apple.controlcenter "NSStatusItem Visible Sound" -bool true

# Show Wi-Fi in menu bar
$ defaults write com.apple.controlcenter "NSStatusItem Visible WiFi" -bool true

# Clock format (customize as needed)
$ defaults write com.apple.menuextra.clock DateFormat -string "EEE d MMM HH:mm:ss"

# Show date in menu bar
$ defaults write com.apple.menuextra.clock ShowDate -int 1

# Flash time separators
$ defaults write com.apple.menuextra.clock FlashDateSeparators -bool false
```

## Text Edit Customizations

```bash
# Default to plain text
$ defaults write com.apple.TextEdit RichText -int 0

# Open and save as UTF-8
$ defaults write com.apple.TextEdit PlainTextEncoding -int 4
$ defaults write com.apple.TextEdit PlainTextEncodingForWrite -int 4
```

## Disk Utility

```bash
# Show all devices
$ defaults write com.apple.DiskUtility SidebarShowAllDevices -bool true

# Enable advanced features
$ defaults write com.apple.DiskUtility advanced-image-options -bool true
```

## Mission Control and Spaces

```bash
# Don't automatically rearrange Spaces
$ defaults write com.apple.dock mru-spaces -bool false

# Group windows by application
$ defaults write com.apple.dock expose-group-apps -bool true

# Hot corners (actions: 0=none, 2=Mission Control, 3=App Windows, 4=Desktop, etc.)
# Bottom left corner: Mission Control
$ defaults write com.apple.dock wvous-bl-corner -int 2
$ defaults write com.apple.dock wvous-bl-modifier -int 0

# Bottom right corner: Desktop
$ defaults write com.apple.dock wvous-br-corner -int 4
$ defaults write com.apple.dock wvous-br-modifier -int 0

$ killall Dock
```

## Security and Privacy

```bash
# Require password immediately after sleep
$ defaults write com.apple.screensaver askForPassword -int 1
$ defaults write com.apple.screensaver askForPasswordDelay -int 0

# Enable firewall
$ sudo defaults write /Library/Preferences/com.apple.alf globalstate -int 1

# Enable firewall stealth mode
$ sudo defaults write /Library/Preferences/com.apple.alf stealthenabled -int 1

# Disable crash reporter dialog
$ defaults write com.apple.CrashReporter DialogType -string "none"
```

## Reading Preferences from Different Sources

```bash
# Read from app container
$ defaults read ~/Library/Containers/com.apple.Safari/Data/Library/Preferences/com.apple.Safari.plist

# Read system-wide preferences
$ sudo defaults read /Library/Preferences/com.apple.loginwindow

# Read from specific plist file
$ defaults read /System/Library/CoreServices/SystemVersion.plist

# Read raw plist (bypassing defaults)
$ plutil -p ~/Library/Preferences/com.apple.finder.plist

# Convert binary plist to XML
$ plutil -convert xml1 -o - ~/Library/Preferences/com.apple.finder.plist
```

## Finding Hidden Preferences

```bash
# Watch for preference changes
$ defaults read com.apple.finder > before.txt
# Change a setting in the GUI
$ defaults read com.apple.finder > after.txt
$ diff before.txt after.txt

# Use defaults to export, compare
$ defaults export com.apple.finder - > finder_prefs.xml

# Find keys containing a term
$ defaults read com.apple.finder | grep -i "show"

# Read all domains, search for term
$ defaults domains | tr ',' '\n' | while read domain; do
    defaults read "$domain" 2>/dev/null | grep -l "searchterm" && echo "$domain"
done
```

## Applying Changes: The Restart Requirement

Different changes require different restarts:

| Domain | Restart Command |
|--------|-----------------|
| Finder | `killall Finder` |
| Dock | `killall Dock` |
| SystemUIServer | `killall SystemUIServer` |
| Safari | `killall Safari` |
| Other apps | `killall AppName` |
| Global changes | Log out and back in |
| System changes | Restart |

## Backup and Restore Preferences

```bash
# Export preferences
$ defaults export com.apple.finder ~/finder-backup.plist

# Import preferences
$ defaults import com.apple.finder ~/finder-backup.plist

# Backup all preferences
$ cp -R ~/Library/Preferences ~/preferences-backup

# Export to human-readable format
$ defaults export com.apple.finder - | plutil -convert xml1 -o ~/finder-backup.xml -
```

## Complete Setup Script Example

```bash
#!/bin/bash
# macOS customization script

echo "Configuring macOS preferences..."

# Close System Preferences to prevent conflicts
osascript -e 'tell application "System Preferences" to quit'

# ========== Finder ==========
defaults write com.apple.finder AppleShowAllFiles -bool true
defaults write com.apple.finder AppleShowAllExtensions -bool true
defaults write com.apple.finder ShowPathbar -bool true
defaults write com.apple.finder ShowStatusBar -bool true
defaults write com.apple.finder _FXShowPosixPathInTitle -bool true
defaults write com.apple.finder _FXSortFoldersFirst -bool true
defaults write com.apple.finder FXEnableExtensionChangeWarning -bool false
defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true

# ========== Dock ==========
defaults write com.apple.dock autohide -bool true
defaults write com.apple.dock autohide-delay -float 0
defaults write com.apple.dock autohide-time-modifier -float 0.3
defaults write com.apple.dock tilesize -int 48
defaults write com.apple.dock show-recents -bool false
defaults write com.apple.dock mineffect -string "scale"

# ========== Screenshots ==========
mkdir -p "${HOME}/Screenshots"
defaults write com.apple.screencapture location -string "${HOME}/Screenshots"
defaults write com.apple.screencapture type -string "png"
defaults write com.apple.screencapture disable-shadow -bool true

# ========== Input ==========
defaults write NSGlobalDomain KeyRepeat -int 2
defaults write NSGlobalDomain InitialKeyRepeat -int 15
defaults write NSGlobalDomain AppleKeyboardUIMode -int 3
defaults write NSGlobalDomain NSAutomaticSpellingCorrectionEnabled -bool false
defaults write NSGlobalDomain NSAutomaticQuoteSubstitutionEnabled -bool false
defaults write NSGlobalDomain NSAutomaticDashSubstitutionEnabled -bool false

# ========== Safari ==========
defaults write com.apple.Safari IncludeDevelopMenu -bool true
defaults write com.apple.Safari ShowFullURLInSmartSearchField -bool true
defaults write com.apple.Safari AutoOpenSafeDownloads -bool false

# ========== Security ==========
defaults write com.apple.screensaver askForPassword -int 1
defaults write com.apple.screensaver askForPasswordDelay -int 0

# ========== Apply Changes ==========
echo "Restarting affected applications..."
for app in "Finder" "Dock" "SystemUIServer" "Safari"; do
    killall "${app}" &> /dev/null
done

echo "Done. Some changes may require logout/restart."
```

## Summary

The `defaults` command gives you complete control over macOS preferences:

| Task | Command |
|------|---------|
| Read all preferences | `defaults read domain` |
| Read specific key | `defaults read domain key` |
| Write value | `defaults write domain key -type value` |
| Delete key | `defaults delete domain key` |
| Export preferences | `defaults export domain file.plist` |
| Import preferences | `defaults import domain file.plist` |
| List domains | `defaults domains` |
| Find preferences | `defaults find searchterm` |

Key domains:

| Domain | Purpose |
|--------|---------|
| `com.apple.finder` | Finder preferences |
| `com.apple.dock` | Dock preferences |
| `com.apple.Safari` | Safari preferences |
| `NSGlobalDomain` | Global/system-wide preferences |
| `com.apple.screencapture` | Screenshot preferences |
| `com.apple.desktopservices` | Desktop services (.DS_Store, etc.) |

Remember to restart the appropriate application or service after making changes. With `defaults`, you can automate your entire macOS setup and maintain consistent configurations across machines.
