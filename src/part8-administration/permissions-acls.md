# The Permissions Model: Unix Meets ACLs

macOS implements a layered permission system that combines traditional Unix permissions with Access Control Lists (ACLs). Understanding both layers is essential for proper file security management, especially when troubleshooting access issues.

## The Three Layers of Access Control

macOS evaluates file access through three layers:

```
1. System Integrity Protection (SIP)
        │
        ▼ (if allowed)
2. Access Control Lists (ACLs)
        │
        ▼ (if no ACL match)
3. Traditional Unix Permissions
```

This chapter focuses on layers 2 and 3. SIP is covered in its own chapter.

## Traditional Unix Permissions

### The Basics

Unix permissions use a three-tier model:

```bash
$ ls -l document.txt
-rw-r--r--  1 david  staff  1024 Jan 15 10:00 document.txt
```

Breaking down `-rw-r--r--`:

| Position | Meaning |
|----------|---------|
| 1 | File type (`-` file, `d` directory, `l` symlink) |
| 2-4 | Owner permissions (user) |
| 5-7 | Group permissions |
| 8-10 | Other permissions (everyone else) |

Permission bits:

| Symbol | Octal | Meaning for Files | Meaning for Directories |
|--------|-------|-------------------|------------------------|
| r | 4 | Read contents | List contents |
| w | 2 | Modify contents | Create/delete files |
| x | 1 | Execute | Enter (cd into) |

### Reading Permissions

```bash
# Long listing
$ ls -l /Users/david/
total 0
drwx------+  5 david  staff   160 Jan 15 10:00 Desktop
drwx------+  8 david  staff   256 Jan 15 10:00 Documents
drwx------+  3 david  staff    96 Jan 15 10:00 Downloads
drwx------@ 85 david  staff  2720 Jan 15 10:00 Library
drwx------   6 david  staff   192 Jan 15 10:00 Movies
drwx------+  4 david  staff   128 Jan 15 10:00 Music
drwx------+  5 david  staff   160 Jan 15 10:00 Pictures
drwxr-xr-x+  4 david  staff   128 Jan 15 10:00 Public

# Note the + and @ symbols:
# + indicates ACLs present
# @ indicates extended attributes present
```

### Numeric (Octal) Permissions

Each permission set converts to a number 0-7:

```bash
# Calculate: r(4) + w(2) + x(1)
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0

# Common permission sets
-rw-r--r--  = 644  (owner read/write, others read)
-rwxr-xr-x  = 755  (executable, world readable)
-rw-------  = 600  (owner only)
drwx------  = 700  (private directory)
drwxr-xr-x  = 755  (public directory)
```

### Changing Permissions with chmod

```bash
# Symbolic mode
$ chmod u+x script.sh           # Add execute for owner
$ chmod g-w file.txt            # Remove write for group
$ chmod o=r file.txt            # Set others to read only
$ chmod a+r file.txt            # Add read for all (a = all)
$ chmod u=rwx,g=rx,o=r file.txt # Set all at once

# Numeric mode
$ chmod 755 script.sh           # rwxr-xr-x
$ chmod 644 document.txt        # rw-r--r--
$ chmod 600 secret.key          # rw-------
$ chmod 700 private_dir         # rwx------

# Recursive
$ chmod -R 755 directory/       # Apply to all contents
$ chmod -R u+w directory/       # Add write for owner recursively
```

### Changing Ownership with chown

```bash
# Change owner
$ sudo chown newowner file.txt

# Change owner and group
$ sudo chown newowner:newgroup file.txt

# Change only group
$ sudo chown :newgroup file.txt
$ chgrp newgroup file.txt       # Alternative

# Recursive
$ sudo chown -R david:staff directory/

# Follow symlinks
$ sudo chown -H david symlink   # Affect target of symlink

# Don't follow symlinks
$ sudo chown -h david symlink   # Affect symlink itself
```

### Special Permission Bits

macOS supports the special Unix permission bits:

```bash
# setuid (4xxx) - Run as file owner
$ ls -l /usr/bin/sudo
-r-s--x--x  1 root  wheel  378848 Jan  1 00:00 /usr/bin/sudo
#   ^-- 's' indicates setuid

# setgid (2xxx) - Run as file group / inherit directory group
$ chmod 2755 shared_dir/

# Sticky bit (1xxx) - Only owner can delete files in directory
$ ls -ld /tmp
drwxrwxrwt  12 root  wheel  384 Jan 15 10:00 /tmp
#        ^-- 't' indicates sticky bit

# Set sticky bit
$ chmod 1777 shared_dir/
$ chmod +t shared_dir/
```

### Default Permissions: umask

The `umask` determines default permissions for new files:

```bash
# View current umask
$ umask
022

# What this means:
# Files: 666 - 022 = 644 (rw-r--r--)
# Dirs:  777 - 022 = 755 (rwxr-xr-x)

# More restrictive umask
$ umask 077
# Files: 666 - 077 = 600 (rw-------)
# Dirs:  777 - 077 = 700 (rwx------)

# Set in shell profile (~/.zshrc)
umask 022
```

## Access Control Lists (ACLs)

ACLs provide fine-grained permissions beyond the user/group/other model. macOS uses NFSv4-style ACLs.

### Viewing ACLs

```bash
# View ACLs with ls
$ ls -le ~/Documents
total 0
drwx------+ 3 david  staff  96 Jan 15 10:00 Projects
 0: group:everyone deny delete

# More detailed view
$ ls -le document.txt
-rw-r--r--+ 1 david  staff  1024 Jan 15 10:00 document.txt
 0: user:alice allow read
 1: group:developers allow read,write
 2: group:everyone deny write
```

ACL entries are numbered and evaluated in order.

### ACL Entry Format

Each ACL entry has:

```
<type>:<name> <action> <permissions>
```

Types:
- `user:username` - Specific user
- `group:groupname` - Specific group

Actions:
- `allow` - Grant permission
- `deny` - Explicitly deny permission

Permissions for files:

| Permission | Meaning |
|------------|---------|
| read | Read file contents |
| write | Modify file contents |
| execute | Execute file |
| delete | Delete file |
| append | Append to file |
| readattr | Read attributes |
| writeattr | Write attributes |
| readextattr | Read extended attributes |
| writeextattr | Write extended attributes |
| readsecurity | Read ACL |
| writesecurity | Modify ACL |
| chown | Change ownership |

Permissions for directories:

| Permission | Meaning |
|------------|---------|
| list | List directory contents |
| search | Access files in directory |
| add_file | Create files |
| add_subdirectory | Create subdirectories |
| delete_child | Delete items in directory |
| readattr | Read attributes |
| writeattr | Write attributes |
| readextattr | Read extended attributes |
| writeextattr | Write extended attributes |
| readsecurity | Read ACL |
| writesecurity | Modify ACL |
| chown | Change ownership |

### Adding ACL Entries

```bash
# Grant read access to specific user
$ chmod +a "user:alice allow read" document.txt

# Grant read/write to a group
$ chmod +a "group:developers allow read,write" project/

# Deny write to everyone
$ chmod +a "group:everyone deny write" readonly.txt

# Add ACL at specific position (0 = first)
$ chmod +a# 0 "user:bob deny write" file.txt

# Grant full control
$ chmod +a "user:admin allow read,write,execute,delete,append,readattr,writeattr,readextattr,writeextattr,readsecurity,writesecurity,chown" file.txt
```

### Inheritance (Directories)

Directory ACLs can be inherited by new files:

```bash
# Add inherited ACL for files
$ chmod +a "group:developers allow read,write,file_inherit" project/

# Add inherited ACL for directories
$ chmod +a "group:developers allow read,write,execute,directory_inherit" project/

# Both inheritance types
$ chmod +a "group:developers allow read,write,file_inherit,directory_inherit" project/

# Limit inheritance to one level
$ chmod +a "group:developers allow read,write,file_inherit,limit_inherit" project/
```

Inheritance flags:

| Flag | Meaning |
|------|---------|
| file_inherit | Apply to new files |
| directory_inherit | Apply to new subdirectories |
| limit_inherit | Don't propagate beyond direct children |
| only_inherit | Don't apply to directory itself, only children |

### Modifying ACL Entries

```bash
# View current ACLs
$ ls -le file.txt
-rw-r--r--+ 1 david  staff  1024 Jan 15 10:00 file.txt
 0: user:alice allow read
 1: group:developers allow read,write

# Remove specific entry by index
$ chmod -a# 0 file.txt

# Remove entry by content
$ chmod -a "user:alice allow read" file.txt

# Remove all ACLs
$ chmod -N file.txt

# Replace an entry at position
$ chmod =a# 0 "user:alice allow read,write" file.txt
```

### Reordering ACLs

Order matters. Entries are evaluated first to last, and first match wins:

```bash
# Current order (deny evaluated before allow)
$ ls -le file.txt
 0: group:everyone deny write
 1: user:alice allow write

# Alice CANNOT write because deny comes first

# Reorder to allow Alice
$ chmod -a "group:everyone deny write" file.txt
$ chmod +a "user:alice allow write" file.txt
$ chmod +a "group:everyone deny write" file.txt

# New order
$ ls -le file.txt
 0: user:alice allow write
 1: group:everyone deny write

# Now Alice CAN write
```

### Recursive ACL Operations

```bash
# Add ACL recursively
$ chmod -R +a "group:developers allow read" project/

# Remove all ACLs recursively
$ chmod -RN project/
```

## Common Permission Patterns

### Shared Project Directory

```bash
# Create shared directory
$ mkdir /Users/Shared/project
$ sudo chown :developers /Users/Shared/project
$ sudo chmod 2775 /Users/Shared/project

# Add ACL for team access with inheritance
$ sudo chmod +a "group:developers allow read,write,execute,delete,add_file,add_subdirectory,delete_child,file_inherit,directory_inherit" /Users/Shared/project
```

### Read-Only for Most, Write for Few

```bash
# Base permissions: readable by all
$ chmod 644 document.txt

# ACL: specific users can write
$ chmod +a "user:editor allow write" document.txt
$ chmod +a "group:admins allow write" document.txt
```

### Private User Directories

```bash
# Standard macOS home directory permissions
$ ls -ld ~
drwxr-x---+ 65 david  staff  2080 Jan 15 10:00 /Users/david

# ACL allows group access (for sharing)
$ ls -le ~
 0: group:everyone deny delete
```

### Web Server Content

```bash
# Web-accessible but protected
$ sudo chown -R www:www /var/www/html
$ sudo chmod -R 755 /var/www/html
$ sudo find /var/www/html -type f -exec chmod 644 {} \;

# Writable upload directory
$ sudo chmod 775 /var/www/html/uploads
$ sudo chmod +a "group:www allow write,add_file,delete_child" /var/www/html/uploads
```

## Troubleshooting Permissions

### Permission Denied

```bash
# Check file permissions
$ ls -la file.txt
-rw------- 1 root wheel 0 Jan 15 10:00 file.txt

# Check ACLs
$ ls -le file.txt

# Check your effective UID/GID
$ id
uid=501(david) gid=20(staff)

# Check if SIP is blocking
$ ls -lO file.txt
-rw-r--r--  1 root  wheel  restricted file.txt
#                          ^-- restricted flag = SIP protected
```

### Resetting Permissions

```bash
# Reset to standard file permissions
$ chmod 644 file.txt
$ chmod -N file.txt  # Remove ACLs

# Reset directory
$ chmod 755 directory/
$ chmod -RN directory/  # Remove all ACLs

# Reset home directory permissions
$ sudo diskutil resetUserPermissions / $(id -u)
```

### Finding Permission Issues

```bash
# Find files you don't own
$ find /path -not -user $(whoami) 2>/dev/null

# Find files with unusual permissions
$ find /path -perm -002 -type f  # World-writable files
$ find /path -perm -4000 -type f  # Setuid files

# Find files with ACLs
$ ls -leR /path 2>/dev/null | grep -B1 "^[[:space:]]*[0-9]:"
```

## Extended Attributes and Flags

macOS also uses extended attributes and file flags:

### Extended Attributes

```bash
# View extended attributes
$ ls -l@ file.txt
-rw-r--r--@ 1 david  staff  1024 Jan 15 10:00 file.txt
    com.apple.quarantine	     57

# List all extended attributes
$ xattr file.txt
com.apple.quarantine

# View attribute value
$ xattr -p com.apple.quarantine file.txt
0083;5f123456;Safari;XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX

# Remove quarantine attribute
$ xattr -d com.apple.quarantine file.txt

# Remove all extended attributes
$ xattr -c file.txt
```

### File Flags (chflags)

```bash
# View flags
$ ls -lO /System
drwxr-xr-x  restricted /System

# Common flags
$ chflags hidden file.txt     # Hide from Finder
$ chflags nohidden file.txt   # Unhide

$ chflags uchg file.txt       # User immutable (can't modify)
$ chflags nouchg file.txt     # Remove immutable

$ sudo chflags schg file.txt  # System immutable (root only)
$ sudo chflags noschg file.txt

# The 'restricted' flag indicates SIP protection
# Cannot be changed without disabling SIP
```

## Comparison with Linux

| Feature | Linux | macOS |
|---------|-------|-------|
| Basic permissions | Same | Same |
| View ACLs | `getfacl` | `ls -le` |
| Set ACLs | `setfacl` | `chmod +a` |
| ACL format | POSIX ACLs | NFSv4 ACLs |
| Extended attributes | `getfattr`/`setfattr` | `xattr` |
| File flags | `chattr` | `chflags` |
| Mandatory access | SELinux, AppArmor | SIP, Sandbox |

## Summary

macOS permissions combine multiple systems:

| Layer | Tool | Purpose |
|-------|------|---------|
| Unix permissions | `chmod`, `chown` | Basic access control |
| ACLs | `chmod +a`, `ls -le` | Fine-grained access |
| Extended attributes | `xattr` | Metadata (quarantine, etc.) |
| File flags | `chflags` | Special protection |
| SIP | `csrutil` | System protection |

Key commands:

```bash
# View everything
$ ls -laeO@ file.txt

# Manage permissions
$ chmod 644 file.txt
$ chmod +a "user:alice allow read" file.txt
$ chmod -N file.txt

# Manage ownership
$ sudo chown user:group file.txt

# Manage attributes
$ xattr -l file.txt
$ xattr -d com.apple.quarantine file.txt

# Manage flags
$ chflags hidden file.txt
```

Understanding all these layers ensures you can properly secure files and troubleshoot access problems on macOS.
