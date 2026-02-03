# Running Linux Containers on macOS

Containers have revolutionized software development, but there's a fundamental challenge on macOS: containers are a Linux technology. Features like cgroups, namespaces, and overlay filesystems are Linux kernel features that don't exist in macOS's XNU kernel. Running Linux containers on macOS requires virtualization to provide a Linux kernel. This chapter explains how different container solutions accomplish this and helps you choose the right one for your workflow.

## How Containers Work on macOS

On Linux, containers run directly on the host kernel:

```
┌─────────────────────────────────────────────────┐
│                 Linux Host                       │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │Container 1│ │Container 2│ │Container 3│     │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘     │
│        │             │             │            │
│  ┌─────┴─────────────┴─────────────┴─────┐     │
│  │          Linux Kernel                  │     │
│  │    (cgroups, namespaces, overlayfs)   │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘
```

On macOS, a Linux VM sits between containers and the host:

```
┌─────────────────────────────────────────────────┐
│                  macOS Host                      │
│  ┌───────────────────────────────────────────┐  │
│  │              Linux VM                      │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐     │  │
│  │  │Container│ │Container│ │Container│     │  │
│  │  └────┬────┘ └────┬────┘ └────┬────┘     │  │
│  │       │           │           │          │  │
│  │  ┌────┴───────────┴───────────┴────┐     │  │
│  │  │        Linux Kernel              │     │  │
│  │  └──────────────────────────────────┘     │  │
│  └───────────────────────────────────────────┘  │
│                       │                          │
│           Hypervisor / Virtualization            │
│  ┌───────────────────────────────────────────┐  │
│  │     Virtualization.framework / QEMU        │  │
│  └───────────────────────────────────────────┘  │
│                       │                          │
│  ┌───────────────────────────────────────────┐  │
│  │              XNU Kernel                    │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

This architecture has implications for performance, file sharing, and networking.

## Virtualization Technologies

### Apple Virtualization.framework

Apple's native virtualization framework, introduced in macOS Big Sur and significantly enhanced for Apple Silicon:

- **Apple Silicon**: Near-native performance, runs ARM64 Linux
- **Intel**: Uses hardware virtualization (VT-x)
- **Features**: Fast boot, file sharing, networking, Rosetta 2 for x86 emulation

Used by: Docker Desktop, Colima, Lima

### QEMU

Open-source machine emulator and virtualizer:

- **Emulation**: Can run x86 Linux on ARM Mac (slower)
- **Virtualization**: Fast when using HVF (Hypervisor.framework) or Virtualization.framework
- **Flexibility**: Supports many architectures

Used by: UTM, Lima (optional), Podman Machine

### Hypervisor.framework

Apple's lower-level virtualization API (Intel only):

- Used by older container solutions on Intel Macs
- More direct hardware access
- Being superseded by Virtualization.framework

## Docker Desktop

Docker Desktop is the official Docker solution for macOS. It provides a complete container development environment.

### Installation

```bash
# Via Homebrew Cask
$ brew install --cask docker

# Or download from docker.com
```

### Architecture

```
┌────────────────────────────────────────────────┐
│                Docker Desktop                   │
│  ┌──────────────────────────────────────────┐  │
│  │            GUI / Menu Bar App             │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │        Docker Engine (in Linux VM)        │  │
│  │  ┌──────────────────────────────────────┐│  │
│  │  │   containerd → containers            ││  │
│  │  └──────────────────────────────────────┘│  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │    Linux VM (custom LinuxKit distro)     │  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │    Virtualization.framework / HVF        │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Configuration

Docker Desktop settings (accessible via GUI or `~/.docker/config.json`):

```bash
# View current configuration
$ cat ~/.docker/config.json

# Resource allocation (GUI or settings file)
# CPUs: Number of CPU cores for the VM
# Memory: RAM allocation (default: 2GB)
# Disk: Virtual disk size

# Example config.json
{
  "auths": {},
  "credHelpers": {
    "docker.io": "desktop"
  },
  "currentContext": "desktop-linux"
}
```

### File Sharing

Docker Desktop provides multiple file sharing mechanisms:

```bash
# Volume mounts use gRPC FUSE (default) or VirtioFS
# Check current mechanism in Docker Desktop settings

# Mount a local directory
$ docker run -v ~/Projects/myapp:/app -it ubuntu bash

# VirtioFS (faster, macOS 12.5+)
# Enable in: Docker Desktop → Settings → General → VirtioFS
```

Performance tip: Use `.dockerignore` to exclude files that don't need to be in the build context:

```bash
# .dockerignore
node_modules
.git
*.log
.DS_Store
```

### Networking

```bash
# Host networking mode isn't fully supported
# Containers use bridge networking by default

# Publish ports to macOS host
$ docker run -p 8080:80 nginx

# Access from Mac at localhost:8080

# Container-to-container networking
$ docker network create mynet
$ docker run --network mynet --name db postgres
$ docker run --network mynet myapp  # Can reach db by hostname
```

### Docker Compose

```bash
# Install (included with Docker Desktop)
$ docker compose version

# Example compose file
$ cat docker-compose.yml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app
    depends_on:
      - db
  db:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:

# Start services
$ docker compose up -d
```

### Pros and Cons

**Pros:**
- Official Docker product, best documentation
- Integrated Kubernetes
- GUI for management
- Automatic updates

**Cons:**
- Resource intensive (runs constantly in background)
- Commercial license required for larger companies
- Can be slow for file-heavy operations

## Colima: Lightweight Docker Alternative

Colima provides Docker and containerd runtimes in a lightweight VM, using Lima under the hood.

### Installation

```bash
# Install Colima and Docker CLI
$ brew install colima docker docker-compose

# Start Colima (creates and starts VM)
$ colima start

# Verify Docker works
$ docker ps
```

### Architecture

```
┌────────────────────────────────────────────────┐
│                   Colima                        │
│  ┌──────────────────────────────────────────┐  │
│  │              Lima VM                      │  │
│  │  ┌──────────────────────────────────────┐│  │
│  │  │    Docker daemon / containerd        ││  │
│  │  └──────────────────────────────────────┘│  │
│  │  ┌──────────────────────────────────────┐│  │
│  │  │    Alpine Linux (minimal)            ││  │
│  │  └──────────────────────────────────────┘│  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │       Virtualization.framework           │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Configuration

```bash
# Start with custom resources
$ colima start --cpu 4 --memory 8 --disk 60

# View current configuration
$ colima status

# Use specific runtime
$ colima start --runtime docker     # Docker (default)
$ colima start --runtime containerd # containerd + nerdctl

# Use Rosetta 2 for x86 emulation (Apple Silicon)
$ colima start --arch x86_64 --vm-type vz --vz-rosetta

# Multiple profiles for different projects
$ colima start --profile work --cpu 4 --memory 8
$ colima start --profile personal --cpu 2 --memory 4

# Switch between profiles
$ colima list
$ docker context use colima-work
```

### File Sharing

```bash
# By default, mounts home directory
$ colima start  # Mounts ~ into VM

# Custom mounts
$ colima start --mount ~/Projects:w  # Read-write mount

# VirtioFS (faster, Apple Silicon + macOS 13+)
$ colima start --vm-type vz --mount-type virtiofs
```

### Managing Colima

```bash
# Start/stop
$ colima start
$ colima stop

# SSH into the VM
$ colima ssh

# Delete VM (removes all containers and images)
$ colima delete

# Update Colima itself
$ brew upgrade colima
```

### Pros and Cons

**Pros:**
- Lightweight, starts quickly
- Free and open source
- Uses Docker CLI (drop-in replacement)
- Multiple profiles for different resource needs

**Cons:**
- Less polished than Docker Desktop
- No built-in Kubernetes (but easy to add)
- Community supported

## Podman: Daemonless Containers

Podman runs containers without a central daemon, providing a more Unix-like approach. Each container is a child process of the podman command.

### Installation

```bash
# Install Podman
$ brew install podman

# Initialize the Podman machine (creates VM)
$ podman machine init

# Start the machine
$ podman machine start

# Verify it's running
$ podman info
```

### Architecture

```
┌────────────────────────────────────────────────┐
│                   Podman                        │
│  ┌──────────────────────────────────────────┐  │
│  │           Podman Machine                  │  │
│  │  ┌──────────────────────────────────────┐│  │
│  │  │   podman (runs containers directly)  ││  │
│  │  └──────────────────────────────────────┘│  │
│  │  ┌──────────────────────────────────────┐│  │
│  │  │         Fedora CoreOS                ││  │
│  │  └──────────────────────────────────────┘│  │
│  └──────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────┐  │
│  │              QEMU + HVF                  │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

### Docker Compatibility

Podman is CLI-compatible with Docker:

```bash
# Most Docker commands work identically
$ podman pull nginx
$ podman run -d -p 8080:80 nginx
$ podman ps
$ podman stop <container>

# Create alias for complete compatibility
$ alias docker=podman

# Or use the podman-docker package
$ brew install podman-docker
# Creates /usr/local/bin/docker → podman
```

### Podman Compose

```bash
# Install podman-compose
$ pip3 install podman-compose

# Or use Docker Compose with Podman
$ podman compose up -d  # Works with docker-compose.yml
```

### Rootless Containers

Podman's design supports rootless containers:

```bash
# Run container as non-root (default on macOS)
$ podman run -it alpine whoami
root  # Root inside container, but mapped to user outside
```

### Managing Podman Machines

```bash
# List machines
$ podman machine list

# Stop machine
$ podman machine stop

# SSH into machine
$ podman machine ssh

# Configure machine resources
$ podman machine init --cpus 4 --memory 4096 --disk-size 50

# Remove machine
$ podman machine rm
```

### Pros and Cons

**Pros:**
- Daemonless architecture
- Docker CLI compatible
- Rootless by default
- Podman Desktop GUI available

**Cons:**
- Fewer features than Docker Desktop
- Some Docker Compose edge cases may not work
- Smaller community

## Lima: Linux Machines for macOS

Lima creates Linux virtual machines with automatic file sharing and port forwarding. It's the foundation for Colima but can be used directly for running Linux VMs.

### Installation

```bash
$ brew install lima
```

### Creating a VM

```bash
# Create default VM (Ubuntu)
$ limactl create default

# Start VM
$ limactl start default

# Run command in VM
$ lima uname -a
Linux lima-default 5.15.0-xx-generic ...

# Get a shell in VM
$ lima
user@lima-default:~$

# Run Linux binaries directly
$ lima apt update
$ lima docker run hello-world
```

### Pre-configured Templates

Lima includes templates for common use cases:

```bash
# List available templates
$ limactl create --list-templates

# Docker template
$ limactl create --name docker template://docker
$ limactl start docker

# Kubernetes (k3s)
$ limactl create --name k3s template://k3s
$ limactl start k3s

# Fedora
$ limactl create --name fedora template://fedora

# Arch Linux
$ limactl create --name arch template://archlinux
```

### Custom Configuration

```yaml
# my-linux.yaml
images:
  - location: "https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img"
    arch: "x86_64"
  - location: "https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-arm64.img"
    arch: "aarch64"

cpus: 4
memory: "8GiB"
disk: "50GiB"

mounts:
  - location: "~/Projects"
    writable: true

containerd:
  system: true
  user: false

provision:
  - mode: system
    script: |
      #!/bin/bash
      apt-get update
      apt-get install -y build-essential git

portForwards:
  - guestPort: 3000
    hostPort: 3000
```

```bash
# Create VM from custom config
$ limactl create --name myvm my-linux.yaml
$ limactl start myvm
```

### File Sharing

Lima provides automatic file sharing:

```bash
# Default: home directory is mounted read-only
# /tmp/lima is mounted read-write

# In VM, macOS home is at:
$ lima ls /Users/yourname/

# Writable mount configuration in yaml:
mounts:
  - location: "~/Projects"
    writable: true
```

### Pros and Cons

**Pros:**
- Very flexible, runs full Linux distributions
- Lightweight VMs
- Good file sharing
- Foundation for other tools

**Cons:**
- More manual setup
- Not specifically for containers (though supports them)
- Less integrated container workflow

## Comparison Matrix

| Feature | Docker Desktop | Colima | Podman | Lima |
|---------|---------------|--------|--------|------|
| Setup Complexity | Easy | Easy | Medium | Medium |
| Resource Usage | High | Low | Medium | Low |
| Docker CLI | Native | Native | Compatible | Manual |
| GUI | Yes | No | Optional | No |
| Kubernetes | Built-in | Add-on | Optional | Templates |
| License | Commercial* | Free | Free | Free |
| File Performance | Good | Good | Adequate | Good |
| Apple Silicon | Yes | Yes | Yes | Yes |
| x86 Emulation | Yes | Via Rosetta | Via QEMU | Via QEMU |

*Free for personal use and small businesses

## Performance Optimization

### General Tips

```bash
# Use volume mounts sparingly for large directories
# Instead, use named volumes for node_modules, vendor, etc.

# docker-compose.yml
services:
  app:
    volumes:
      - .:/app
      - node_modules:/app/node_modules  # Named volume, not bind mount

volumes:
  node_modules:
```

### VirtioFS for Better File Performance

```bash
# Docker Desktop: Enable in Settings → General → VirtioFS

# Colima: Use VZ runtime with VirtioFS
$ colima start --vm-type vz --mount-type virtiofs
```

### Multi-Architecture Builds

For building images that work on both ARM and x86:

```bash
# Enable buildx (Docker Desktop has this by default)
$ docker buildx create --use

# Build for multiple platforms
$ docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .

# Build and push to registry
$ docker buildx build --platform linux/amd64,linux/arm64 \
    -t registry/myapp:latest --push .
```

### Rosetta 2 for x86 Containers

Apple Silicon can run x86 Linux binaries via Rosetta 2:

```bash
# Colima with Rosetta
$ colima start --arch x86_64 --vm-type vz --vz-rosetta

# Docker Desktop: Enable in Settings → Features in development → Rosetta

# Now x86 images run with Rosetta translation
$ docker run --platform linux/amd64 ubuntu uname -m
x86_64
```

## Quick Start Recommendations

**For Docker Desktop users wanting a free alternative:**
```bash
$ brew install colima docker
$ colima start
# Use docker as normal
```

**For minimal resource usage:**
```bash
$ brew install colima docker
$ colima start --cpu 2 --memory 4 --disk 30
$ colima stop  # When not in use
```

**For full Linux development environment:**
```bash
$ brew install lima
$ limactl create --name dev template://ubuntu
$ limactl start dev
$ lima  # Enter Linux shell
```

**For rootless/daemonless preference:**
```bash
$ brew install podman
$ podman machine init
$ podman machine start
```

## Summary

All container solutions on macOS require a Linux VM layer. The choice depends on your needs:

- **Docker Desktop**: Best for teams standardized on Docker, need Kubernetes, want GUI
- **Colima**: Best for Docker users wanting lightweight, free alternative
- **Podman**: Best for those preferring daemonless architecture, OCI standards focus
- **Lima**: Best for running full Linux VMs, maximum flexibility

Key points:
1. Containers on macOS always run in a Linux VM
2. Apple's Virtualization.framework provides near-native performance
3. File sharing between macOS and containers has performance overhead
4. VirtioFS significantly improves file sharing performance
5. Rosetta 2 enables x86 container support on Apple Silicon
