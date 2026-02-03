# Filesystem Hierarchy: Where macOS Diverges

If you're coming from Linux, you expect files to be in certain places. `/etc` has configuration, `/var` has variable data, `/opt` has optional packages. macOS follows some of these conventions but diverges significantly in others. Understanding the macOS filesystem hierarchy helps you find files and understand where to put things.

## The macOS Root Filesystem

```bash
$ ls -la /
total 17
drwxr-xr-x   20 root  wheel    640 Dec 11 11:23 .
drwxr-xr-x   20 root  wheel    640 Dec 11 11:23 ..
lrwxr-xr-x    1 root  wheel     36 Dec 11 11:23 System -> /System/Volumes/Data/System
drwxrwxr-x   45 root  admin   1440 Jan 15 14:20 Applications
drwxr-xr-x   70 root  wheel   2240 Dec 11 11:23 Library
drwxr-xr-x@   9 root  wheel    288 Dec 11 11:23 System
drwxr-xr-x    6 root  wheel    192 Dec 11 11:23 Users
drwxr-xr-x    4 root  wheel    128 Jan 17 09:00 Volumes
drwxr-xr-x@   2 root  wheel     64 Dec 11 11:23 bin
drwxr-xr-x    2 root  wheel     64 Dec 11 11:23 cores
dr-xr-xr-x    4 root  wheel   5207 Jan 15 10:30 dev
lrwxr-xr-x@   1 root  wheel     11 Dec 11 11:23 etc -> private/etc
lrwxr-xr-x    1 root  wheel     25 Dec 11 11:23 home -> /System/Volumes/Data/home
drwxr-xr-x    2 root  wheel     64 Dec 11 11:23 opt
drwxr-xr-x    6 root  wheel    192 Dec 11 11:23 private
drwxr-xr-x@  64 root  wheel   2048 Dec 11 11:23 sbin
lrwxr-xr-x@   1 root  wheel     11 Dec 11 11:23 tmp -> private/tmp
drwxr-xr-x@  11 root  wheel    352 Dec 11 11:23 usr
lrwxr-xr-x@   1 root  wheel     11 Dec 11 11:23 var -> private/var
```

Note the symbolic links: `/etc`, `/tmp`, and `/var` are links to `/private/*`.

## Unix-Standard Directories

### /bin and /sbin

Essential system binaries:

```bash
$ ls /bin
[        cp       df       ed       ksh      ls       mv       rm       stty     zsh
bash     csh      echo     expr     launchctl mkdir    pax      rmdir    sync
cat      dash     hostname kill     link     ln       pwd      sh       tcsh     unlink
chmod    date     sleep    test     wait4path
```

These are Apple-provided, BSD-derived utilities. On modern macOS (Catalina+), they're read-only, protected by System Integrity Protection, and on a sealed system volume.

### /usr

User utilities and libraries:

```bash
/usr/
├── bin/          # User commands (not essential for single-user mode)
├── include/      # Header files (symlink to SDK when Xcode installed)
├── lib/          # Libraries
├── libexec/      # Support binaries for system programs
├── local/        # Local additions (Homebrew on Intel Macs)
├── sbin/         # System administration commands
├── share/        # Architecture-independent data
└── standalone/   # Standalone resources
```

**Important**: `/usr/local` is writable and is where user-installed software traditionally goes. Homebrew uses it on Intel Macs.

### /etc (→ /private/etc)

System configuration:

```bash
$ ls /etc | head
afpovertcp.cfg
apache2
asl
asl.conf
autofs.conf
auto_home
auto_master
bashrc
bashrc_Apple_Terminal
cups
```

Unlike Linux, many macOS services don't use `/etc` for configuration—they use property lists in `/Library/Preferences` or `/Library/LaunchDaemons`.

### /var (→ /private/var)

Variable data:

```bash
$ ls /var
agentx     db         empty      folders    install    log        mail       msgs       networkd   root       rpc        run        spool      tmp        vm         yp
audit      at         backups    db         empty      folders    install    jabberd    lib        log        mail       msgs       networkd   protected  root       rpc        run        screensharing  select     spool      tmp        vm         yp
```

Notable contents:
- `/var/log` — System logs (though modern macOS uses unified logging)
- `/var/db` — System databases
- `/var/folders` — Per-user temporary caches
- `/var/tmp` — Temporary files that survive reboots

### /tmp (→ /private/tmp)

Temporary files:

```bash
$ ls -la /tmp
lrwxr-xr-x@ 1 root  wheel  11 Dec 11 11:23 /tmp -> private/tmp
```

Unlike some Unix systems, `/tmp` may not be cleaned on every reboot. Use the proper temp directory API:

```bash
# Get user-specific temp directory
$ echo $TMPDIR
/var/folders/xx/xxxxxxxxxx/T/

# This is per-user and properly permissioned
```

## macOS-Specific Directories

### /Applications

GUI applications:

```bash
$ ls /Applications
App Store.app
Automator.app
Calculator.app
...
```

Applications are bundles (directories that appear as files in Finder):

```bash
$ file /Applications/Safari.app
/Applications/Safari.app: directory

$ ls /Applications/Safari.app/Contents/
_CodeSignature  Frameworks      Info.plist      MacOS           PkgInfo         Resources       XPCServices
```

### /Library

System-wide resources and configuration:

```bash
/Library/
├── Application Support/   # App-specific data (system-wide)
├── Audio/                 # Audio plug-ins and presets
├── Caches/               # System caches
├── ColorPickers/         # Color picker modules
├── Components/           # System components
├── Contextual Menu Items/
├── CoreMediaIO/          # Video device drivers
├── Desktop Pictures/     # System wallpapers
├── Documentation/        # System documentation
├── Extensions/           # Kernel extensions (deprecated)
├── Fonts/                # System-wide fonts
├── Frameworks/           # System-wide frameworks
├── Graphics/             # Graphics drivers
├── Image Capture/        # Scanner plug-ins
├── Input Methods/        # Input method editors
├── Internet Plug-Ins/    # Browser plug-ins
├── iTunes/               # iTunes-related resources
├── Java/                 # Java installations
├── Keyboard Layouts/     # Custom keyboard layouts
├── Keychains/            # System keychain
├── LaunchAgents/         # System-wide login agents
├── LaunchDaemons/        # System daemons
├── Logs/                 # Application logs
├── Mail/                 # Mail.app resources
├── OpenDirectory/        # Directory services
├── PDF Services/         # PDF workflow actions
├── Perl/                 # Perl modules
├── Preferences/          # System-wide preferences
├── Printers/             # Printer drivers
├── PrivilegedHelperTools/ # Helper tools running as root
├── Python/               # Python packages
├── QuickLook/            # QuickLook generators
├── QuickTime/            # QuickTime components
├── Receipts/             # Package receipts
├── Ruby/                 # Ruby gems
├── Sandbox/              # Sandboxing profiles
├── Screen Savers/        # Screen savers
├── ScriptingAdditions/   # AppleScript additions
├── Scripts/              # System scripts
├── Security/             # Security resources
├── Speech/               # Speech recognition/synthesis
├── Spelling/             # Spelling dictionaries
├── Spotlight/            # Spotlight importers
├── StartupItems/         # Legacy startup items (deprecated)
├── SystemMigration/      # Migration assistant data
├── SystemProfiler/       # System profiler plugins
├── Updates/              # Software update data
├── User Pictures/        # User account pictures
├── Video/                # Video drivers
└── WebServer/            # Apache web server files
```

### /System/Library

Apple's system resources—**protected by SIP**:

```bash
$ ls /System/Library | head
Accessibility
Accounts
Address Book Plug-Ins
Ambiances
AppleRAIDConsoleUI
Assistants
Audio
Automator
BridgeSupport
CFMSupport
```

**Never modify files here.** Changes are blocked by System Integrity Protection.

### ~/Library

Per-user resources (in your home directory):

```bash
~/Library/
├── Application Support/   # App-specific data
├── Caches/               # App caches (safe to delete)
├── Containers/           # Sandboxed app data
├── Cookies/              # Browser cookies
├── Fonts/                # User-installed fonts
├── Group Containers/     # Shared sandboxed data
├── Keychains/            # User keychains
├── LaunchAgents/         # User login agents
├── Logs/                 # User app logs
├── Mail/                 # Mail.app data
├── Messages/             # Messages.app data
├── Preferences/          # User preferences (plists)
├── Saved Application State/ # App state for restore
└── Services/             # User-installed services
```

This directory is **hidden in Finder by default**. Access it:

```bash
# From Terminal
$ open ~/Library

# Or in Finder: Go → Go to Folder → ~/Library
# Or hold Option while clicking Go menu
```

### /System/Volumes

macOS Catalina+ uses multiple APFS volumes:

```bash
$ ls /System/Volumes
Data        Preboot     Recovery    Update      VM
```

- **Data**: User data, applications, mutable system data
- **Preboot**: Boot-time data
- **Recovery**: Recovery OS
- **Update**: Software update staging
- **VM**: Swap files

The system uses "firmlinks" to make these volumes appear as a unified filesystem.

### /Volumes

Mount point for all volumes:

```bash
$ ls /Volumes
Macintosh HD            # Boot volume
Macintosh HD - Data     # Data volume (if visible)
ExternalDrive           # External drives appear here
DiskImage               # Mounted disk images
```

## Comparison with FHS (Linux)

| Purpose | FHS (Linux) | macOS |
|---------|-------------|-------|
| User binaries | /usr/bin | /usr/bin |
| System binaries | /sbin | /sbin |
| Config files | /etc | /etc + /Library/Preferences |
| Variable data | /var | /var (→ /private/var) |
| Temp files | /tmp | /tmp (→ /private/tmp) + $TMPDIR |
| User homes | /home | /Users |
| Root's home | /root | /var/root |
| Mount points | /mnt, /media | /Volumes |
| Optional packages | /opt | /opt (mostly unused) |
| Third-party | /opt, /usr/local | /Applications, /usr/local |
| Libraries | /usr/lib | /usr/lib + /Library/Frameworks |
| Headers | /usr/include | /usr/include (SDK symlink) |
| Docs | /usr/share/doc | /Library/Documentation |

## Homebrew Locations

Homebrew has different install locations:

### Intel Macs

```bash
/usr/local/
├── bin/         # Symlinks to installed binaries
├── Cellar/      # Installed formula versions
├── etc/         # Configuration files
├── include/     # Header files
├── lib/         # Libraries
├── opt/         # Symlinks to latest versions
├── sbin/        # System binaries
├── share/       # Architecture-independent data
└── var/         # Variable data (logs, databases)
```

### Apple Silicon Macs

```bash
/opt/homebrew/
├── bin/
├── Cellar/
├── etc/
├── include/
├── lib/
├── opt/
├── sbin/
├── share/
└── var/
```

The different location avoids conflicts with Rosetta 2 (Intel emulation).

## Finding Files

### Using locate (if enabled)

```bash
# Enable locate database
$ sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.locate.plist

# Update database
$ sudo /usr/libexec/locate.updatedb

# Find files
$ locate nginx.conf
```

### Using mdfind (Spotlight)

```bash
# Find by name
$ mdfind -name "nginx.conf"

# Find by content
$ mdfind "error handling"

# Find by type
$ mdfind "kMDItemKind == 'Application'"
```

### Using find

```bash
# Traditional Unix find
$ find /usr -name "*.h" -type f 2>/dev/null
```

## Summary

The macOS filesystem hierarchy combines Unix traditions with Apple innovations:

**Unix-familiar**:
- `/bin`, `/sbin`, `/usr`, `/etc`, `/var`, `/tmp` work as expected
- `/usr/local` is available for local software

**macOS-specific**:
- `/Applications` for GUI applications
- `/Library` hierarchy for system/user resources
- `/System` protected by SIP
- `/Volumes` for all mounted volumes
- `/Users` instead of `/home`
- `/private` containing the actual `/etc`, `/var`, `/tmp`

**Key differences from Linux**:
- Configuration often in property lists, not `/etc` text files
- Three-tier Library structure (`/System/Library`, `/Library`, `~/Library`)
- Homebrew in `/opt/homebrew` on Apple Silicon
- Multiple APFS volumes presenting as unified filesystem

Understanding this hierarchy helps you navigate macOS, find configuration files, and work effectively across Unix and macOS environments.
