# Native Installers: pkg and dmg

macOS has its own installation formats that predate third-party package managers. Understanding `.pkg` installers and `.dmg` disk images helps you manage software that doesn't come through Homebrew or MacPorts, including Xcode Command Line Tools and many commercial applications.

## Package Types Overview

| Format | Description | Management |
|--------|-------------|------------|
| `.pkg` | Installer package | pkgutil, installer |
| `.dmg` | Disk image containing `.app` | hdiutil, manual |
| `.app` | Application bundle | Manual drag to /Applications |
| `.mpkg` | Meta-package (multiple .pkg) | installer |

## .pkg Installers

### What pkg Files Contain

A `.pkg` file is an archive containing:
- Payload (files to install)
- Scripts (pre/post install)
- Bill of Materials (file manifest)
- Package info

```bash
# Examine pkg contents
$ pkgutil --payload-files /path/to/package.pkg
$ pkgutil --bom /path/to/package.pkg
```

### Installing pkg Files

```bash
# GUI installation
$ open package.pkg

# Command-line installation (requires sudo)
$ sudo installer -pkg package.pkg -target /
installer: Package name is My Package
installer: Installing at base path /
installer: The install was successful.

# Install to specific volume
$ sudo installer -pkg package.pkg -target /Volumes/OtherDisk

# Verbose installation
$ sudo installer -pkg package.pkg -target / -verbose
```

### Managing Installed Packages

```bash
# List all installed packages
$ pkgutil --pkgs
com.apple.pkg.CLTools_Executables
com.apple.pkg.CLTools_SDK_macOS14
...

# Filter by pattern
$ pkgutil --pkgs | grep -i xcode

# Package information
$ pkgutil --pkg-info com.apple.pkg.CLTools_Executables
package-id: com.apple.pkg.CLTools_Executables
version: 15.0.0.0.1.1694021235
volume: /
location: /
install-time: 1694000000

# Files installed by a package
$ pkgutil --files com.apple.pkg.CLTools_Executables | head
Library/Developer/CommandLineTools/
Library/Developer/CommandLineTools/SDKs/
Library/Developer/CommandLineTools/SDKs/MacOSX.sdk
...

# Find which package owns a file
$ pkgutil --file-info /usr/bin/git
volume: /
path: /usr/bin/git
pkgid: com.apple.pkg.CLTools_Executables
...
```

### Removing Packages

macOS doesn't have a built-in package uninstaller. Removal requires:

```bash
# 1. Find installed files
$ pkgutil --files com.example.package > /tmp/files.txt

# 2. Review the file list
$ cat /tmp/files.txt

# 3. Remove files (carefully!)
$ cd /
$ sudo rm -rf $(pkgutil --files com.example.package)

# 4. Forget the package receipt
$ sudo pkgutil --forget com.example.package
```

**Warning**: Be very careful with `rm -rf`. Always review file lists first.

### Xcode Command Line Tools

The most common `.pkg` on developer Macs:

```bash
# Check if installed
$ xcode-select -p
/Library/Developer/CommandLineTools

# Install
$ xcode-select --install
# Opens GUI prompt

# Or via softwareupdate
$ softwareupdate --list | grep "Command Line Tools"
$ softwareupdate -i "Command Line Tools for Xcode-15.0"

# Switch between Xcode and CLI tools
$ sudo xcode-select --switch /Applications/Xcode.app
$ sudo xcode-select --switch /Library/Developer/CommandLineTools

# Reset
$ sudo xcode-select --reset
```

### Creating pkg Files

For distributing your own software:

```bash
# Create package from directory
$ pkgbuild --root /path/to/files \
           --identifier com.example.myapp \
           --version 1.0.0 \
           --install-location /Applications \
           myapp.pkg

# With scripts
$ pkgbuild --root /path/to/files \
           --identifier com.example.myapp \
           --version 1.0.0 \
           --scripts /path/to/scripts \
           --install-location /Applications \
           myapp.pkg

# Create distribution (combines multiple packages)
$ productbuild --distribution distribution.xml \
               --package-path /path/to/packages \
               final-installer.pkg
```

## .dmg Disk Images

### Working with DMG Files

```bash
# Mount disk image
$ hdiutil attach application.dmg
/dev/disk4          GUID_partition_scheme
/dev/disk4s1        Apple_APFS
/dev/disk5s1        Apple_APFS                   /Volumes/Application

# List mounted images
$ hdiutil info | grep image-path
image-path      : /path/to/application.dmg

# Unmount
$ hdiutil detach /Volumes/Application
# or
$ hdiutil detach /dev/disk5
```

### Installing from DMG

Typical DMG workflow:

```bash
# Mount
$ hdiutil attach MyApp.dmg

# Copy app to Applications
$ cp -R /Volumes/MyApp/MyApp.app /Applications/

# Unmount
$ hdiutil detach /Volumes/MyApp

# Optional: delete dmg
$ rm MyApp.dmg
```

### DMG Scripted Installation

```bash
#!/bin/bash
# install-from-dmg.sh

DMG_PATH="$1"
APP_NAME="$2"

# Mount
MOUNT_POINT=$(hdiutil attach "$DMG_PATH" -nobrowse | grep "/Volumes" | tail -1 | cut -f3)

# Copy
cp -R "$MOUNT_POINT"/*.app /Applications/ 2>/dev/null || \
cp -R "$MOUNT_POINT/$APP_NAME" /Applications/

# Unmount
hdiutil detach "$MOUNT_POINT"

echo "Installed to /Applications"
```

### Creating DMG Files

```bash
# Create DMG from folder
$ hdiutil create -srcfolder /path/to/MyApp.app \
                 -volname "MyApp" \
                 -format UDZO \
                 MyApp.dmg

# Create empty DMG
$ hdiutil create -size 100m \
                 -fs HFS+ \
                 -volname "MyApp" \
                 MyApp.dmg

# Create read-write DMG (for customization)
$ hdiutil create -srcfolder /path/to/MyApp.app \
                 -volname "MyApp" \
                 -format UDRW \
                 MyApp-rw.dmg

# Convert to compressed read-only
$ hdiutil convert MyApp-rw.dmg -format UDZO -o MyApp.dmg
```

### Internet-Enabled DMG

DMGs can auto-extract:

```bash
# Make DMG internet-enabled
$ hdiutil internet-enable -yes MyApp.dmg
```

When downloaded via Safari, the app is extracted and DMG deleted automatically.

## Application Bundles (.app)

### Bundle Structure

```bash
$ ls -la /Applications/Safari.app/Contents/
total 24
drwxr-xr-x   7 root  wheel   224 Dec 11 11:23 .
drwxr-xr-x   3 root  wheel    96 Dec 11 11:23 ..
drwxr-xr-x   3 root  wheel    96 Dec 11 11:23 _CodeSignature
-rw-r--r--   1 root  wheel  8572 Dec 11 11:23 Info.plist
drwxr-xr-x   3 root  wheel    96 Dec 11 11:23 MacOS
-rw-r--r--   1 root  wheel     8 Dec 11 11:23 PkgInfo
drwxr-xr-x  73 root  wheel  2336 Dec 11 11:23 Resources
drwxr-xr-x   5 root  wheel   160 Dec 11 11:23 XPCServices

# Key files
/Contents/Info.plist         # Application metadata
/Contents/MacOS/             # Executable
/Contents/Resources/         # App resources
/Contents/_CodeSignature/    # Code signature
```

### App Metadata

```bash
# View Info.plist
$ plutil -p /Applications/Safari.app/Contents/Info.plist | head -20
{
  "BuildMachineOSBuild" => "23A344"
  "CFBundleDevelopmentRegion" => "English"
  "CFBundleExecutable" => "Safari"
  "CFBundleIdentifier" => "com.apple.Safari"
  "CFBundleInfoDictionaryVersion" => "6.0"
  "CFBundleName" => "Safari"
  "CFBundlePackageType" => "APPL"
  ...
}

# Get specific values
$ defaults read /Applications/Safari.app/Contents/Info.plist CFBundleVersion
17617.1.17.11.9
```

### Installing Applications

```bash
# Copy to Applications
$ cp -R /path/to/MyApp.app /Applications/

# Or per-user
$ cp -R /path/to/MyApp.app ~/Applications/

# Remove quarantine (for trusted apps)
$ xattr -d com.apple.quarantine /Applications/MyApp.app
```

### Uninstalling Applications

```bash
# Remove app bundle
$ rm -rf /Applications/MyApp.app

# Also remove:
# - ~/Library/Application Support/MyApp
# - ~/Library/Preferences/com.example.myapp.plist
# - ~/Library/Caches/com.example.myapp

# Find related files
$ mdfind "kMDItemCFBundleIdentifier == 'com.example.myapp'"
$ mdfind -name "myapp"
```

## Software Update

### Command-Line Software Updates

```bash
# List available updates
$ softwareupdate --list
Software Update found the following new or updated software:
* Label: macOS Sonoma 14.2.1-23C71
...

# Install all updates
$ sudo softwareupdate --install --all

# Install specific update
$ sudo softwareupdate --install "macOS Sonoma 14.2.1-23C71"

# Download only (don't install)
$ sudo softwareupdate --download --all

# Install and restart if needed
$ sudo softwareupdate --install --all --restart
```

### Managing Update Behavior

```bash
# Check automatic update settings
$ defaults read /Library/Preferences/com.apple.SoftwareUpdate

# Enable automatic updates
$ sudo defaults write /Library/Preferences/com.apple.SoftwareUpdate AutomaticCheckEnabled -bool true
$ sudo defaults write /Library/Preferences/com.apple.SoftwareUpdate AutomaticDownload -bool true

# Disable automatic updates
$ sudo defaults write /Library/Preferences/com.apple.SoftwareUpdate AutomaticCheckEnabled -bool false
```

## App Store from Command Line

### mas (Mac App Store CLI)

```bash
# Install mas
$ brew install mas

# Sign in (may require App Store sign-in first)
$ mas signin email@example.com

# Search
$ mas search Xcode
497799835  Xcode (15.0)

# Install by ID
$ mas install 497799835

# List installed
$ mas list

# Outdated apps
$ mas outdated

# Upgrade all
$ mas upgrade
```

## Summary

Native macOS installation methods:

| Method | Use Case | Management |
|--------|----------|------------|
| `.pkg` | System tools, CLT | pkgutil, installer |
| `.dmg` | App distribution | hdiutil |
| `.app` | Direct app bundle | cp, rm |
| App Store | Sandboxed apps | mas |
| softwareupdate | macOS/system | softwareupdate |

Key commands:

| Task | Command |
|------|---------|
| Install pkg | `sudo installer -pkg file.pkg -target /` |
| List packages | `pkgutil --pkgs` |
| Package files | `pkgutil --files com.example.pkg` |
| Forget package | `sudo pkgutil --forget com.example.pkg` |
| Mount dmg | `hdiutil attach file.dmg` |
| Unmount dmg | `hdiutil detach /Volumes/Name` |
| System updates | `softwareupdate --list` |
| Install CLT | `xcode-select --install` |

Native installers complement package managers for software that requires system-level installation or comes directly from vendors.
