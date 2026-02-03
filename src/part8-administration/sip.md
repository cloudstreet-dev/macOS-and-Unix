# System Integrity Protection (SIP)

System Integrity Protection, introduced in OS X El Capitan (10.11), is macOS's kernel-level protection mechanism that prevents even root from modifying critical system files. Understanding SIP is essential for any macOS administrator, especially when troubleshooting software that seems to inexplicably fail despite having proper permissions.

## What SIP Protects

SIP protects several categories of system resources:

### Protected Directories

```bash
# These directories are protected by SIP
/System/
/usr/          # Except /usr/local/
/bin/
/sbin/
/var/          # Some subdirectories

# These are NOT protected
/usr/local/
/Applications/
/Library/
~/
```

Even as root, you cannot modify protected paths:

```bash
$ sudo touch /System/test
touch: /System/test: Operation not permitted

$ sudo rm /bin/ls
rm: /bin/ls: Operation not permitted

$ sudo mv /usr/bin/zip /usr/bin/zip.bak
mv: rename /usr/bin/zip to /usr/bin/zip.bak: Operation not permitted
```

### Protected System Processes

SIP prevents:
- Attaching debuggers to system processes
- Modifying running system processes
- Loading unsigned kernel extensions (kexts)
- Injecting code into protected processes

```bash
# Cannot attach to system processes
$ sudo lldb -p $(pgrep Finder)
error: attach failed: attach failed (Not allowed to attach to process.
Look in the console messages (Console.app) for possible reason.)
```

### Kernel Extension Restrictions

```bash
# View loaded kernel extensions
$ kextstat | head -5
Index Refs Address            Size       Wired      Name (Version)
    1  148 0                  0          0          com.apple.kpi.bsd (23.2.0)
    2   18 0                  0          0          com.apple.kpi.dsep (23.2.0)
    3  181 0                  0          0          com.apple.kpi.iokit (23.2.0)
    4    0 0                  0          0          com.apple.kpi.kasan (23.2.0)

# Loading unsigned kexts is blocked by SIP
$ sudo kextload /path/to/unsigned.kext
/path/to/unsigned.kext failed to load - (libkern/kext) not loadable...
```

### The Restricted Flag

Protected files carry a special `restricted` flag:

```bash
# View restricted flag
$ ls -lO /bin/ls
-rwxr-xr-x  1 root  wheel  restricted,compressed 134800 Jan  1 00:00 /bin/ls

$ ls -lO /System
drwxr-xr-x  6 root  wheel  restricted 192 Jan  1 00:00 /System

# Your files don't have this flag
$ ls -lO ~/Desktop
drwx------+ 5 david  staff  - 160 Jan 15 10:00 /Users/david/Desktop
```

## Checking SIP Status

```bash
# Check if SIP is enabled
$ csrutil status
System Integrity Protection status: enabled.

# On systems with SIP disabled
$ csrutil status
System Integrity Protection status: disabled.

# Check authenticated-root status (macOS 11+)
$ csrutil authenticated-root status
Authenticated Root status: enabled
```

## What SIP Prevents You From Doing

### System Modifications

```bash
# Cannot modify system binaries
$ sudo vim /usr/bin/python3
E: Unable to write file

# Cannot create files in protected directories
$ sudo mkdir /System/mydir
mkdir: /System/mydir: Operation not permitted

# Cannot modify boot configuration
$ sudo nvram boot-args="some value"
nvram: Error setting variable - 'boot-args': (iokit/common) not permitted
```

### Code Injection

```bash
# Cannot inject into protected processes
$ sudo DYLD_INSERT_LIBRARIES=/path/to/lib.dylib /System/Applications/Finder.app/Contents/MacOS/Finder
# DYLD_INSERT_LIBRARIES is ignored for protected binaries
```

### Kernel Extension Loading

```bash
# Cannot load unsigned kexts
$ sudo kextload unsigned.kext
# Blocked by SIP

# Cannot modify kext cache
$ sudo touch /System/Library/Extensions
touch: /System/Library/Extensions: Operation not permitted
```

## When You Might Need to Disable SIP

Disabling SIP should be rare and temporary. Valid reasons include:

1. **Installing specific kernel extensions** (legacy drivers)
2. **Security research** (analyzing malware, reverse engineering)
3. **Modifying system behavior** for development
4. **Recovering from system issues** where SIP is in the way

**Invalid reasons** (there's usually a better way):
- Installing software that wants to modify `/usr/bin`
- General "I want full control" sentiment
- A Stack Overflow answer told you to

## How to Disable SIP

SIP can only be disabled from Recovery Mode:

### Intel Macs

1. Restart your Mac
2. Hold **Command + R** during startup to boot into Recovery Mode
3. From the menu bar, select **Utilities > Terminal**
4. Disable SIP:

```bash
# Disable all SIP protections
$ csrutil disable
Successfully disabled System Integrity Protection. Please restart the machine for the changes to take effect.

# Or disable specific protections only
$ csrutil enable --without kext      # Allow unsigned kexts
$ csrutil enable --without fs        # Allow filesystem modifications
$ csrutil enable --without debug     # Allow debugging system processes
$ csrutil enable --without nvram     # Allow NVRAM modifications
```

5. Restart: `reboot`

### Apple Silicon Macs

1. Shut down your Mac completely
2. Press and hold the **Power button** until "Loading startup options" appears
3. Click **Options**, then **Continue**
4. If prompted, select a user and enter their password
5. From the menu bar, select **Utilities > Terminal**
6. Disable SIP:

```bash
$ csrutil disable
```

**Note:** Apple Silicon Macs have additional security considerations. You may also need to:

```bash
# Allow kernel extensions from identified developers
$ spctl kext-consent add <TEAM-ID>

# Change security policy (Startup Security Utility)
# Options: Full Security, Reduced Security, Permissive Security
```

7. Restart

### Re-enabling SIP

Always re-enable SIP when you're done:

1. Boot into Recovery Mode (same method as above)
2. Open Terminal
3. Enable SIP:

```bash
$ csrutil enable
Successfully enabled System Integrity Protection. Please restart the machine for the changes to take effect.
```

4. Restart

## Partial SIP Configuration

You can selectively disable SIP features:

```bash
# From Recovery Mode Terminal:

# See all options
$ csrutil enable --help

# Common partial configurations:
$ csrutil enable --without kext        # Kernel extensions only
$ csrutil enable --without fs          # Filesystem protection only
$ csrutil enable --without debug       # Process debugging only
$ csrutil enable --without nvram       # NVRAM protection only
$ csrutil enable --without dtrace      # DTrace restrictions only

# Multiple options
$ csrutil enable --without kext --without debug

# Check current configuration
$ csrutil status
System Integrity Protection status: enabled (Custom Configuration).

Configuration:
    Apple Internal: disabled
    Kext Signing: disabled
    Filesystem Protections: enabled
    Debugging Restrictions: enabled
    DTrace Restrictions: enabled
    NVRAM Protections: enabled
    BaseSystem Verification: enabled
```

## SIP and Development

### DTrace Limitations

With SIP enabled, DTrace cannot instrument protected processes:

```bash
# This works (your own process)
$ sudo dtrace -n 'syscall:::entry { @[execname] = count(); }'

# This won't show system process details
# Many probes are restricted
```

### Debugging Limitations

```bash
# Cannot debug protected processes
$ sudo lldb -n Finder
error: attach failed: attach failed (Not allowed to attach to process.)

# Can debug your own processes
$ lldb ./myapp
(lldb) target create "./myapp"
Current executable set to './myapp' (arm64).
```

### Building Software

Most development works fine with SIP enabled:

```bash
# /usr/local is not protected - Homebrew works fine
$ brew install wget

# /Applications is not protected
$ cp -r MyApp.app /Applications/

# Your home directory is not protected
$ make install PREFIX=$HOME/local
```

## SIP Bypass Attempts (Don't Do This)

Historical SIP bypasses have been patched:
- Exploiting installer packages (fixed)
- Using `csrutil` outside Recovery Mode (never worked properly)
- Kernel vulnerability exploitation (patched)

Attempting to bypass SIP:
- May indicate malware
- Voids any support from Apple
- Can brick your system
- Will be patched in future updates

## Authenticated Root (macOS 11+)

macOS Big Sur introduced Signed System Volume (SSV) and Authenticated Root:

```bash
# Check authenticated root status
$ csrutil authenticated-root status
Authenticated Root status: enabled

# The system volume is cryptographically sealed
$ diskutil list
...
/dev/disk3 (synthesized):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      APFS Container Scheme -                      +994.7 GB   disk3
   1:                APFS Volume Macintosh HD            14.9 GB    disk3s1
   2:              APFS Snapshot com.apple.os.update-... 14.9 GB    disk3s1s1
...
```

The system runs from a cryptographically verified snapshot. Even with SIP disabled, system modifications won't persist across reboots without additional steps:

```bash
# To make system modifications persist (macOS 11+):
# 1. Disable SIP and authenticated-root
$ csrutil disable
$ csrutil authenticated-root disable

# 2. Make your modifications
# 3. Create a new snapshot
$ sudo bless --folder /Volumes/Macintosh\ HD/System/Library/CoreServices --bootefi --create-snapshot

# 4. Re-enable protections when done
```

## Troubleshooting SIP Issues

### Operation Not Permitted

```bash
$ sudo some-command
Operation not permitted

# Check if it's SIP-related
$ ls -lO /path/to/file
# Look for 'restricted' flag

# Or check csrutil
$ csrutil status
```

### Third-Party Software Issues

Some software legitimately needs SIP disabled:
- Legacy virtualization software
- Certain security/analysis tools
- Some kernel extension drivers

Check if the vendor has:
1. A newer version that works with SIP
2. A System Extension (modern kext replacement)
3. Official guidance on SIP requirements

### Kext Loading Failures

```bash
# Check why kext won't load
$ sudo kextutil -v /path/to/kext.kext
Kext rejected due to system policy

# Check kext consent
$ sudo spctl kext-consent status
Kernel Extension User Consent: ENABLED

$ sudo spctl kext-consent list
ABCDE12345  # Allowed team IDs
```

Modern macOS prefers System Extensions over kexts:

```bash
# View system extensions
$ systemextensionsctl list
1 extension(s)
--- com.apple.system_extension.network_extension
enabled	active	teamID	bundleID (version)	name	[state]
*	*	ABC123	com.example.ext (1.0/1)	MyExtension	[activated enabled]
```

## Best Practices

1. **Keep SIP enabled** for daily use
2. **Use /usr/local** for custom software
3. **Use Homebrew** instead of modifying system paths
4. **If you must disable SIP:**
   - Do it for the minimum time necessary
   - Disable only the specific protection needed
   - Re-enable immediately after
   - Document what you did and why
5. **Never disable SIP** on production machines
6. **Consider virtualization** for testing that requires SIP disabled

## Summary

SIP is a fundamental security feature of modern macOS:

| Aspect | Details |
|--------|---------|
| Purpose | Protect system integrity from malware and mistakes |
| Protected | /System, /usr, /bin, /sbin, kernel, system processes |
| Unprotected | /usr/local, /Applications, /Library, ~/  |
| Check status | `csrutil status` |
| Disable | Recovery Mode only |
| Best practice | Keep enabled, use /usr/local |

Key commands:

```bash
# Check status
$ csrutil status

# View restricted flag
$ ls -lO /System

# From Recovery Mode only:
$ csrutil disable
$ csrutil enable
$ csrutil enable --without kext
```

SIP may occasionally frustrate you, but it's a crucial defense against both malware and accidental system damage. Work with it, not against it.
