# Cross-Platform Interoperability

Modern software development rarely happens on a single platform. Developers routinely build on macOS, deploy to Linux servers, share files with colleagues on different operating systems, and run containers that virtualize entirely different environments. macOS's Unix heritage makes it an excellent development platform for cross-platform work, but its differences from Linux and BSD systems can cause friction if you don't understand them.

This section provides practical guidance for developers who work across platforms, covering everything from file sharing quirks to running Linux containers natively on your Mac.

## The Interoperability Challenge

macOS is Unix-certified and shares much with Linux and BSD, but important differences remain:

```
┌─────────────────────────────────────────────────────────────────┐
│                   Platform Comparison                            │
├─────────────────────┬──────────────────┬────────────────────────┤
│        macOS        │      Linux       │         BSD            │
├─────────────────────┼──────────────────┼────────────────────────┤
│ BSD userland        │ GNU userland     │ BSD userland           │
│ HFS+/APFS           │ ext4/xfs/btrfs   │ UFS/ZFS                │
│ launchd             │ systemd/init     │ rc.d                   │
│ Case-insensitive*   │ Case-sensitive   │ Case-sensitive         │
│ Extended attributes │ xattrs (diff.)   │ xattrs (diff.)         │
│ Frameworks          │ .so libraries    │ .so libraries          │
│ Mach-O binaries     │ ELF binaries     │ ELF binaries           │
│ Apple Silicon/Intel │ x86/ARM/etc.     │ x86/ARM/etc.           │
└─────────────────────┴──────────────────┴────────────────────────┘
* Default; case-sensitive option available
```

These differences affect:

- **File sharing**: Case sensitivity, extended attributes, and line endings cause problems
- **Shell scripts**: Command flags differ between BSD and GNU utilities
- **Containers**: Linux containers need a VM layer on macOS
- **Remote work**: SSH, VNC, and network protocols have platform-specific considerations

## Common Cross-Platform Workflows

### Development Workflow: Code on Mac, Deploy to Linux

The most common pattern for web developers:

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   macOS Local    │───▶│   Git/GitHub     │───▶│  Linux Server    │
│   Development    │    │                  │    │  (Production)    │
└──────────────────┘    └──────────────────┘    └──────────────────┘
        │                                               ▲
        │                                               │
        └───────────────────────────────────────────────┘
                    SSH / Container Testing
```

Key considerations:
- Line endings (CRLF vs LF)
- Case sensitivity in filenames
- Shell script portability
- Environment parity with containers

### Container-Based Development

Running Linux containers on macOS:

```
┌─────────────────────────────────────────────────────────────────┐
│                         macOS Host                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Linux VM (Docker/Colima/Lima)                 │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │ Container 1 │  │ Container 2 │  │ Container 3 │       │  │
│  │  │   (app)     │  │   (db)      │  │   (cache)   │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                    Volume mounts (with translation)              │
│                              │                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    macOS Filesystem                        │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Multi-Machine Environment

Working with multiple systems:

```bash
# SSH config for quick access
$ cat ~/.ssh/config
Host dev
    HostName dev-server.example.com
    User developer

Host prod-*
    User deploy
    IdentityFile ~/.ssh/deploy_key
    ProxyJump bastion.example.com

# Quick connections
$ ssh dev
$ ssh prod-web-1
```

## Quick Compatibility Reference

### File Systems and Paths

| Issue | macOS | Linux | Solution |
|-------|-------|-------|----------|
| Case sensitivity | Insensitive | Sensitive | Test on Linux or case-sensitive volume |
| Path separator | `/` | `/` | Same |
| Home directory | `/Users/name` | `/home/name` | Use `$HOME` or `~` |
| Temp directory | `/tmp` (→ `/private/tmp`) | `/tmp` | Use `$TMPDIR` or `/tmp` |
| Extended attrs | Yes (complex) | Yes (simpler) | Strip with `xattr -rc` before sharing |

### Shell Commands

| Task | macOS (BSD) | Linux (GNU) | Portable |
|------|-------------|-------------|----------|
| List files | `ls` | `ls` | Same (basic flags) |
| Extended list | `ls -l@` | `ls -l` | Check for attrs separately |
| sed in-place | `sed -i ''` | `sed -i` | `sed -i.bak` |
| Date format | BSD date | GNU date | Use `date +%FORMAT` |
| Find with delete | `find -delete` | `find -delete` | Same |
| readlink | BSD (limited) | GNU (full) | Install GNU coreutils |
| stat format | `stat -f` | `stat -c` | Use format flags carefully |

### Network Services

| Service | macOS | Linux | Notes |
|---------|-------|-------|-------|
| SSH | OpenSSH | OpenSSH | Mostly identical |
| VNC | Screen Sharing | TigerVNC/x11vnc | Different implementations |
| SMB | Built-in | Samba | macOS client works with Linux servers |
| NFS | Built-in | Built-in | Some option differences |
| mDNS/Bonjour | Built-in | Avahi | Compatible protocols |

## What You'll Learn in This Part

**[File Sharing with Linux and BSD](./file-sharing.md)** covers the pitfalls of moving files between systems, including case sensitivity, extended attributes, line endings, and the best portable formats like ExFAT.

**[Running Linux Containers on macOS](./linux-containers.md)** explains how Docker Desktop, Colima, Podman, and Lima work, including the virtualization layer required to run Linux containers on macOS.

**[Virtual Machines on macOS](./virtual-machines.md)** covers running full Linux distributions on your Mac using UTM, Parallels, VMware, and Apple's Virtualization.framework.

**[Cross-Platform Script Compatibility](./script-compatibility.md)** provides techniques for writing shell scripts that work on both macOS and Linux, covering POSIX compliance, shebang lines, OS detection, and portable patterns.

**[SSH Configuration and Usage](./ssh.md)** dives deep into SSH configuration, key management, agent forwarding, ProxyJump, multiplexing, and advanced features for developers.

**[Remote Access: VNC and ARD](./remote-access.md)** covers accessing your Mac remotely and using your Mac to access other systems via VNC, Apple Remote Desktop, and other protocols.

**[Working with NFS and SMB](./nfs-smb.md)** explains mounting network shares, configuring autofs for automatic mounting, and troubleshooting common permission issues.

## Quick Interoperability Checklist

Before sharing files or scripts across platforms:

```bash
# Check for case-sensitivity issues in a directory
find . -type f | sort -f | uniq -di

# Remove extended attributes before archiving
xattr -rc directory/

# Convert line endings to Unix format
find . -name "*.sh" -exec dos2unix {} \;
# Or using sed:
find . -name "*.sh" -exec sed -i '' 's/\r$//' {} \;

# Test script POSIX compliance
shellcheck --shell=sh script.sh

# Check shebang portability
head -1 script.sh
# Use: #!/usr/bin/env bash
# Not: #!/usr/local/bin/bash
```

The following chapters provide detailed guidance on each of these areas, helping you work seamlessly across macOS, Linux, and BSD systems.
