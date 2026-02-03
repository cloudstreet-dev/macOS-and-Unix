# Writing Portable Shell Scripts

Scripts that work perfectly on Linux often break on macOS. This chapter provides techniques for writing shell scripts that run correctly on both platforms, covering common pitfalls and their solutions.

## The Challenge

A script that runs on one platform might fail on another due to:

- Different command-line tools (GNU vs BSD)
- Different default shells (zsh vs bash)
- Different file system behaviors
- Missing commands
- Different option flags for the same command

## Start with the Shebang

The first line determines which interpreter runs your script:

```bash
#!/bin/bash            # Explicit bash (best for portability)
#!/bin/sh              # POSIX shell (most portable, but limited)
#!/usr/bin/env bash    # Find bash in PATH (handles different locations)
#!/bin/zsh             # zsh (macOS default since Catalina)
```

Recommendations:

```bash
# Most portable - uses whatever bash is available
#!/usr/bin/env bash

# For scripts requiring bash features
#!/usr/bin/env bash
set -euo pipefail      # Strict mode

# For maximum portability (POSIX only)
#!/bin/sh
# Then use only POSIX-compliant syntax
```

Note: macOS ships with Bash 3.2 (from 2007) due to licensing. For Bash 4+ features, users need Homebrew's bash:

```bash
# Check bash version
$ /bin/bash --version
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin23)

$ /opt/homebrew/bin/bash --version
GNU bash, version 5.2.21(1)-release (aarch64-apple-darwin23)
```

## Detecting the Operating System

```bash
#!/usr/bin/env bash

# Method 1: uname
OS=$(uname -s)
case "$OS" in
    Linux*)  PLATFORM="Linux";;
    Darwin*) PLATFORM="macOS";;
    CYGWIN*) PLATFORM="Cygwin";;
    MINGW*)  PLATFORM="MinGW";;
    *)       PLATFORM="Unknown";;
esac

# Method 2: Check for specific files
if [ -f /etc/os-release ]; then
    # Linux (most distributions)
    . /etc/os-release
    echo "Linux: $NAME $VERSION"
elif [ -f /System/Library/CoreServices/SystemVersion.plist ]; then
    # macOS
    echo "macOS: $(sw_vers -productVersion)"
fi

# Method 3: OSTYPE variable (bash)
case "$OSTYPE" in
    darwin*)  echo "macOS" ;;
    linux*)   echo "Linux" ;;
    bsd*)     echo "BSD" ;;
    msys*)    echo "Windows/MSYS" ;;
    *)        echo "Unknown: $OSTYPE" ;;
esac
```

## Handling sed Differences

The `-i` flag is the most common issue:

```bash
# WRONG - GNU syntax fails on macOS
sed -i 's/old/new/g' file.txt

# PORTABLE SOLUTION 1: Use a function
sed_i() {
    if [[ "$OSTYPE" == darwin* ]]; then
        sed -i '' "$@"
    else
        sed -i "$@"
    fi
}

# Usage
sed_i 's/old/new/g' file.txt

# PORTABLE SOLUTION 2: Use a temp file (most portable)
sed 's/old/new/g' file.txt > file.tmp && mv file.tmp file.txt

# PORTABLE SOLUTION 3: Use perl (if available)
perl -i -pe 's/old/new/g' file.txt
```

More sed portability:

```bash
# Newlines - GNU sed understands \n, BSD doesn't
# Portable: use a literal newline or $'\n'
sed 's/$/\
/' file.txt

# Or use printf
nl=$'\n'
sed "s/$/$nl/" file.txt

# Extended regex - use -E on both (works on modern BSD and GNU)
sed -E 's/[0-9]+/NUMBER/g' file.txt
```

## Handling grep Differences

```bash
# WRONG - Perl regex not available on BSD
grep -P '\d+' file.txt

# PORTABLE - Use extended regex with POSIX character classes
grep -E '[0-9]+' file.txt

# Word boundaries
# GNU: \bword\b
# BSD: [[:<:]]word[[:>:]]
# PORTABLE: Use -w flag
grep -w 'word' file.txt

# Portable function for patterns
grep_digits() {
    grep -E '[0-9]+' "$@"
}
```

## Handling date Differences

Date parsing differs dramatically:

```bash
# Get date N days ago
get_date_ago() {
    local days=$1
    if [[ "$OSTYPE" == darwin* ]]; then
        date -v-${days}d +%Y-%m-%d
    else
        date -d "$days days ago" +%Y-%m-%d
    fi
}

# Parse a date string
parse_date() {
    local datestr=$1
    local format=$2
    if [[ "$OSTYPE" == darwin* ]]; then
        date -j -f "$format" "$datestr" +%s
    else
        date -d "$datestr" +%s
    fi
}

# Get current timestamp (portable)
date +%s

# Format current date (portable)
date +%Y-%m-%d
date +"%Y-%m-%d %H:%M:%S"
```

## Handling stat Differences

```bash
# Get file size
get_file_size() {
    local file=$1
    if [[ "$OSTYPE" == darwin* ]]; then
        stat -f%z "$file"
    else
        stat -c%s "$file"
    fi
}

# Get modification time
get_mtime() {
    local file=$1
    if [[ "$OSTYPE" == darwin* ]]; then
        stat -f%m "$file"
    else
        stat -c%Y "$file"
    fi
}

# MOST PORTABLE: Use wc or ls
file_size=$(wc -c < "$file")
# or
file_size=$(ls -l "$file" | awk '{print $5}')
```

## Handling readlink Differences

```bash
# Get canonical path
get_realpath() {
    local path=$1
    if command -v realpath &> /dev/null; then
        realpath "$path"
    elif command -v greadlink &> /dev/null; then
        greadlink -f "$path"
    elif [[ "$OSTYPE" == darwin* ]]; then
        # macOS without coreutils
        python3 -c "import os; print(os.path.realpath('$path'))"
    else
        readlink -f "$path"
    fi
}

# Or use this POSIX-compliant function
realpath_posix() {
    local path=$1
    if [ -d "$path" ]; then
        (cd "$path" && pwd -P)
    else
        (cd "$(dirname "$path")" && echo "$(pwd -P)/$(basename "$path")")
    fi
}
```

## Handling mktemp Differences

```bash
# PORTABLE: Always use a template
tmpfile=$(mktemp /tmp/myscript.XXXXXX)
tmpdir=$(mktemp -d /tmp/myscript.XXXXXX)

# Cleanup on exit
cleanup() {
    rm -rf "$tmpfile" "$tmpdir"
}
trap cleanup EXIT
```

## Handling Array Differences

Bash arrays work the same, but be aware of version differences:

```bash
# Bash 3.2 (macOS default) vs Bash 4+
# Associative arrays require Bash 4+
# declare -A map  # Fails on macOS default bash

# PORTABLE: Check bash version
if ((BASH_VERSINFO[0] >= 4)); then
    declare -A map
    map[key]="value"
else
    # Use a different approach
    echo "Warning: Associative arrays not supported, using files"
fi

# Regular arrays work on both
arr=("one" "two" "three")
for item in "${arr[@]}"; do
    echo "$item"
done
```

## Finding Commands

```bash
# Check if command exists
command_exists() {
    command -v "$1" &> /dev/null
}

# Find preferred command
find_editor() {
    for editor in nvim vim vi nano; do
        if command_exists "$editor"; then
            echo "$editor"
            return
        fi
    done
    echo "cat"  # Fallback
}

# Use GNU tool if available, fall back to BSD
SED=$(command -v gsed || command -v sed)
GREP=$(command -v ggrep || command -v grep)
DATE=$(command -v gdate || command -v date)
```

## Clipboard Operations

```bash
# Copy to clipboard
copy_to_clipboard() {
    if [[ "$OSTYPE" == darwin* ]]; then
        pbcopy
    elif command_exists xclip; then
        xclip -selection clipboard
    elif command_exists xsel; then
        xsel --clipboard --input
    else
        echo "No clipboard tool available" >&2
        return 1
    fi
}

# Paste from clipboard
paste_from_clipboard() {
    if [[ "$OSTYPE" == darwin* ]]; then
        pbpaste
    elif command_exists xclip; then
        xclip -selection clipboard -o
    elif command_exists xsel; then
        xsel --clipboard --output
    else
        echo "No clipboard tool available" >&2
        return 1
    fi
}

# Usage
echo "Hello" | copy_to_clipboard
paste_from_clipboard
```

## Opening Files and URLs

```bash
# Open file with default application
open_file() {
    if [[ "$OSTYPE" == darwin* ]]; then
        open "$@"
    elif command_exists xdg-open; then
        xdg-open "$@"
    else
        echo "No 'open' command available" >&2
        return 1
    fi
}

# Open URL in browser
open_url() {
    local url=$1
    if [[ "$OSTYPE" == darwin* ]]; then
        open "$url"
    elif command_exists xdg-open; then
        xdg-open "$url"
    elif command_exists sensible-browser; then
        sensible-browser "$url"
    else
        echo "Cannot open URL: $url" >&2
        return 1
    fi
}
```

## Network Operations

```bash
# Get primary IP address
get_ip() {
    if [[ "$OSTYPE" == darwin* ]]; then
        ipconfig getifaddr en0 2>/dev/null || \
        ipconfig getifaddr en1 2>/dev/null
    else
        hostname -I | awk '{print $1}'
    fi
}

# Check if port is open
check_port() {
    local host=$1
    local port=$2
    if command_exists nc; then
        nc -zv "$host" "$port" 2>&1
    elif command_exists timeout; then
        timeout 1 bash -c "echo > /dev/tcp/$host/$port" 2>/dev/null
    else
        (echo > /dev/tcp/"$host"/"$port") 2>/dev/null
    fi
}
```

## Process Management

```bash
# Get process ID by name
get_pid() {
    local name=$1
    if [[ "$OSTYPE" == darwin* ]]; then
        pgrep -f "$name"
    else
        pgrep -f "$name"
    fi
}

# Kill process by name
kill_by_name() {
    local name=$1
    if [[ "$OSTYPE" == darwin* ]]; then
        pkill -f "$name"
    else
        pkill -f "$name"
    fi
}

# Check if process is running
is_running() {
    local name=$1
    pgrep -f "$name" > /dev/null 2>&1
}
```

## Complete Portable Script Template

```bash
#!/usr/bin/env bash
#
# script_name.sh - Description of what this script does
#
# Usage: script_name.sh [options] arguments
#

set -euo pipefail

# ============================================================================
# Configuration
# ============================================================================

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SCRIPT_NAME="$(basename "$0")"

# Detect OS
case "$OSTYPE" in
    darwin*)  OS="macos" ;;
    linux*)   OS="linux" ;;
    *)        OS="unknown" ;;
esac

# ============================================================================
# Utility Functions
# ============================================================================

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2
}

error() {
    log "ERROR: $*"
    exit 1
}

command_exists() {
    command -v "$1" &> /dev/null
}

require_command() {
    if ! command_exists "$1"; then
        error "Required command not found: $1"
    fi
}

# ============================================================================
# OS-Specific Functions
# ============================================================================

# Portable sed in-place
sed_i() {
    if [[ "$OS" == "macos" ]]; then
        sed -i '' "$@"
    else
        sed -i "$@"
    fi
}

# Portable date manipulation
date_ago() {
    local days=$1
    if [[ "$OS" == "macos" ]]; then
        date -v-${days}d +%Y-%m-%d
    else
        date -d "$days days ago" +%Y-%m-%d
    fi
}

# Portable file size
file_size() {
    wc -c < "$1" | tr -d ' '
}

# Portable realpath
get_realpath() {
    if command_exists realpath; then
        realpath "$1"
    elif [[ "$OS" == "macos" ]] && command_exists greadlink; then
        greadlink -f "$1"
    else
        (cd "$(dirname "$1")" && echo "$(pwd)/$(basename "$1")")
    fi
}

# ============================================================================
# Main Script
# ============================================================================

usage() {
    cat << EOF
Usage: $SCRIPT_NAME [options] <arguments>

Description of what this script does.

Options:
    -h, --help      Show this help message
    -v, --verbose   Enable verbose output
    -d, --debug     Enable debug mode

Examples:
    $SCRIPT_NAME file.txt
    $SCRIPT_NAME -v directory/

EOF
}

main() {
    local verbose=false
    local debug=false

    # Parse arguments
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)
                usage
                exit 0
                ;;
            -v|--verbose)
                verbose=true
                shift
                ;;
            -d|--debug)
                debug=true
                set -x
                shift
                ;;
            --)
                shift
                break
                ;;
            -*)
                error "Unknown option: $1"
                ;;
            *)
                break
                ;;
        esac
    done

    # Validate arguments
    if [[ $# -lt 1 ]]; then
        usage
        error "Missing required argument"
    fi

    # Check dependencies
    require_command sed
    require_command grep

    # Main logic here
    log "Running on $OS"
    log "Processing: $*"

    # Your code here...
}

# Run main function
main "$@"
```

## Testing Portability

### Test with Docker

```bash
# Test on Linux
docker run --rm -v "$PWD:/work" -w /work alpine:latest sh ./script.sh
docker run --rm -v "$PWD:/work" -w /work ubuntu:latest bash ./script.sh

# Test on different shells
docker run --rm -v "$PWD:/work" -w /work bash:5.2 bash ./script.sh
docker run --rm -v "$PWD:/work" -w /work bash:3.2 bash ./script.sh
```

### Use shellcheck

```bash
# Install shellcheck
brew install shellcheck

# Check your script
shellcheck script.sh

# Fix common issues
shellcheck -f diff script.sh | patch -p1
```

### Use shfmt

```bash
# Install shfmt
brew install shfmt

# Format script
shfmt -w script.sh

# Check for issues
shfmt -d script.sh
```

## Summary: Portability Checklist

Before distributing a script:

1. **Shebang**: Use `#!/usr/bin/env bash` or `#!/bin/sh` for POSIX
2. **sed -i**: Use temp files or detect OS
3. **grep**: Avoid `-P`, use `-E` with POSIX classes
4. **date**: Abstract into functions
5. **stat**: Use `wc -c` or `ls` for file size
6. **readlink -f**: Provide fallback function
7. **Arrays**: Avoid associative arrays for Bash 3.2 compatibility
8. **Test**: Test on both macOS and Linux
9. **Lint**: Run shellcheck

Common compatibility functions:

| Task | Portable Approach |
|------|-------------------|
| In-place sed | Temp file or OS detection |
| File size | `wc -c < file` |
| Canonical path | Custom function |
| Date math | OS-specific functions |
| Clipboard | OS detection (pbcopy vs xclip) |
| Open file | OS detection (open vs xdg-open) |
| Command check | `command -v cmd` |
