# Understanding macOS Filesystems

Filesystems are where Unix heritage meets Apple innovation. While macOS supports familiar Unix filesystem concepts—files, directories, permissions, symbolic links—it implements them atop storage technologies that differ significantly from traditional Unix systems.

Understanding macOS filesystems means understanding three layers:

1. **APFS**: The modern storage technology providing advanced features like snapshots, clones, and space sharing
2. **The VFS layer**: How macOS presents a unified filesystem interface to applications
3. **macOS conventions**: Extended attributes, metadata, and organizational patterns unique to Mac

## The Evolution of Mac Storage

The journey from original Macintosh to modern macOS represents a dramatic evolution:

**1984-1998**: The original Macintosh File System (MFS) and its successor HFS
- Resource forks and data forks as first-class concepts
- Case-insensitive by design
- No Unix permissions (Mac OS had no concept of users)

**1998-2017**: HFS+ (Mac OS Extended)
- Journaling for data integrity
- Support for Unix permissions (added for Mac OS X)
- Still case-insensitive by default
- Resource forks maintained for compatibility

**2017-Present**: APFS (Apple File System)
- Built for flash storage and SSDs
- Native encryption
- Snapshots and cloning
- Space sharing between volumes
- Case-sensitive option more practical

## What You'll Learn in This Part

**[APFS: Apple's Modern Filesystem](./apfs.md)** explains the architecture and capabilities of Apple's current filesystem, including features like space sharing, snapshots, and encryption.

**[HFS+ Legacy and Migration](./hfs-plus.md)** covers the previous filesystem format, why you might still encounter it, and how to handle legacy volumes.

**[Case Sensitivity: Options and Implications](./case-sensitivity.md)** addresses one of the most misunderstood aspects of macOS filesystems—why case-insensitivity is the default and when it matters.

**[Extended Attributes and Resource Forks](./extended-attributes.md)** explores macOS's rich file metadata system, including quarantine attributes, Finder information, and the legacy of resource forks.

**[The Metadata Files: .DS_Store and ._AppleDouble](./metadata-files.md)** explains those mysterious files that appear everywhere and how to manage them.

**[Filesystem Hierarchy: Where macOS Diverges](./hierarchy.md)** maps out where things live on macOS compared to the Filesystem Hierarchy Standard used by Linux.

**[Disk Management from the Command Line](./disk-management.md)** teaches you to manage disks, partitions, and volumes using Terminal commands.

**[Volumes, Containers, and Snapshots](./volumes-containers.md)** dives deep into APFS's container architecture and how to work with snapshots.

## Why Filesystems Matter

As a Unix user, you might assume filesystems are interchangeable—files are files, directories are directories. On macOS, this assumption can lead to problems:

- Copy a file to a FAT32 USB drive and mysterious `._` files appear
- Git reports changes to files you didn't modify (case sensitivity issues)
- Scripts fail because they assume `/opt` exists or `/tmp` persists across reboots
- Files downloaded from the internet won't run (quarantine attributes)

Understanding macOS filesystems helps you avoid these pitfalls and leverage features like snapshots and clones that traditional Unix filesystems lack.
