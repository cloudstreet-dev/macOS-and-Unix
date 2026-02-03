# Virtual Machines on macOS

While containers provide lightweight isolation for running Linux applications, virtual machines offer complete operating system virtualization. This is essential when you need to run a full Linux desktop environment, test installers, run Windows, or develop for architectures that differ from your host. macOS supports virtualization through multiple solutions, from Apple's native Virtualization.framework to established products like Parallels and VMware.

## Virtualization Technologies on macOS

### Apple Virtualization.framework

Apple's native hypervisor, introduced in macOS Big Sur (11.0) and significantly enhanced in Monterey (12.0) and later:

```
┌─────────────────────────────────────────────────┐
│                  macOS Host                      │
│  ┌───────────────────────────────────────────┐  │
│  │            Guest VM (Linux/macOS)          │  │
│  │  ┌─────────────────────────────────────┐  │  │
│  │  │         Virtualized Hardware         │  │  │
│  │  │   CPU, Memory, Disk, Network, GPU    │  │  │
│  │  └─────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │       Virtualization.framework            │  │
│  │  (Native Apple hypervisor technology)     │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │              XNU Kernel                   │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

**Features:**
- Near-native performance on Apple Silicon
- Low latency, efficient memory sharing
- Rosetta 2 support for running x86 Linux on ARM
- VirtioFS for fast file sharing
- GPU acceleration (limited)

**Limitations:**
- macOS guests only on Apple Silicon
- No Windows guests (Apple Silicon)
- Limited legacy hardware emulation

### Hypervisor.framework

The lower-level hypervisor API, primarily for Intel Macs:

- Direct CPU virtualization (VT-x)
- Used by Docker Desktop (Intel), VMware, Parallels
- Being superseded by Virtualization.framework

### QEMU

Open-source machine emulator:

- **Emulation mode**: Run any architecture (slow)
- **Virtualization mode**: Near-native speed when combined with Apple's hypervisors
- Can emulate x86 on ARM Mac (and vice versa)

## UTM: Free and Open Source

UTM is a full-featured virtualization app built on QEMU with a native macOS interface.

### Installation

```bash
# Via Homebrew
$ brew install --cask utm

# Or download from https://getutm.app
```

### Creating a Linux VM

1. **Download an ISO**: Get a Linux distribution ISO (e.g., Ubuntu, Fedora)

2. **Create new VM in UTM**:
   - Click "Create a New Virtual Machine"
   - Select "Virtualize" (for ARM Linux on Apple Silicon)
   - Select "Emulate" (for x86 Linux on Apple Silicon, slower)
   - Select Linux as the operating system
   - Browse to your ISO file

3. **Configure Resources**:
   - Memory: 4GB+ recommended
   - CPU Cores: 2-4 for good performance
   - Storage: 30GB+ for development

### Using UTM from Terminal

UTM supports automation via AppleScript and the `utmctl` command:

```bash
# List VMs
$ utmctl list

# Start a VM
$ utmctl start "Ubuntu 24.04"

# Stop a VM
$ utmctl stop "Ubuntu 24.04"

# Get VM IP address (for SSH)
$ utmctl ip "Ubuntu 24.04"
192.168.64.5
```

### UTM Virtualization vs Emulation

| Mode | Use Case | Performance |
|------|----------|-------------|
| Virtualize | ARM Linux on Apple Silicon | Near-native |
| Virtualize | x86 Linux on Intel Mac | Near-native |
| Emulate | x86 Linux on Apple Silicon | Slow (~10% native) |
| Emulate | ARM Linux on Intel Mac | Slow |

### Shared Folders

UTM supports shared folders via SPICE WebDAV or VirtioFS:

```bash
# In guest Linux, mount VirtioFS share (if configured)
$ sudo mkdir /mnt/share
$ sudo mount -t virtiofs share /mnt/share

# Or use WebDAV mount
$ sudo mount -t davfs http://localhost:9843 /mnt/share
```

### Pros and Cons

**Pros:**
- Free and open source
- Native macOS app with good UI
- Supports both virtualization and emulation
- Can run Windows, Linux, other OSes

**Cons:**
- Emulation is slow for x86 on ARM
- No GPU passthrough
- Snapshots less seamless than commercial options

## Parallels Desktop

Parallels is a commercial virtualization solution known for excellent performance and macOS integration.

### Installation

```bash
# Via Homebrew
$ brew install --cask parallels

# Or download from https://www.parallels.com
```

### Key Features

- **Coherence Mode**: Run Linux apps as if they were macOS apps
- **Automatic optimization**: Adjusts VM settings for best performance
- **Snapshots**: Create and manage VM states
- **Rosetta 2 Linux**: Run x86 binaries in ARM Linux guests

### Creating a Linux VM

Parallels automates much of the setup:

```bash
# Command-line VM creation
$ prlctl create "Ubuntu Dev" --distribution ubuntu
$ prlctl set "Ubuntu Dev" --cpus 4 --memsize 8192
$ prlctl start "Ubuntu Dev"
```

Or through the GUI:
1. File → New
2. Download or select Linux ISO
3. Configure resources
4. Install

### Parallels Command-Line Tools

```bash
# List VMs
$ prlctl list -a
UUID                                    STATUS       IP_ADDR         NAME
{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}  running      10.211.55.3     Ubuntu Dev

# Start/stop/suspend
$ prlctl start "Ubuntu Dev"
$ prlctl stop "Ubuntu Dev"
$ prlctl suspend "Ubuntu Dev"

# Create snapshot
$ prlctl snapshot "Ubuntu Dev" --name "Clean Install"

# Restore snapshot
$ prlctl snapshot-switch "Ubuntu Dev" --id "snap-id"

# SSH into VM
$ ssh user@$(prlctl list -o ip "Ubuntu Dev" 2>/dev/null | tail -1)
```

### Shared Folders

```bash
# Configure shared folder
$ prlctl set "Ubuntu Dev" --shf-host on
$ prlctl set "Ubuntu Dev" --shared-folder-add shared --path ~/Shared

# In Linux guest, access at:
# /media/psf/shared
```

### Networking Modes

```bash
# Shared networking (NAT) - default
$ prlctl set "Ubuntu Dev" --device-set net0 --type shared

# Bridged networking (VM on same network as host)
$ prlctl set "Ubuntu Dev" --device-set net0 --type bridged

# Host-only networking
$ prlctl set "Ubuntu Dev" --device-set net0 --type host-only
```

### Pros and Cons

**Pros:**
- Excellent performance, especially on Apple Silicon
- Best Windows support on Apple Silicon
- Great macOS integration (Coherence mode)
- Commercial support

**Cons:**
- Commercial license required (subscription model)
- Can be expensive for occasional use
- Runs background services

## VMware Fusion

VMware Fusion is another commercial virtualization option with enterprise features.

### Installation

```bash
# Via Homebrew
$ brew install --cask vmware-fusion

# Download from https://www.vmware.com/products/fusion.html
# Free "Fusion Player" license available for personal use
```

### Creating a Linux VM

```bash
# Using vmrun command-line tool
$ vmrun -T fusion start /path/to/vm.vmx

# Or through GUI:
# File → New → Create Custom VM → Linux
```

### VMware Command-Line Interface

```bash
# vmrun is the primary CLI tool
$ vmrun list  # List running VMs

# Start VM
$ vmrun -T fusion start "/path/to/vm.vmx"

# Stop VM
$ vmrun -T fusion stop "/path/to/vm.vmx"

# Soft stop (guest shutdown)
$ vmrun -T fusion stop "/path/to/vm.vmx" soft

# Suspend
$ vmrun -T fusion suspend "/path/to/vm.vmx"

# Take snapshot
$ vmrun -T fusion snapshot "/path/to/vm.vmx" "Snapshot Name"

# Revert to snapshot
$ vmrun -T fusion revertToSnapshot "/path/to/vm.vmx" "Snapshot Name"

# Run command in guest
$ vmrun -T fusion -gu username -gp password \
    runProgramInGuest "/path/to/vm.vmx" /bin/ls

# Copy file to guest
$ vmrun -T fusion -gu username -gp password \
    copyFileFromHostToGuest "/path/to/vm.vmx" \
    ~/local/file.txt /home/user/file.txt
```

### Shared Folders

VMware Fusion uses HGFS (Host-Guest File System):

```bash
# In Linux guest, install open-vm-tools
$ sudo apt install open-vm-tools open-vm-tools-desktop

# Mount shared folder
$ sudo mkdir /mnt/hgfs
$ sudo mount -t fuse.vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other

# Add to /etc/fstab for automatic mounting
# .host:/    /mnt/hgfs    fuse.vmhgfs-fuse    allow_other    0    0
```

### Pros and Cons

**Pros:**
- Enterprise-grade virtualization
- Good Linux guest tools
- Free personal use license
- Established product

**Cons:**
- Apple Silicon support still evolving
- Heavier than UTM
- Can conflict with other hypervisors

## Using Virtualization.framework Directly

For developers wanting programmatic control, Apple's Virtualization.framework can be used directly.

### Swift Example

```swift
import Virtualization

// Create VM configuration
let config = VZVirtualMachineConfiguration()

// CPU
config.cpuCount = 4

// Memory
config.memorySize = 4 * 1024 * 1024 * 1024  // 4GB

// Boot loader (Linux)
let bootLoader = VZLinuxBootLoader(kernelURL: kernelURL)
bootLoader.initialRamdiskURL = initrdURL
bootLoader.commandLine = "console=hvc0"
config.bootLoader = bootLoader

// Storage
let diskAttachment = try VZDiskImageStorageDeviceAttachment(
    url: diskURL,
    readOnly: false
)
config.storageDevices = [VZVirtioBlockDeviceConfiguration(attachment: diskAttachment)]

// Network
let network = VZNATNetworkDeviceAttachment()
config.networkDevices = [VZVirtioNetworkDeviceConfiguration(attachment: network)]

// Create and start VM
let vm = VZVirtualMachine(configuration: config)
try await vm.start()
```

### Projects Using Virtualization.framework

Several open-source projects wrap Virtualization.framework:

- **Tart**: VM management focused on macOS guests for CI
- **VirtualBuddy**: GUI for macOS VMs
- **Lima**: Configurable Linux VMs (covered in containers chapter)

```bash
# Tart for macOS VMs (good for CI)
$ brew install cirruslabs/cli/tart
$ tart clone ghcr.io/cirruslabs/macos-ventura-base:latest ventura-dev
$ tart run ventura-dev
```

## Running Linux Distributions

### Ubuntu

Ubuntu provides official ARM64 images for Apple Silicon:

```bash
# Download Ubuntu Server for ARM
# https://ubuntu.com/download/server/arm

# For desktop, use Ubuntu Desktop ARM64
# https://cdimage.ubuntu.com/jammy/daily-live/current/
```

### Fedora

Fedora has excellent ARM64 support:

```bash
# Download Fedora Server/Workstation for aarch64
# https://getfedora.org/

# Fedora runs very well on Apple Silicon
```

### Debian

Debian supports ARM64:

```bash
# https://www.debian.org/distrib/
# Download arm64 netinst or DVD image
```

### Arch Linux

Arch Linux ARM for Apple Silicon:

```bash
# https://archlinuxarm.org/
# Requires more manual setup but works well
```

## x86 Linux on Apple Silicon

Running x86 Linux on Apple Silicon Macs is possible but involves trade-offs.

### Emulation (QEMU/UTM)

Full emulation is slow but compatible:

```bash
# In UTM, select "Emulate" instead of "Virtualize"
# Performance: ~10% of native
# Use case: Testing x86-specific software
```

### Rosetta 2 Translation

Better option: ARM Linux with Rosetta for x86 binaries:

```bash
# In an ARM Linux VM (Parallels, UTM, Lima)
# Install Rosetta (if supported)

# On Parallels:
# Rosetta is auto-configured for supported distros

# On Lima with Rosetta:
$ limactl create --name rosetta template://default --rosetta

# Now x86 binaries run with Rosetta translation
$ lima
user@lima-rosetta:~$ arch
aarch64
user@lima-rosetta:~$ /usr/bin/x86_binary  # Runs via Rosetta
```

Rosetta in Linux provides much better performance than full emulation (typically 70-80% of native ARM performance for translated x86 code).

## VM Networking

### NAT Networking

Default mode; VM shares host's network connection:

```
┌─────────────────────────────────────────┐
│              macOS Host                  │
│  ┌───────────────────────────────────┐  │
│  │     VM (192.168.64.x)             │  │
│  └─────────────┬─────────────────────┘  │
│                │ NAT                     │
│  ┌─────────────┴─────────────────────┐  │
│  │  Host Network (192.168.1.x)       │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

```bash
# VM can access internet
# Host can't directly reach VM (port forwarding needed)

# Forward port 22 from host to VM (UTM)
# Configure in UTM VM settings → Network → Port Forward
```

### Bridged Networking

VM appears as separate device on network:

```
┌────────────────────────────────────────────────┐
│                Network Switch                   │
├──────────────┬──────────────┬─────────────────┤
│ macOS Host   │     VM       │  Other Devices   │
│ 192.168.1.10 │ 192.168.1.20 │  192.168.1.x    │
└──────────────┴──────────────┴─────────────────┘
```

```bash
# VM gets own IP from DHCP
# Other network devices can reach VM directly
# Useful for testing network services
```

### Host-Only Networking

Private network between host and VMs:

```bash
# VM can communicate with host and other VMs
# VM cannot access external network
# Good for isolated development environments
```

## Shared Folders and File Sharing

### VirtioFS (Fastest)

Available with Virtualization.framework:

```bash
# In Linux guest
$ sudo mount -t virtiofs sharename /mnt/shared

# Add to /etc/fstab
# sharename /mnt/shared virtiofs rw 0 0
```

### SSHFS (Flexible)

Mount directories over SSH:

```bash
# On host, enable Remote Login (SSH)
# System Preferences → Sharing → Remote Login

# In Linux guest
$ sudo apt install sshfs
$ sshfs user@192.168.64.1:/Users/user/Projects /mnt/projects

# user@host-ip:/path /mount/point fuse.sshfs defaults 0 0
```

### NFS (Traditional)

Reliable for larger setups:

```bash
# On macOS host, export directory
$ sudo bash -c 'echo "/Users/user/Shared -network 192.168.64.0 -mask 255.255.255.0" >> /etc/exports'
$ sudo nfsd restart

# In Linux guest
$ sudo mount -t nfs 192.168.64.1:/Users/user/Shared /mnt/shared
```

## Development Workflow with VMs

### Set Up SSH Access

```bash
# In VM, install and enable SSH
$ sudo apt install openssh-server
$ sudo systemctl enable ssh

# Get VM IP
$ hostname -I
192.168.64.5

# From macOS, add to ~/.ssh/config
Host dev-vm
    HostName 192.168.64.5
    User developer
    ForwardAgent yes

# Now connect easily
$ ssh dev-vm
```

### VS Code Remote Development

```bash
# Install Remote - SSH extension in VS Code
# Connect to VM: Cmd+Shift+P → Remote-SSH: Connect to Host

# Or use devcontainers
# Define .devcontainer/devcontainer.json
# Use VM as Docker host
```

### Synced Development

```bash
# Option 1: Edit on Mac, run in VM via shared folder
# Option 2: Full dev environment in VM, SSH from Mac
# Option 3: Use VS Code Remote for best of both

# Mount project from Mac to VM
$ sshfs user@192.168.64.1:/Users/user/Projects ~/projects
$ cd ~/projects/myapp
$ npm run dev
```

## Performance Tips

### Allocate Resources Appropriately

```bash
# For development VMs:
# - CPU: Half of physical cores (4 on 8-core Mac)
# - Memory: 8GB for comfortable IDE use
# - Disk: SSD, 50GB+ for development tools

# For testing/CI VMs:
# - CPU: 2 cores sufficient
# - Memory: 4GB usually enough
# - Disk: 20GB for minimal installs
```

### Use Snapshots Wisely

```bash
# Create baseline snapshot after initial setup
# Use for quick rollback during testing
# Delete old snapshots to recover disk space

# Parallels
$ prlctl snapshot "VM" --name "Baseline"

# VMware
$ vmrun snapshot "/path/to/vm.vmx" "Baseline"
```

### Headless Operation

```bash
# Run VMs headlessly for server workloads
# Less resource usage without GUI

# UTM: Window → Minimize or run via utmctl
# Parallels: prlctl start "VM" --headless
# VMware: vmrun start "/path/to/vm.vmx" nogui
```

## Summary

| Solution | License | Best For | Apple Silicon |
|----------|---------|----------|---------------|
| UTM | Free | Personal use, testing | Good (QEMU) |
| Parallels | Commercial | Developers, Windows users | Excellent |
| VMware Fusion | Commercial/Free* | Enterprise, existing VMware | Good |
| Virtualization.framework | N/A (API) | Custom solutions, Lima/Tart | Native |

*Free personal use license available

Key considerations:
1. **Apple Silicon**: Native ARM VMs are fast; x86 requires emulation or Rosetta
2. **File sharing**: VirtioFS is fastest, SSHFS is most flexible
3. **Networking**: NAT for isolation, bridged for network testing
4. **Snapshots**: Essential for development and testing workflows
5. **Resources**: Balance VM allocation with host system needs
