# Extended Attributes and Resource Forks

macOS files carry metadata beyond the basic Unix permissions and timestamps. Extended attributes (xattrs) store arbitrary key-value pairs, while resource forks—a legacy from classic Mac OS—persist in modern macOS as a special extended attribute. Understanding this metadata system is essential for proper file handling.

## Extended Attributes Overview

Extended attributes are named metadata associated with files:

```bash
# List extended attributes on a file
$ xattr file.txt
com.apple.quarantine

# List with values
$ xattr -l file.txt
com.apple.quarantine: 0083;65a12345;Safari;12345678-1234-1234-1234-123456789012

# View in hex
$ xattr -px com.apple.quarantine file.txt
00 38 33 3B 36 35 61 31 32 33 34 35 3B 53 61 66
...
```

### Common Extended Attributes

| Attribute | Purpose |
|-----------|---------|
| `com.apple.quarantine` | Marks files downloaded from internet |
| `com.apple.FinderInfo` | Finder metadata (color labels, etc.) |
| `com.apple.ResourceFork` | Legacy resource fork data |
| `com.apple.metadata:*` | Spotlight metadata |
| `com.apple.lastuseddate#PS` | Last used timestamp |
| `com.apple.macl` | macOS Access Control List |
| `com.apple.provenance` | File provenance information |

## The Quarantine Attribute

Files downloaded from the internet are "quarantined":

```bash
# Download a file
$ curl -O https://example.com/file.zip

# Check quarantine status
$ xattr -l file.zip
com.apple.quarantine: 0083;65a12345;curl;12345678-...

# First time opening triggers Gatekeeper
$ open file.zip
# "file.zip" is from an unidentified developer...
```

### Quarantine Fields

The quarantine attribute contains:
- Flags (hex): What checks have been performed
- Timestamp: When downloaded (hex Unix timestamp)
- Agent: Application that downloaded it
- UUID: Unique identifier

### Removing Quarantine

```bash
# Remove quarantine from a single file
$ xattr -d com.apple.quarantine file.zip

# Remove from application bundle and contents
$ xattr -dr com.apple.quarantine /Applications/SomeApp.app

# Recursive removal for a directory
$ xattr -dr com.apple.quarantine ~/Downloads/extracted-folder/
```

### Quarantine Flags

```bash
# Flag values:
# 0000: No quarantine (translocated app)
# 0001: Origin URL checked
# 0002: User consent obtained
# 0040: Notarization checked (macOS 10.15+)
# 0080: Hardened runtime checked
```

## FinderInfo Attribute

Finder metadata is stored in `com.apple.FinderInfo`:

```bash
# View FinderInfo
$ xattr -px com.apple.FinderInfo file.txt

# This 32-byte structure contains:
# - File type and creator codes (legacy)
# - Finder flags (label color, extension hidden, etc.)
# - Position (for icon placement)
```

### Setting Finder Tags/Labels

```bash
# Set color label using tag command (macOS 10.9+)
$ tag -a Red file.txt

# View tags
$ tag -l file.txt
file.txt    Red

# Using mdls to see color
$ mdls -name kMDItemUserTags file.txt
kMDItemUserTags = (
    "Red"
)
```

Note: The tag command is third-party. Install via `brew install tag`.

## Resource Forks

Resource forks are a legacy concept from classic Mac OS where files had two "forks":

- **Data fork**: The main content (what Unix sees as the file)
- **Resource fork**: Structured data (icons, code, dialog layouts, etc.)

### Modern Resource Fork Access

Resource forks are stored in the `com.apple.ResourceFork` extended attribute:

```bash
# Check if a file has a resource fork
$ xattr file | grep ResourceFork
com.apple.ResourceFork

# Access resource fork via named fork syntax
$ ls -l file/..namedfork/rsrc

# Or via attribute
$ xattr -p com.apple.ResourceFork file | xxd | head
```

### When Resource Forks Appear

Modern software rarely uses resource forks, but you'll encounter them in:
- Classic Mac OS applications/files
- Some disk images
- Files created by older software
- Cross-platform transfers from old Mac files

```bash
# Get resource fork size (if any)
$ ls -l@ file | grep -A1 "ResourceFork"
    com.apple.ResourceFork       1234
```

### Preserving Resource Forks

Standard Unix tools may not preserve resource forks:

```bash
# cp preserves extended attributes including resource forks
$ cp -p file /destination/

# tar can preserve extended attributes
$ tar --xattrs -cvf archive.tar file

# ditto preserves everything (recommended for Mac-to-Mac)
$ ditto source destination

# rsync requires special flags
$ rsync -avX source/ destination/
# -X preserves extended attributes on macOS
```

## Working with Extended Attributes

### Reading Attributes

```bash
# List attribute names
$ xattr file.txt
com.apple.quarantine
com.apple.metadata:kMDItemWhereFroms

# Print attribute value (human readable)
$ xattr -p com.apple.quarantine file.txt
0083;65a12345;Safari;...

# Print as hex
$ xattr -px com.apple.quarantine file.txt
00 38 33 3B 36 35 61 31 ...
```

### Writing Attributes

```bash
# Set an attribute (string value)
$ xattr -w user.comment "My comment" file.txt

# Set an attribute (hex value)
$ xattr -wx user.binary "48454C4C4F" file.txt

# Verify
$ xattr -p user.comment file.txt
My comment
```

### Removing Attributes

```bash
# Remove specific attribute
$ xattr -d com.apple.quarantine file.txt

# Remove all attributes
$ xattr -c file.txt

# Remove recursively
$ xattr -rc directory/
```

### Copying Attributes

```bash
# Copy attributes from one file to another
$ xattr -l source.txt > /tmp/attrs
# (Manual process - no direct copy command)

# Or use ditto for full preservation
$ ditto source.txt dest.txt
```

## Extended Attributes and Unix Tools

### Tools That Preserve Attributes

| Tool | Preserves xattrs | Notes |
|------|------------------|-------|
| `cp` | Yes (with -p) | macOS cp includes -p in default behavior |
| `mv` | Yes | Within same filesystem |
| `rsync -X` | Yes | Requires -X flag |
| `tar --xattrs` | Yes | Requires flag |
| `ditto` | Yes | Recommended for Mac-to-Mac |
| `asr` | Yes | Apple Software Restore |

### Tools That Don't Preserve Attributes

| Tool | Issue | Workaround |
|------|-------|------------|
| `scp` | Drops attributes | Use rsync -avzX |
| `git` | Not version controlled | Document requirements |
| `curl`/`wget` | Adds quarantine | Remove with xattr -d |

### Attributes and SSH/SCP

When copying files to remote systems:

```bash
# Standard scp loses attributes
$ scp file.txt remote:~/

# Use rsync instead
$ rsync -avzX file.txt remote:~/

# Note: Remote must support xattrs
# Linux ext4 supports them; many remote systems do not
```

## Metadata Spotlight Integration

Some extended attributes feed into Spotlight:

```bash
# View Spotlight metadata
$ mdls file.txt
kMDItemContentType         = "public.plain-text"
kMDItemContentTypeTree     = (
    "public.plain-text",
    "public.text",
    ...
)
kMDItemDisplayName         = "file.txt"
kMDItemFSCreatorCode       = ""
kMDItemFSFinderFlags       = 0
kMDItemFSHasCustomIcon     = (null)
...

# View WhereFroms (download URLs)
$ mdls -name kMDItemWhereFroms file.txt
kMDItemWhereFroms = (
    "https://example.com/file.txt",
    "https://example.com/"
)
```

### Clearing Download History

```bash
# Remove WhereFroms metadata
$ xattr -d com.apple.metadata:kMDItemWhereFroms file.txt
```

## AppleDouble Files

When copying files with extended attributes to filesystems that don't support them (FAT32, SMB shares, etc.), macOS creates "AppleDouble" files:

```bash
# Copy file to FAT32 USB drive
$ cp file.txt /Volumes/USB_FAT32/

# Results in:
# /Volumes/USB_FAT32/file.txt        (data fork)
# /Volumes/USB_FAT32/._file.txt      (extended attributes)
```

The `._` files contain serialized extended attributes.

### Preventing AppleDouble Files

```bash
# Use dot_clean to merge AppleDouble files
$ dot_clean /Volumes/USB_FAT32/

# Prevent creation (strip attributes during copy)
$ cp file.txt /Volumes/USB_FAT32/
$ xattr -c /Volumes/USB_FAT32/file.txt
```

### Cleaning Up AppleDouble Files

```bash
# Find AppleDouble files
$ find /Volumes/USB_FAT32 -name '._*' -type f

# Remove them
$ find /Volumes/USB_FAT32 -name '._*' -type f -delete

# Or use dot_clean to merge them back
$ dot_clean /Volumes/USB_FAT32
```

## Summary

Extended attributes on macOS provide:

- **Quarantine tracking** for security (Gatekeeper)
- **Finder metadata** for GUI integration
- **Resource forks** for legacy compatibility
- **Spotlight integration** for search
- **Custom metadata** for applications

When working with files:

1. Be aware that attributes exist and affect behavior
2. Use appropriate tools (`ditto`, `rsync -X`) when preservation matters
3. Know how to inspect (`xattr -l`) and remove (`xattr -d`) attributes
4. Understand AppleDouble files when working with non-Mac filesystems
5. Remember that quarantine affects executable files from the internet

Extended attributes are integral to the macOS experience, even when working from the command line.
