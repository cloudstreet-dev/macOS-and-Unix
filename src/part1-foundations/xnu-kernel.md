# The XNU Kernel Architecture

XNU—"X is Not Unix" (a recursive acronym in the tradition of GNU)—is the kernel at the heart of Darwin. XNU is neither a traditional monolithic kernel like Linux nor a pure microkernel like Mach was intended to be. It's a hybrid that combines elements of both approaches.

## What Is a Kernel?

Before diving into XNU's specifics, let's clarify what a kernel does:

- **Process management**: Creating, scheduling, and terminating processes
- **Memory management**: Allocating RAM, implementing virtual memory
- **Device drivers**: Interfacing with hardware
- **System calls**: Providing services to user-space programs
- **IPC**: Facilitating communication between processes

How these responsibilities are organized defines a kernel's architecture.

## Kernel Architecture Styles

### Monolithic Kernels

Linux and traditional BSD use monolithic kernels:

- All kernel services run in kernel space
- Efficient: no context switches between kernel components
- Risk: a bug in any driver can crash the entire system

### Microkernels

Pure microkernels (like MINIX or L4) take a different approach:

- Minimal kernel: only scheduling, IPC, basic memory management
- Services (filesystems, drivers) run as user-space processes
- Robust: service crashes don't bring down the kernel
- Overhead: IPC between services adds latency

### XNU: The Hybrid Approach

XNU combines both:

- **Mach layer**: Microkernel-derived component handling IPC, virtual memory, scheduling
- **BSD layer**: Monolithic component providing Unix APIs, networking, filesystems
- **IOKit**: Object-oriented driver framework in kernel space

```
┌─────────────────────────────────────────────────────────┐
│                    User Space                            │
│  Applications, Daemons, Shell commands                   │
├─────────────────────────────────────────────────────────┤
│                    System Calls                          │
├───────────────┬────────────────────────┬────────────────┤
│   BSD Layer   │      IOKit/Drivers     │                │
│  (POSIX APIs, │    (Device drivers,    │                │
│   networking, │   hardware abstraction)│                │
│  filesystems) │                        │                │
├───────────────┴────────────────────────┴────────────────┤
│                    Mach Layer                            │
│  (IPC, virtual memory, scheduling, timers)               │
├─────────────────────────────────────────────────────────┤
│                    Hardware                              │
└─────────────────────────────────────────────────────────┘
```

## The Mach Layer

XNU's Mach component derives from Mach 3.0, developed at Carnegie Mellon University. However, XNU doesn't use Mach as a true microkernel—the BSD layer runs in kernel space, not as a user-space server.

### Mach Abstractions

Mach provides several fundamental abstractions:

**Tasks**: The unit of resource ownership. A task contains threads and owns resources like memory and ports. In Unix terms, a task roughly corresponds to a process.

```bash
# View Mach task information
$ sudo launchctl procinfo 1
{
    "task_info" : {
        "suspend_count" : 0,
        "virtual_size" : 418099200,
        "resident_size" : 5423104,
        ...
    }
}
```

**Threads**: The unit of execution. Tasks contain one or more threads.

**Ports**: The fundamental IPC mechanism. Ports are endpoints for message passing.

```bash
# List Mach ports for a process
$ sudo lsmp -p $$
```

**Messages**: Data passed between ports. All Mach IPC occurs through message passing.

**Memory Objects**: Abstractions for memory that can be mapped into task address spaces.

### Mach IPC

Mach's message-passing IPC is powerful but complex:

```c
// Simplified Mach message send
mach_msg_header_t msg;
msg.msgh_bits = MACH_MSGH_BITS(MACH_MSG_TYPE_COPY_SEND, 0);
msg.msgh_size = sizeof(msg);
msg.msgh_remote_port = destination_port;
msg.msgh_local_port = MACH_PORT_NULL;
mach_msg(&msg, MACH_SEND_MSG, sizeof(msg), 0,
         MACH_PORT_NULL, MACH_MSG_TIMEOUT_NONE, MACH_PORT_NULL);
```

While application developers rarely use Mach IPC directly (preferring higher-level APIs like XPC), it underpins many macOS facilities.

### Virtual Memory

Mach's virtual memory system provides:

- **Sparse address spaces**: Processes can have large virtual address spaces
- **Copy-on-write**: Efficient process forking and memory sharing
- **Memory-mapped files**: Files accessible as memory regions
- **External pagers**: Pluggable systems for backing memory with storage

```bash
# Examine Mach VM statistics
$ vm_stat
Mach Virtual Memory Statistics: (page size of 16384 bytes)
Pages free:                               32148.
Pages active:                            402844.
Pages inactive:                          373521.
Pages speculative:                        11276.
Pages throttled:                              0.
Pages wired down:                         96482.
Pages purgeable:                          23159.
"Translation faults":                 135941690.
Pages copy-on-write:                    4972318.
Pages zero filled:                     62938502.
Pages reactivated:                      4058508.
Pages purged:                            789235.
...
```

### Scheduling

Mach handles thread scheduling with multiple scheduling policies:

- **Timeshare**: Standard interactive scheduling
- **Fixed priority**: Real-time scheduling with fixed priorities
- **Round-robin**: Equal time slices

```bash
# View thread scheduling information
$ sudo spindump -noProcesses 1 -stdout | head -50
```

## The BSD Layer

The BSD layer provides the Unix personality that applications see:

### POSIX APIs

Standard Unix system calls are implemented in the BSD layer:

- `fork()`, `exec()`, `wait()`
- `open()`, `read()`, `write()`, `close()`
- `socket()`, `bind()`, `listen()`, `accept()`
- `kill()`, signal handling

### Process Model

The BSD layer maintains the traditional Unix process model atop Mach tasks:

```bash
# BSD process information
$ ps aux | head -5
USER   PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
root     1   0.0  0.1 410045680  18512   ??  Ss   Tue10AM   3:21.67 /sbin/launchd

# Underlying Mach task information
$ sudo taskinfo 1
```

Each BSD process corresponds to a Mach task, but the BSD layer provides the familiar Unix semantics.

### Networking Stack

The BSD layer includes a complete TCP/IP networking stack derived from FreeBSD:

```bash
# View network statistics (BSD-style)
$ netstat -s | head -30
tcp:
    14548388 packets sent
        9522841 data packets (5765485687 bytes)
        96487 data packets (87239347 bytes) retransmitted
        0 resend initiated by MTU discovery
        3896370 ack-only packets (2451961 delayed)
...
```

### Filesystems

Filesystem implementations live in the BSD layer:

- **APFS**: Apple File System (modern default)
- **HFS+**: Hierarchical File System Plus (legacy)
- **NFS**: Network File System
- **SMB**: Server Message Block
- **devfs**: Device filesystem
- **Various FUSE filesystems**: Via macFUSE

### The VFS Layer

The Virtual File System layer abstracts filesystem differences:

```bash
# List mounted filesystems
$ mount
/dev/disk3s1s1 on / (apfs, sealed, local, read-only, journaled)
devfs on /dev (devfs, local, nobrowse)
/dev/disk3s6 on /System/Volumes/VM (apfs, local, noexec, journaled, noatime, nobrowse)
/dev/disk3s2 on /System/Volumes/Preboot (apfs, local, journaled, nobrowse)
/dev/disk3s4 on /System/Volumes/Update (apfs, local, journaled, nobrowse)
/dev/disk3s5 on /System/Volumes/Data (apfs, local, journaled, nobrowse, protect)
```

## IOKit: The Driver Framework

IOKit is XNU's object-oriented framework for device drivers:

### Design Philosophy

- Written in a restricted subset of C++ (Embedded C++)
- Object-oriented: drivers inherit from base classes
- Uses a registry to match drivers with devices
- Power management integrated at the framework level

### The IORegistry

IOKit maintains a database of devices and their relationships:

```bash
# View the IORegistry
$ ioreg -l | head -50
+-o Root  <class IORegistryEntry, id 0x100000100, retain 36>
  +-o MacBookPro18,3  <class IOPlatformExpertDevice, id 0x100000116, registered, matched, active, busy 0 (10112 ms), retain 61>
    | {
    |   "IOBusyInterest" = "IOCommand is not serializable"
    |   "IOPlatformSerialNumber" = "..."
    |   "IOPolledInterface" = "SMCPolledInterface is not serializable"
    |   "IOPlatformUUID" = "..."
...

# List USB devices through IOKit
$ ioreg -p IOUSB -l -w0

# List storage devices
$ ioreg -c IOMedia -l
```

### Kernel Extensions (Kexts)

Traditionally, third-party drivers were loaded as kernel extensions:

```bash
# List loaded kexts
$ kextstat | head -10
Index Refs Address            Size       Wired      Name (Version) UUID <Linked Against>
    1  169 0                  0          0          com.apple.kpi.bsd (23.4.0)
    2   18 0                  0          0          com.apple.kpi.dsep (23.4.0)
    3  196 0                  0          0          com.apple.kpi.iokit (23.4.0)
...
```

However, modern macOS strongly discourages kexts in favor of:

- **System Extensions**: User-space drivers for many device types
- **DriverKit**: A framework for building drivers that run outside the kernel

This shift improves security and stability by moving driver code out of kernel space.

## System Calls in XNU

XNU provides system calls through multiple interfaces:

### Mach Traps

Low-level Mach operations:

```c
// Example Mach trap numbers
#define MACH_MSG_TRAP         -31
#define MACH_REPLY_PORT       -26
#define THREAD_SELF_TRAP      -27
#define TASK_SELF_TRAP        -28
```

### BSD System Calls

Standard Unix system calls:

```bash
# View system call numbers
$ grep -E "^#define\s+SYS_" /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/syscall.h | head -20
#define SYS_syscall        0
#define SYS_exit           1
#define SYS_fork           2
#define SYS_read           3
#define SYS_write          4
#define SYS_open           5
#define SYS_close          6
#define SYS_wait4          7
...
```

### Machine-Dependent Calls

Architecture-specific operations.

## Kernel Debugging and Introspection

Several tools expose XNU internals:

### sysctl

The `sysctl` interface queries and sets kernel parameters:

```bash
# View kernel information
$ sysctl kern.ostype kern.osrelease kern.hostname
kern.ostype: Darwin
kern.osrelease: 23.4.0
kern.hostname: MacBook-Pro.local

# List all kernel parameters
$ sysctl -a | wc -l
    1847

# View memory parameters
$ sysctl vm
vm.loadavg: { 1.83 2.07 2.15 }
vm.swapusage: total = 0.00M  used = 0.00M  free = 0.00M  (encrypted)
vm.cs_validation: 1
...
```

### DTrace (Where Available)

On systems where it's available, DTrace provides deep kernel tracing:

```bash
# Trace system calls (requires SIP adjustment)
$ sudo dtrace -n 'syscall:::entry { @[execname] = count(); }'
```

Note: System Integrity Protection restricts DTrace on modern macOS.

### Instruments

Apple's Instruments application provides profiling using kernel trace facilities:

```bash
# Command-line profiling
$ xctrace record --template 'Time Profiler' --launch -- /path/to/app
```

## XNU Source Code Structure

For those interested in exploring the code:

```
xnu/
├── bsd/           # BSD layer (processes, filesystems, networking)
├── iokit/         # IOKit driver framework
├── libkern/       # Kernel C++ runtime
├── libsa/         # Standalone library
├── osfmk/         # Mach layer (scheduling, IPC, VM)
├── pexpert/       # Platform expert (hardware abstraction)
└── security/      # Security policies (MAC framework)
```

The source is available at opensource.apple.com, though building it requires significant effort.

## Why XNU's Architecture Matters

Understanding XNU helps explain several macOS behaviors:

1. **Why Mach-O binaries?** XNU uses Mach-O format, not ELF like Linux
2. **Why different process tools?** BSD and Mach layers provide overlapping but different views
3. **Why complex memory semantics?** Mach's VM model has unique characteristics
4. **Why DriverKit?** Moving away from kernel extensions reflects security priorities

The hybrid nature of XNU—combining Mach's IPC and memory management with BSD's Unix compatibility—creates a unique system that's familiar yet different from both Linux and traditional BSD.
