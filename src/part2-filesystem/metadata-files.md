# The Metadata Files: .DS_Store and ._AppleDouble

Every Mac user eventually notices them: `.DS_Store` files appearing in directories, `._` prefix files cluttering USB drives, and mysterious metadata files showing up in archives. These files serve legitimate purposes but can be annoying when they leak into version control or cross-platform file sharing.

## .DS_Store Files

### What They Are

`.DS_Store` (Desktop Services Store) files are created by Finder to store custom folder attributes:

- Window position and size
- Icon positions (when using icon view)
- View settings (icon/list/column/gallery view)
- Background color or image
- Sort order and column widths
- Custom icons

```bash
# Find .DS_Store files
$ find ~/Documents -name '.DS_Store' -type f 2>/dev/null | head
/Users/david/Documents/.DS_Store
/Users/david/Documents/Projects/.DS_Store
...

# View basic info
$ file .DS_Store
.DS_Store: Apple Desktop Services Store

# Size is usually small
$ ls -la .DS_Store
-rw-r--r--@ 1 david  staff  6148 Jan 15 10:30 .DS_Store
```

### .DS_Store Contents

The file format is proprietary binary, but tools exist to examine it:

```bash
# Install ds_store Python library
$ pip install ds_store

# Or use a basic hex dump
$ xxd .DS_Store | head
00000000: 0000 0001 4275 6431 0000 3000 0000 0800  ....Bud1..0.....
00000010: 0000 2000 0000 0008 0000 0000 0000 0000  .. .............
...
```

The file contains:
- File/folder names in the directory
- Per-item display settings
- Finder view preferences
- Spotlight comments (sometimes)

### Preventing .DS_Store Creation

#### On Network Volumes

Finder can be configured to not create `.DS_Store` on network drives:

```bash
# Prevent on network volumes
$ defaults write com.apple.desktopservices DSDontWriteNetworkStores true

# Prevent on USB volumes
$ defaults write com.apple.desktopservices DSDontWriteUSBStores true

# Restart Finder to apply
$ killall Finder
```

This doesn't prevent local `.DS_Store` creation—only network and USB volumes.

#### Globally (Not Recommended)

There's no supported way to prevent local `.DS_Store` creation without breaking Finder functionality.

### Cleaning Up .DS_Store

```bash
# Delete .DS_Store files in current directory tree
$ find . -name '.DS_Store' -type f -delete

# Dry run first (just show what would be deleted)
$ find . -name '.DS_Store' -type f -print

# Delete from a specific path, including hidden directories
$ find /path/to/folder -name '.DS_Store' -delete
```

### .DS_Store and Git

These files should almost always be gitignored:

```bash
# Add to global gitignore
$ echo '.DS_Store' >> ~/.gitignore_global
$ git config --global core.excludesfile ~/.gitignore_global

# Add to project .gitignore
$ echo '.DS_Store' >> .gitignore

# Remove already-tracked .DS_Store files
$ git rm --cached $(git ls-files -i --exclude-standard '*.DS_Store')
$ git commit -m "Remove .DS_Store files"
```

## AppleDouble Files (._prefix files)

### What They Are

When macOS copies files to filesystems that don't support extended attributes (FAT32, exFAT, SMB shares, etc.), it creates companion files with `._` prefix:

```bash
# Copy to FAT32 drive
$ cp photo.jpg /Volumes/USB_DRIVE/

# Results in:
$ ls -la /Volumes/USB_DRIVE/
-rwxr-xr-x  1 david  staff  1048576 Jan 15 10:30 photo.jpg
-rwxr-xr-x  1 david  staff     4096 Jan 15 10:30 ._photo.jpg
```

### What They Contain

AppleDouble files store:
- Extended attributes that couldn't be preserved
- Resource forks (if present)
- Finder metadata
- ACLs

```bash
# View what's in an AppleDouble file
$ xattr -l ._photo.jpg
# (Shows nothing - attributes are encoded in file content)

# Decode structure (basic view)
$ xxd ._photo.jpg | head
00000005 26 00 02 00 00 00 00 00  00 00 00 00 00 00 00 00  |&...............|
00000015 00 00 00 00 00 00 00 00  00 02 00 00 00 09 00 00  |................|
...
```

### When They Appear

- Copying to FAT32/exFAT USB drives
- Copying to SMB/CIFS network shares
- Creating ZIP files (sometimes)
- Copying to NFS shares without xattr support
- Burning to data CDs/DVDs

### The dot_clean Command

macOS provides `dot_clean` to merge AppleDouble files:

```bash
# Merge AppleDouble files back into their parent files
# (Only works when copying back to APFS/HFS+)
$ dot_clean /Volumes/USB_DRIVE/

# Merge and remove AppleDouble files
$ dot_clean -m /Volumes/USB_DRIVE/

# Verbose output
$ dot_clean -v /Volumes/USB_DRIVE/
Merging ._photo.jpg
...
```

### Cleaning Up AppleDouble Files

```bash
# Find all AppleDouble files
$ find /Volumes/USB_DRIVE -name '._*' -type f

# Delete them
$ find /Volumes/USB_DRIVE -name '._*' -type f -delete

# Delete .DS_Store AND AppleDouble files
$ find /Volumes/USB_DRIVE \( -name '.DS_Store' -o -name '._*' \) -delete
```

### Preventing AppleDouble Files

#### Use ditto with --norsrc

```bash
# Copy without resource forks/AppleDouble
$ ditto --norsrc source.txt /Volumes/USB_DRIVE/

# Copy entire directory
$ ditto --norsrc ~/Documents/folder /Volumes/USB_DRIVE/folder
```

#### Strip Attributes Before Copy

```bash
# Copy then strip
$ cp file.txt /Volumes/USB_DRIVE/
$ xattr -c /Volumes/USB_DRIVE/file.txt
# This removes the need for AppleDouble companion
```

#### Use Archive Formats

When sharing files, use formats that either:
- Don't preserve Mac metadata (simple tar, zip)
- Explicitly preserve it (tar with --xattrs, ditto archives)

```bash
# Create cross-platform-friendly tar
$ tar -cvf archive.tar files/
# No metadata preserved, no ._files

# Or strip metadata first
$ find files/ -name '._*' -delete
$ find files/ -name '.DS_Store' -delete
$ tar -cvf archive.tar files/
```

## __MACOSX Folders in ZIP Files

When Finder creates ZIP files, it may include a `__MACOSX` folder:

```bash
# Create ZIP with Finder
$ cd ~/Documents
# Right-click folder → Compress "folder"

# Extract on Windows/Linux reveals:
# folder/
# __MACOSX/
#   folder/
#     ._file1.txt
#     ._file2.txt
```

### Creating Clean ZIPs

```bash
# Use zip command with exclusions
$ zip -r archive.zip folder/ -x "*.DS_Store" -x "__MACOSX/*" -x "._*"

# Or use ditto to create clean ZIP
$ ditto -c -k --sequesterRsrc --keepParent folder archive.zip
# --sequesterRsrc puts resource forks in __MACOSX (standard Mac behavior)

# For truly clean ZIP (no Mac metadata at all):
$ cd folder && zip -r ../archive.zip . -x "*.DS_Store" -x "*.git*" -x "._*"
```

### Extracting and Cleaning

```bash
# Extract ZIP
$ unzip archive.zip

# Remove Mac metadata
$ find . -name '__MACOSX' -type d -exec rm -rf {} +
$ find . -name '._*' -type f -delete
$ find . -name '.DS_Store' -type f -delete
```

## Best Practices

### For Version Control

Standard `.gitignore` entries:

```gitignore
# macOS metadata
.DS_Store
.AppleDouble
.LSOverride

# Icon must end with two \r
Icon

# Thumbnails
._*

# Files that might appear in the root of a volume
.DocumentRevisions-V100
.fseventsd
.Spotlight-V100
.TemporaryItems
.Trashes
.VolumeIcon.icns
.com.apple.timemachine.donotpresent

# Directories potentially created on remote AFP share
.AppleDB
.AppleDesktop
Network Trash Folder
Temporary Items
.apdisk
```

### For File Sharing

When sharing files with non-Mac users:

1. **Clean before sharing**:
```bash
find /path/to/share -name '.DS_Store' -delete
find /path/to/share -name '._*' -delete
```

2. **Use appropriate tools**:
```bash
# For cross-platform ZIP
zip -r archive.zip folder/ -x "*.DS_Store" -x "._*"
```

3. **Configure Finder**:
```bash
defaults write com.apple.desktopservices DSDontWriteNetworkStores true
defaults write com.apple.desktopservices DSDontWriteUSBStores true
```

### For External Drives

If you regularly use drives with non-Mac systems:

```bash
# Configure once
defaults write com.apple.desktopservices DSDontWriteUSBStores true

# After use, clean up
dot_clean /Volumes/DRIVE_NAME
find /Volumes/DRIVE_NAME -name '.DS_Store' -delete
find /Volumes/DRIVE_NAME -name '._*' -delete
```

## Summary

macOS metadata files serve real purposes but require management:

| File Type | Purpose | When to Remove |
|-----------|---------|----------------|
| `.DS_Store` | Finder view settings | Version control, cross-platform sharing |
| `._*` files | Extended attributes on non-Mac filesystems | When no longer needed, cross-platform sharing |
| `__MACOSX/` | Resource forks in ZIP files | Cross-platform distribution |

Key commands:
- `find . -name '.DS_Store' -delete` — Clean `.DS_Store` files
- `find . -name '._*' -delete` — Clean AppleDouble files
- `dot_clean /path` — Merge AppleDouble files back
- `defaults write com.apple.desktopservices DSDontWriteUSBStores true` — Prevent on USB

These files are a side effect of macOS's rich metadata system—useful locally but often unwanted when files leave the Mac ecosystem.
