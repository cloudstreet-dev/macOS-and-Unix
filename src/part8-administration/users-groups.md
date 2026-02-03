# User and Group Management

macOS doesn't use `/etc/passwd` and `useradd` like Linux. Instead, it relies on Directory Services, a flexible system that can manage local accounts and integrate with directory servers like Active Directory or LDAP. Understanding this difference is essential for macOS system administration.

## Directory Services Architecture

On macOS, all user and group information flows through Directory Services (previously called Open Directory):

```
Application
    │
    ▼
Directory Services API
    │
    ├── Local Directory (/var/db/dslocal/)
    │       └── Local users, groups, computers
    │
    ├── Active Directory
    │       └── Domain users, groups
    │
    └── LDAP
            └── Network directory users
```

The key commands for interacting with Directory Services are:

| Command | Purpose |
|---------|---------|
| `dscl` | Directory Service command line utility |
| `sysadminctl` | Modern user management tool (10.10+) |
| `dseditgroup` | Group membership management |
| `dscacheutil` | Query and flush directory cache |
| `id` | Display user and group IDs |

## Understanding UIDs and GIDs

macOS uses Unix UIDs (User IDs) and GIDs (Group IDs), but with some Apple-specific conventions:

```bash
# View your UID and GIDs
$ id
uid=501(david) gid=20(staff) groups=20(staff),12(everyone),61(localaccounts),
79(_appserverusr),80(admin),81(_appserveradm),98(_lpadmin),701(com.apple.sharepoint.group.1),
33(_appstore),100(_lpoperator),204(_developer),250(_analyticsusers),395(com.apple.access_ftp),
398(com.apple.access_screensharing),399(com.apple.access_ssh)

# View another user's info
$ id -P admin
admin:********:501:20::0:0:Admin User:/Users/admin:/bin/zsh
```

Standard macOS UID ranges:

| UID Range | Purpose |
|-----------|---------|
| 0 | root |
| 1-199 | System accounts (daemon, nobody, etc.) |
| 200-400 | System services |
| 401-500 | Reserved |
| 501+ | Regular users |

First user created during setup gets UID 501:

```bash
$ dscl . -read /Users/david UniqueID
UniqueID: 501
```

## Listing Users and Groups

### Using dscl

```bash
# List all local users
$ dscl . -list /Users
_amavisd
_appleevents
_applepay
...
daemon
david
Guest
nobody
root

# List users with UIDs
$ dscl . -list /Users UniqueID
_amavisd                211
_appleevents            55
_applepay               260
...
david                   501
root                    0

# List only "real" users (UID >= 500)
$ dscl . -list /Users UniqueID | awk '$2 >= 500 {print $1}'
david
admin

# List all local groups
$ dscl . -list /Groups
admin
everyone
staff
wheel
...

# List groups with GIDs
$ dscl . -list /Groups PrimaryGroupID
admin                   80
everyone                12
staff                   20
wheel                   0
```

### Using dscacheutil

```bash
# Query user information
$ dscacheutil -q user -a name david
name: david
password: ********
uid: 501
gid: 20
dir: /Users/david
shell: /bin/zsh
gecos: David

# Query group
$ dscacheutil -q group -a name admin
name: admin
password: *
gid: 80
users: root david

# Flush directory cache
$ sudo dscacheutil -flushcache
```

## Reading User Attributes

Each user has many attributes in Directory Services:

```bash
# Read all attributes for a user
$ dscl . -read /Users/david
AppleMetaNodeLocation: /Local/Default
GeneratedUID: XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
NFSHomeDirectory: /Users/david
Password: ********
PrimaryGroupID: 20
RealName: David
RecordName: david
RecordType: dsRecTypeStandard:Users
UniqueID: 501
UserShell: /bin/zsh
...

# Read specific attribute
$ dscl . -read /Users/david RealName
RealName: David

$ dscl . -read /Users/david UserShell
UserShell: /bin/zsh

$ dscl . -read /Users/david NFSHomeDirectory
NFSHomeDirectory: /Users/david

# Check if user is admin
$ dscl . -read /Groups/admin GroupMembership
GroupMembership: root david

# Raw value only
$ dscl . -read /Users/david UniqueID | awk '{print $2}'
501
```

## Creating Users

### Using sysadminctl (Recommended)

`sysadminctl` is the modern tool for user management:

```bash
# Create a standard user (interactive password)
$ sudo sysadminctl -addUser newuser -fullName "New User" -password -
Enter password for new user newuser:
Confirm password for new user newuser:
Creating user record...
User record created successfully.

# Create admin user
$ sudo sysadminctl -addUser newadmin -fullName "New Admin" -password - -admin

# Create user with specific UID
$ sudo sysadminctl -addUser newuser -fullName "New User" -UID 601 -password -

# Create user with specified shell
$ sudo sysadminctl -addUser newuser -fullName "New User" -shell /bin/bash -password -

# Create hidden user (doesn't appear on login screen)
$ sudo sysadminctl -addUser hiddenuser -fullName "Hidden User" -password - -admin
$ sudo dscl . create /Users/hiddenuser IsHidden 1
```

### Using dscl (Manual Method)

For more control, create users with `dscl`:

```bash
# Find next available UID
$ dscl . -list /Users UniqueID | awk '{print $2}' | sort -n | tail -1
501
# So next UID would be 502

# Create user record
$ sudo dscl . -create /Users/newuser

# Set required attributes
$ sudo dscl . -create /Users/newuser UserShell /bin/zsh
$ sudo dscl . -create /Users/newuser RealName "New User"
$ sudo dscl . -create /Users/newuser UniqueID 502
$ sudo dscl . -create /Users/newuser PrimaryGroupID 20
$ sudo dscl . -create /Users/newuser NFSHomeDirectory /Users/newuser

# Set password
$ sudo dscl . -passwd /Users/newuser
New Password:

# Create home directory
$ sudo createhomedir -c -u newuser

# Or manually create home directory from template
$ sudo cp -R /System/Library/User\ Template/English.lproj /Users/newuser
$ sudo chown -R newuser:staff /Users/newuser
```

## Modifying Users

```bash
# Change user's full name
$ sudo dscl . -change /Users/david RealName "David" "David Smith"

# Change shell
$ sudo dscl . -change /Users/david UserShell /bin/zsh /bin/bash

# Or create/replace attribute
$ sudo dscl . -create /Users/david UserShell /bin/bash

# Change password
$ sudo dscl . -passwd /Users/david
New Password:

# User changes own password
$ passwd
Changing password for david.
Old Password:
New Password:
Retype New Password:

# Disable user (prevent login)
$ sudo pwpolicy -u newuser -setpolicy "isDisabled=1"

# Re-enable user
$ sudo pwpolicy -u newuser -setpolicy "isDisabled=0"
```

## Deleting Users

```bash
# Delete user with sysadminctl
$ sudo sysadminctl -deleteUser olduser
Deleting user record...
User "olduser" deleted.

# Delete user AND home directory
$ sudo sysadminctl -deleteUser olduser -secure

# Delete user with dscl
$ sudo dscl . -delete /Users/olduser

# Manually remove home directory
$ sudo rm -rf /Users/olduser
```

## Group Management

### Viewing Groups

```bash
# List all groups
$ dscl . -list /Groups PrimaryGroupID
admin                   80
daemon                  1
everyone                12
kmem                    2
localaccounts           61
mail                    6
network                 69
nobody                  -2
nogroup                 -1
operator                5
staff                   20
sys                     3
tty                     4
utmp                    45
wheel                   0
...

# Read group details
$ dscl . -read /Groups/admin
AppleMetaNodeLocation: /Local/Default
GeneratedUID: ABCDEFAB-CDEF-ABCD-EFAB-CDEF00000050
GroupMembership: root david
Password: *
PrimaryGroupID: 80
RealName: Administrators
RecordName: admin BUILTIN\Administrators
RecordType: dsRecTypeStandard:Groups

# List group members
$ dscl . -read /Groups/admin GroupMembership
GroupMembership: root david

# Check if user is in group
$ dseditgroup -o checkmember -m david admin
yes david is a member of admin
```

### Creating Groups

```bash
# Create new group
$ sudo dscl . -create /Groups/developers

# Set group ID
$ sudo dscl . -create /Groups/developers PrimaryGroupID 1001

# Set group name
$ sudo dscl . -create /Groups/developers RealName "Developers"

# Add group password (rarely needed)
$ sudo dscl . -create /Groups/developers Password "*"
```

### Managing Group Membership

```bash
# Add user to group (using dseditgroup)
$ sudo dseditgroup -o edit -a david -t user admin
$ sudo dseditgroup -o edit -a david -t user developers

# Remove user from group
$ sudo dseditgroup -o edit -d david -t user developers

# Add user to group (using dscl)
$ sudo dscl . -append /Groups/developers GroupMembership david

# Remove user from group (using dscl)
$ sudo dscl . -delete /Groups/developers GroupMembership david

# List all groups a user belongs to
$ id -Gn david
staff everyone localaccounts _appserverusr admin _appserveradm _lpadmin com.apple.sharepoint.group.1 _appstore _lpoperator _developer _analyticsusers com.apple.access_ftp com.apple.access_screensharing com.apple.access_ssh
```

### Deleting Groups

```bash
# Delete group
$ sudo dscl . -delete /Groups/developers
```

## Special macOS Groups

macOS has several important groups:

| Group | GID | Purpose |
|-------|-----|---------|
| wheel | 0 | Traditional Unix superuser group |
| admin | 80 | macOS administrators (can use sudo) |
| staff | 20 | Default group for regular users |
| everyone | 12 | All users including guests |
| localaccounts | 61 | All local (non-network) accounts |
| _developer | 204 | Can debug other processes |

Adding a user to `admin` makes them a macOS administrator:

```bash
# Make user an admin
$ sudo dseditgroup -o edit -a username -t user admin

# Remove admin rights
$ sudo dseditgroup -o edit -d username -t user admin

# Verify admin status
$ dseditgroup -o checkmember -m username admin
```

## Working with Network Directories

### Checking Directory Configuration

```bash
# List configured directory nodes
$ dscl -list /
Active Directory
BSD Configuration and target
Cache
Local
Search
Contact Search

# Read from specific node
$ dscl "/Active Directory/DOMAIN" -list /Users
```

### Active Directory Integration

```bash
# Check AD binding status
$ dsconfigad -show
Active Directory Forest          = corp.example.com
Active Directory Domain          = corp.example.com
Computer Account                 = MY-MAC$
...

# Bind to Active Directory (run from GUI or carefully from CLI)
$ sudo dsconfigad -add corp.example.com -username admin -password - -computer MY-MAC

# Remove AD binding
$ sudo dsconfigad -remove -username admin -password -

# Force AD cache refresh
$ sudo dscacheutil -flushcache
```

### Directory Service Paths

```bash
# Local directory data location
/var/db/dslocal/

# Per-node data
/var/db/dslocal/nodes/Default/
    ├── aliases/
    ├── computers/
    ├── config/
    ├── groups/
    └── users/

# User plist files
$ sudo ls /var/db/dslocal/nodes/Default/users/
_amavisd.plist
_appleevents.plist
daemon.plist
david.plist
nobody.plist
root.plist
```

## Common Administration Tasks

### Reset a User's Password

```bash
# As admin, reset another user's password
$ sudo dscl . -passwd /Users/targetuser newpassword

# Or interactively
$ sudo dscl . -passwd /Users/targetuser
New Password:

# Using sysadminctl
$ sudo sysadminctl -resetPasswordFor targetuser -newPassword -
```

### Find All Admin Users

```bash
# List all administrators
$ dscl . -read /Groups/admin GroupMembership
GroupMembership: root david admin2

# More detailed
$ dscl . -read /Groups/admin GroupMembership | tr ' ' '\n' | tail -n +2
root
david
admin2
```

### Create Service Account

Service accounts typically have no home directory and can't log in:

```bash
# Create service account
$ sudo dscl . -create /Users/_myservice
$ sudo dscl . -create /Users/_myservice UserShell /usr/bin/false
$ sudo dscl . -create /Users/_myservice RealName "My Service"
$ sudo dscl . -create /Users/_myservice UniqueID 401
$ sudo dscl . -create /Users/_myservice PrimaryGroupID 401
$ sudo dscl . -create /Users/_myservice NFSHomeDirectory /var/empty

# Create matching group
$ sudo dscl . -create /Groups/_myservice
$ sudo dscl . -create /Groups/_myservice PrimaryGroupID 401

# Set password to * (can't login)
$ sudo dscl . -create /Users/_myservice Password "*"
```

### List Users Who Can SSH

```bash
# SSH access is controlled by the com.apple.access_ssh group
$ dscl . -read /Groups/com.apple.access_ssh GroupMembership 2>/dev/null || echo "Group doesn't exist - all users can SSH"

# Add user to SSH access group
$ sudo dseditgroup -o edit -a username -t user com.apple.access_ssh

# Remove SSH access
$ sudo dseditgroup -o edit -d username -t user com.apple.access_ssh
```

## Comparison with Linux

| Task | Linux | macOS |
|------|-------|-------|
| List users | `cat /etc/passwd` | `dscl . -list /Users` |
| Add user | `useradd` | `sysadminctl -addUser` |
| Delete user | `userdel` | `sysadminctl -deleteUser` |
| Modify user | `usermod` | `dscl . -change` |
| Add to group | `usermod -aG group user` | `dseditgroup -o edit -a user -t user group` |
| List groups | `cat /etc/group` | `dscl . -list /Groups` |
| Change password | `passwd user` | `dscl . -passwd /Users/user` |
| User info | `id user` | `id user` (same) |

## Summary

macOS user management through Directory Services is more complex than Linux's `/etc/passwd` approach, but offers advantages:

- Unified interface for local and network accounts
- Rich metadata on user records
- Integration with enterprise directories
- Consistent API across different backends

Key commands to remember:

| Command | Purpose |
|---------|---------|
| `dscl . -list /Users` | List all users |
| `dscl . -read /Users/name` | Read user details |
| `sysadminctl -addUser` | Create new user |
| `sysadminctl -deleteUser` | Delete user |
| `dseditgroup -o edit -a user -t user group` | Add to group |
| `id username` | Show UID/GIDs |
| `dscacheutil -flushcache` | Clear directory cache |

Master these tools, and you'll manage macOS users as effectively as any Linux sysadmin manages their systems.
