# Library Paths and Linking

macOS handles dynamic libraries differently from Linux. Understanding these differences is essential for compiling software, debugging linking issues, and distributing applications.

## Dynamic Libraries: dylib vs so

| Platform | Extension | Format |
|----------|-----------|--------|
| macOS | `.dylib` | Mach-O |
| Linux | `.so` | ELF |
| Windows | `.dll` | PE |

### Library Naming Conventions

```bash
# macOS naming
libfoo.dylib                    # Current version symlink
libfoo.1.dylib                  # Major version
libfoo.1.2.3.dylib              # Full version

# Linux naming (for comparison)
libfoo.so                       # Symlink
libfoo.so.1                     # Major version
libfoo.so.1.2.3                 # Full version

# macOS also supports .so for compatibility
# Some ports create .so files, but native libraries use .dylib
```

### Static Libraries

Static libraries use the same format on both platforms:

```bash
# Archive format
libfoo.a

# Create static library
$ ar rcs libfoo.a foo.o bar.o

# View contents
$ ar -t libfoo.a
foo.o
bar.o
```

## Library Search Paths

### Default Search Order

The dynamic linker (`dyld`) searches in this order:

1. `DYLD_LIBRARY_PATH` (if set and SIP disabled)
2. `LD_LIBRARY_PATH` (fallback, if set)
3. Paths embedded in the binary (rpath, @executable_path, etc.)
4. `/usr/lib`
5. `/usr/local/lib`

### Viewing Search Paths

```bash
# Show dyld search paths
$ dyld_info -search_paths /usr/bin/some_binary

# Or check man page for full algorithm
$ man dyld
```

### DYLD_LIBRARY_PATH

Similar to Linux's `LD_LIBRARY_PATH` but with restrictions:

```bash
# Set library path
$ export DYLD_LIBRARY_PATH=/opt/homebrew/lib:/usr/local/lib

# Run program with modified path
$ DYLD_LIBRARY_PATH=/custom/lib ./myprogram
```

**Important**: System Integrity Protection (SIP) clears `DYLD_*` variables for system binaries and when calling protected executables.

```bash
# This won't work for system binaries
$ DYLD_LIBRARY_PATH=/custom sudo somecommand  # Variable cleared
```

### DYLD_FALLBACK_LIBRARY_PATH

Used when library isn't found in standard locations:

```bash
$ export DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/lib:$HOME/lib:/usr/local/lib:/usr/lib
```

## The otool Command

`otool` is the macOS equivalent of `ldd` and `readelf`:

### View Linked Libraries

```bash
$ otool -L /usr/bin/python3
/usr/bin/python3:
        /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation
        /usr/lib/libncurses.5.4.dylib
        /usr/lib/libSystem.B.dylib

# For a Homebrew binary
$ otool -L /opt/homebrew/bin/wget
/opt/homebrew/bin/wget:
        /opt/homebrew/opt/openssl@3/lib/libssl.3.dylib
        /opt/homebrew/opt/openssl@3/lib/libcrypto.3.dylib
        /opt/homebrew/opt/libidn2/lib/libidn2.0.dylib
        /usr/lib/libz.1.dylib
        /usr/lib/libiconv.2.dylib
        /usr/lib/libSystem.B.dylib
```

### View Load Commands

```bash
# All load commands
$ otool -l /opt/homebrew/bin/wget | head -50

# Just the library paths
$ otool -l /opt/homebrew/bin/wget | grep -A2 LC_LOAD_DYLIB
          cmd LC_LOAD_DYLIB
      cmdsize 64
         name /opt/homebrew/opt/openssl@3/lib/libssl.3.dylib
```

### View Symbols

```bash
# External symbols
$ nm /opt/homebrew/lib/libssl.dylib | head
0000000000001234 T _SSL_connect
0000000000001456 T _SSL_read
...

# Undefined symbols (needed from other libraries)
$ nm -u /opt/homebrew/bin/wget
                 U _SSL_connect
                 U _SSL_read
```

## Install Names

Every macOS dynamic library has an "install name"---the path recorded inside the library itself:

```bash
# View install name
$ otool -D /opt/homebrew/lib/libssl.dylib
/opt/homebrew/lib/libssl.dylib:
/opt/homebrew/opt/openssl@3/lib/libssl.3.dylib

# The install name is embedded in the library
# When you link against it, this path is recorded in your binary
```

### How Install Names Work

1. Library is built with an install name
2. When you link against the library, the install name is copied to your binary
3. At runtime, dyld uses this path to find the library

```bash
# Example: Building and linking

# Library specifies its install name during build
$ clang -dynamiclib -install_name /usr/local/lib/libfoo.dylib \
    -o libfoo.dylib foo.c

# Program links against library
$ clang -o myprogram main.c -L. -lfoo

# The install name is recorded
$ otool -L myprogram
myprogram:
        /usr/local/lib/libfoo.dylib  # Install name, not build path!
        /usr/lib/libSystem.B.dylib
```

## install_name_tool

Modify install names in binaries and libraries:

### Change Install Name

```bash
# Change a library's own install name
$ install_name_tool -id /new/path/libfoo.dylib libfoo.dylib

# Verify
$ otool -D libfoo.dylib
libfoo.dylib:
/new/path/libfoo.dylib
```

### Change Dependency Path

```bash
# Change where a binary looks for a library
$ install_name_tool -change \
    /old/path/libfoo.dylib \
    /new/path/libfoo.dylib \
    myprogram

# Verify
$ otool -L myprogram
```

### Add rpath

```bash
# Add runtime search path
$ install_name_tool -add_rpath /opt/homebrew/lib myprogram

# Delete rpath
$ install_name_tool -delete_rpath /old/path myprogram

# View rpaths
$ otool -l myprogram | grep -A2 LC_RPATH
          cmd LC_RPATH
      cmdsize 32
         path /opt/homebrew/lib
```

## @executable_path, @loader_path, @rpath

Special path prefixes for relocatable binaries:

### @executable_path

Relative to the main executable:

```bash
# Install name
@executable_path/../lib/libfoo.dylib

# If executable is /Applications/MyApp.app/Contents/MacOS/MyApp
# Library resolves to /Applications/MyApp.app/Contents/lib/libfoo.dylib
```

### @loader_path

Relative to the binary containing the reference (useful for plugin systems):

```bash
# Install name
@loader_path/../Frameworks/libfoo.dylib

# Resolves relative to whatever binary loads this library
# Different from @executable_path for plugins/bundles
```

### @rpath

Resolved using the binary's rpath list:

```bash
# Install name
@rpath/libfoo.dylib

# dyld searches each rpath entry:
# If rpath contains /opt/homebrew/lib, tries /opt/homebrew/lib/libfoo.dylib
```

### Using @rpath

```bash
# Build library with @rpath install name
$ clang -dynamiclib -install_name @rpath/libfoo.dylib \
    -o libfoo.dylib foo.c

# Build executable with rpath
$ clang -o myprogram main.c -L. -lfoo \
    -Wl,-rpath,/usr/local/lib \
    -Wl,-rpath,/opt/homebrew/lib

# Multiple rpaths are searched in order
$ otool -l myprogram | grep -A2 LC_RPATH
          cmd LC_RPATH
      cmdsize 32
         path /usr/local/lib
--
          cmd LC_RPATH
      cmdsize 40
         path /opt/homebrew/lib
```

## dyld Debugging

### DYLD_PRINT_LIBRARIES

See which libraries are loaded:

```bash
$ DYLD_PRINT_LIBRARIES=1 ./myprogram
dyld[12345]: <1A23B456-...> /usr/lib/libSystem.B.dylib
dyld[12345]: <2B34C567-...> /opt/homebrew/lib/libssl.3.dylib
...
```

### DYLD_PRINT_SEARCHING

See the search process:

```bash
$ DYLD_PRINT_SEARCHING=1 ./myprogram
dyld[12345]: find path "/opt/homebrew/lib/libfoo.dylib"
dyld[12345]:   found: "/opt/homebrew/lib/libfoo.dylib"
```

Note: These only work with SIP disabled or for non-system binaries.

### dyld_info

Modern tool for analyzing binaries:

```bash
# Show all dyld info
$ dyld_info -all /opt/homebrew/bin/wget

# Just dependencies
$ dyld_info -dependents /opt/homebrew/bin/wget

# Exports
$ dyld_info -exports /opt/homebrew/lib/libssl.dylib
```

## Common Linking Problems

### Library Not Found

```bash
$ ./myprogram
dyld[12345]: Library not loaded: /usr/local/lib/libfoo.dylib
  Referenced from: <UUID> /path/to/myprogram
  Reason: tried: '/usr/local/lib/libfoo.dylib' (no such file)
```

Solutions:

```bash
# 1. Install the library
$ brew install foo

# 2. Create symlink
$ sudo ln -s /opt/homebrew/lib/libfoo.dylib /usr/local/lib/

# 3. Fix the binary's path
$ install_name_tool -change \
    /usr/local/lib/libfoo.dylib \
    /opt/homebrew/lib/libfoo.dylib \
    myprogram

# 4. Add rpath
$ install_name_tool -add_rpath /opt/homebrew/lib myprogram
```

### Wrong Architecture

```bash
$ ./myprogram
dyld[12345]: Library not loaded: libfoo.dylib
  Reason: tried: 'libfoo.dylib' (mach-o file, but is an incompatible architecture)
```

Check architectures:

```bash
$ file myprogram
myprogram: Mach-O 64-bit executable arm64

$ file /path/to/libfoo.dylib
libfoo.dylib: Mach-O 64-bit dynamically linked shared library x86_64

# Need to rebuild for matching architecture
```

### Symbol Not Found

```bash
$ ./myprogram
dyld[12345]: Symbol not found: _some_function
  Referenced from: <UUID> /path/to/myprogram
  Expected in: <UUID> /path/to/libfoo.dylib
```

Library version mismatch---library doesn't have expected symbol:

```bash
# Check what symbols library exports
$ nm /path/to/libfoo.dylib | grep some_function

# May need newer library version
$ brew upgrade foo
```

## Creating Dynamic Libraries

### Basic Dynamic Library

```bash
# Compile to object file
$ clang -c -fPIC foo.c -o foo.o

# Create dynamic library
$ clang -dynamiclib \
    -install_name @rpath/libfoo.dylib \
    -o libfoo.dylib \
    foo.o

# With version info
$ clang -dynamiclib \
    -install_name @rpath/libfoo.1.dylib \
    -current_version 1.2.3 \
    -compatibility_version 1.0.0 \
    -o libfoo.1.2.3.dylib \
    foo.o

# Create symlinks
$ ln -s libfoo.1.2.3.dylib libfoo.1.dylib
$ ln -s libfoo.1.dylib libfoo.dylib
```

### Version Numbers

```bash
# View version info
$ otool -L libfoo.dylib
libfoo.dylib:
        @rpath/libfoo.1.dylib (compatibility version 1.0.0, current version 1.2.3)
```

- **Current version**: Actual version of the library
- **Compatibility version**: Minimum version needed by binaries linked against it

## TBD Files (Text-Based Stubs)

Modern macOS uses text-based stub files instead of actual library binaries in SDKs:

```bash
$ cat /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/lib/libSystem.tbd
--- !tapi-tbd
tbd-version:     4
targets:         [ x86_64-macos, arm64-macos ]
install-name:    '/usr/lib/libSystem.B.dylib'
exports:
  - targets:     [ x86_64-macos, arm64-macos ]
    symbols:     [ __exit, _abort, ... ]
```

TBD files:
- Contain metadata about library (exports, version)
- Much smaller than full dylib
- Used at link time, not runtime
- Runtime uses actual dylib

## Linking at Build Time

### Compiler Flags

```bash
# Link against library
$ clang -o myprogram main.c -lfoo

# Specify library path
$ clang -o myprogram main.c -L/opt/homebrew/lib -lfoo

# Link against framework
$ clang -o myprogram main.c -framework CoreFoundation

# Specify framework path
$ clang -o myprogram main.c -F/path/to/frameworks -framework MyFramework

# Add rpath at link time
$ clang -o myprogram main.c -lfoo -Wl,-rpath,/opt/homebrew/lib
```

### Static vs Dynamic

```bash
# Force static linking of specific library
$ clang -o myprogram main.c /path/to/libfoo.a

# Let linker choose (prefers dynamic)
$ clang -o myprogram main.c -L/path/to/lib -lfoo

# Force all libraries static (not fully supported on macOS)
# macOS always dynamically links system libraries
```

## Summary

| Tool | Purpose |
|------|---------|
| `otool -L` | View linked libraries |
| `otool -D` | View install name |
| `otool -l` | View load commands |
| `nm` | View symbols |
| `install_name_tool` | Modify paths |
| `dyld_info` | Modern binary analysis |
| `file` | Check architecture |

| Path Prefix | Meaning |
|-------------|---------|
| `@executable_path` | Relative to main executable |
| `@loader_path` | Relative to loading binary |
| `@rpath` | Search rpath list |

Key differences from Linux:

| Aspect | Linux | macOS |
|--------|-------|-------|
| Extension | `.so` | `.dylib` |
| Path variable | `LD_LIBRARY_PATH` | `DYLD_LIBRARY_PATH` |
| View dependencies | `ldd` | `otool -L` |
| Modify paths | `patchelf` | `install_name_tool` |
| Embedded path | RPATH/RUNPATH | Install name |

Understanding install names and rpaths is crucial for creating portable macOS applications and debugging library loading issues.
