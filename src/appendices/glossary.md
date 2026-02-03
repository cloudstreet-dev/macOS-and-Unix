# Appendix D: Glossary

This glossary defines key terms related to macOS, Unix, and the command line.

---

## A

**ACL (Access Control List)**
: An extended permission system beyond traditional Unix permissions (user/group/other). ACLs allow fine-grained control over file access for multiple users and groups. On macOS, view with `ls -le` and modify with `chmod +a`.

**APFS (Apple File System)**
: Apple's modern filesystem introduced in 2017, replacing HFS+. Features include native encryption, snapshots, space sharing between volumes, copy-on-write, and crash protection. Optimized for flash/SSD storage.

**Apple Silicon**
: Apple's custom ARM-based processors for Mac (M1, M2, M3, M4 series). These chips use a different architecture (arm64) than Intel Macs (x86_64), affecting binary compatibility and some system paths.

**Application Bundle**
: A directory structure with a `.app` extension that contains a macOS application. The bundle includes the executable, resources, frameworks, and metadata in a standardized layout. Appears as a single file in Finder.

**ARD (Apple Remote Desktop)**
: Apple's remote administration tool for managing Mac computers. Uses the VNC protocol for screen sharing with additional management features.

---

## B

**Bash (Bourne Again Shell)**
: A Unix shell and command language, formerly the default shell on macOS (through Catalina). Replaced by zsh as the default in macOS Catalina (10.15).

**Bonjour**
: Apple's implementation of zero-configuration networking (mDNS/DNS-SD). Allows automatic discovery of devices and services on a local network without manual configuration. The command `dns-sd` provides CLI access.

**BSD (Berkeley Software Distribution)**
: A Unix operating system derivative developed at UC Berkeley. macOS's userland (command-line tools) derives from FreeBSD. Key BSD characteristics include the BSD license, specific command-line tool behaviors, and system call conventions.

**Bundle Identifier**
: A reverse-DNS format string uniquely identifying a macOS application (e.g., `com.apple.Safari`). Used in preferences, entitlements, code signing, and system services.

---

## C

**Catalina**
: macOS 10.15 (2019), notable for switching the default shell to zsh, introducing a read-only system volume, deprecating 32-bit app support, and introducing stricter security controls.

**Clang**
: The C/C++/Objective-C compiler used on macOS, part of the LLVM project. Replaced GCC as the default compiler. Invoked via `clang` or through `gcc` (which is actually Clang on macOS).

**Code Signing**
: The process of digitally signing executables and apps to verify their identity and integrity. Required for apps distributed through the App Store and increasingly enforced for other software by Gatekeeper.

**Coreutils**
: GNU's implementation of basic Unix utilities (ls, cp, cat, etc.). Can be installed via Homebrew to get GNU-style behavior, which differs from macOS's BSD-based versions.

**Copy-on-Write (COW)**
: A resource management technique where copies share the same data until one is modified. APFS uses COW extensively, making file copies and snapshots space-efficient.

---

## D

**Daemon**
: A background process that runs without user interaction, typically started at boot time. On macOS, daemons are managed by launchd and configured in `/Library/LaunchDaemons/`.

**Darwin**
: The open-source Unix foundation of macOS, iOS, and other Apple operating systems. Darwin includes the XNU kernel, BSD userland components, and various frameworks. Available at opensource.apple.com.

**DHCP (Dynamic Host Configuration Protocol)**
: A network protocol for automatically assigning IP addresses and network configuration to devices.

**Disk Image (.dmg)**
: A file format that contains the contents of a disk or volume. Used for software distribution on macOS. Mounted with `hdiutil attach`.

**Disk Utility**
: macOS's GUI tool for managing disks, volumes, and disk images. Command-line equivalent is `diskutil`.

**DNS (Domain Name System)**
: The system that translates human-readable domain names to IP addresses. Configure on macOS with `networksetup -setdnsservers`.

**DriverKit**
: Apple's modern framework for building device drivers that run in user space rather than the kernel, improving system security and stability. Introduced in macOS Catalina.

---

## E

**Entitlement**
: A key-value pair embedded in a code signature that grants specific capabilities to an application (e.g., network access, file access, hardware access). Central to macOS's security model.

**Environment Variable**
: A named value available to processes, used to configure behavior. Common examples: `PATH`, `HOME`, `SHELL`. Set with `export VAR=value`.

**Extended Attributes**
: Metadata attached to files beyond standard permissions and timestamps. On macOS, used for Finder info, quarantine flags, and resource forks. View with `xattr`, list with `ls -l@`.

---

## F

**FileVault**
: macOS's full-disk encryption feature using XTS-AES-128 encryption. Encrypts the entire startup volume. Check status with `fdesetup status`.

**Finder**
: macOS's default file manager and graphical shell. Access the current directory in Finder with `open .`.

**Firmware**
: Low-level software stored in hardware that initializes the system before the OS loads. On Apple Silicon Macs, includes iBoot. On Intel Macs, uses EFI.

**Fork**
: In Unix, creating a child process by duplicating the parent. Also refers to resource forks, a legacy macOS method of storing structured data alongside file contents.

**Framework**
: A bundle containing a dynamic shared library along with its headers, resources, and documentation. The macOS equivalent of Linux's shared libraries with packaging. Located in `/System/Library/Frameworks/` and `/Library/Frameworks/`.

**FUSE (Filesystem in Userspace)**
: A mechanism for implementing filesystems in user space. macFUSE allows mounting alternative filesystems (NTFS, sshfs, etc.) on macOS.

---

## G

**Gatekeeper**
: macOS's security feature that verifies apps come from identified developers and haven't been tampered with. Uses code signing and notarization. Check status with `spctl --status`.

**GCD (Grand Central Dispatch)**
: Apple's technology for managing concurrent operations through work queues. Available to developers for efficient multi-threading.

**GNU**
: "GNU's Not Unix" - a project to create a free Unix-like operating system. GNU tools (grep, sed, awk) often have different options than BSD equivalents on macOS.

---

## H

**HFS+ (Hierarchical File System Plus)**
: The legacy filesystem used on macOS before APFS. Still supported for compatibility. Also known as Mac OS Extended.

**Homebrew**
: The most popular package manager for macOS. Installs software to `/opt/homebrew/` (Apple Silicon) or `/usr/local/` (Intel). Run `brew install <package>`.

---

## I

**Installer Package (.pkg)**
: A file format for distributing software installers on macOS. Contains installation scripts, payload, and configuration. Install with `installer -pkg`.

**IOKit**
: The object-oriented framework for device drivers in macOS. Drivers interact with hardware through IOKit APIs. View the hardware registry with `ioreg`.

**IPC (Inter-Process Communication)**
: Mechanisms for processes to communicate. On macOS, includes Mach ports, Unix signals, pipes, XPC, and distributed notifications.

---

## K

**Kernel**
: The core of an operating system that manages hardware resources and provides services to applications. macOS uses the XNU kernel.

**Kernel Extension (kext)**
: A loadable kernel module on macOS. Being deprecated in favor of System Extensions and DriverKit for security reasons. List with `kextstat`.

**Keychain**
: macOS's secure storage for passwords, certificates, and encryption keys. Access from CLI with `security` command.

---

## L

**Launch Agent**
: A background process managed by launchd that runs in the user session context. Configured in `~/Library/LaunchAgents/` or `/Library/LaunchAgents/`.

**Launch Daemon**
: A background process managed by launchd that runs at system level (as root or specified user). Configured in `/Library/LaunchDaemons/`.

**launchctl**
: The command-line interface for interacting with launchd. Use to load, unload, and manage services.

**launchd**
: macOS's init system and service manager (PID 1). Manages system and user services, replacing traditional Unix init, cron, and other systems.

**LLDB**
: The debugger used on macOS, part of the LLVM project. Replaced GDB. Invoked with `lldb`.

**LLVM**
: A compiler infrastructure project that includes Clang (C/C++ compiler), LLDB (debugger), and related tools. The foundation of Apple's development toolchain.

---

## M

**Mach**
: The microkernel that forms the lower layer of XNU. Provides fundamental services like IPC, memory management, and scheduling. Originally developed at Carnegie Mellon University.

**Mach-O**
: The executable file format used on macOS and iOS. Different from Linux's ELF format. View information with `otool` or `file`.

**MacPorts**
: An alternative package manager for macOS (besides Homebrew). Uses `/opt/local/` prefix.

**man (manual)**
: The Unix documentation system. View documentation with `man <command>`.

**mDNS (Multicast DNS)**
: A protocol for resolving hostnames to IP addresses on small networks without a central DNS server. Part of Bonjour.

---

## N

**NeXTSTEP**
: The operating system developed by NeXT (Steve Jobs's company after leaving Apple). Became the foundation for macOS when Apple acquired NeXT in 1997.

**Notarization**
: Apple's process of scanning software for malicious content and issues before distribution. Required for software distributed outside the App Store to run without Gatekeeper warnings.

**NVRAM (Non-Volatile RAM)**
: Memory that retains data when powered off. Stores boot-time settings. Access with `nvram` command.

---

## O

**Objective-C**
: A programming language that adds object-oriented features to C. Historically the primary language for macOS and iOS development, now largely supplanted by Swift.

---

## P

**PATH**
: An environment variable containing a colon-separated list of directories where the shell looks for executable commands.

**PID (Process ID)**
: A unique number identifying a running process. View with `ps` or Activity Monitor.

**Pipe**
: A mechanism for connecting the output of one command to the input of another. Created with `|` (e.g., `ls | grep foo`).

**plist (Property List)**
: An XML or binary format for storing structured data on macOS. Used for preferences, launch agent/daemon configuration, and app settings. Edit with `defaults` or `plutil`.

**POSIX**
: Portable Operating System Interface - a family of standards for Unix-like operating systems. macOS is POSIX-compliant, ensuring compatibility with standard Unix tools and APIs.

---

## Q

**Quarantine**
: macOS's mechanism for marking downloaded files. Triggers Gatekeeper checks when the file is first opened. View with `xattr -l` (look for `com.apple.quarantine`).

---

## R

**Recovery Mode**
: A boot environment for troubleshooting and system maintenance. Access by holding power button (Apple Silicon) or Cmd+R (Intel) during startup.

**Resource Fork**
: A legacy macOS feature storing structured data alongside a file's data fork. Largely replaced by extended attributes but still exists for compatibility.

**Root User**
: The superuser account with full system access. On macOS, disabled by default. Use `sudo` to execute commands with root privileges.

**Rosetta 2**
: Apple's translation layer that allows Intel (x86_64) applications to run on Apple Silicon (arm64) Macs.

---

## S

**Sandbox**
: A security mechanism that restricts an application's access to system resources. App Store apps must be sandboxed. Container data stored in `~/Library/Containers/`.

**SDK (Software Development Kit)**
: Tools, libraries, and documentation for developing software. macOS SDKs are included with Xcode or Command Line Tools.

**Shell**
: A command-line interpreter that provides an interface to the operating system. macOS's default is zsh; bash, tcsh, and others are also available.

**Signal**
: A notification sent to a process. Common signals: SIGTERM (terminate), SIGKILL (force kill), SIGINT (interrupt from Ctrl+C), SIGSTOP (suspend).

**SIP (System Integrity Protection)**
: macOS security feature that restricts root access to protected system files and processes. Check with `csrutil status`. Can only be modified from Recovery Mode.

**Snapshot**
: A point-in-time copy of a filesystem state. APFS supports efficient snapshots used by Time Machine and system updates.

**Spotlight**
: macOS's search technology that indexes file contents and metadata. Access from CLI with `mdfind`.

**stderr (Standard Error)**
: The output stream for error messages, file descriptor 2. Redirect with `2>`.

**stdin (Standard Input)**
: The input stream, file descriptor 0. Redirect with `<`.

**stdout (Standard Output)**
: The primary output stream, file descriptor 1. Redirect with `>` or `>>`.

**sudo**
: "Superuser do" - execute a command with elevated privileges. Configuration in `/etc/sudoers`.

**Swift**
: Apple's modern programming language for macOS, iOS, and other Apple platforms. Increasingly used for system utilities.

**Symlink (Symbolic Link)**
: A file that points to another file or directory. Create with `ln -s target linkname`.

**sysctl**
: Command and interface for viewing and modifying kernel parameters. View with `sysctl -a`.

**System Extension**
: Modern replacement for kernel extensions that runs in user space. Used for network extensions, endpoint security, and driver extensions.

---

## T

**TCC (Transparency, Consent, and Control)**
: macOS's privacy protection framework that requires explicit user consent for apps to access sensitive data (contacts, calendar, microphone, screen recording, etc.).

**Terminal**
: An application that provides a text-based interface to the shell. macOS includes Terminal.app; iTerm2 is a popular alternative.

**Time Machine**
: macOS's backup system using APFS snapshots and incremental backups. CLI tool: `tmutil`.

**TTY**
: Historically "teletype," now refers to terminal devices. View current TTY with `tty`.

---

## U

**UID (User ID)**
: A numeric identifier for a user account. Root is UID 0. View with `id`.

**Unified Logging**
: macOS's centralized logging system replacing traditional syslog. Access with `log` command or Console.app.

**Universal Binary**
: An executable containing code for multiple CPU architectures (e.g., arm64 and x86_64). View architectures with `lipo -info` or `file`.

**Unix**
: A family of operating systems originating from AT&T Bell Labs in the 1970s. macOS is a certified Unix operating system.

---

## V

**VFS (Virtual File System)**
: An abstraction layer that provides a common interface for different filesystem implementations.

**Volume**
: A logical storage unit that can be mounted and accessed. On APFS, multiple volumes can share a container.

---

## W

**Watchdog**
: A system that monitors processes and restarts them if they crash. launchd provides watchdog functionality via `KeepAlive` in plist configurations.

---

## X

**x86_64**
: The 64-bit Intel/AMD processor architecture. Used in Intel Macs (2006-2020).

**Xattr**
: Command for viewing and manipulating extended attributes. Common use: `xattr -l file` to list attributes.

**Xcode**
: Apple's integrated development environment (IDE) for macOS, iOS, and other Apple platforms. Includes compilers, debuggers, and Interface Builder.

**Xcode Command Line Tools**
: A standalone package of development tools (compilers, headers, utilities) without the full Xcode IDE. Install with `xcode-select --install`.

**XNU**
: "X is Not Unix" - the hybrid kernel at the core of macOS. Combines Mach microkernel, BSD components, and IOKit.

**XPC (Cross-Process Communication)**
: Apple's modern IPC mechanism for communication between processes, particularly for sandboxed apps and system services.

**XProtect**
: macOS's built-in malware detection and prevention system. Automatically updated by Apple.

---

## Z

**zsh (Z Shell)**
: The default shell on macOS since Catalina. Compatible with bash but adds features like improved tab completion, spelling correction, and plugin support.

---

## Version Names Quick Reference

| Version | Name | Release Year | Notable Changes |
|---------|------|--------------|-----------------|
| 10.0 | Cheetah | 2001 | First Mac OS X release |
| 10.4 | Tiger | 2005 | Spotlight, Dashboard |
| 10.5 | Leopard | 2007 | Time Machine, Spaces |
| 10.6 | Snow Leopard | 2009 | Grand Central Dispatch |
| 10.7 | Lion | 2011 | Full-screen apps, Launchpad |
| 10.9 | Mavericks | 2013 | Free upgrades begin |
| 10.11 | El Capitan | 2015 | System Integrity Protection |
| 10.12 | Sierra | 2016 | Siri, APFS introduced |
| 10.13 | High Sierra | 2017 | APFS default for SSD |
| 10.14 | Mojave | 2018 | Dark mode, privacy controls |
| 10.15 | Catalina | 2019 | zsh default, read-only system |
| 11 | Big Sur | 2020 | Apple Silicon support |
| 12 | Monterey | 2021 | Shortcuts, Focus modes |
| 13 | Ventura | 2022 | Stage Manager |
| 14 | Sonoma | 2023 | Desktop widgets |
| 15 | Sequoia | 2024 | iPhone mirroring |

---

## Filesystem Path Quick Reference

| Path | Contents |
|------|----------|
| `/` | Root directory |
| `/Applications/` | GUI applications |
| `/Library/` | System-wide resources |
| `/System/` | macOS system files (protected) |
| `/Users/` | User home directories |
| `/Volumes/` | Mounted volumes |
| `/bin/`, `/sbin/` | Essential commands |
| `/usr/bin/`, `/usr/sbin/` | User commands |
| `/usr/local/` | Locally installed (Intel Homebrew) |
| `/opt/homebrew/` | Homebrew (Apple Silicon) |
| `/etc/` | Configuration (symlink to /private/etc) |
| `/var/` | Variable data (symlink to /private/var) |
| `/tmp/` | Temporary files (symlink to /private/tmp) |
| `~/Library/` | User resources and preferences |
