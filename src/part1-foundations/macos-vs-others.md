# macOS vs Linux vs BSD: Key Differences

If you're coming to macOS from Linux or BSD, many things will feel familiar—but the differences can be surprising and sometimes frustrating. This chapter catalogs the key differences you'll encounter, explaining not just what differs but why.

## Kernel and System Architecture

### Kernel Type

| System | Kernel | Type |
|--------|--------|------|
| macOS | XNU | Hybrid (Mach + BSD) |
| Linux | Linux | Monolithic (with modules) |
| FreeBSD | FreeBSD | Monolithic (with modules) |
| OpenBSD | OpenBSD | Monolithic |

**Impact**: macOS uses Mach-O binary format instead of ELF. Tools that work with binaries (nm, objdump, ldd) have different equivalents on macOS.

```bash
# Linux: View shared libraries
$ ldd /bin/ls

# macOS: View shared libraries (different tool)
$ otool -L /bin/ls
/bin/ls:
    /usr/lib/libutil.dylib (compatibility version 1.0.0, current version 1.0.0)
    /usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
    /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1336.61.1)
```

### Init System

| System | Init System | Config Format |
|--------|-------------|---------------|
| macOS | launchd | Property lists (XML/binary) |
| Linux (modern) | systemd | Unit files (INI-like) |
| Linux (traditional) | SysVinit | Shell scripts |
| FreeBSD | rc.d | Shell scripts |

**Impact**: Service management is completely different on macOS. There's no `systemctl`, no `/etc/init.d/`, no `chkconfig`.

```bash
# Linux (systemd)
$ sudo systemctl start nginx
$ sudo systemctl enable nginx

# macOS (launchd)
$ sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.nginx.plist
$ sudo launchctl enable system/homebrew.mxcl.nginx
```

## Command-Line Tools

### BSD vs GNU Commands

macOS ships with BSD versions of common commands. These often have different options than GNU versions:

#### sed

```bash
# GNU sed (Linux): In-place edit
$ sed -i 's/old/new/g' file.txt

# BSD sed (macOS): Requires backup extension (use '' for no backup)
$ sed -i '' 's/old/new/g' file.txt
```

#### grep

```bash
# GNU grep: Perl regex with -P
$ grep -P '\d{3}-\d{4}' file.txt

# BSD grep (macOS): No -P option, use -E for extended regex
$ grep -E '[0-9]{3}-[0-9]{4}' file.txt
```

#### ls

```bash
# GNU ls: Color output with --color
$ ls --color=auto

# BSD ls (macOS): Color with -G
$ ls -G

# GNU ls: Human-readable with -h requires -l
$ ls -lh

# BSD ls: -h works without -l (but usually used with -l anyway)
$ ls -lh
```

#### date

```bash
# GNU date: Specific date formatting
$ date -d "2024-01-15" +%s

# BSD date (macOS): Different syntax
$ date -j -f "%Y-%m-%d" "2024-01-15" +%s
```

#### stat

```bash
# GNU stat
$ stat --format="%s" file.txt

# BSD stat (macOS)
$ stat -f "%z" file.txt
```

#### readlink

```bash
# GNU readlink: -f to canonicalize
$ readlink -f /path/to/symlink

# BSD readlink (macOS): No -f, but realpath exists (macOS 12.3+)
$ realpath /path/to/symlink
# Or for older macOS:
$ python3 -c "import os; print(os.path.realpath('/path/to/symlink'))"
```

### macOS-Specific Commands

macOS includes commands with no Linux/BSD equivalent:

```bash
# Copy to clipboard
$ echo "Hello" | pbcopy
$ pbpaste

# Open files with default application
$ open file.pdf
$ open -a Safari https://apple.com
$ open .  # Open current directory in Finder

# Spotlight search
$ mdfind "search term"
$ mdfind -name "filename"
$ mdfind "kMDItemContentType == 'public.jpeg'"

# Text-to-speech
$ say "Hello, world"

# Manage system preferences
$ open "x-apple.systempreferences:"

# Screen capture
$ screencapture -i screenshot.png

# Airport (Wi-Fi) utility
$ /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -s

# Disk utility
$ diskutil list
$ diskutil info disk0
```

### Network Commands

Networking tools differ significantly:

```bash
# Linux: Modern networking
$ ip addr show
$ ip route
$ ss -tuln

# macOS: BSD networking
$ ifconfig
$ netstat -rn
$ netstat -an | grep LISTEN
```

Note: `ip` doesn't exist on macOS (unless installed via Homebrew). macOS uses traditional BSD tools like `ifconfig`, `netstat`, and `route`.

```bash
# Linux: Network manager
$ nmcli device status

# macOS: networksetup
$ networksetup -listallhardwareports
$ networksetup -getinfo "Wi-Fi"
```

## Filesystem Differences

### Filesystem Types

| System | Default FS | Features |
|--------|-----------|----------|
| macOS | APFS | Snapshots, encryption, space sharing, clones |
| Linux | ext4 / XFS / Btrfs | Varies by filesystem |
| FreeBSD | ZFS / UFS | ZFS has similar features to APFS |

### Case Sensitivity

macOS is case-insensitive but case-preserving by default:

```bash
# On default macOS filesystem:
$ touch File.txt
$ ls file.txt
File.txt           # Finds it! Different from Linux
```

This can cause issues with Git repositories from case-sensitive systems.

### Filesystem Hierarchy

Key differences from the Filesystem Hierarchy Standard (FHS):

| Purpose | Linux/BSD FHS | macOS |
|---------|--------------|-------|
| Applications | /usr/bin, /opt | /Applications |
| System config | /etc | /etc (Unix) + /Library/Preferences (macOS) |
| User config | ~/.config | ~/Library |
| User data | Various | ~/Documents, ~/Library/Application Support |
| Temp files | /tmp | /tmp → /private/tmp |
| Variable data | /var | /var → /private/var |
| System libraries | /usr/lib | /usr/lib + /System/Library/Frameworks |
| Third-party | /opt, /usr/local | /usr/local, /opt/homebrew (ARM) |

### Extended Attributes

macOS uses extended attributes heavily:

```bash
# View extended attributes
$ xattr file.txt
com.apple.quarantine

# List with values
$ xattr -l file.txt
com.apple.quarantine: 0083;5f4a7c14;Safari;...

# Remove quarantine attribute
$ xattr -d com.apple.quarantine file.txt

# Clear all attributes
$ xattr -c file.txt
```

Linux has extended attributes too, but they're less pervasive:

```bash
# Linux extended attributes
$ getfattr -d file.txt
$ setfattr -n user.comment -v "test" file.txt
```

### Resource Forks and Metadata

macOS preserves metadata when copying files:

```bash
# macOS: Copy with metadata
$ cp -p file.txt dest/          # Preserves basic metadata
$ ditto source/ dest/           # Preserves everything including resource forks

# Problem: copying to non-HFS+/APFS filesystems
$ cp file.txt /Volumes/USB_FAT32/
# Creates file.txt and ._file.txt (AppleDouble file)
```

Those `._*` files you see on USB drives? That's macOS preserving metadata on filesystems that don't support it natively.

## Package Management

### No Native Package Manager

Unlike most Linux distributions and BSD systems, macOS doesn't include a built-in package manager for third-party software:

| System | Native Package Manager |
|--------|----------------------|
| macOS | None (uses App Store for GUI apps) |
| Debian/Ubuntu | apt |
| Red Hat/Fedora | dnf/yum |
| Arch | pacman |
| FreeBSD | pkg |
| OpenBSD | pkg_add |

**Solutions for macOS**:

```bash
# Homebrew (most popular)
$ brew install wget

# MacPorts (alternative)
$ sudo port install wget
```

## User and Permission Differences

### User IDs

| User | Linux | macOS |
|------|-------|-------|
| root | UID 0 | UID 0 |
| First user | UID 1000 | UID 501 |
| Nobody | UID 65534 | UID -2 |

### Groups

macOS has different default groups:

```bash
$ id
uid=501(david) gid=20(staff) groups=20(staff),12(everyone),61(localaccounts),...
```

The primary group for users is `staff` (GID 20), not a user-specific group.

### System Integrity Protection (SIP)

macOS restricts what even root can do:

```bash
# This fails even as root (SIP protected):
$ sudo rm /System/Library/CoreServices/Finder.app
rm: /System/Library/CoreServices/Finder.app: Operation not permitted

# Check SIP status
$ csrutil status
System Integrity Protection status: enabled.
```

Linux and BSD have no equivalent to SIP—root can do anything.

### Sudo Configuration

macOS doesn't use `/etc/sudoers` by default for most permissions. Instead, it uses admin group membership:

```bash
# Check who can sudo
$ grep -E "^%admin|^%wheel" /etc/sudoers
%admin		ALL = (ALL) ALL
```

## Process and System Management

### Process Listing

```bash
# Both work on macOS:
$ ps aux                    # BSD syntax
$ ps -ef                    # POSIX syntax

# Linux-specific options don't work:
$ ps --forest              # Error on macOS
```

### System Information

```bash
# Linux
$ cat /proc/cpuinfo
$ free -h
$ lsb_release -a

# macOS (no /proc filesystem)
$ sysctl -n machdep.cpu.brand_string
$ vm_stat                   # Memory (in pages, not bytes)
$ sw_vers                   # macOS version
```

### Signals

Signal numbers can differ:

```bash
# Check signal numbers
$ kill -l
```

Most common signals (SIGTERM=15, SIGKILL=9, SIGHUP=1) are the same.

## Development Environment

### Compiler

```bash
# Linux (usually GCC)
$ gcc --version
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0

# macOS (Clang masquerading as gcc)
$ gcc --version
Apple clang version 15.0.0 (clang-1500.3.9.4)
```

The `gcc` command on macOS is actually Clang. This usually doesn't matter, but some software with GCC-specific features may need adjustment.

### Libraries

```bash
# Linux: Shared library extension
libfoo.so, libfoo.so.1, libfoo.so.1.0.0

# macOS: Shared library extension
libfoo.dylib, libfoo.1.dylib, libfoo.1.0.0.dylib

# Linux: Find library
$ ldconfig -p | grep libssl
$ ldd /usr/bin/openssl

# macOS: Find library
$ otool -L /usr/bin/openssl
```

### Library Paths

```bash
# Linux
$ echo $LD_LIBRARY_PATH
$ cat /etc/ld.so.conf

# macOS
$ echo $DYLD_LIBRARY_PATH           # Runtime
$ echo $DYLD_FALLBACK_LIBRARY_PATH  # Fallback
```

## Virtualization and Containers

### Docker

```bash
# Linux: Docker runs natively
$ docker run alpine echo "Hello"

# macOS: Docker runs in a Linux VM
$ docker run alpine echo "Hello"
# (Actually runs in a hypervisor-backed Linux VM)
```

Docker Desktop on macOS uses a Linux virtual machine because containers are a Linux kernel feature.

### Native Virtualization

```bash
# macOS: Virtualization.framework (Apple Silicon) or Hypervisor.framework
# No direct CLI, used by apps like UTM, Parallels, Docker Desktop

# Linux: KVM
$ virsh list --all
```

## Summary Table

| Feature | macOS | Linux | FreeBSD |
|---------|-------|-------|---------|
| Kernel | XNU (hybrid) | Linux (monolithic) | FreeBSD (monolithic) |
| Init | launchd | systemd (usually) | rc.d |
| Package mgr | Homebrew (3rd party) | apt/dnf/pacman | pkg |
| Shell default | zsh | bash (usually) | sh/tcsh |
| Filesystem | APFS | ext4/XFS/Btrfs | ZFS/UFS |
| Case sensitive | No (default) | Yes | Configurable |
| Binary format | Mach-O | ELF | ELF |
| Commands | BSD | GNU | BSD |
| Firewall | pf | iptables/nftables | pf/ipfw |
| Root restriction | SIP | SELinux/AppArmor | securelevel |

Understanding these differences helps you translate your Unix knowledge to macOS effectively. When something doesn't work as expected, check whether it's a BSD vs GNU difference, a filesystem assumption, or a macOS-specific security feature.
