# File Operations and Manipulation

File operations on macOS have unique considerations: extended attributes, resource forks, ACLs, and macOS-specific tools like `ditto`. This chapter covers practical file operations while handling macOS-specific metadata correctly.

## cp: Copying Files

### Basic Operations (Mostly Compatible)

```bash
# Copy file (compatible)
$ cp source.txt dest.txt

# Copy directory recursively (compatible)
$ cp -R source_dir/ dest_dir/

# Preserve permissions and timestamps (compatible)
$ cp -p source.txt dest.txt

# Interactive mode - prompt before overwrite (compatible)
$ cp -i source.txt dest.txt

# Verbose output (compatible)
$ cp -v source.txt dest.txt
```

### macOS-Specific Considerations

```bash
# Preserve extended attributes and resource forks
$ cp -p source.txt dest.txt    # -p includes xattrs on macOS

# Explicitly preserve all metadata
$ cp -a source_dir/ dest_dir/   # Same as -pPR (preserves everything)

# Don't follow symbolic links (compatible)
$ cp -P symlink dest/

# Copy without extended attributes (macOS specific)
# There's no direct flag; use ditto or strip after
```

### Progress Indicator

```bash
# GNU cp has --progress (not available on macOS)
$ cp --progress large_file.iso /dest/    # Linux
cp: illegal option -- -                   # macOS

# macOS alternatives:

# 1. Use rsync with progress
$ rsync --progress large_file.iso /dest/

# 2. Use pv (pipe viewer, install via brew)
$ brew install pv
$ pv source.iso > dest/source.iso

# 3. Use macOS cp with Ctrl+T for status
$ cp large_file.iso /dest/
# Press Ctrl+T during copy to see progress
load: 2.45  cmd: cp 12345 uninterruptible 0.00u 1.23s
source.iso -> /dest/source.iso  45%
```

### Handling Conflicts

```bash
# Don't overwrite existing (compatible)
$ cp -n source.txt dest.txt

# Force overwrite (compatible)
$ cp -f source.txt dest.txt

# Update only if source is newer - GNU only
$ cp -u source.txt dest.txt

# macOS alternative for update behavior
$ rsync -u source.txt dest.txt
```

## mv: Moving and Renaming

### Basic Operations (Compatible)

```bash
# Move file
$ mv source.txt /new/location/

# Rename file
$ mv oldname.txt newname.txt

# Move directory
$ mv source_dir/ /new/location/

# Interactive mode
$ mv -i source.txt dest.txt

# Don't overwrite existing
$ mv -n source.txt dest.txt

# Force overwrite
$ mv -f source.txt dest.txt

# Verbose
$ mv -v source.txt dest.txt
```

### Update Mode (GNU only)

```bash
# GNU mv - only if source is newer
$ mv -u source.txt dest.txt

# macOS alternative
$ rsync -u --remove-source-files source.txt dest.txt
```

### Moving Across Volumes

When moving files between volumes, `mv` copies then deletes (can't just rename):

```bash
# Moving between volumes preserves metadata on macOS
$ mv file.txt /Volumes/ExternalDisk/

# For large moves, rsync gives progress
$ rsync -av --progress --remove-source-files source/ /Volumes/External/dest/
```

## rm: Removing Files

### Basic Operations (Compatible)

```bash
# Remove file
$ rm file.txt

# Remove directory and contents
$ rm -r directory/

# Force remove (no prompts)
$ rm -f file.txt

# Interactive mode
$ rm -i file.txt

# Verbose
$ rm -v file.txt
```

### Secure Delete

```bash
# Secure delete - macOS (deprecated but still works)
$ rm -P file.txt    # 3-pass overwrite before delete

# Note: On SSDs with TRIM, secure delete is less effective
# FileVault encryption is more reliable for security
```

### Safe Remove Practices

```bash
# Trash instead of delete (use trash-cli)
$ brew install trash-cli
$ trash file.txt    # Moves to Trash instead of deleting

# Or use AppleScript
$ osascript -e 'tell app "Finder" to delete POSIX file "'$(pwd)/file.txt'"'
```

## ditto: macOS's Superior Copy Tool

`ditto` is Apple's copy utility, designed for macOS-specific needs:

```bash
# Basic copy (preserves everything)
$ ditto source.txt dest.txt
$ ditto source_dir/ dest_dir/

# Copy with verbose output
$ ditto -V source_dir/ dest_dir/

# Preserve extended attributes and ACLs (default)
$ ditto source/ dest/

# Flatten to tar archive
$ ditto -c -k --sequesterRsrc source_dir/ archive.zip

# Extract from archive
$ ditto -x -k archive.zip dest_dir/
```

### Why Use ditto Over cp?

```bash
# ditto advantages:
# 1. Properly handles resource forks
# 2. Preserves HFS+ metadata
# 3. Can create/extract ZIP archives
# 4. Works correctly with bundles (apps)

# Copy an application bundle
$ ditto /Applications/TextEdit.app ~/Desktop/TextEdit.app

# cp -r might miss resource forks or metadata
# ditto is the safer choice for macOS files
```

### Creating Archives with ditto

```bash
# Create ZIP archive
$ ditto -c -k --sequesterRsrc --keepParent source_dir/ archive.zip

# Options:
#   -c : create archive
#   -k : use PKZip format
#   --sequesterRsrc : store resource forks in __MACOSX
#   --keepParent : include parent directory in archive

# Create archive without __MACOSX metadata
$ ditto -c -k --norsrc source_dir/ archive.zip
```

## rsync: Synchronization

macOS includes rsync, but it's an older version. The Homebrew version has more features:

```bash
# Check version
$ rsync --version
rsync  version 2.6.9  protocol version 29    # System version (old)

# Install newer version
$ brew install rsync
$ /opt/homebrew/bin/rsync --version
rsync  version 3.2.7  protocol version 31    # Much newer
```

### Basic Synchronization

```bash
# Sync directories
$ rsync -av source/ dest/

# Options breakdown:
#   -a : archive mode (preserves permissions, timestamps, etc.)
#   -v : verbose

# With progress
$ rsync -av --progress source/ dest/

# Dry run (show what would happen)
$ rsync -av --dry-run source/ dest/

# Delete files in dest that aren't in source
$ rsync -av --delete source/ dest/
```

### Preserving macOS Metadata

```bash
# System rsync doesn't have extended attribute support
# Homebrew rsync does

# Preserve extended attributes
$ rsync -avX source/ dest/    # -X for extended attributes

# Full macOS preservation (Homebrew rsync 3.x)
$ rsync -av --xattrs --fileflags source/ dest/

# For cross-platform transfers, you might want to strip metadata
$ rsync -av --no-xattrs source/ dest/
```

### Remote Synchronization

```bash
# Copy to remote (SSH)
$ rsync -av source/ user@remote:/path/to/dest/

# Copy from remote
$ rsync -av user@remote:/path/to/source/ dest/

# With compression for slow links
$ rsync -avz source/ user@remote:/dest/

# Limit bandwidth (KB/s)
$ rsync -av --bwlimit=1000 source/ user@remote:/dest/

# Resume interrupted transfer
$ rsync -av --partial --progress source/ dest/

# Combine for reliable large transfers
$ rsync -avz --partial --progress --bwlimit=5000 source/ user@remote:/dest/
```

### Excluding Files

```bash
# Exclude patterns
$ rsync -av --exclude='*.log' --exclude='.DS_Store' source/ dest/

# Exclude from file
$ rsync -av --exclude-from='exclude.txt' source/ dest/

# Common exclusions for macOS
$ rsync -av \
    --exclude='.DS_Store' \
    --exclude='._*' \
    --exclude='.Spotlight-*' \
    --exclude='.Trashes' \
    --exclude='.fseventsd' \
    source/ dest/
```

## Extended Attributes (xattr)

macOS stores additional metadata in extended attributes:

```bash
# List extended attributes
$ xattr file.txt
com.apple.quarantine

# View attribute content
$ xattr -p com.apple.quarantine file.txt
0083;5f3e8bc0;Chrome;XXXXXXXX

# Remove quarantine attribute (common need for downloaded files)
$ xattr -d com.apple.quarantine file.txt

# Remove all extended attributes
$ xattr -c file.txt

# Recursive operations
$ xattr -r -d com.apple.quarantine app_folder/

# Copy preserving xattrs
$ cp -p file.txt dest.txt    # -p preserves xattrs

# List with details
$ xattr -l file.txt
com.apple.quarantine: 0083;5f3e8bc0;Chrome;XXXXXXXX
com.apple.lastuseddate#PS: ... (binary data)
```

### Common Extended Attributes

| Attribute | Purpose |
|-----------|---------|
| `com.apple.quarantine` | Downloaded file, triggers Gatekeeper |
| `com.apple.lastuseddate#PS` | Last opened date |
| `com.apple.metadata:kMDItemWhereFroms` | Download URL |
| `com.apple.FinderInfo` | Finder metadata (labels, etc.) |
| `com.apple.ResourceFork` | Classic resource fork data |

### Stripping Metadata for Transfer

```bash
# Remove all macOS metadata before sharing
$ xattr -cr directory/

# Or use tar with no extended attributes
$ COPYFILE_DISABLE=1 tar cvf archive.tar directory/

# Copy without metadata using rsync
$ rsync -av --no-xattrs source/ dest/
```

## find: Finding Files

### Basic Usage (Mostly Compatible)

```bash
# Find by name (compatible)
$ find . -name "*.txt"

# Find by type (compatible)
$ find . -type f    # Files
$ find . -type d    # Directories

# Find by modification time (compatible)
$ find . -mtime -7    # Modified in last 7 days

# Find by size (compatible)
$ find . -size +100M    # Larger than 100MB

# Execute command on results (compatible)
$ find . -name "*.tmp" -exec rm {} \;

# Delete found files (compatible)
$ find . -name "*.tmp" -delete
```

### macOS-Specific Options

```bash
# Extended regex (BSD find)
$ find -E . -regex ".*\.(jpg|png|gif)"

# GNU find uses -regextype
$ find . -regextype posix-extended -regex ".*\.(jpg|png|gif)"

# Find with extended attributes
$ find . -xattrname com.apple.quarantine

# Find files by Spotlight metadata
$ mdfind -onlyin . "kMDItemFSSize > 1000000"
```

### Handling Spaces in Filenames

```bash
# Using -print0 and xargs -0 (compatible)
$ find . -name "*.txt" -print0 | xargs -0 rm

# Using -exec (compatible)
$ find . -name "*.txt" -exec rm {} \;
```

## mkdir: Creating Directories

```bash
# Create directory (compatible)
$ mkdir newdir

# Create parent directories as needed (compatible)
$ mkdir -p path/to/new/dir

# Set permissions on creation (compatible)
$ mkdir -m 755 newdir

# Verbose (compatible)
$ mkdir -v newdir
```

## ln: Creating Links

```bash
# Create hard link (compatible)
$ ln original.txt hardlink.txt

# Create symbolic link (compatible)
$ ln -s /path/to/original symlink

# Force overwrite existing link (compatible)
$ ln -sf /new/target symlink

# Create relative symbolic link
$ ln -s ../sibling/file.txt symlink
```

### macOS Link Behavior

```bash
# macOS symbolic links work across volumes
$ ln -s /Volumes/External/file.txt local_link

# Aliases vs Symbolic Links
# Finder aliases are NOT symbolic links
# They track file moves; symlinks don't

# Create Finder alias from command line
$ osascript -e 'tell application "Finder" to make alias file to POSIX file "/path/to/original" at POSIX file "/path/to/location"'
```

## chmod, chown: Permissions

### Basic Usage (Compatible)

```bash
# Change permissions (compatible)
$ chmod 755 script.sh
$ chmod u+x script.sh
$ chmod -R 644 directory/

# Change owner (compatible)
$ sudo chown user:group file.txt
$ sudo chown -R user:group directory/
```

### macOS Specific

```bash
# macOS also has ACLs
$ ls -le file.txt    # Shows ACL entries

# Remove ACLs
$ chmod -N file.txt

# View with full permissions
$ ls -l@ file.txt    # Shows extended attributes
```

## Touch: Create/Update Timestamps

```bash
# Create empty file or update timestamp (compatible)
$ touch file.txt

# Set specific modification time (BSD syntax)
$ touch -t 202401151200 file.txt    # YYYYMMDDhhmm

# Use another file's timestamp (compatible)
$ touch -r reference.txt target.txt

# Only update if file exists (compatible)
$ touch -c file.txt
```

## Practical Examples

### Backup a Directory Preserving Metadata

```bash
# Best method on macOS
$ ditto -V source_dir/ backup_dir/

# Or with rsync
$ rsync -av --progress source_dir/ backup_dir/
```

### Clean Up .DS_Store Files

```bash
# Find and delete
$ find . -name ".DS_Store" -delete

# Also remove ._* AppleDouble files
$ find . -name "._*" -delete

# Prevent .DS_Store on network volumes
$ defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true
```

### Remove Quarantine from Downloads

```bash
# Single file
$ xattr -d com.apple.quarantine download.dmg

# Entire directory
$ xattr -dr com.apple.quarantine ~/Downloads/

# From an app
$ xattr -cr /Applications/SomeApp.app
```

### Sync with External Drive

```bash
# Initial sync
$ rsync -av --progress ~/Documents/ /Volumes/Backup/Documents/

# Update (only changed files)
$ rsync -av --progress --delete ~/Documents/ /Volumes/Backup/Documents/
```

### Find Large Files

```bash
# Using find
$ find ~ -type f -size +100M 2>/dev/null

# Using Spotlight (faster)
$ mdfind -onlyin ~ "kMDItemFSSize > 104857600"

# Sorted by size
$ find ~ -type f -size +100M -exec ls -lh {} \; 2>/dev/null | sort -k5 -h
```

## Summary: macOS File Operations

| Task | Best Command | Notes |
|------|--------------|-------|
| Copy files | `ditto` or `cp -p` | ditto preserves everything |
| Sync directories | `rsync -av` | Use Homebrew rsync for xattr support |
| Large file copy | `rsync --progress` | Shows progress |
| Move to Trash | `trash` (brew) | Safer than rm |
| Remove quarantine | `xattr -d com.apple.quarantine` | Common need |
| Archive directory | `ditto -c -k` | Creates ZIP with metadata |
| Strip metadata | `xattr -cr` | Before sharing |
| Find files | `mdfind` | Uses Spotlight, much faster |

Understanding these tools and their macOS-specific behaviors helps you manage files effectively while preserving the metadata that macOS applications expect.
