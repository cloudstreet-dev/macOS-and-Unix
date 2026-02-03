# Unix Commands on macOS

If you've used Linux, you already know the core Unix commands: `ls`, `grep`, `sed`, `find`, `awk`. The good news is these commands exist on macOS. The challenging news is they often behave differently. macOS ships with BSD versions of these tools, while Linux uses GNU versions. These differences will trip you up until you understand them.

## The BSD Heritage

macOS descends from BSD Unix, specifically FreeBSD via NeXTSTEP. This heritage means:

```bash
# Check the version of common tools
$ ls --version
ls: illegal option -- -
usage: ls [-ABCFGHILOPRSTUWabcdefghiklmnopqrstuvwxy1%,] ...

$ /bin/ls --version  # BSD ls doesn't have --version
ls: illegal option -- -

# On Linux, this works:
$ ls --version
ls (GNU coreutils) 9.1
```

The BSD tools are older, follow stricter POSIX standards, and often have fewer features than their GNU counterparts. But they're also leaner and more portable.

## Why It Matters

Consider this common task—in-place editing with sed:

```bash
# On Linux (GNU sed)
$ sed -i 's/foo/bar/g' file.txt

# On macOS (BSD sed) - same command fails
$ sed -i 's/foo/bar/g' file.txt
sed: 1: "file.txt": invalid command code f

# The BSD way
$ sed -i '' 's/foo/bar/g' file.txt
```

BSD sed requires an argument after `-i` (the backup extension), even if it's empty. This single difference breaks countless scripts when moving from Linux to macOS.

## What You'll Learn in This Part

**[BSD vs GNU: The Command Divide](./bsd-vs-gnu.md)** explains the fundamental differences between BSD and GNU command-line tools, with specific examples of where they diverge.

**[Essential macOS-Specific Commands](./macos-commands.md)** covers commands unique to macOS like `pbcopy`, `open`, `mdfind`, `say`, and `defaults`—tools with no direct Linux equivalent.

**[Installing and Using GNU Coreutils](./gnu-coreutils.md)** shows how to install GNU versions via Homebrew while keeping BSD tools available, giving you the best of both worlds.

**[Text Processing: sed, awk, and grep](./text-processing.md)** provides a detailed comparison of text processing tools, showing equivalent commands for BSD and GNU versions.

**[File Operations and Manipulation](./file-operations.md)** covers file commands like `cp`, `mv`, `rm`, and macOS-specific tools like `ditto`, including extended attribute handling.

**[Networking Commands](./networking-commands.md)** explains network diagnostics and configuration from the command line, noting what's different from Linux.

**[System Information Commands](./system-info.md)** shows how to query system details using macOS-specific commands like `system_profiler`, `sw_vers`, and `sysctl`.

**[Writing Portable Shell Scripts](./portable-scripts.md)** teaches techniques for writing scripts that work on both macOS and Linux without modification.

## Quick Comparison

Here's a taste of what differs between macOS and Linux:

| Task | macOS (BSD) | Linux (GNU) |
|------|-------------|-------------|
| In-place sed | `sed -i '' 's/a/b/'` | `sed -i 's/a/b/'` |
| Extended regex in grep | `grep -E` | `grep -E` or `grep -P` |
| Copy to clipboard | `pbcopy` | `xclip` or `xsel` |
| Open file with default app | `open file.pdf` | `xdg-open file.pdf` |
| Find files by content | `mdfind "query"` | `locate` or `find + grep` |
| Colored ls | `ls -G` | `ls --color` |
| Date formatting | `date -j -f` | `date -d` |
| Network interfaces | `ifconfig`, `networksetup` | `ip addr` |
| Service management | `launchctl` | `systemctl` |

## The Solutions

You have several approaches:

1. **Learn both**: Know the BSD and GNU syntax for common operations
2. **Install GNU tools**: Use Homebrew to get GNU coreutils
3. **Write portable scripts**: Use POSIX-compatible syntax that works everywhere
4. **Use aliases**: Create aliases that normalize behavior across platforms

Most experienced macOS users combine all four approaches. The following chapters show you how.

## Checking What You Have

See what versions of tools are installed:

```bash
# Default tools are in /usr/bin
$ which sed
/usr/bin/sed

$ which grep
/usr/bin/grep

# After installing GNU tools via Homebrew
$ which gsed  # GNU sed
/opt/homebrew/bin/gsed

$ which ggrep  # GNU grep
/opt/homebrew/bin/ggrep
```

Check if you have Homebrew's GNU tools:

```bash
$ brew list | grep -E '^(coreutils|gnu-sed|grep|gawk)$'
coreutils
gnu-sed
grep
gawk
```

## A Note on Examples

Throughout this part, examples show both BSD (macOS default) and GNU (Linux-style) syntax when they differ. Commands are tested on:
- macOS Sonoma 14.x with BSD tools
- macOS with GNU coreutils via Homebrew

When a command works the same on both platforms, only one version is shown.
