# File Sharing with Linux and BSD

Moving files between macOS and Linux or BSD systems seems straightforward until you encounter the edge cases: files that vanish because of case sensitivity, mysterious extended attributes polluting your archives, or scripts that fail because of invisible line ending differences. This chapter covers these common issues and provides solutions for reliable cross-platform file sharing.

## Case Sensitivity: The Silent Problem

By default, macOS uses a case-insensitive (but case-preserving) filesystem. Linux and BSD use case-sensitive filesystems. This difference causes real problems.

### The Problem

```bash
# On macOS (case-insensitive)
$ touch README.md readme.md
$ ls
README.md  # Only one file exists!

# On Linux (case-sensitive)
$ touch README.md readme.md
$ ls
README.md  readme.md  # Two distinct files
```

When you clone a repository on macOS that contains both `File.txt` and `file.txt` (created on Linux), one will silently overwrite the other.

### Detecting Case Conflicts

Check for case conflicts before they cause problems:

```bash
# Find potential case conflicts in current directory
find . -type f | sort -f | uniq -di

# More comprehensive check (shows conflicting pairs)
find . -type f -print0 | \
  xargs -0 -n1 basename | \
  sort -f | uniq -di | while read name; do
    find . -iname "$name" -type f
    echo "---"
  done

# Check a git repository for case conflicts
git ls-files | sort -f | uniq -di
```

### Solutions

**Option 1: Create a Case-Sensitive Volume**

For development that must match Linux behavior:

```bash
# Create a case-sensitive APFS volume
$ diskutil apfs addVolume disk1 "Case-sensitive APFS" CaseSensitive

# Mount point will be at /Volumes/CaseSensitive
$ cd /Volumes/CaseSensitive
$ touch README.md readme.md
$ ls
README.md  readme.md  # Both files exist
```

**Option 2: Create a Case-Sensitive Disk Image**

For a portable solution:

```bash
# Create a sparse image with case-sensitive filesystem
$ hdiutil create -size 10g -fs "Case-sensitive APFS" \
    -type SPARSE -volname "DevWork" ~/DevWork.sparseimage

# Mount it
$ hdiutil attach ~/DevWork.sparseimage
/dev/disk4          	GUID_partition_scheme
/dev/disk4s1        	EFI
/dev/disk4s2        	Apple_APFS
/dev/disk5          	EFI
/dev/disk5s1        	XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX	/Volumes/DevWork

# Use it
$ cd /Volumes/DevWork
```

**Option 3: Git Configuration**

Configure Git to warn about case issues:

```bash
# Enable case-sensitivity checking
$ git config core.ignoreCase false

# This will cause Git to notice case-only renames
```

**Option 4: Rename Files to Avoid Conflicts**

The simplest solution when possible:

```bash
# On Linux, rename conflicting files
$ mv file.txt file_lower.txt
$ mv File.txt File_upper.txt
```

## Extended Attributes and Resource Forks

macOS extensively uses extended attributes for metadata. These cause problems when sharing files with Linux/BSD.

### The Problem

```bash
# On macOS, files acquire extended attributes
$ ls -l@ file.txt
-rw-r--r--@ 1 user staff 100 Jan 1 12:00 file.txt
    com.apple.quarantine	     57
    com.apple.lastuseddate#PS	     16

# When archived, these create ._ files (AppleDouble format)
$ tar czf archive.tar.gz folder/
$ tar tzf archive.tar.gz | grep "._"
folder/._file.txt
folder/._image.png
```

On Linux, these `._` files litter your directories:

```bash
# On Linux after extracting
$ ls -la
-rw-r--r-- 1 user user  100 Jan  1 12:00 file.txt
-rw-r--r-- 1 user user  289 Jan  1 12:00 ._file.txt  # What is this?
```

### Understanding Extended Attributes

View extended attributes on macOS:

```bash
# List extended attributes
$ xattr file.txt
com.apple.quarantine
com.apple.lastuseddate#PS

# View attribute contents
$ xattr -p com.apple.quarantine file.txt
0083;5f4c1234;Safari;1234ABCD-5678-EFGH-IJKL-MNOPQRSTUVWX

# Common macOS extended attributes
# com.apple.quarantine     - File downloaded from internet
# com.apple.FinderInfo     - Finder metadata
# com.apple.ResourceFork   - Classic Mac resource data
# com.apple.lastuseddate#PS - Last opened date
# com.apple.metadata:*     - Spotlight metadata
```

### Removing Extended Attributes

Before sharing files with other systems:

```bash
# Remove all extended attributes from a file
$ xattr -c file.txt

# Remove all extended attributes recursively
$ xattr -rc folder/

# Remove specific attribute
$ xattr -d com.apple.quarantine file.txt

# Remove quarantine attribute from downloaded app
$ xattr -dr com.apple.quarantine /Applications/SomeApp.app
```

### Creating Clean Archives

For archives meant for Linux/BSD systems:

```bash
# Method 1: Use tar with --no-xattrs (if available)
$ tar --no-xattrs -czf archive.tar.gz folder/

# Method 2: Strip attributes first
$ xattr -rc folder/
$ tar czf archive.tar.gz folder/

# Method 3: Use COPYFILE_DISABLE to prevent AppleDouble files
$ COPYFILE_DISABLE=1 tar czf archive.tar.gz folder/

# Method 4: For zip files
$ zip -r -X archive.zip folder/  # -X excludes extra file attributes

# Verify archive is clean
$ tar tzf archive.tar.gz | grep "._"
# Should return nothing
```

### Cleaning Received Archives

When you receive archives with `._` files:

```bash
# Remove AppleDouble files
$ find . -name "._*" -delete

# Alternative: use dot_clean utility
$ dot_clean folder/
# This merges ._ files back into their parent files or removes them
```

## Line Endings: The Invisible Difference

Different operating systems use different line endings:

- Unix/Linux/macOS: `LF` (`\n`, `0x0A`)
- Windows: `CRLF` (`\r\n`, `0x0D 0x0A`)
- Classic Mac (pre-OS X): `CR` (`\r`, `0x0D`)

### Detecting Line Ending Issues

```bash
# Check file line endings
$ file script.sh
script.sh: Bourne-Again shell script, ASCII text executable

$ file windows_script.sh
windows_script.sh: Bourne-Again shell script, ASCII text executable, with CRLF line terminators

# Using cat -v to see carriage returns
$ cat -v windows_script.sh | head -2
#!/bin/bash^M
echo "Hello"^M

# Using hexdump
$ hexdump -C script.sh | head -2
00000000  23 21 2f 62 69 6e 2f 62  61 73 68 0a              |#!/bin/bash.|
```

### Converting Line Endings

```bash
# Using dos2unix (install via Homebrew)
$ brew install dos2unix

# Convert Windows to Unix
$ dos2unix file.txt

# Convert Unix to Windows
$ unix2dos file.txt

# Convert multiple files
$ dos2unix *.sh

# Without dos2unix, use sed
# Remove CR characters (Windows → Unix)
$ sed -i '' 's/\r$//' file.txt

# Using tr
$ tr -d '\r' < windows_file.txt > unix_file.txt

# Using Perl (often more reliable for binary safety)
$ perl -pi -e 's/\r\n/\n/g' file.txt
```

### Git Configuration for Line Endings

Configure Git to handle line endings automatically:

```bash
# For cross-platform projects (recommended)
# This normalizes to LF in repository, converts to native on checkout
$ git config core.autocrlf input   # On macOS/Linux
$ git config core.autocrlf true    # On Windows

# Or use .gitattributes in the repository (better):
$ cat .gitattributes
# Set default behavior
* text=auto

# Explicitly declare text files
*.sh text eol=lf
*.py text eol=lf
*.md text eol=lf

# Declare binary files
*.png binary
*.jpg binary
*.pdf binary

# Windows batch files need CRLF
*.bat text eol=crlf
```

### Fixing Line Endings in a Git Repository

```bash
# Normalize all text files in repository
$ git add --renormalize .
$ git commit -m "Normalize line endings"
```

## ExFAT: The Universal Filesystem

When you need to share files via USB drives or SD cards, ExFAT is the best choice for cross-platform compatibility.

### Why ExFAT?

| Filesystem | macOS | Linux | Windows | Max File Size |
|------------|-------|-------|---------|---------------|
| HFS+ | R/W | Read* | No | 8 EB |
| APFS | R/W | Read* | No | 8 EB |
| NTFS | Read | R/W | R/W | 16 TB |
| FAT32 | R/W | R/W | R/W | 4 GB |
| **ExFAT** | R/W | R/W | R/W | 16 EB |

*With third-party tools

ExFAT is:
- Native read/write on all three platforms
- No 4GB file size limit (unlike FAT32)
- Good for large external drives

### Formatting as ExFAT

```bash
# Find disk identifier
$ diskutil list
/dev/disk4 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *32.0 GB    disk4
   1:             Windows_FAT_32 UNTITLED                32.0 GB    disk4s1

# Format as ExFAT (WARNING: erases all data)
$ diskutil eraseDisk ExFAT SharedDrive /dev/disk4
Started erase on disk4
Unmounting disk
Creating the partition map
Waiting for partitions to activate
Formatting disk4s2 as ExFAT with name SharedDrive
Volume name      : SharedDrive
Partition offset : 411648 sectors (210763776 bytes)
Volume size      : 62078976 sectors (31784435712 bytes)
Bytes per sector : 512
Bytes per cluster: 32768
FAT offset       : 2048 sectors (1048576 bytes)
# FAT sectors    : 7680
Number of FATs   : 1
Cluster offset   : 9728 sectors (4980736 bytes)
# Clusters       : 969666
Volume Serial Number: XXXX-XXXX
Mounting disk
Finished erase on disk4
```

### ExFAT Limitations on macOS

ExFAT on macOS has some restrictions:

```bash
# No extended attributes support
$ xattr -w user.test "value" /Volumes/SharedDrive/file.txt
xattr: file.txt: No such xattr: user.test

# No symbolic links
$ ln -s file.txt /Volumes/SharedDrive/link.txt
ln: /Volumes/SharedDrive/link.txt: Function not implemented

# No hard links
$ ln file.txt /Volumes/SharedDrive/hardlink.txt
ln: /Volumes/SharedDrive/hardlink.txt: Operation not supported
```

## Network Share Considerations

When sharing files over the network, format differences still matter.

### SMB Shares from Linux

When accessing Linux SMB (Samba) shares from macOS:

```bash
# Mount with specific options
$ mount_smbfs //user@server/share /mnt/share

# For better compatibility with Linux servers
$ mount_smbfs -o nounix //user@server/share /mnt/share
```

Common issues:
- **Permission mapping**: Unix permissions don't map perfectly to SMB
- **Case sensitivity**: Linux share is case-sensitive; macOS client may cache incorrectly
- **Extended attributes**: May not be preserved across SMB

### NFS Shares

NFS is often more Unix-friendly:

```bash
# Mount NFS share
$ sudo mount -t nfs server:/export/share /mnt/share

# With specific options for Linux NFS servers
$ sudo mount -t nfs -o vers=3,resvport server:/export/share /mnt/share
```

See the [Working with NFS and SMB](./nfs-smb.md) chapter for detailed coverage.

## Archive Formats for Cross-Platform Sharing

### Recommended: tar.gz Without macOS Metadata

```bash
# Clean, cross-platform archive
$ xattr -rc folder/
$ COPYFILE_DISABLE=1 tar czf archive.tar.gz folder/
```

### Alternative: zip Without macOS Extras

```bash
# Create clean zip
$ zip -r -X archive.zip folder/

# Exclude macOS metadata files
$ zip -r archive.zip folder/ -x "*.DS_Store" -x "*__MACOSX*" -x "*._*"
```

### Receiving Archives from Other Systems

```bash
# Linux tar archives work fine on macOS
$ tar xzf linux-archive.tar.gz

# If you encounter extended attribute errors
$ tar xzf archive.tar.gz --no-xattrs 2>/dev/null
```

## Practical Workflow: Syncing Project with Linux Server

A complete workflow for keeping a project in sync:

```bash
#!/bin/bash
# sync-to-server.sh - Sync project to Linux server cleanly

PROJECT_DIR="$HOME/projects/myapp"
REMOTE_HOST="dev@linux-server"
REMOTE_DIR="/home/dev/projects/myapp"

# Remove extended attributes
xattr -rc "$PROJECT_DIR"

# Sync with rsync, excluding macOS-specific files
rsync -avz --delete \
    --exclude='.DS_Store' \
    --exclude='._*' \
    --exclude='.AppleDouble' \
    --exclude='__MACOSX' \
    --exclude='.Spotlight-*' \
    --exclude='.Trashes' \
    --exclude='.fseventsd' \
    "$PROJECT_DIR/" \
    "$REMOTE_HOST:$REMOTE_DIR/"
```

## Global Git Configuration

Configure Git globally for cross-platform work:

```bash
# ~/.gitconfig additions
$ cat >> ~/.gitconfig << 'EOF'

[core]
    # Handle line endings
    autocrlf = input

    # Warn about case issues
    ignoreCase = false

    # Ignore extended attribute changes
    ignoreStat = false

[transfer]
    # Stricter checking
    fsckObjects = true
EOF
```

## Summary

| Issue | Detection | Solution |
|-------|-----------|----------|
| Case sensitivity | `find . \| sort -f \| uniq -di` | Use case-sensitive volume or rename files |
| Extended attributes | `ls -l@`, `xattr -l` | `xattr -rc` before sharing |
| AppleDouble files | `find . -name "._*"` | `COPYFILE_DISABLE=1` or `dot_clean` |
| Line endings | `file` command, `cat -v` | `dos2unix` or configure `.gitattributes` |
| External drives | N/A | Format as ExFAT for universal compatibility |

Key practices:
1. **Clean files before sharing**: `xattr -rc folder/`
2. **Use clean archive creation**: `COPYFILE_DISABLE=1 tar czf`
3. **Configure Git properly**: `.gitattributes` with explicit line ending rules
4. **Test on target platform**: Especially for case sensitivity issues
5. **Use ExFAT for shared drives**: Works everywhere without drivers
