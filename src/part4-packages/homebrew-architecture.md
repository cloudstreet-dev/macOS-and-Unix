# Homebrew: Architecture and Internals

Homebrew describes itself as "The Missing Package Manager for macOS." To use it effectively—especially when things go wrong—you need to understand its architecture: how formulae define packages, how the Cellar stores installations, how taps extend the available packages, and how bottles provide pre-compiled binaries.

## Installation Locations

Homebrew's location depends on your Mac's architecture:

### Apple Silicon (M1/M2/M3)

```
/opt/homebrew/
├── bin/           → Symlinks to installed binaries
├── Cellar/        → Installed packages (versioned)
├── etc/           → Configuration files
├── include/       → Header files
├── lib/           → Libraries
├── opt/           → Symlinks to latest versions
├── sbin/          → System binaries
├── share/         → Architecture-independent data
└── var/           → Variable data (logs, databases)
```

### Intel Macs

```
/usr/local/
├── bin/
├── Cellar/
├── etc/
...
```

### Why Different Locations?

Apple Silicon Macs use `/opt/homebrew` to:
- Avoid conflicts with Rosetta 2 (Intel emulation)
- Allow running both native and Intel Homebrew
- Follow Unix convention (`/opt` for optional software)

## Core Concepts

### Formulae

A formula is a Ruby script defining how to install a package:

```bash
# View a formula
$ brew cat wget
```

```ruby
class Wget < Formula
  desc "Internet file retriever"
  homepage "https://www.gnu.org/software/wget/"
  url "https://ftp.gnu.org/gnu/wget/wget-1.21.4.tar.gz"
  sha256 "81542f5cefb8faacc39bbbc6c82ded..."

  depends_on "openssl@3"
  depends_on "pkg-config" => :build

  def install
    system "./configure", "--prefix=#{prefix}",
                          "--with-ssl=openssl",
                          "--with-openssl-dir=#{Formula["openssl@3"].opt_prefix}"
    system "make", "install"
  end

  test do
    system "#{bin}/wget", "-O", "-", "https://example.com"
  end
end
```

Key elements:
- `desc`: Package description
- `homepage`: Project website
- `url`: Source code download location
- `sha256`: Checksum for verification
- `depends_on`: Dependencies
- `install`: Build and installation commands
- `test`: Verification test

### The Cellar

Installed packages live in the Cellar, organized by name and version:

```bash
$ ls /opt/homebrew/Cellar/wget/
1.21.4/

$ ls /opt/homebrew/Cellar/wget/1.21.4/
bin/        etc/        lib/        share/

$ ls /opt/homebrew/Cellar/wget/1.21.4/bin/
wget
```

This versioned structure allows:
- Multiple versions installed simultaneously
- Easy rollback (`brew switch wget 1.21.3`)
- Clean uninstallation

### Symlinks and opt

Homebrew creates symlinks for easy access:

```bash
# Symlink in bin
$ ls -la /opt/homebrew/bin/wget
lrwxr-xr-x  1 user  admin  32 Jan 15 10:00 /opt/homebrew/bin/wget -> ../Cellar/wget/1.21.4/bin/wget

# Stable reference in opt
$ ls -la /opt/homebrew/opt/wget
lrwxr-xr-x  1 user  admin  22 Jan 15 10:00 /opt/homebrew/opt/wget -> ../Cellar/wget/1.21.4
```

The `opt` directory provides stable paths regardless of version:
- `/opt/homebrew/opt/openssl` always points to current OpenSSL
- Useful for configuring build systems

### Bottles

Bottles are pre-compiled binary packages:

```bash
# Install from bottle (default)
$ brew install wget
==> Downloading https://ghcr.io/v2/homebrew/core/wget/manifests/1.21.4
==> Pouring wget--1.21.4.arm64_sonoma.bottle.tar.gz
```

Without bottles, Homebrew compiles from source:

```bash
# Force compilation from source
$ brew install --build-from-source wget
==> Downloading https://ftp.gnu.org/gnu/wget/wget-1.21.4.tar.gz
==> ./configure --prefix=/opt/homebrew/Cellar/wget/1.21.4 ...
==> make install
```

Bottles are:
- Pre-compiled for specific macOS versions
- Built for both Intel and Apple Silicon
- Stored on GitHub Container Registry
- Much faster than source compilation

### Taps

Taps are third-party formula repositories:

```bash
# List tapped repositories
$ brew tap
homebrew/core
homebrew/cask
homebrew/services

# Add a tap
$ brew tap hashicorp/tap

# Now you can install from it
$ brew install hashicorp/tap/terraform

# View tap contents
$ brew tap-info hashicorp/tap
```

Default taps (pre-installed):
- `homebrew/core`: Main formula repository
- `homebrew/cask`: GUI application installers

Tap mechanics:
```bash
# Taps clone Git repositories
$ ls ~/Library/Homebrew/Library/Taps/
homebrew/
hashicorp/

# You can create your own tap
$ brew tap-new myname/mytap
```

### Casks

Casks install macOS GUI applications:

```bash
# Install application
$ brew install --cask visual-studio-code

# Casks typically:
# 1. Download .dmg or .zip
# 2. Extract .app bundle
# 3. Move to /Applications
```

Cask definition example:

```ruby
cask "visual-studio-code" do
  version "1.85.0"
  sha256 "abc123..."

  url "https://update.code.visualstudio.com/#{version}/darwin-arm64/stable"
  name "Visual Studio Code"
  homepage "https://code.visualstudio.com/"

  app "Visual Studio Code.app"

  zap trash: [
    "~/Library/Application Support/Code",
    "~/.vscode",
  ]
end
```

## Package Lifecycle

### Installation

```bash
$ brew install python@3.11

# What happens:
# 1. Resolve dependencies
# 2. Download bottle (or source)
# 3. Verify checksum
# 4. Extract to Cellar
# 5. Create symlinks
# 6. Run post-install script
```

### Upgrading

```bash
$ brew upgrade python@3.11

# What happens:
# 1. Download new version
# 2. Install to new Cellar directory
# 3. Update symlinks
# 4. Keep old version (until cleanup)
```

### Uninstallation

```bash
$ brew uninstall python@3.11

# What happens:
# 1. Remove symlinks
# 2. Remove Cellar directory
# 3. Optional: remove unused dependencies
```

## Dependencies

### Viewing Dependencies

```bash
# Direct dependencies
$ brew deps wget
gettext
libidn2
openssl@3

# Full dependency tree
$ brew deps --tree wget
wget
├── gettext
├── libidn2
│   ├── gettext
│   └── libunistring
└── openssl@3
    └── ca-certificates

# Why is something installed?
$ brew uses --installed openssl@3
curl
python@3.11
wget
```

### Dependency Types

- **Required**: Must be installed
- **Build**: Only needed during compilation
- **Optional**: Enables additional features
- **Recommended**: Suggested but not required

```ruby
# In formula
depends_on "cmake" => :build          # Build-time only
depends_on "openssl@3"                # Runtime required
depends_on "readline" => :optional    # Optional feature
```

## Services

Homebrew can manage background services:

```bash
# Start a service
$ brew services start postgresql@14

# Stop a service
$ brew services stop postgresql@14

# Restart
$ brew services restart postgresql@14

# List services
$ brew services list
Name          Status  User File
postgresql@14 started user ~/Library/LaunchAgents/homebrew.mxcl.postgresql@14.plist

# Service files are launchd plists
$ cat ~/Library/LaunchAgents/homebrew.mxcl.postgresql@14.plist
```

## Important Files and Directories

```bash
# Homebrew itself
/opt/homebrew/                          # Prefix (Apple Silicon)
/opt/homebrew/bin/brew                  # Main executable
/opt/homebrew/Homebrew/                 # Homebrew Git repository

# Installed packages
/opt/homebrew/Cellar/                   # All installations
/opt/homebrew/opt/                      # Stable symlinks

# Configuration
/opt/homebrew/etc/                      # Config files

# Cache
~/Library/Caches/Homebrew/              # Downloaded files
~/Library/Caches/Homebrew/downloads/    # Bottles and sources

# Logs
~/Library/Logs/Homebrew/                # Build logs

# Taps
$(brew --repository)/Library/Taps/      # Tap repositories
```

## Under the Hood

### Git-Based Design

Homebrew itself is a Git repository:

```bash
# View Homebrew version
$ brew --version
Homebrew 4.2.0

# Update Homebrew (git pull)
$ brew update
==> Updating Homebrew...

# View homebrew commits
$ cd $(brew --repository) && git log --oneline -5
```

Formula repositories are also Git-based:

```bash
# View formula history
$ brew log wget
```

### Keg-Only Packages

Some packages are "keg-only"—installed but not symlinked:

```bash
$ brew info openssl@3
==> openssl@3: stable 3.2.0 (bottled)
...
openssl@3 is keg-only, which means it was not symlinked into /opt/homebrew,
because macOS provides LibreSSL.

# Access keg-only packages
$ /opt/homebrew/opt/openssl@3/bin/openssl version

# Or add to PATH
export PATH="/opt/homebrew/opt/openssl@3/bin:$PATH"
```

Keg-only reasons:
- Conflicts with macOS system software
- Multiple versions available
- Not meant for direct use

### Linking and Unlinking

```bash
# Unlink (remove symlinks but keep installed)
$ brew unlink python@3.11

# Link (create symlinks)
$ brew link python@3.11

# Force link keg-only formula (use carefully)
$ brew link --force openssl@3
```

## Diagnostic Commands

```bash
# Check Homebrew health
$ brew doctor
Your system is ready to brew.

# Show configuration
$ brew config
HOMEBREW_VERSION: 4.2.0
ORIGIN: https://github.com/Homebrew/brew
HEAD: abc1234
Core tap ORIGIN: https://github.com/Homebrew/homebrew-core
...

# Debug information
$ brew --env
HOMEBREW_CC: clang
HOMEBREW_CXX: clang++
...

# List all installed packages
$ brew list

# List with versions
$ brew list --versions

# Show what would be upgraded
$ brew outdated
```

## Summary

Homebrew architecture key concepts:

| Component | Purpose | Location |
|-----------|---------|----------|
| Formula | Package definition | Taps (Git repos) |
| Cellar | Installed packages | `/opt/homebrew/Cellar/` |
| Bottles | Pre-built binaries | GitHub Container Registry |
| Taps | Formula repositories | `~/.../Taps/` |
| Casks | GUI app installers | homebrew/cask tap |
| opt | Stable symlinks | `/opt/homebrew/opt/` |

Understanding this architecture helps you:
- Troubleshoot installation issues
- Manage multiple versions
- Create your own formulae
- Work with keg-only packages
- Understand why things are where they are
