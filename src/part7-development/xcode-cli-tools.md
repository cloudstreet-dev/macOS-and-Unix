# Xcode Command Line Tools

The Xcode Command Line Tools package provides essential development utilities for macOS. You don't need the full Xcode IDE to compile code, run scripts, or use Unix development tools. The standalone Command Line Tools package gives you compilers, linkers, headers, and utilities in a much smaller download.

## What's Included

The Command Line Tools package contains:

### Compilers and Build Tools

| Tool | Description |
|------|-------------|
| `clang` | C, C++, Objective-C compiler |
| `clang++` | C++ compiler (same as `clang` with C++ mode) |
| `swift` | Swift compiler |
| `ld` | The linker |
| `as` | Assembler |
| `ar` | Archive tool (static libraries) |
| `libtool` | Library creation tool |
| `make` | Build automation |
| `cmake` | Cross-platform build system (recent versions) |

### Development Utilities

| Tool | Description |
|------|-------------|
| `git` | Version control |
| `svn` | Subversion (legacy) |
| `lldb` | Debugger |
| `nm` | Symbol table viewer |
| `otool` | Object file viewer |
| `lipo` | Universal binary tool |
| `codesign` | Code signing |
| `install_name_tool` | Library path modifier |
| `dsymutil` | Debug symbol utility |
| `dwarfdump` | DWARF debug info viewer |
| `strip` | Remove symbols from binaries |
| `size` | Display binary section sizes |

### Headers and SDKs

The tools include macOS SDK headers necessary for compiling programs:

```bash
# SDK location
$ xcrun --show-sdk-path
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk

# SDK version
$ xcrun --show-sdk-version
14.4
```

### Other Utilities

| Tool | Description |
|------|-------------|
| `xcrun` | Run tools from active developer directory |
| `xcode-select` | Manage developer tool paths |
| `xcodebuild` | Build Xcode projects (limited without full Xcode) |
| `instruments` | Command-line profiling (requires Xcode) |

## Installing Command Line Tools

### Interactive Installation

The simplest method triggers an installation dialog:

```bash
$ xcode-select --install
```

This opens a dialog offering to install the Command Line Tools. Click "Install" and wait for the download (approximately 1-2 GB).

### Trigger via Missing Tool

Running any development command without installed tools triggers the prompt:

```bash
$ git --version
xcode-select: note: no developer tools were found at '/Applications/Xcode.app'
# Installation dialog appears
```

### Download from Apple Developer

For specific versions or offline installation:

1. Visit [developer.apple.com/download/more/](https://developer.apple.com/download/more/)
2. Sign in with Apple ID (free account works)
3. Search for "Command Line Tools"
4. Download the .dmg for your macOS version
5. Install the package

### Verify Installation

```bash
$ xcode-select -p
/Library/Developer/CommandLineTools

$ clang --version
Apple clang version 15.0.0 (clang-1500.3.9.4)
Target: arm64-apple-darwin23.4.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin

$ git --version
git version 2.39.3 (Apple Git-146)
```

## Xcode vs Command Line Tools

You can develop on macOS with either:

1. **Command Line Tools only** (standalone)
2. **Full Xcode** (includes Command Line Tools)

### Command Line Tools Only

Pros:
- Smaller download (~2 GB vs ~12+ GB)
- No IDE required
- Sufficient for most Unix development
- Faster installation

Cons:
- No iOS/watchOS/tvOS development
- No Interface Builder
- Limited Instruments profiling
- No Simulator
- No Xcode IDE features

### Full Xcode

Required for:
- iOS, iPadOS, watchOS, tvOS, visionOS development
- SwiftUI previews
- Full Instruments profiling
- Metal shader development
- App Store submission
- Simulator usage

### Installation Locations

```bash
# Command Line Tools (standalone)
/Library/Developer/CommandLineTools/
├── Library/
│   └── Frameworks/         # Development frameworks
├── SDKs/
│   └── MacOSX.sdk         # macOS SDK
└── usr/
    ├── bin/               # Tools (clang, git, etc.)
    ├── include/           # C headers
    └── lib/               # Libraries

# Xcode (full)
/Applications/Xcode.app/
└── Contents/
    └── Developer/
        ├── Platforms/     # iOS, macOS, etc.
        ├── SDKs/
        ├── Toolchains/
        └── usr/
            └── bin/       # Tools
```

## Switching Between Xcode and Command Line Tools

The `xcode-select` command controls which developer directory is active:

### Check Current Selection

```bash
$ xcode-select -p
/Applications/Xcode.app/Contents/Developer
# or
/Library/Developer/CommandLineTools
```

### Switch to Xcode

```bash
$ sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

### Switch to Command Line Tools

```bash
$ sudo xcode-select -s /Library/Developer/CommandLineTools
```

### Reset to Default

```bash
$ sudo xcode-select -r
```

### Why Switch?

Common reasons to switch:

| Scenario | Use |
|----------|-----|
| Building iOS apps | Xcode |
| Homebrew formula development | Command Line Tools (often) |
| Compiling Unix software | Either works |
| Using different SDK version | Switch as needed |
| CI/CD server | Command Line Tools (smaller) |

## The xcrun Command

`xcrun` executes tools from the active developer directory, ensuring you use the correct version:

```bash
# Run clang from current developer directory
$ xcrun clang -v

# Find tool location
$ xcrun -f clang
/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang

# Show SDK path
$ xcrun --show-sdk-path
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX14.4.sdk

# Run with specific SDK
$ xcrun --sdk macosx clang -v

# List available SDKs
$ xcodebuild -showsdks
macOS SDKs:
        macOS 14.4                      -sdk macosx14.4

iOS SDKs:
        iOS 17.4                        -sdk iphoneos17.4
...
```

### Using xcrun in Scripts

For portable scripts that work with either Xcode or Command Line Tools:

```bash
#!/bin/bash
# Use xcrun to find compiler
CC=$(xcrun -f clang)
SDK=$(xcrun --show-sdk-path)

$CC -isysroot "$SDK" -o myprogram myprogram.c
```

## SDK Management

### Listing Installed SDKs

```bash
# With Xcode
$ ls /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/
MacOSX.sdk  MacOSX14.4.sdk

# With Command Line Tools
$ ls /Library/Developer/CommandLineTools/SDKs/
MacOSX.sdk  MacOSX14.4.sdk
```

### SDK Contents

```bash
$ ls $(xcrun --show-sdk-path)
Entitlements.plist  SDKSettings.json  System/
Library/            SDKSettings.plist usr/
```

The SDK contains:
- **System/**: System framework stubs
- **usr/**: Headers and libraries
- **Library/**: Frameworks

### Targeting Specific SDK Versions

```bash
# Compile against specific SDK
$ clang -isysroot $(xcrun --show-sdk-path --sdk macosx14.4) program.c

# Set deployment target (minimum OS version)
$ clang -mmacosx-version-min=12.0 program.c
```

## Updating Command Line Tools

### Check for Updates

```bash
$ softwareupdate --list
Software Update found the following new or updated software:
* Command Line Tools for Xcode-15.3
```

### Install Updates

```bash
$ softwareupdate --install "Command Line Tools for Xcode-15.3"
```

### Reinstall After macOS Upgrade

After major macOS upgrades, you may need to reinstall:

```bash
# Remove existing tools
$ sudo rm -rf /Library/Developer/CommandLineTools

# Reinstall
$ xcode-select --install
```

## Troubleshooting

### "No Developer Tools Found"

```bash
$ git status
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools)
```

Solution:
```bash
$ xcode-select --install
```

### "Agreeing to Xcode License"

```bash
$ clang
Agreeing to the Xcode/iOS license requires admin privileges, please run "sudo xcodebuild -license"
```

Solution:
```bash
$ sudo xcodebuild -license accept
```

### Wrong SDK or Tools Version

```bash
# Check what's active
$ xcode-select -p
$ xcrun --show-sdk-version

# Switch if needed
$ sudo xcode-select -s /path/to/correct/developer/directory
```

### Headers Not Found After macOS Update

```bash
$ clang program.c
fatal error: 'stdio.h' file not found
```

Reinstall Command Line Tools:
```bash
$ sudo rm -rf /Library/Developer/CommandLineTools
$ xcode-select --install
```

### Multiple Xcode Versions

You can have multiple Xcode versions installed:

```bash
# List installations
$ ls /Applications/ | grep Xcode
Xcode.app
Xcode-15.2.app

# Switch between them
$ sudo xcode-select -s /Applications/Xcode-15.2.app/Contents/Developer
```

## Command Line Tools Components

### Examining Package Contents

```bash
# List package contents
$ pkgutil --files com.apple.pkg.CLTools_Executables | head -20
Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework
Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework/Headers
Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework/Python3
...
```

### Package Receipts

```bash
# List installed developer packages
$ pkgutil --pkgs | grep -i cltools
com.apple.pkg.CLTools_Executables
com.apple.pkg.CLTools_SDK_macOS14
com.apple.pkg.CLTools_SwiftBackDeploy
com.apple.pkg.CLTools_macOS_SDK
```

## Summary

| Task | Command |
|------|---------|
| Install CLI tools | `xcode-select --install` |
| Check installation path | `xcode-select -p` |
| Switch to Xcode | `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` |
| Switch to CLI tools | `sudo xcode-select -s /Library/Developer/CommandLineTools` |
| Reset to default | `sudo xcode-select -r` |
| Find tool path | `xcrun -f <tool>` |
| Get SDK path | `xcrun --show-sdk-path` |
| Run tool with SDK | `xcrun --sdk macosx <command>` |

The Command Line Tools package provides everything needed for Unix-style development on macOS without the full Xcode installation. For most developers who don't need iOS development or the Xcode IDE, it's the right choice.
