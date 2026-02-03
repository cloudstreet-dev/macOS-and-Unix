# Frameworks vs Unix Libraries

macOS introduces "frameworks"---a packaging concept that bundles libraries, headers, and resources into a single directory structure. Understanding frameworks is essential for macOS development, as they're used throughout the system and differ significantly from traditional Unix shared libraries.

## What Is a Framework?

A framework is a bundle (directory with a specific structure) containing:

- Dynamic library (the actual code)
- Header files
- Resources (images, strings, etc.)
- Version management
- Metadata

```
CoreFoundation.framework/
├── CoreFoundation         → Versions/Current/CoreFoundation
├── Headers                → Versions/Current/Headers/
├── Resources              → Versions/Current/Resources/
└── Versions/
    ├── A/
    │   ├── CoreFoundation      # The actual dylib
    │   ├── Headers/
    │   │   ├── CoreFoundation.h
    │   │   └── ...
    │   └── Resources/
    │       ├── Info.plist
    │       └── ...
    └── Current            → A/
```

## Frameworks vs Traditional Libraries

| Aspect | Unix Library | macOS Framework |
|--------|--------------|-----------------|
| Structure | Single file + separate headers | Bundle directory |
| Headers | `/usr/include` or `/usr/local/include` | Inside framework |
| Versioning | Symlinks (libfoo.so.1) | Versions/ directory |
| Resources | N/A | Included |
| Self-contained | No | Yes |
| Discoverability | pkg-config, manual | Built-in |

## Framework Locations

### System Frameworks

```bash
$ ls /System/Library/Frameworks/
Accelerate.framework/
AppKit.framework/
CoreFoundation.framework/
Foundation.framework/
Security.framework/
...
```

These are protected by System Integrity Protection.

### Local Frameworks

```bash
# System-wide third-party frameworks
$ ls /Library/Frameworks/

# User frameworks
$ ls ~/Library/Frameworks/
```

### SDK Frameworks

```bash
$ ls $(xcrun --show-sdk-path)/System/Library/Frameworks/
```

## Examining Frameworks

### Framework Structure

```bash
$ ls -la /System/Library/Frameworks/CoreFoundation.framework/
total 0
lrwxr-xr-x  CoreFoundation -> Versions/Current/CoreFoundation
lrwxr-xr-x  Headers -> Versions/Current/Headers
lrwxr-xr-x  Resources -> Versions/Current/Resources
drwxr-xr-x  Versions/

$ ls /System/Library/Frameworks/CoreFoundation.framework/Versions/
A/
Current    -> A/
```

### Framework Binary

```bash
# The binary is inside
$ file /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation: Mach-O universal binary...

# View linked libraries
$ otool -L /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
```

### Framework Headers

```bash
$ ls /System/Library/Frameworks/CoreFoundation.framework/Headers/
CFArray.h
CFBase.h
CFBundle.h
CoreFoundation.h
...
```

## Using Frameworks

### Compiling with Frameworks

```bash
# Link against framework
$ clang -framework CoreFoundation main.c -o myprogram

# Multiple frameworks
$ clang -framework CoreFoundation -framework Security main.c -o myprogram

# Framework search path
$ clang -F/Library/Frameworks -framework MyFramework main.c
```

### Include Headers

```c
// Import all framework headers
#include <CoreFoundation/CoreFoundation.h>

// Or specific header
#include <CoreFoundation/CFString.h>

// In Objective-C
#import <Foundation/Foundation.h>

// In Swift
import Foundation
```

### Finding Framework Headers

```bash
# System frameworks (via SDK)
$ ls $(xcrun --show-sdk-path)/System/Library/Frameworks/CoreFoundation.framework/Headers/

# Include path for compiler
$ clang -F$(xcrun --show-sdk-path)/System/Library/Frameworks ...
```

## Linking Comparison

### Traditional Unix Library

```bash
# Compile
$ clang -c main.c -I/usr/local/include

# Link
$ clang -o myprogram main.o -L/usr/local/lib -lfoo

# Runtime needs:
# - Library in search path
# - Or DYLD_LIBRARY_PATH set
```

### Framework

```bash
# Compile and link
$ clang -framework CoreFoundation main.c -o myprogram

# Runtime:
# - Framework found automatically in standard locations
# - Path embedded in binary
```

### Examining Linked Frameworks

```bash
$ otool -L myprogram
myprogram:
        /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation
        /usr/lib/libSystem.B.dylib
```

## Creating Your Own Framework

### Framework Structure

```bash
# Create framework structure
$ mkdir -p MyFramework.framework/Versions/A/{Headers,Resources}
$ cd MyFramework.framework/Versions

# Create symlinks
$ ln -s A Current
$ cd ..
$ ln -s Versions/Current/Headers Headers
$ ln -s Versions/Current/Resources Resources
$ ln -s Versions/Current/MyFramework MyFramework
```

### Build the Library

```bash
# Compile source
$ clang -c -fPIC myframework.c -o myframework.o

# Create dynamic library
$ clang -dynamiclib \
    -install_name @rpath/MyFramework.framework/Versions/A/MyFramework \
    -o MyFramework.framework/Versions/A/MyFramework \
    myframework.o

# Copy headers
$ cp myframework.h MyFramework.framework/Versions/A/Headers/
```

### Info.plist

Create `MyFramework.framework/Versions/A/Resources/Info.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>
    <string>com.example.MyFramework</string>
    <key>CFBundleName</key>
    <string>MyFramework</string>
    <key>CFBundleVersion</key>
    <string>1.0</string>
    <key>CFBundleShortVersionString</key>
    <string>1.0</string>
    <key>CFBundlePackageType</key>
    <string>FMWK</string>
</dict>
</plist>
```

### Using Your Framework

```bash
# From parent directory of MyFramework.framework
$ clang -F. -framework MyFramework main.c -o myprogram -Wl,-rpath,@executable_path
```

## Umbrella Frameworks

Some Apple frameworks are "umbrella frameworks" that re-export other frameworks:

```c
// CoreServices includes many sub-frameworks
#include <CoreServices/CoreServices.h>
// This gives you FSEvents, LaunchServices, etc.
```

```bash
$ ls /System/Library/Frameworks/CoreServices.framework/Frameworks/
FSEvents.framework/
LaunchServices.framework/
Metadata.framework/
...
```

## Framework vs dylib: When to Use Which

### Use Frameworks When:

- Distributing macOS/iOS apps
- Packaging headers with library
- Including resources (images, plists)
- Building Apple platform applications
- Need versioning support

### Use dylib When:

- Porting Unix software
- Maximum compatibility with Unix build systems
- Simple library without resources
- Cross-platform code

## XCFrameworks

Modern Apple development uses XCFrameworks for multi-platform support:

```
MyLibrary.xcframework/
├── ios-arm64/
│   └── MyLibrary.framework/
├── ios-arm64_x86_64-simulator/
│   └── MyLibrary.framework/
└── macos-arm64_x86_64/
    └── MyLibrary.framework/
```

XCFrameworks bundle frameworks for multiple platforms and architectures.

### Creating XCFrameworks

```bash
# From multiple framework builds
$ xcodebuild -create-xcframework \
    -framework build/ios/MyFramework.framework \
    -framework build/ios-simulator/MyFramework.framework \
    -framework build/macos/MyFramework.framework \
    -output MyFramework.xcframework
```

## System Framework Details

### Core Frameworks

| Framework | Purpose |
|-----------|---------|
| CoreFoundation | C-level foundation (strings, collections) |
| Foundation | Objective-C foundation (NSObject, etc.) |
| AppKit | macOS GUI |
| UIKit | iOS/iPadOS GUI |
| Security | Cryptography, keychain |
| CoreGraphics | 2D graphics |
| CoreData | Object persistence |
| CoreML | Machine learning |

### Finding Framework Documentation

```bash
# Headers are the documentation
$ ls /System/Library/Frameworks/CoreFoundation.framework/Headers/

# Or use Apple's documentation
$ open "https://developer.apple.com/documentation/corefoundation"
```

## Private Frameworks

Apple has private frameworks not meant for public use:

```bash
$ ls /System/Library/PrivateFrameworks/
```

These are:
- Not documented
- Subject to change without notice
- Rejected from App Store
- Sometimes useful for system utilities

## Weak Linking

Link against frameworks that may not exist at runtime:

```bash
$ clang -weak_framework NewFramework main.c -o myprogram
```

In code:
```c
if (@available(macOS 14.0, *)) {
    // Use new framework features
} else {
    // Fallback
}
```

## Framework Search Order

When linking with `-framework`:

1. `-F` specified paths
2. `FRAMEWORK_SEARCH_PATHS` in Xcode
3. System framework directories:
   - `$(SDKROOT)/System/Library/Frameworks`
   - `/System/Library/Frameworks`
   - `/Library/Frameworks`

## Common Issues

### Framework Not Found

```bash
$ clang -framework NonExistent main.c
ld: framework not found NonExistent
```

Solution:
```bash
# Check if framework exists
$ find /System/Library/Frameworks -name "*.framework" | grep -i name

# Add search path
$ clang -F/path/to/frameworks -framework MyFramework main.c
```

### Header Not Found

```bash
$ clang main.c
main.c:1:10: fatal error: 'MyFramework/MyFramework.h' file not found
```

Solution:
```bash
# Add framework path (also adds headers)
$ clang -F/path/to/frameworks -framework MyFramework main.c
```

### Old SDK/Missing Symbols

```bash
Undefined symbols: "_NewFunction"
```

Check your SDK version:
```bash
$ xcrun --show-sdk-version

# Use newer SDK
$ clang -isysroot $(xcrun --show-sdk-path --sdk macosx14.0) ...
```

## Summary

| Aspect | Framework | dylib |
|--------|-----------|-------|
| Structure | Bundle (directory) | Single file |
| Headers | Included | Separate |
| Resources | Supported | N/A |
| Versioning | Built-in | Symlinks |
| Compile flag | `-framework Name` | `-lname` |
| Search path | `-F/path` | `-L/path` |
| Apple APIs | Required | Possible |

Frameworks are central to macOS development. While Unix libraries work fine for portable software, frameworks provide a more integrated experience for macOS-native applications.
