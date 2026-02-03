# Compiling Software from Source

Compiling software from source on macOS follows familiar Unix patterns but with important differences. This chapter covers the practical aspects of building open-source software, common pitfalls, and macOS-specific solutions.

## Basic Build Process

Most Unix software follows the configure-make-install pattern:

```bash
# Download and extract
$ curl -LO https://example.com/software-1.0.tar.gz
$ tar xzf software-1.0.tar.gz
$ cd software-1.0

# Configure, build, install
$ ./configure --prefix=/usr/local
$ make
$ sudo make install
```

On macOS, this often requires adjustments.

## Prerequisites

### Install Command Line Tools

```bash
$ xcode-select --install
```

### Install Common Build Dependencies

```bash
# Essential build tools
$ brew install autoconf automake libtool pkg-config cmake

# Common libraries
$ brew install openssl readline zlib
```

## The Configure Step

### Basic Configure Usage

```bash
# Show available options
$ ./configure --help

# Common options
$ ./configure \
    --prefix=/usr/local \
    --with-ssl=/opt/homebrew/opt/openssl@3 \
    --disable-static \
    --enable-shared
```

### Important Configure Options

| Option | Purpose |
|--------|---------|
| `--prefix=PATH` | Installation directory |
| `--with-PACKAGE=PATH` | Path to dependency |
| `--without-PACKAGE` | Disable optional feature |
| `--enable-FEATURE` | Enable optional feature |
| `--disable-FEATURE` | Disable feature |
| `--build=TRIPLE` | Build system type |
| `--host=TRIPLE` | Target system type |

### macOS-Specific Configure Issues

#### Finding Homebrew Libraries

Configure scripts often can't find Homebrew-installed libraries:

```bash
# Problem
$ ./configure
checking for openssl... no
configure: error: OpenSSL not found

# Solution: Tell configure where to look
$ ./configure \
    --with-openssl=/opt/homebrew/opt/openssl@3 \
    LDFLAGS="-L/opt/homebrew/opt/openssl@3/lib" \
    CPPFLAGS="-I/opt/homebrew/opt/openssl@3/include"
```

#### Using pkg-config

```bash
# Set PKG_CONFIG_PATH for Homebrew packages
$ export PKG_CONFIG_PATH="/opt/homebrew/lib/pkgconfig:$PKG_CONFIG_PATH"

# Verify package is found
$ pkg-config --modversion openssl
3.2.1

# Use in configure
$ ./configure PKG_CONFIG_PATH="/opt/homebrew/lib/pkgconfig"
```

#### Setting Environment Variables

```bash
# Comprehensive approach for Homebrew
$ export LDFLAGS="-L/opt/homebrew/lib"
$ export CPPFLAGS="-I/opt/homebrew/include"
$ export CFLAGS="-I/opt/homebrew/include"
$ export PKG_CONFIG_PATH="/opt/homebrew/lib/pkgconfig"

$ ./configure --prefix=/usr/local
```

### Apple Silicon Specifics

```bash
# Verify architecture
$ uname -m
arm64

# Build for native architecture
$ ./configure --build=aarch64-apple-darwin

# Force x86_64 (via Rosetta)
$ arch -x86_64 ./configure --build=x86_64-apple-darwin
```

## CMake Projects

Many modern projects use CMake instead of autoconf:

```bash
# Basic CMake build
$ mkdir build && cd build
$ cmake ..
$ make
$ sudo make install

# With options
$ cmake .. \
    -DCMAKE_INSTALL_PREFIX=/usr/local \
    -DCMAKE_BUILD_TYPE=Release \
    -DOPENSSL_ROOT_DIR=/opt/homebrew/opt/openssl@3

# Using Ninja (faster)
$ brew install ninja
$ cmake -G Ninja ..
$ ninja
$ sudo ninja install
```

### CMake on macOS

```bash
# Tell CMake about Homebrew
$ cmake .. \
    -DCMAKE_PREFIX_PATH="/opt/homebrew" \
    -DCMAKE_INSTALL_PREFIX="/usr/local"

# Build universal binary
$ cmake .. \
    -DCMAKE_OSX_ARCHITECTURES="arm64;x86_64"

# Target specific macOS version
$ cmake .. \
    -DCMAKE_OSX_DEPLOYMENT_TARGET="12.0"
```

## Meson Projects

```bash
# Install Meson
$ brew install meson ninja

# Basic build
$ meson setup build
$ cd build
$ ninja
$ sudo ninja install

# With options
$ meson setup build \
    --prefix=/usr/local \
    -Doption=value
```

## Common Build Errors and Solutions

### Missing Header Files

```bash
# Error
fatal error: 'openssl/ssl.h' file not found

# Solution: Add include path
$ CPPFLAGS="-I/opt/homebrew/include" ./configure
# or
$ ./configure CPPFLAGS="-I/opt/homebrew/opt/openssl@3/include"
```

### Library Not Found

```bash
# Error
ld: library not found for -lssl

# Solution: Add library path
$ LDFLAGS="-L/opt/homebrew/lib" ./configure
# or
$ ./configure LDFLAGS="-L/opt/homebrew/opt/openssl@3/lib"
```

### SDK Headers Not Found

After macOS upgrade, SDK headers may be missing:

```bash
# Error
fatal error: 'stdio.h' file not found

# Solution: Reinstall Command Line Tools
$ sudo rm -rf /Library/Developer/CommandLineTools
$ xcode-select --install
```

### Incompatible Function Signatures

macOS uses BSD libc, which differs from GNU libc:

```c
// Linux (GNU)
char *strsignal(int sig);

// macOS (BSD)
const char *strsignal(int sig);  // Note: const return
```

Fix: Check for macOS in code or use compatibility shims.

### Missing GNU Tools

Some software expects GNU-specific tool behavior:

```bash
# Error
sed: 1: invalid command code

# Solution: Use GNU sed
$ brew install gnu-sed
$ export PATH="/opt/homebrew/opt/gnu-sed/libexec/gnubin:$PATH"
```

### Clock Functions

```bash
# Error
error: use of undeclared identifier 'CLOCK_MONOTONIC'

# macOS uses different clock APIs
# Fix usually requires code changes or compatibility patches
```

### Undefined Symbols for Architecture

```bash
# Error
Undefined symbols for architecture arm64:
  "_some_function", referenced from:

# Solutions:
# 1. Missing library
$ ./configure LDFLAGS="-L/path/to/lib -lmissing"

# 2. Wrong architecture
$ file libsome.dylib  # Check architecture
$ ./configure --build=arm64-apple-darwin
```

## Working with Patches

### Applying Patches

```bash
# Standard patch
$ patch -p1 < fix.patch

# Dry run (test without applying)
$ patch -p1 --dry-run < fix.patch

# Reverse a patch
$ patch -R -p1 < fix.patch
```

### Common macOS Patches

Many projects have macOS-specific issues. Check:

```bash
# Homebrew formula (may contain patches)
$ brew cat software-name

# MacPorts portfile
$ port cat software-name
```

### Creating Patches

```bash
# Create patch from changes
$ diff -u original.c modified.c > fix.patch

# From git
$ git diff > changes.patch
```

## Build System Specifics

### Make

```bash
# Parallel build (faster)
$ make -j$(sysctl -n hw.ncpu)

# Verbose output
$ make V=1
# or
$ make VERBOSE=1

# Clean build
$ make clean
$ make distclean  # Also removes configure output
```

### Autotools Regeneration

If you modify configure.ac or Makefile.am:

```bash
# Regenerate configure script
$ autoreconf -fi

# May need:
$ brew install autoconf automake libtool
```

## Installation Locations

### Standard Prefixes

| Prefix | Use |
|--------|-----|
| `/usr/local` | Traditional Unix (less common now) |
| `/opt/homebrew` | Homebrew on Apple Silicon |
| `/opt/local` | MacPorts |
| `$HOME/.local` | User-local installation |

### Avoiding System Directories

Never install to:
- `/usr` (System Integrity Protection)
- `/System` (SIP protected)
- `/bin`, `/sbin` (SIP protected)

```bash
# This will fail
$ sudo ./configure --prefix=/usr
$ sudo make install
error: Read-only file system

# Use /usr/local or custom prefix instead
$ ./configure --prefix=/usr/local
```

## Environment Setup for Building

### Comprehensive Build Environment

```bash
# Create a build environment script: build-env.sh

# Homebrew paths
export HOMEBREW_PREFIX="/opt/homebrew"
export PATH="$HOMEBREW_PREFIX/bin:$PATH"

# Compiler flags
export CFLAGS="-I$HOMEBREW_PREFIX/include"
export CXXFLAGS="-I$HOMEBREW_PREFIX/include"
export CPPFLAGS="-I$HOMEBREW_PREFIX/include"
export LDFLAGS="-L$HOMEBREW_PREFIX/lib"

# pkg-config
export PKG_CONFIG_PATH="$HOMEBREW_PREFIX/lib/pkgconfig"

# Prefer Homebrew's versions
export PATH="$HOMEBREW_PREFIX/opt/gnu-sed/libexec/gnubin:$PATH"
export PATH="$HOMEBREW_PREFIX/opt/grep/libexec/gnubin:$PATH"
export PATH="$HOMEBREW_PREFIX/opt/coreutils/libexec/gnubin:$PATH"
```

### Using the Environment

```bash
$ source build-env.sh
$ ./configure --prefix=/usr/local
$ make -j$(sysctl -n hw.ncpu)
```

## Example: Compiling a Real Package

Let's walk through compiling tmux from source:

```bash
# Install dependencies
$ brew install libevent ncurses

# Download tmux
$ curl -LO https://github.com/tmux/tmux/releases/download/3.4/tmux-3.4.tar.gz
$ tar xzf tmux-3.4.tar.gz
$ cd tmux-3.4

# Configure with Homebrew dependencies
$ ./configure \
    --prefix=/usr/local \
    CPPFLAGS="-I/opt/homebrew/include -I/opt/homebrew/include/ncurses" \
    LDFLAGS="-L/opt/homebrew/lib"

# Build
$ make -j$(sysctl -n hw.ncpu)

# Test (optional)
$ ./tmux -V
tmux 3.4

# Install
$ sudo make install

# Verify
$ /usr/local/bin/tmux -V
tmux 3.4
```

## Static vs Dynamic Linking

### Default: Dynamic Linking

```bash
$ ./configure
$ make
$ otool -L myprogram
myprogram:
    /usr/lib/libSystem.B.dylib
    /opt/homebrew/opt/openssl@3/lib/libssl.3.dylib
    /opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib
```

### Static Linking (where possible)

```bash
# Request static linking
$ ./configure --enable-static --disable-shared

# Or link specific libraries statically
$ LDFLAGS="-static-libgcc" ./configure
```

Note: Full static linking isn't possible on macOS due to system library requirements.

## Troubleshooting Checklist

When a build fails:

1. **Check prerequisites**
   ```bash
   $ xcode-select -p
   $ brew doctor
   ```

2. **Verify dependencies are installed**
   ```bash
   $ pkg-config --exists libname && echo "Found"
   ```

3. **Check config.log**
   ```bash
   $ tail -100 config.log  # Shows why configure failed
   ```

4. **Search for macOS-specific issues**
   - Check project's issue tracker
   - Search Homebrew formula
   - Check MacPorts portfile

5. **Try Homebrew's formula**
   ```bash
   $ brew install --verbose software-name
   # Observe how Homebrew builds it
   ```

6. **Check architecture**
   ```bash
   $ uname -m
   $ file /opt/homebrew/lib/libdependency.dylib
   ```

## Summary

Building from source on macOS:

| Aspect | Consideration |
|--------|---------------|
| Compiler | Clang (not GCC) |
| Headers | May need explicit paths |
| Libraries | Often in `/opt/homebrew` |
| pkg-config | Set `PKG_CONFIG_PATH` |
| GNU tools | Install via Homebrew if needed |
| Architecture | arm64 or x86_64 |
| Installation | `/usr/local` or custom prefix |
| SIP | Can't write to system directories |

The key to successful compilation on macOS is understanding where dependencies are installed and how to tell the build system about them.
