# MacPorts: An Alternative Approach

MacPorts predates Homebrew and takes a different philosophical approach to package management on macOS. While Homebrew focuses on ease of use and leveraging macOS system libraries, MacPorts prioritizes isolation, reproducibility, and Unix tradition. Understanding MacPorts helps you make informed choices about which tool to use—and some software is only available in MacPorts.

## MacPorts Philosophy

MacPorts follows principles from the BSD ports tradition:

1. **Self-contained**: Installs entirely under `/opt/local`, separate from system
2. **Build from source**: Compiles everything by default
3. **Explicit variants**: Fine-grained control over build options
4. **Full dependency trees**: Doesn't rely on macOS system libraries

This approach offers predictability and control at the cost of longer installation times.

## Installation

### Prerequisites

MacPorts requires Xcode or Command Line Tools:

```bash
$ xcode-select --install
```

### Installing MacPorts

Download the installer from [macports.org](https://www.macports.org/install.php) for your macOS version, or install manually:

```bash
# Download (check macports.org for current version)
$ curl -O https://distfiles.macports.org/MacPorts/MacPorts-2.8.1.tar.bz2
$ tar xf MacPorts-2.8.1.tar.bz2
$ cd MacPorts-2.8.1

# Build and install
$ ./configure
$ make
$ sudo make install
```

### Post-Installation

Add to your PATH in `~/.zprofile`:

```bash
export PATH="/opt/local/bin:/opt/local/sbin:$PATH"
export MANPATH="/opt/local/share/man:$MANPATH"
```

Verify:

```bash
$ source ~/.zprofile
$ port version
Version: 2.8.1
```

## Basic Usage

### Searching for Ports

```bash
# Search by name
$ port search postgresql
postgresql15 @15.4 (databases)
    PostgreSQL is a highly-extensible, object-relational database...

# Search descriptions too
$ port search --description "web server"

# List all ports
$ port list
```

### Getting Information

```bash
$ port info nginx
nginx @1.24.0 (www)
Variants:             debug, gzip_static, http2, mail, realip...

Description:          Nginx is a high performance HTTP server
Homepage:             https://nginx.org/

Build Dependencies:   pcre
Library Dependencies: openssl, pcre, zlib
Platforms:            darwin
License:              BSD
Maintainers:          Email: ...
```

### Installing Ports

```bash
# Install (requires sudo)
$ sudo port install nginx

# Install with variants
$ sudo port install nginx +http2 +realip

# Install specific version
$ sudo port install postgresql15 @15.4

# Dry run (see what would happen)
$ port -y install nginx
```

### Updating

```bash
# Update port definitions
$ sudo port selfupdate

# Upgrade all outdated ports
$ sudo port upgrade outdated

# Upgrade specific port
$ sudo port upgrade nginx

# See what's outdated
$ port outdated
```

### Uninstalling

```bash
# Uninstall a port
$ sudo port uninstall nginx

# Uninstall with dependents
$ sudo port uninstall --follow-dependents nginx

# Remove inactive ports (old versions)
$ sudo port uninstall inactive
```

## Variants

Variants are build options that customize port installation:

```bash
# List variants for a port
$ port variants vim
vim has the variants:
   athena: Use Athena widgets for GUI
   big: Build big features
   cscope: Enable cscope interface
   gtk2: Enable GTK2 GUI
   gtk3: Enable GTK3 GUI
   huge: Build huge features
   ...

# Install with variant
$ sudo port install vim +huge +python311

# Install without default variant
$ sudo port install vim -x11

# View installed variants
$ port installed vim
  vim @9.0.1000_0+huge+python311 (active)
```

### Default Variants

Configure default variants in `/opt/local/etc/macports/variants.conf`:

```conf
# Always build with these variants
+no_x11
+quartz

# Never build with these
-debug
```

## Port Lifecycle

### Version Management

```bash
# List installed versions
$ port installed
  nginx @1.24.0_0 (active)
  nginx @1.22.0_0

# Activate specific version
$ sudo port activate nginx @1.22.0_0

# Deactivate (without uninstalling)
$ sudo port deactivate nginx
```

### Cleaning

```bash
# Clean build artifacts for a port
$ sudo port clean nginx

# Clean all ports
$ sudo port clean --all all

# Clean work directories, distribution files, and archives
$ sudo port clean --all --dist nginx
```

## File Locations

```
/opt/local/
├── bin/              # Binaries
├── etc/              # Configuration files
├── include/          # Header files
├── lib/              # Libraries
├── libexec/          # Support programs
├── sbin/             # System binaries
├── share/            # Architecture-independent files
├── var/              # Variable data (logs, databases)
│   └── macports/     # MacPorts-specific
│       ├── build/    # Build directories
│       └── sources/  # Downloaded sources
└── Library/
    └── Frameworks/   # macOS frameworks
```

## Maintenance

### Regular Maintenance

```bash
# Full update cycle
$ sudo port selfupdate
$ sudo port upgrade outdated
$ sudo port uninstall inactive
$ sudo port clean --all all
```

### Reclaiming Disk Space

```bash
# Remove inactive ports
$ sudo port uninstall inactive

# Remove unrequested ports (installed as dependencies but no longer needed)
$ sudo port uninstall leaves

# Clean all working directories and sources
$ sudo port clean --all installed
$ sudo rm -rf /opt/local/var/macports/distfiles/*
```

### Checking Installation

```bash
# Verify installed ports
$ port -v installed

# Check for broken ports
$ port diagnose

# List dependencies
$ port deps nginx
$ port rdeps nginx  # Recursive
$ port dependents nginx  # What depends on this
```

## MacPorts vs Homebrew

### Key Differences

| Aspect | MacPorts | Homebrew |
|--------|----------|----------|
| Installation prefix | `/opt/local` | `/opt/homebrew` or `/usr/local` |
| Default build | From source | Pre-built bottles |
| System integration | Self-contained | Uses macOS libraries |
| Sudo required | Yes | No |
| Build customization | Variants | Limited options |
| Package count | ~27,000 | ~6,000 formulae |
| Speed | Slower (compiles) | Faster (bottles) |

### When to Use MacPorts

- **Need specific build options**: Variants offer fine-grained control
- **Isolation is important**: Complete separation from system
- **Package only in MacPorts**: Some software not in Homebrew
- **Reproducible builds**: Same configuration everywhere
- **Server environments**: Predictable, isolated installations

### When to Use Homebrew

- **Quick installation**: Bottles are fast
- **macOS integration**: Works with system libraries
- **Casual use**: Simpler command syntax
- **GUI apps**: Casks are convenient
- **Most common packages**: Available and well-maintained

## Coexistence

You can run both MacPorts and Homebrew:

```bash
# Different paths, different prefixes
/opt/local/bin/python3          # MacPorts
/opt/homebrew/bin/python3        # Homebrew

# Adjust PATH to prefer one
export PATH="/opt/homebrew/bin:$PATH"       # Prefer Homebrew
# or
export PATH="/opt/local/bin:$PATH"          # Prefer MacPorts
```

**Caveats**:
- Don't mix libraries from both in builds
- Be explicit about which version you're using
- Can cause confusion with shared dependencies

## Creating Custom Ports

### Portfile Basics

```ruby
# /opt/local/var/macports/sources/.../myport/Portfile

PortSystem          1.0

name                myapp
version             1.0.0
categories          devel
platforms           darwin
maintainers         {example.com:admin}
description         My application
long_description    ${description} with more details

homepage            https://example.com/myapp
master_sites        https://example.com/downloads/

checksums           rmd160  abc123... \
                    sha256  def456...

depends_lib         port:openssl

configure.args      --with-ssl=${prefix}

post-destroot {
    xinstall -d ${destroot}${prefix}/share/doc/${name}
    xinstall -m 644 ${worksrcpath}/README ${destroot}${prefix}/share/doc/${name}
}
```

### Local Portfile Repository

```bash
# Create local repository
$ mkdir -p ~/ports/mycat/myport
$ cp Portfile ~/ports/mycat/myport/

# Add to sources.conf
$ sudo nano /opt/local/etc/macports/sources.conf
# Add line:
file:///Users/username/ports

# Update index
$ cd ~/ports
$ portindex

# Now install
$ sudo port install myport
```

## Troubleshooting

### Common Issues

**Port fails to build**:
```bash
# View build log
$ port logfile nginx
$ less /opt/local/var/macports/logs/_opt_local_var_macports_sources_.../nginx/main.log

# Try clean build
$ sudo port clean nginx
$ sudo port install nginx
```

**Dependency conflicts**:
```bash
# See dependency graph
$ port deps nginx

# Force rebuild of dependencies
$ sudo port -n upgrade --force nginx
```

**Stuck port**:
```bash
# Force uninstall
$ sudo port -f uninstall nginx

# Clean state
$ sudo port clean nginx
$ sudo rm -rf /opt/local/var/macports/build/*nginx*
```

### Recovering from Problems

```bash
# Selfupdate when things are broken
$ sudo port -d selfupdate

# Restore ports from backup
$ sudo port -f restore /path/to/backup.tar.gz

# Migration after macOS upgrade
$ sudo port migrate
```

## Summary

MacPorts provides:
- Self-contained installation in `/opt/local`
- Build-from-source with variants for customization
- Large package collection
- Traditional Unix approach

Best for users who:
- Need specific build configurations
- Prefer isolation from system
- Have time for source compilation
- Need packages not in Homebrew

Commands reference:

| Action | Command |
|--------|---------|
| Search | `port search name` |
| Info | `port info name` |
| Install | `sudo port install name` |
| Update | `sudo port selfupdate` |
| Upgrade | `sudo port upgrade outdated` |
| Uninstall | `sudo port uninstall name` |
| Clean | `sudo port uninstall inactive` |
