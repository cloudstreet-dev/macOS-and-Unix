# Appendix B: Configuration File Locations

This appendix documents where configuration files are located on macOS, how they differ from Linux conventions, and where to find common settings.

---

## macOS Directory Structure Overview

macOS uses a hybrid filesystem hierarchy combining traditional Unix paths with Apple-specific locations:

```
/
├── Applications/              # GUI applications
├── Library/                   # System-wide resources and preferences
├── System/                    # macOS system files (read-only with SIP)
├── Users/                     # User home directories
│   └── <username>/
│       ├── Applications/      # User-installed apps
│       ├── Desktop/
│       ├── Documents/
│       ├── Downloads/
│       ├── Library/           # User-specific preferences and data
│       └── ...
├── Volumes/                   # Mounted volumes
├── bin/                       # Essential user commands
├── etc/                       # System configuration (symlink to /private/etc)
├── private/                   # Actual location of /etc, /var, /tmp
├── sbin/                      # Essential system commands
├── tmp/                       # Temporary files (symlink)
├── usr/                       # Secondary hierarchy
│   ├── bin/                   # User commands
│   ├── lib/                   # Libraries
│   ├── local/                 # Locally installed software
│   └── share/                 # Architecture-independent data
└── var/                       # Variable data (symlink to /private/var)
```

---

## Shell Configuration Files

### Zsh (Default Shell)

| File | Scope | Purpose |
|------|-------|---------|
| `/etc/zshenv` | System | Read for all zsh invocations |
| `/etc/zprofile` | System | Read for login shells |
| `/etc/zshrc` | System | Read for interactive shells |
| `/etc/zlogin` | System | Read after zprofile for login shells |
| `~/.zshenv` | User | User environment (all invocations) |
| `~/.zprofile` | User | User login shell setup |
| `~/.zshrc` | User | User interactive shell config |
| `~/.zlogin` | User | User post-login commands |
| `~/.zlogout` | User | Logout cleanup |

**Load order for interactive login shell:**
1. `/etc/zshenv`
2. `~/.zshenv`
3. `/etc/zprofile`
4. `~/.zprofile`
5. `/etc/zshrc`
6. `~/.zshrc`
7. `/etc/zlogin`
8. `~/.zlogin`

### Bash

| File | Scope | Purpose |
|------|-------|---------|
| `/etc/profile` | System | Login shell initialization |
| `/etc/bashrc` | System | Non-login interactive shells |
| `~/.bash_profile` | User | Login shell (read instead of .profile) |
| `~/.bash_login` | User | Login shell (fallback) |
| `~/.profile` | User | Login shell (fallback if no .bash_profile) |
| `~/.bashrc` | User | Interactive non-login shells |
| `~/.bash_logout` | User | Logout cleanup |

> **Note**: On macOS, Terminal.app opens login shells by default, so `~/.bash_profile` is used rather than `~/.bashrc`. Many users source `.bashrc` from `.bash_profile` for consistency.

### Shell-Agnostic

| File | Purpose |
|------|---------|
| `~/.inputrc` | GNU Readline configuration |
| `/etc/paths` | System PATH directories (one per line) |
| `/etc/paths.d/*` | Additional PATH directories |
| `/etc/manpaths` | Manual page paths |
| `/etc/manpaths.d/*` | Additional manual paths |

---

## Application Preferences (Property Lists)

macOS applications store preferences in property list (plist) files:

### User Preferences

| Location | Contents |
|----------|----------|
| `~/Library/Preferences/` | User app preferences (.plist files) |
| `~/Library/Preferences/ByHost/` | Machine-specific user preferences |

Common preference files:
```
~/Library/Preferences/
├── com.apple.finder.plist           # Finder settings
├── com.apple.dock.plist             # Dock settings
├── com.apple.Terminal.plist         # Terminal.app settings
├── com.apple.Safari.plist           # Safari settings
├── .GlobalPreferences.plist         # Global user defaults
├── com.googlecode.iterm2.plist      # iTerm2 settings
└── com.microsoft.VSCode.plist       # VS Code settings
```

### System Preferences

| Location | Contents |
|----------|----------|
| `/Library/Preferences/` | System-wide app preferences |
| `/Library/Preferences/SystemConfiguration/` | Network and system config |

### Reading and Modifying Preferences

```bash
# Read all preferences for an app
$ defaults read com.apple.finder

# Read specific key
$ defaults read com.apple.finder ShowHardDrivesOnDesktop

# Write a preference
$ defaults write com.apple.finder ShowHardDrivesOnDesktop -bool true

# Delete a preference (revert to default)
$ defaults delete com.apple.finder ShowHardDrivesOnDesktop

# Export plist to XML (readable)
$ plutil -convert xml1 -o - ~/Library/Preferences/com.apple.finder.plist

# List all domains
$ defaults domains | tr ',' '\n'
```

---

## Launch Agents and Daemons

### Directories by Scope

| Location | Runs As | Loaded When |
|----------|---------|-------------|
| `~/Library/LaunchAgents/` | Current user | User logs in |
| `/Library/LaunchAgents/` | Current user | User logs in |
| `/System/Library/LaunchAgents/` | Current user | User logs in (Apple only) |
| `/Library/LaunchDaemons/` | root (or specified user) | System boot |
| `/System/Library/LaunchDaemons/` | root | System boot (Apple only) |

### Example LaunchAgent Locations

```
~/Library/LaunchAgents/
├── com.example.myagent.plist        # User-installed agent
├── homebrew.mxcl.postgresql.plist   # Homebrew service
└── com.docker.helper.plist          # Docker helper

/Library/LaunchDaemons/
├── com.example.mydaemon.plist       # Third-party daemon
├── com.docker.vmnetd.plist          # Docker networking daemon
└── org.postgresql.postgres.plist    # PostgreSQL daemon
```

### Managing Services

```bash
# List user agents
$ launchctl list

# Load an agent
$ launchctl load ~/Library/LaunchAgents/com.example.agent.plist

# Unload an agent
$ launchctl unload ~/Library/LaunchAgents/com.example.agent.plist

# Bootstrap (modern method)
$ launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.agent.plist

# Bootout (modern method)
$ launchctl bootout gui/$(id -u)/com.example.agent
```

---

## Application Support and Data

### User Application Data

| Location | Contents |
|----------|----------|
| `~/Library/Application Support/` | App data, caches, state |
| `~/Library/Caches/` | Cached data (safe to delete) |
| `~/Library/Logs/` | Application logs |
| `~/Library/Containers/` | Sandboxed app data |
| `~/Library/Group Containers/` | Shared data between related apps |
| `~/Library/Saved Application State/` | Window positions, open documents |

Common Application Support paths:
```
~/Library/Application Support/
├── Code/                    # VS Code
├── Firefox/                 # Firefox profiles
├── Google/Chrome/           # Chrome profiles
├── Slack/                   # Slack data
├── iTerm2/                  # iTerm2 data
├── JetBrains/               # IntelliJ, PyCharm, etc.
└── com.apple.TCC/           # Privacy database
```

### System Application Data

| Location | Contents |
|----------|----------|
| `/Library/Application Support/` | System-wide app data |
| `/Library/Caches/` | System caches |
| `/Library/Logs/` | System and app logs |

---

## Development Tools

### Xcode and Command Line Tools

| Location | Contents |
|----------|----------|
| `/Applications/Xcode.app/` | Xcode IDE |
| `/Library/Developer/CommandLineTools/` | CLI tools (headers, compilers) |
| `~/Library/Developer/` | User developer data |
| `~/Library/Developer/Xcode/` | Xcode user data |
| `~/Library/Developer/Xcode/DerivedData/` | Build products |
| `/Applications/Xcode.app/Contents/Developer/Platforms/` | SDKs |

### SDK and Header Locations

```bash
# Find active developer directory
$ xcode-select -p
/Library/Developer/CommandLineTools

# Find SDK path
$ xcrun --show-sdk-path
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk

# SDK locations
/Library/Developer/CommandLineTools/SDKs/
├── MacOSX.sdk -> MacOSX14.sdk
├── MacOSX14.sdk/
│   └── usr/include/        # System headers
└── MacOSX13.sdk/
```

### Homebrew

| Location (Apple Silicon) | Location (Intel) | Contents |
|--------------------------|------------------|----------|
| `/opt/homebrew/` | `/usr/local/` | Homebrew prefix |
| `/opt/homebrew/bin/` | `/usr/local/bin/` | Executable symlinks |
| `/opt/homebrew/Cellar/` | `/usr/local/Cellar/` | Installed packages |
| `/opt/homebrew/Caskroom/` | `/usr/local/Caskroom/` | Installed casks |
| `/opt/homebrew/etc/` | `/usr/local/etc/` | Configuration files |
| `/opt/homebrew/var/` | `/usr/local/var/` | Variable data |

```bash
# Find Homebrew prefix
$ brew --prefix
/opt/homebrew

# Find package location
$ brew --prefix postgresql
/opt/homebrew/opt/postgresql

# Find config file location
$ brew --prefix/etc/nginx/nginx.conf
```

### Language Version Managers

| Tool | Configuration | Versions Location |
|------|---------------|-------------------|
| pyenv | `~/.pyenv/` | `~/.pyenv/versions/` |
| rbenv | `~/.rbenv/` | `~/.rbenv/versions/` |
| nvm | `~/.nvm/` | `~/.nvm/versions/` |
| nodenv | `~/.nodenv/` | `~/.nodenv/versions/` |
| rustup | `~/.rustup/` | `~/.rustup/toolchains/` |

---

## Network Configuration

### DNS

| Location | Purpose |
|----------|---------|
| `/etc/resolv.conf` | DNS resolver configuration (often managed by macOS) |
| `/etc/hosts` | Static hostname mappings |
| `/Library/Preferences/SystemConfiguration/preferences.plist` | Network service configuration |

```bash
# View DNS configuration
$ scutil --dns

# View network configuration
$ cat /Library/Preferences/SystemConfiguration/preferences.plist | plutil -convert xml1 -o - -
```

### SSH

| Location | Purpose |
|----------|---------|
| `~/.ssh/config` | User SSH client configuration |
| `~/.ssh/known_hosts` | Known host keys |
| `~/.ssh/authorized_keys` | Keys allowed to log in as you |
| `~/.ssh/id_rsa`, `~/.ssh/id_ed25519` | Private keys |
| `~/.ssh/id_rsa.pub`, `~/.ssh/id_ed25519.pub` | Public keys |
| `/etc/ssh/ssh_config` | System-wide SSH client config |
| `/etc/ssh/sshd_config` | SSH server configuration |

### Firewall

| Location | Purpose |
|----------|---------|
| `/etc/pf.conf` | Packet Filter configuration |
| `/etc/pf.anchors/` | PF anchor rules |
| `/Library/Preferences/com.apple.alf.plist` | Application Layer Firewall settings |

---

## Security

### Keychain

| Location | Purpose |
|----------|---------|
| `~/Library/Keychains/login.keychain-db` | User login keychain |
| `/Library/Keychains/System.keychain` | System keychain |
| `/System/Library/Keychains/SystemRootCertificates.keychain` | Root certificates |

### TCC (Privacy) Database

| Location | Purpose |
|----------|---------|
| `~/Library/Application Support/com.apple.TCC/TCC.db` | User privacy permissions |
| `/Library/Application Support/com.apple.TCC/TCC.db` | System privacy permissions |

### Gatekeeper

```bash
# Gatekeeper configuration
$ spctl --status
assessments enabled

# Security assessment policy
/var/db/SystemPolicy
/var/db/SystemPolicyConfiguration
```

---

## Logs

### Unified Logging

macOS uses a unified logging system. Logs are stored in a binary format:

| Location | Purpose |
|----------|---------|
| `/var/db/diagnostics/` | Log data store |
| `/var/db/uuidtext/` | Log text data |

```bash
# View recent logs
$ log show --last 1h

# Stream live logs
$ log stream

# Filter by process
$ log show --predicate 'process == "Safari"' --last 30m

# Filter by subsystem
$ log show --predicate 'subsystem == "com.apple.network"' --last 1h
```

### Traditional Log Files

| Location | Contents |
|----------|----------|
| `/var/log/system.log` | General system log (limited) |
| `/var/log/install.log` | Installation logs |
| `/var/log/wifi.log` | Wi-Fi diagnostics |
| `~/Library/Logs/` | User application logs |
| `/Library/Logs/` | System application logs |
| `/Library/Logs/DiagnosticReports/` | Crash reports |
| `~/Library/Logs/DiagnosticReports/` | User crash reports |

---

## Comparison: macOS vs Linux

### Configuration File Locations

| Purpose | macOS | Linux |
|---------|-------|-------|
| Shell config | `~/.zshrc` | `~/.bashrc`, `~/.zshrc` |
| App preferences | `~/Library/Preferences/*.plist` | `~/.config/`, `~/.<app>` |
| App data | `~/Library/Application Support/` | `~/.local/share/`, `~/.<app>` |
| App cache | `~/Library/Caches/` | `~/.cache/` |
| System services | `/Library/LaunchDaemons/` | `/etc/systemd/system/` |
| User services | `~/Library/LaunchAgents/` | `~/.config/systemd/user/` |
| Fonts | `~/Library/Fonts/`, `/Library/Fonts/` | `~/.fonts/`, `/usr/share/fonts/` |
| Binaries | `/usr/local/bin/`, `/opt/homebrew/bin/` | `/usr/local/bin/` |
| Service config | Embedded in .plist | `/etc/<service>/` |

### Service Configuration

**Linux (systemd)**:
```
/etc/systemd/system/myservice.service
/etc/myservice/config.conf          # Separate config file
```

**macOS (launchd)**:
```
/Library/LaunchDaemons/com.example.myservice.plist
# Configuration often embedded in plist or in:
/Library/Application Support/MyService/config.conf
```

### Path Differences

| Purpose | macOS | Linux |
|---------|-------|-------|
| Package manager prefix | `/opt/homebrew/` (ARM), `/usr/local/` (Intel) | Varies by distro |
| System binaries | `/bin/`, `/sbin/`, `/usr/bin/` | Same (or merged `/usr/bin/`) |
| Temp files | `/private/tmp/` (`/tmp/` symlink) | `/tmp/` |
| Variable data | `/private/var/` (`/var/` symlink) | `/var/` |
| Optional software | `/opt/` | `/opt/` |
| Mounts | `/Volumes/` | `/mnt/`, `/media/` |

---

## Finding Configuration Files

### Using Spotlight

```bash
# Find all plist files
$ mdfind -name ".plist"

# Find config files in Library
$ mdfind -onlyin ~/Library "kMDItemFSName == '*.conf'"

# Find preference files for an app
$ mdfind "kMDItemContentType == 'com.apple.property-list'" | grep -i appname
```

### Using find

```bash
# Find all hidden config files in home
$ find ~ -maxdepth 1 -name ".*" -type f

# Find all plist files in Library
$ find ~/Library -name "*.plist" 2>/dev/null

# Find all configuration directories
$ find ~ -maxdepth 2 -type d -name ".*" 2>/dev/null
```

### Common Locations to Check

When troubleshooting an application, check these locations:

```bash
# Preferences
~/Library/Preferences/com.<company>.<app>.plist
~/Library/Preferences/<app>.plist

# Application Support
~/Library/Application Support/<App Name>/

# Caches
~/Library/Caches/<App Name>/
~/Library/Caches/com.<company>.<app>/

# Containers (sandboxed apps)
~/Library/Containers/<bundle-id>/

# Logs
~/Library/Logs/<App Name>/

# Launch Agents (if app has background services)
~/Library/LaunchAgents/com.<company>.<app>.plist
```

---

## Quick Reference: Key Locations

### User Configuration

```
~/.zshrc                              # Zsh configuration
~/.ssh/config                         # SSH client configuration
~/.gitconfig                          # Git configuration
~/Library/Preferences/                # Application preferences (plist)
~/Library/Application Support/        # Application data
~/Library/LaunchAgents/               # User launch agents
```

### System Configuration

```
/etc/hosts                            # Host name mappings
/etc/paths                            # System PATH
/etc/shells                           # Valid login shells
/etc/ssh/                             # SSH server configuration
/Library/LaunchDaemons/               # System daemons
/Library/Preferences/                 # System-wide app preferences
```

### Development

```
/Library/Developer/CommandLineTools/  # Xcode CLI tools
/opt/homebrew/                        # Homebrew (Apple Silicon)
/usr/local/                           # Homebrew (Intel) / local software
~/.pyenv/, ~/.rbenv/, ~/.nvm/         # Language version managers
```

### Homebrew Services

```
/opt/homebrew/etc/                    # Configuration files
/opt/homebrew/var/                    # Data files
/opt/homebrew/var/log/                # Log files
~/Library/LaunchAgents/homebrew.*     # Service plists
```
