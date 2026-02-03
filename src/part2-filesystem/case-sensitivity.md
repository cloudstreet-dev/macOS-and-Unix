# Case Sensitivity: Options and Implications

One of the most surprising aspects of macOS for Unix users is that the filesystem is case-insensitive by default. `File.txt` and `file.txt` refer to the same file. This design choice dates back to the original Macintosh and persists today, though case-sensitive options exist.

## Understanding Case Insensitivity

### Case-Insensitive, Case-Preserving

macOS filesystems (APFS and HFS+) are **case-insensitive but case-preserving**:

```bash
# Create a file
$ echo "content" > Hello.txt

# Reference it with different case - same file!
$ cat hello.txt
content

# But the original case is preserved
$ ls
Hello.txt

# You cannot create a second file that differs only in case
$ touch HELLO.txt
touch: HELLO.txt: No such file or directory
# Actually, this silently succeeds but doesn't create a new file
$ ls
Hello.txt
```

### Unicode Normalization

macOS also normalizes Unicode characters, treating different representations of the same character as identical:

```bash
# 'é' can be encoded as:
# - U+00E9 (precomposed: single character)
# - U+0065 U+0301 (decomposed: 'e' + combining acute accent)

# macOS treats these as the same filename
```

This affects filenames with accented characters, especially in cross-platform contexts.

## Why Case Insensitivity?

### Historical Reasons

The original Macintosh (1984) targeted non-technical users. Case sensitivity was seen as confusing—why should "Document" and "document" be different?

### Practical Reasons

Many Mac applications, including Finder, assume case insensitivity:

- File extensions: `.TXT` should equal `.txt`
- Application bundles: `Safari.app` and `safari.app` shouldn't be different
- User expectations: Most users don't think about case

### Compatibility

Changing to case-sensitive by default would break:
- Existing software assuming case insensitivity
- User workflows and muscle memory
- Cross-platform workflows where users expect Mac behavior

## Case-Sensitive Volumes

macOS does support case-sensitive filesystems:

```bash
# Check if a volume is case-sensitive
$ diskutil info / | grep "Name (Bundle)"
   Name (Bundle):                  APFS
# Standard APFS is case-insensitive

$ diskutil info / | grep "File System Personality"
   File System Personality:        APFS
# vs "Case-sensitive APFS" for case-sensitive volumes
```

### Creating Case-Sensitive Volumes

```bash
# Create a case-sensitive APFS volume
$ sudo diskutil apfs addVolume disk3 "Case-sensitive APFS" "CaseSensitive"

# Erase a disk with case-sensitive APFS
$ sudo diskutil eraseDisk "Case-sensitive APFS" "CaseSensitiveDisk" GPT disk2

# Create a case-sensitive HFS+
$ sudo diskutil eraseDisk JHFS+X "CaseSensitiveDisk" GPT disk2
# The X in JHFS+X denotes case-sensitive
```

### Testing Case Sensitivity

```bash
# Check if current directory is case-sensitive
$ touch test_CASE test_case 2>/dev/null
$ count=$(ls test_* 2>/dev/null | wc -l | tr -d ' ')
$ rm -f test_CASE test_case 2>/dev/null
$ if [ "$count" = "2" ]; then
    echo "Case-sensitive"
else
    echo "Case-insensitive"
fi
```

Or more simply:

```bash
# This tells you the filesystem type
$ diskutil info "$(df . | tail -1 | awk '{print $1}')" | grep "Personality"
```

## Problems with Case Insensitivity

### Git and Version Control

This is the most common pain point. Repositories from case-sensitive systems (Linux) can have issues:

```bash
# On a Linux server
$ ls
Makefile
makefile

# Clone to case-insensitive Mac
$ git clone git@server:repo.git
# Only one file appears! Git sees both but filesystem conflates them

# Git status shows changed files that aren't "really" changed
$ git status
On branch main
Changes not staged for commit:
    modified:   Makefile
```

**Workaround**: Use a case-sensitive disk image for development:

```bash
# Create sparse disk image with case-sensitive filesystem
$ hdiutil create -size 50g -fs "Case-sensitive APFS" -type SPARSE -volname "DevWork" ~/dev.sparseimage

# Mount it
$ hdiutil attach ~/dev.sparseimage
/dev/disk4s1    Apple_APFS
/dev/disk5s1    APFS Volume    /Volumes/DevWork

# Clone repositories there
$ cd /Volumes/DevWork
$ git clone git@server:problematic-repo.git
```

### Build Systems and Dependencies

Some software requires case sensitivity:

```bash
# Building software that expects case-sensitive filesystem
$ ./configure
checking for case-sensitive filesystem... no
configure: error: This package requires a case-sensitive filesystem.
```

### Filename Conflicts

Files from case-sensitive systems may conflict:

```bash
# Archive from Linux might contain:
#   config.h
#   Config.h
#
# Extracting on macOS loses one file

# Check for conflicts before extracting
$ tar -tf archive.tar.gz | sort -f | uniq -di
# Lists filenames that would conflict
```

## Problems with Case Sensitivity

Using a case-sensitive boot volume causes its own problems:

### Adobe Software

Adobe applications (Photoshop, Illustrator, etc.) famously refuse to install on case-sensitive volumes:

```
"This product cannot be installed on a case-sensitive file system."
```

### Steam and Games

Many games and game launchers assume case insensitivity:

```bash
# Game looks for "/path/to/data.dat"
# Actually stored as "/path/to/Data.dat"
# Works on Windows, fails on case-sensitive Mac
```

### Apple's Own Applications

Some older Apple software had issues with case-sensitive volumes. While modern macOS mostly works, testing revealed enough edge cases that Apple kept case-insensitive as default.

## Best Practices

### For Most Users

Keep the default case-insensitive filesystem. It's what macOS is designed for and tested with.

### For Developers

Options for handling case-sensitive requirements:

**Option 1: Case-sensitive disk image (Recommended)**

```bash
# Create sparse image that grows as needed
$ hdiutil create -size 100g -fs "Case-sensitive APFS" \
    -type SPARSE -volname "Code" ~/code.sparseimage

# Add to login items or create a mount script
$ cat > ~/mount-code.sh << 'EOF'
#!/bin/bash
if [ ! -d "/Volumes/Code" ]; then
    hdiutil attach ~/code.sparseimage
fi
EOF
```

**Option 2: Separate case-sensitive volume**

```bash
# Add volume to existing APFS container
$ sudo diskutil apfs addVolume disk3 "Case-sensitive APFS" "Code"
```

**Option 3: Linux virtual machine or container**

For maximum compatibility with Linux-centric projects:

```bash
# Use Docker for builds
$ docker run -v "$(pwd):/code" -w /code alpine make
```

### For Cross-Platform Projects

When maintaining projects used on both case-sensitive and case-insensitive systems:

1. **Establish naming conventions**: Always use lowercase, or establish explicit conventions
2. **Add CI checks**: Detect case-only differences in commits

```bash
# Git hook to check for case conflicts
#!/bin/bash
git ls-files | sort -f | uniq -di && exit 1
```

3. **Document requirements**: If case sensitivity matters, document it clearly

## Detecting and Handling Case Issues

### Find Case Conflicts in a Directory

```bash
# Find files/directories that differ only in case
$ find . -maxdepth 1 -print | sort -f | uniq -di

# More thorough check
$ find . -print | sed 's|.*/||' | sort -f | uniq -di
```

### Git Configuration for Case Issues

```bash
# Tell Git to notice case-only renames
$ git config core.ignorecase false

# This can cause issues on case-insensitive filesystems
# as Git may report phantom changes
```

### Safe Renaming

On case-insensitive systems, renaming with case changes requires two steps:

```bash
# This fails (same file)
$ mv File.txt file.txt

# Two-step rename
$ mv File.txt File.txt.tmp
$ mv File.txt.tmp file.txt

# Or use git for tracked files
$ git mv File.txt file.txt
# Git handles the two-step internally
```

## Summary

Case sensitivity on macOS is a trade-off:

**Case-insensitive (default)** provides:
- Maximum compatibility with Mac software
- Familiar behavior for most users
- Fewer surprises with existing workflows

**Case-sensitive** provides:
- Linux/Unix compatibility
- Correct handling of repositories from case-sensitive systems
- Required by some build systems

For most users, the default case-insensitive filesystem is correct. Developers working with cross-platform code should consider case-sensitive disk images or volumes for their development work, while keeping the boot volume case-insensitive for maximum application compatibility.
