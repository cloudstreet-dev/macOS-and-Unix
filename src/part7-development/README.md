# Development on macOS

macOS is a powerful development platform that combines the familiarity of Unix with Apple's unique tooling and conventions. For developers coming from Linux or other Unix systems, macOS presents a familiar environment with important differences that affect how you compile code, link libraries, debug programs, and build for multiple architectures.

This part covers the practical aspects of developing software on macOS, from installing the toolchain to profiling production code.

## The macOS Development Stack

The development environment on macOS consists of several layers:

```
┌─────────────────────────────────────────┐
│     Your Application / Project          │
├─────────────────────────────────────────┤
│  Frameworks (Cocoa, Foundation, etc.)   │
├─────────────────────────────────────────┤
│  Libraries (dylib, tbd, static)         │
├─────────────────────────────────────────┤
│  Toolchain (Clang/LLVM, linker, tools)  │
├─────────────────────────────────────────┤
│  Xcode Command Line Tools / Xcode       │
├─────────────────────────────────────────┤
│  Darwin / macOS                         │
└─────────────────────────────────────────┘
```

## Key Differences from Linux Development

| Aspect | Linux | macOS |
|--------|-------|-------|
| Compiler | GCC (usually) | Clang/LLVM (always) |
| Default shell | Bash | Zsh |
| Shared libraries | `.so` files | `.dylib` files |
| Library path | `LD_LIBRARY_PATH` | `DYLD_LIBRARY_PATH` |
| Binary inspection | `ldd`, `readelf` | `otool`, `nm` |
| Debugger | GDB | LLDB |
| Package format | ELF | Mach-O |
| Frameworks | N/A | Native concept |
| Multiple architectures | Separate binaries | Universal binaries |

## What You'll Learn in This Part

**[Xcode Command Line Tools](./xcode-cli-tools.md)** covers installing and managing the essential development tools, including what's included and how to switch between full Xcode and standalone tools.

**[The Clang/LLVM Toolchain](./clang-llvm.md)** explains how macOS uses Clang as its compiler, why `gcc` is actually Clang, and the practical differences from GCC.

**[Compiling Software from Source](./compiling-source.md)** walks through building open-source software on macOS, including common pitfalls and solutions.

**[Library Paths and Linking](./library-linking.md)** demystifies dynamic linking on macOS, covering dylib files, install names, rpath, and the tools to inspect and modify them.

**[Frameworks vs Unix Libraries](./frameworks-vs-libraries.md)** explains Apple's framework concept, how frameworks differ from traditional Unix libraries, and when to use each.

**[Debugging with LLDB](./lldb-debugging.md)** provides a practical guide to macOS's native debugger, with command mappings for GDB users.

**[Performance Profiling Tools](./profiling-tools.md)** covers Instruments, DTrace, sample, and other tools for understanding program behavior.

**[Universal Binaries and Architecture](./universal-binaries.md)** explains fat binaries, the transition to Apple Silicon, and how to build for multiple architectures.

## Prerequisites

Before diving into macOS development, you should:

1. Have Terminal.app or another terminal emulator ready
2. Be comfortable with basic command-line operations
3. Understand fundamental compilation concepts (source, object files, linking)

## A Note on Apple Silicon

The transition from Intel to Apple Silicon affects many aspects of development:

- Universal binaries contain code for both architectures
- Rosetta 2 can run Intel binaries on Apple Silicon
- Different Homebrew installation paths (`/opt/homebrew` vs `/usr/local`)
- Some tools behave differently depending on architecture

Throughout this part, we'll note where architecture matters and how to handle both platforms.
