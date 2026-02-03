# Cross-Platform Script Compatibility

Writing shell scripts that work on both macOS and Linux is challenging. The differences between BSD and GNU utilities, path conventions, and available commands can break scripts that work perfectly on one platform. This chapter provides practical patterns for writing portable shell scripts that work across Unix-like systems.

## The Compatibility Challenge

macOS and Linux differ in several ways that affect shell scripts:

```
┌───────────────────┬──────────────────────┬────────────────────────┐
│     Aspect        │       macOS          │        Linux           │
├───────────────────┼──────────────────────┼────────────────────────┤
│ Core utilities    │ BSD (FreeBSD-based)  │ GNU coreutils          │
│ Default shell     │ zsh (since Catalina) │ bash (usually)         │
│ sed in-place      │ sed -i ''            │ sed -i                 │
│ date command      │ BSD date             │ GNU date               │
│ readlink          │ Limited              │ Full (readlink -f)     │
│ stat format       │ stat -f "%..."       │ stat -c "%..."         │
│ grep              │ BSD grep             │ GNU grep               │
│ xargs             │ BSD xargs            │ GNU xargs              │
│ find              │ BSD find             │ GNU find               │
│ Bash version      │ 3.2 (old)            │ 4.x/5.x (current)      │
└───────────────────┴──────────────────────┴────────────────────────┘
```

## Shebang Lines: The First Line Matters

The shebang line tells the system which interpreter to use.

### Portable Shebang Patterns

```bash
#!/usr/bin/env bash    # Find bash in PATH - portable
#!/bin/bash            # Direct path - may not exist
#!/usr/bin/env sh      # POSIX shell - most portable
#!/usr/bin/env python3 # Python via PATH
#!/usr/bin/env perl    # Perl via PATH
```

### Why Use /usr/bin/env

```bash
# Problem: bash location differs
# macOS: /bin/bash (ancient 3.2), /opt/homebrew/bin/bash (modern)
# Linux: /bin/bash or /usr/bin/bash

# Solution: let env find it
#!/usr/bin/env bash

# env searches PATH and runs the first match
# Users can control which version by modifying PATH
```

### Caveats with env

```bash
# Can't pass arguments to interpreter with env (portably)
#!/usr/bin/env bash -e   # This won't work on all systems

# Instead, set options inside the script
#!/usr/bin/env bash
set -e  # Exit on error
set -u  # Error on undefined variables
set -o pipefail  # Pipeline fails if any command fails
```

### Strict Mode Template

```bash
#!/usr/bin/env bash
#
# script-name.sh - Brief description
#

set -euo pipefail

# Debug mode (uncomment to enable)
# set -x

# Script directory (portable)
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
```

## Detecting the Operating System

Many scripts need to behave differently on different platforms.

### Basic OS Detection

```bash
#!/usr/bin/env bash

detect_os() {
    case "$(uname -s)" in
        Darwin)
            OS="macos"
            ;;
        Linux)
            OS="linux"
            ;;
        FreeBSD)
            OS="freebsd"
            ;;
        CYGWIN*|MINGW*|MSYS*)
            OS="windows"
            ;;
        *)
            OS="unknown"
            ;;
    esac
    echo "$OS"
}

OS=$(detect_os)
echo "Running on: $OS"
```

### Detailed Detection

```bash
#!/usr/bin/env bash

detect_platform() {
    local os arch distro

    # Operating system
    os=$(uname -s | tr '[:upper:]' '[:lower:]')

    # Architecture
    arch=$(uname -m)
    case "$arch" in
        x86_64|amd64)
            arch="amd64"
            ;;
        arm64|aarch64)
            arch="arm64"
            ;;
        armv7l)
            arch="arm"
            ;;
    esac

    # Linux distribution
    if [[ "$os" == "linux" ]]; then
        if [[ -f /etc/os-release ]]; then
            distro=$(. /etc/os-release && echo "$ID")
        elif [[ -f /etc/debian_version ]]; then
            distro="debian"
        elif [[ -f /etc/redhat-release ]]; then
            distro="rhel"
        else
            distro="unknown"
        fi
    fi

    echo "$os $arch ${distro:-}"
}

read -r OS ARCH DISTRO <<< "$(detect_platform)"
echo "OS: $OS, Arch: $ARCH, Distro: $DISTRO"
```

### Using OS Detection

```bash
#!/usr/bin/env bash

OS=$(uname -s)

case "$OS" in
    Darwin)
        # macOS-specific commands
        OPEN_CMD="open"
        CLIPBOARD_COPY="pbcopy"
        CLIPBOARD_PASTE="pbpaste"
        ;;
    Linux)
        # Linux-specific commands
        OPEN_CMD="xdg-open"
        CLIPBOARD_COPY="xclip -selection clipboard"
        CLIPBOARD_PASTE="xclip -selection clipboard -o"
        ;;
esac

# Use the variables
echo "Hello" | $CLIPBOARD_COPY
$OPEN_CMD https://example.com
```

## BSD vs GNU Command Differences

### sed: In-Place Editing

The most common portability issue:

```bash
# macOS (BSD sed) requires suffix for -i
sed -i '' 's/old/new/g' file.txt

# Linux (GNU sed) requires no suffix (or suffix without space)
sed -i 's/old/new/g' file.txt

# Portable solution 1: Use backup extension
sed -i.bak 's/old/new/g' file.txt && rm file.txt.bak

# Portable solution 2: Detect and branch
if [[ "$(uname)" == "Darwin" ]]; then
    sed -i '' 's/old/new/g' file.txt
else
    sed -i 's/old/new/g' file.txt
fi

# Portable solution 3: Create a wrapper function
sedi() {
    if [[ "$(uname)" == "Darwin" ]]; then
        sed -i '' "$@"
    else
        sed -i "$@"
    fi
}

sedi 's/old/new/g' file.txt
```

### date: Format Differences

```bash
# Get epoch timestamp
# macOS:
date +%s

# Get date from timestamp
# macOS (BSD date):
date -r 1609459200 "+%Y-%m-%d"

# Linux (GNU date):
date -d @1609459200 "+%Y-%m-%d"

# Portable approach:
epoch_to_date() {
    local epoch=$1
    local format=${2:-"%Y-%m-%d %H:%M:%S"}

    if date -r 0 &>/dev/null 2>&1; then
        # BSD date (macOS)
        date -r "$epoch" "+$format"
    else
        # GNU date (Linux)
        date -d "@$epoch" "+$format"
    fi
}

epoch_to_date 1609459200

# Date arithmetic
# macOS:
date -v+7d "+%Y-%m-%d"  # 7 days from now

# Linux:
date -d "+7 days" "+%Y-%m-%d"

# Portable date arithmetic:
days_from_now() {
    local days=$1
    local format=${2:-"%Y-%m-%d"}

    if date -v+1d &>/dev/null 2>&1; then
        date -v+"${days}d" "+$format"
    else
        date -d "+${days} days" "+$format"
    fi
}
```

### readlink: Getting Real Paths

```bash
# Get canonical path (resolve symlinks)
# Linux (GNU readlink):
readlink -f /some/path

# macOS: readlink doesn't have -f

# Portable solution 1: Use a function
realpath_portable() {
    if command -v realpath &>/dev/null; then
        realpath "$1"
    elif command -v greadlink &>/dev/null; then
        # GNU coreutils installed via Homebrew
        greadlink -f "$1"
    else
        # Pure bash fallback
        local path="$1"
        cd "$(dirname "$path")" 2>/dev/null || return 1
        path=$(pwd -P)/$(basename "$path")
        # Handle file symlinks
        while [[ -L "$path" ]]; do
            path=$(readlink "$path")
            cd "$(dirname "$path")" 2>/dev/null || return 1
            path=$(pwd -P)/$(basename "$path")
        done
        echo "$path"
    fi
}

# Portable solution 2: Python fallback
realpath() {
    python3 -c "import os; print(os.path.realpath('$1'))"
}
```

### stat: File Information

```bash
# Get file size
# macOS (BSD stat):
stat -f %z file.txt

# Linux (GNU stat):
stat -c %s file.txt

# Portable:
file_size() {
    if stat -f %z "$1" &>/dev/null 2>&1; then
        stat -f %z "$1"
    else
        stat -c %s "$1"
    fi
}

# Get modification time (epoch)
# macOS:
stat -f %m file.txt

# Linux:
stat -c %Y file.txt

# Portable:
file_mtime() {
    if stat -f %m "$1" &>/dev/null 2>&1; then
        stat -f %m "$1"
    else
        stat -c %Y "$1"
    fi
}
```

### grep: Extended Regex and Options

```bash
# Extended regex
# Both platforms support -E (POSIX):
grep -E 'pattern1|pattern2' file.txt

# Perl regex (different options)
# macOS: grep doesn't have -P
# Linux: grep -P uses PCRE

# Portable: use -E (ERE) instead of -P (PCRE)
# Or use Perl directly for complex patterns:
perl -ne 'print if /complex(?=pattern)/' file.txt

# Recursive grep
# Both support -r:
grep -r "pattern" directory/

# Count matches
# Both support -c:
grep -c "pattern" file.txt

# Fixed strings (no regex)
# Both support -F:
grep -F "literal.string" file.txt
```

### xargs: Null Delimiter

```bash
# Handle filenames with spaces/newlines
# macOS and Linux both support -0:
find . -name "*.txt" -print0 | xargs -0 rm

# But -P (parallel) differs:
# Linux: xargs -P 4 (4 parallel processes)
# macOS: Same syntax, but may differ in behavior

# Portable parallel processing:
find . -name "*.txt" -print0 | xargs -0 -P "${JOBS:-4}" command
```

### find: Differences

```bash
# Basic find works the same
find . -name "*.txt"

# Execute command
# Both support -exec:
find . -name "*.txt" -exec grep "pattern" {} \;

# Batch execute (less portable)
# GNU find: -exec command {} +
# BSD find: same, but edge cases differ

# Portable batch:
find . -name "*.txt" -print0 | xargs -0 grep "pattern"

# Delete files
# Both support -delete:
find . -name "*.tmp" -delete

# Time-based (syntax varies)
# Modified in last 7 days (both):
find . -mtime -7

# Minutes (both support -mmin):
find . -mmin -60
```

## POSIX Compliance for Maximum Portability

### POSIX Shell Basics

For maximum portability, write POSIX sh instead of bash:

```sh
#!/bin/sh
# POSIX-compliant shell script

# No arrays (bash feature)
# No [[ ]] (use [ ])
# No $(( )) arithmetic (use expr or bc)
# No process substitution <()
# No here-strings <<<

# POSIX test syntax
if [ "$var" = "value" ]; then
    echo "Match"
fi

# POSIX arithmetic
count=$(expr $count + 1)

# POSIX command substitution
result=$(command)

# POSIX string comparison
[ "$a" = "$b" ]   # Equal
[ "$a" != "$b" ]  # Not equal
[ -z "$a" ]       # Empty
[ -n "$a" ]       # Not empty
```

### Bash vs POSIX Comparison

```bash
# Feature           | Bash              | POSIX sh
# ------------------|-------------------|------------------
# Arrays            | arr=(a b c)       | Not available
# Extended test     | [[ $a == b* ]]    | case statement
# Arithmetic        | $(( x + 1 ))      | expr / bc
# Here-string       | cmd <<< "$var"    | echo "$var" | cmd
# Process subst     | diff <(cmd1) <(cmd2) | temp files
# Regex matching    | [[ $x =~ regex ]] | grep / expr

# POSIX alternatives:

# Instead of [[ $str == pattern* ]]
case "$str" in
    pattern*) echo "Match" ;;
    *) echo "No match" ;;
esac

# Instead of arrays
set -- value1 value2 value3
for item in "$@"; do
    echo "$item"
done

# Instead of here-string
echo "$var" | command
# or
printf '%s\n' "$var" | command

# Instead of process substitution
cmd1 > /tmp/file1.$$
cmd2 > /tmp/file2.$$
diff /tmp/file1.$$ /tmp/file2.$$
rm /tmp/file1.$$ /tmp/file2.$$
```

## Portable Script Patterns

### Temporary Files

```bash
#!/usr/bin/env bash

# Create temp file portably
if command -v mktemp &>/dev/null; then
    TMPFILE=$(mktemp)
else
    TMPFILE="/tmp/script.$$.$RANDOM"
    touch "$TMPFILE"
fi

# Cleanup on exit
cleanup() {
    rm -f "$TMPFILE"
}
trap cleanup EXIT

# Use temp file
echo "data" > "$TMPFILE"
```

### Command Availability

```bash
#!/usr/bin/env bash

# Check if command exists
command_exists() {
    command -v "$1" &>/dev/null
}

# Require commands
require_commands() {
    local missing=()
    for cmd in "$@"; do
        if ! command_exists "$cmd"; then
            missing+=("$cmd")
        fi
    done

    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "Error: Missing required commands: ${missing[*]}" >&2
        exit 1
    fi
}

require_commands git curl jq

# Find alternative commands
find_command() {
    for cmd in "$@"; do
        if command_exists "$cmd"; then
            echo "$cmd"
            return 0
        fi
    done
    return 1
}

# Find a download tool
DOWNLOAD_CMD=$(find_command curl wget fetch)
if [[ -z "$DOWNLOAD_CMD" ]]; then
    echo "No download tool found"
    exit 1
fi
```

### Cross-Platform Download

```bash
#!/usr/bin/env bash

download() {
    local url=$1
    local dest=$2

    if command -v curl &>/dev/null; then
        curl -fsSL -o "$dest" "$url"
    elif command -v wget &>/dev/null; then
        wget -q -O "$dest" "$url"
    elif command -v fetch &>/dev/null; then
        fetch -q -o "$dest" "$url"
    else
        echo "No download tool found" >&2
        return 1
    fi
}

download "https://example.com/file.tar.gz" "file.tar.gz"
```

### Getting Script Directory

```bash
#!/usr/bin/env bash

# Works in bash (handles symlinks, sourced scripts)
get_script_dir() {
    local source="${BASH_SOURCE[0]}"
    local dir

    # Resolve symlinks
    while [[ -L "$source" ]]; do
        dir=$(cd -P "$(dirname "$source")" && pwd)
        source=$(readlink "$source")
        # Handle relative symlinks
        [[ $source != /* ]] && source="$dir/$source"
    done

    cd -P "$(dirname "$source")" && pwd
}

SCRIPT_DIR=$(get_script_dir)
echo "Script is in: $SCRIPT_DIR"

# POSIX version (simpler, no symlink resolution)
SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
```

### User Input with Defaults

```bash
#!/usr/bin/env bash

# Read with default (works in bash and zsh)
prompt() {
    local message=$1
    local default=$2
    local response

    if [[ -n "$default" ]]; then
        read -r -p "$message [$default]: " response
        echo "${response:-$default}"
    else
        read -r -p "$message: " response
        echo "$response"
    fi
}

# Handle non-interactive environments
prompt_or_default() {
    local varname=$1
    local message=$2
    local default=$3

    if [[ -t 0 ]]; then
        # Interactive
        read -r -p "$message [$default]: " value
        eval "$varname='${value:-$default}'"
    else
        # Non-interactive
        eval "$varname='$default'"
    fi
}

NAME=$(prompt "Enter your name" "Anonymous")
```

### Color Output

```bash
#!/usr/bin/env bash

# Check if terminal supports colors
supports_colors() {
    [[ -t 1 ]] && [[ -n "$TERM" ]] && [[ "$TERM" != "dumb" ]]
}

# Set up colors (or empty strings if not supported)
setup_colors() {
    if supports_colors; then
        RED='\033[0;31m'
        GREEN='\033[0;32m'
        YELLOW='\033[0;33m'
        BLUE='\033[0;34m'
        BOLD='\033[1m'
        NC='\033[0m'  # No Color
    else
        RED=''
        GREEN=''
        YELLOW=''
        BLUE=''
        BOLD=''
        NC=''
    fi
}

setup_colors

# Use colors
echo -e "${RED}Error:${NC} Something went wrong"
echo -e "${GREEN}Success:${NC} Operation completed"
echo -e "${YELLOW}Warning:${NC} Check this"
echo -e "${BLUE}Info:${NC} FYI"

# Or use printf (more portable)
log_error() { printf "${RED}Error:${NC} %s\n" "$*" >&2; }
log_success() { printf "${GREEN}Success:${NC} %s\n" "$*"; }
log_warn() { printf "${YELLOW}Warning:${NC} %s\n" "$*"; }
log_info() { printf "${BLUE}Info:${NC} %s\n" "$*"; }
```

## Complete Portable Script Template

```bash
#!/usr/bin/env bash
#
# portable-script.sh - A cross-platform script template
#
# Usage: portable-script.sh [options] <arguments>
#

set -euo pipefail

# Script metadata
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_VERSION="1.0.0"

# Get script directory (handles symlinks)
get_script_dir() {
    local source="${BASH_SOURCE[0]}"
    while [[ -L "$source" ]]; do
        local dir=$(cd -P "$(dirname "$source")" && pwd)
        source=$(readlink "$source")
        [[ $source != /* ]] && source="$dir/$source"
    done
    cd -P "$(dirname "$source")" && pwd
}
readonly SCRIPT_DIR=$(get_script_dir)

# Detect OS
detect_os() {
    case "$(uname -s)" in
        Darwin) echo "macos" ;;
        Linux) echo "linux" ;;
        *) echo "unknown" ;;
    esac
}
readonly OS=$(detect_os)

# Colors
setup_colors() {
    if [[ -t 1 ]] && [[ -n "${TERM:-}" ]] && [[ "${TERM}" != "dumb" ]]; then
        RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[0;33m'
        BLUE='\033[0;34m'; BOLD='\033[1m'; NC='\033[0m'
    else
        RED=''; GREEN=''; YELLOW=''; BLUE=''; BOLD=''; NC=''
    fi
}
setup_colors

# Logging functions
log_info() { printf "${BLUE}[INFO]${NC} %s\n" "$*"; }
log_success() { printf "${GREEN}[OK]${NC} %s\n" "$*"; }
log_warn() { printf "${YELLOW}[WARN]${NC} %s\n" "$*" >&2; }
log_error() { printf "${RED}[ERROR]${NC} %s\n" "$*" >&2; }
die() { log_error "$*"; exit 1; }

# Command check
require_cmd() {
    command -v "$1" &>/dev/null || die "Required command not found: $1"
}

# Cross-platform sed -i
sedi() {
    if [[ "$OS" == "macos" ]]; then
        sed -i '' "$@"
    else
        sed -i "$@"
    fi
}

# Cleanup
cleanup() {
    # Remove temp files, etc.
    :
}
trap cleanup EXIT

# Usage
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [options] <argument>

A portable cross-platform script template.

Options:
    -h, --help      Show this help message
    -v, --version   Show version
    -d, --debug     Enable debug mode

Examples:
    $SCRIPT_NAME file.txt
    $SCRIPT_NAME --debug file.txt
EOF
}

# Parse arguments
DEBUG=false
POSITIONAL_ARGS=()

while [[ $# -gt 0 ]]; do
    case $1 in
        -h|--help)
            usage
            exit 0
            ;;
        -v|--version)
            echo "$SCRIPT_NAME version $SCRIPT_VERSION"
            exit 0
            ;;
        -d|--debug)
            DEBUG=true
            set -x
            shift
            ;;
        -*)
            die "Unknown option: $1"
            ;;
        *)
            POSITIONAL_ARGS+=("$1")
            shift
            ;;
    esac
done

set -- "${POSITIONAL_ARGS[@]}"

# Main logic
main() {
    log_info "Running on $OS"
    log_info "Script directory: $SCRIPT_DIR"

    if [[ $# -lt 1 ]]; then
        die "Missing required argument. Use --help for usage."
    fi

    local input=$1
    log_info "Processing: $input"

    # Your code here

    log_success "Done!"
}

main "$@"
```

## Testing Script Portability

### Using ShellCheck

```bash
# Install ShellCheck
$ brew install shellcheck   # macOS
$ apt install shellcheck    # Debian/Ubuntu

# Check script for issues
$ shellcheck script.sh

# Check for POSIX compliance
$ shellcheck --shell=sh script.sh

# Exclude specific warnings
$ shellcheck --exclude=SC2086 script.sh
```

### Testing on Multiple Platforms

```bash
# Test in Docker containers
$ docker run --rm -v "$PWD:/scripts" ubuntu:22.04 bash /scripts/test.sh
$ docker run --rm -v "$PWD:/scripts" alpine:3 sh /scripts/test.sh

# Test with different shells
$ bash script.sh
$ zsh script.sh
$ dash script.sh   # POSIX test
```

## Summary

| Challenge | Portable Solution |
|-----------|-------------------|
| Shebang | `#!/usr/bin/env bash` |
| OS detection | `uname -s` + case statement |
| sed in-place | `sed -i.bak` + rm, or wrapper function |
| date formatting | OS-specific wrapper function |
| readlink -f | Pure bash function or Python fallback |
| stat format | OS-specific wrapper function |
| Script directory | `BASH_SOURCE[0]` with symlink resolution |
| Colors | Check `[[ -t 1 ]]` and `$TERM` |
| Downloads | Check for curl/wget/fetch |
| Temp files | `mktemp` with fallback |

Key practices:
1. Use `#!/usr/bin/env bash` for bash scripts
2. Detect OS early and branch for platform-specific code
3. Create wrapper functions for incompatible commands
4. Test with ShellCheck and on multiple platforms
5. Use POSIX sh for maximum portability when bash features aren't needed
6. Document platform requirements in script header
