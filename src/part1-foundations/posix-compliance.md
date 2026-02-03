# POSIX Compliance and Standards

When discussing Unix compatibility, you'll often hear about "POSIX compliance." macOS is notable for being both POSIX-compliant and UNIX-certified—distinctions that matter for software portability and system behavior.

## What Is POSIX?

POSIX (Portable Operating System Interface) is a family of standards specified by the IEEE Computer Society for maintaining compatibility between operating systems. POSIX defines:

- **System interfaces**: C API functions like `open()`, `fork()`, `pthread_create()`
- **Shell and utilities**: Command-line tools and their expected behavior
- **Shell language**: The syntax and features of POSIX shell scripts
- **Threads**: The pthreads API for multi-threaded programming
- **Real-time extensions**: APIs for real-time computing

The goal: write code once, run it on any POSIX-compliant system.

## POSIX vs UNIX Certification

These are related but different:

### POSIX Compliance

A system that implements POSIX interfaces. Self-declared, no formal certification required.

- Linux: POSIX-compliant (mostly)
- FreeBSD: POSIX-compliant
- macOS: POSIX-compliant and certified

### UNIX Certification

A formal certification from The Open Group, which owns the UNIX trademark. Requires:

1. Passing conformance tests
2. Paying certification fees
3. Regular re-certification

Currently certified UNIX systems:
- macOS (UNIX 03)
- IBM AIX
- HP-UX
- Oracle Solaris

**Notable non-certified systems**:
- Linux (too expensive/complex to certify each distribution)
- FreeBSD (philosophically opposed to the process)

### What Certification Means

macOS's UNIX 03 certification means it has passed The Open Group's test suite demonstrating compliance with:

- Single UNIX Specification version 3 (SUS v3)
- POSIX.1-2001
- Related standards

You can verify this:

```bash
# Check POSIX version supported
$ getconf _POSIX_VERSION
200112

# Check UNIX version (SUS)
$ getconf _XOPEN_VERSION
600
```

## POSIX on macOS in Practice

### Standard Headers

macOS provides all required POSIX headers:

```bash
$ ls /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/ | head
_endian.h
_posix_availability.h
_pthread
_select.h
_structs.h
_symbol_aliasing.h
_types
_types.h
acct.h
acl.h
```

### Feature Test Macros

To access POSIX-specific features in C code:

```c
#define _POSIX_C_SOURCE 200112L  // Request POSIX.1-2001 features
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
```

Or for all UNIX features:

```c
#define _XOPEN_SOURCE 600  // Request SUS v3 features
#include <stdlib.h>
```

### Checking POSIX Limits

POSIX defines various system limits that can be queried:

```bash
# Maximum arguments to exec()
$ getconf ARG_MAX
1048576

# Maximum filename length
$ getconf NAME_MAX /
255

# Maximum path length
$ getconf PATH_MAX /
1024

# Number of processors
$ getconf _NPROCESSORS_ONLN
10

# POSIX version
$ getconf _POSIX_VERSION
200112
```

In C code:

```c
#include <unistd.h>
#include <limits.h>

long arg_max = sysconf(_SC_ARG_MAX);
long open_max = sysconf(_SC_OPEN_MAX);
long nprocessors = sysconf(_SC_NPROCESSORS_ONLN);
```

## POSIX Shell Scripting

POSIX defines a portable shell language. Scripts targeting POSIX should:

### Use `/bin/sh`

```bash
#!/bin/sh
# Not #!/bin/bash or #!/bin/zsh
```

On macOS, `/bin/sh` is actually `bash` running in POSIX mode (or `zsh` on newer versions in some contexts):

```bash
$ file /bin/sh
/bin/sh: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64e:Mach-O 64-bit executable arm64e]

$ /bin/sh --version
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin23)
```

### Avoid Bashisms

Common bash features that aren't POSIX:

```bash
# Non-POSIX (bash-specific)
[[ $var == "test" ]]     # Double brackets
array=(one two three)     # Arrays
${var//old/new}          # Pattern substitution
$((var++))               # C-style increment
source file.sh           # source command

# POSIX equivalents
[ "$var" = "test" ]      # Single brackets, single =
# No arrays in POSIX
# Use sed for substitution
var=$((var + 1))         # POSIX arithmetic
. file.sh                # Dot command
```

### Test Your Scripts

Check scripts for POSIX compliance:

```bash
# Install shellcheck
$ brew install shellcheck

# Check a script
$ shellcheck --shell=sh myscript.sh
```

## Common POSIX-Related Issues

### Shebang Paths

POSIX doesn't specify interpreter locations:

```bash
#!/bin/bash              # Works on most Linux, macOS
#!/usr/bin/env bash      # More portable - finds bash in PATH
```

On some systems, `bash` might be in `/usr/local/bin/bash` or elsewhere.

### Command Options

Even when commands exist, options may differ:

```bash
# POSIX requires these options for 'ls'
# -a, -A, -C, -d, -F, -g, -i, -l, -n, -o, -p, -q, -r, -R, -s, -t, -u, -1

# These are NOT required by POSIX:
ls --color     # GNU extension
ls -G          # BSD extension (macOS)
```

### printf vs echo

`echo` behavior varies between systems. POSIX recommends `printf`:

```bash
# Portable
printf '%s\n' "Hello"
printf 'Value: %d\n' 42

# Not portable (echo -n, echo -e behavior varies)
echo -n "No newline"      # Not POSIX
echo -e "Tab:\tHere"     # Not POSIX
```

### Regular Expressions

POSIX defines Basic Regular Expressions (BRE) and Extended Regular Expressions (ERE):

```bash
# BRE (default for grep, sed)
grep 'a\+b'              # + must be escaped
grep 'a\{2,3\}'          # Braces must be escaped

# ERE (grep -E, egrep, awk)
grep -E 'a+b'            # + doesn't need escaping
grep -E 'a{2,3}'         # Braces don't need escaping
```

Perl-compatible regex (PCRE) is NOT part of POSIX:

```bash
grep -P '\d+'            # GNU extension - NOT on macOS
grep -E '[0-9]+'         # POSIX ERE equivalent
```

## Extensions Beyond POSIX

macOS includes extensions beyond POSIX requirements:

### BSD Extensions

```bash
# sysctl - BSD system control
$ sysctl kern.hostname

# BSD-style ps flags
$ ps aux

# Disk management
$ diskutil list
```

### Apple Extensions

```bash
# Spotlight search
$ mdfind "query"

# Clipboard
$ pbcopy < file.txt

# Open files
$ open document.pdf

# System configuration
$ defaults read com.apple.finder
```

These extensions are not portable to Linux or other systems.

## Writing Portable Code

### For Shell Scripts

1. Use `#!/bin/sh` and stick to POSIX features
2. Test with `shellcheck --shell=sh`
3. Avoid command options not listed in POSIX
4. Use `printf` instead of `echo` for complex output
5. Use `[ ]` instead of `[[ ]]`

```bash
#!/bin/sh
# Portable script example

set -e  # Exit on error (POSIX)

# Check if file exists
if [ -f "$1" ]; then
    printf 'Processing: %s\n' "$1"
    # Use cat (POSIX) not bashisms
    cat "$1" | while IFS= read -r line; do
        printf '%s\n' "$line"
    done
else
    printf 'File not found: %s\n' "$1" >&2
    exit 1
fi
```

### For C Programs

1. Include `_POSIX_C_SOURCE` or `_XOPEN_SOURCE` appropriately
2. Check return values (POSIX mandates specific error codes)
3. Use `configure` scripts to detect platform differences
4. Provide fallbacks for non-POSIX features

```c
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>

int main(void) {
    char *cwd = getcwd(NULL, 0);  // POSIX.1-2001
    if (cwd == NULL) {
        perror("getcwd");
        return EXIT_FAILURE;
    }
    printf("Current directory: %s\n", cwd);
    free(cwd);
    return EXIT_SUCCESS;
}
```

### Autoconf and Portable Building

For larger projects, GNU Autotools help detect platform differences:

```bash
# Generate configure script
$ autoreconf -i

# Configure for the current platform
$ ./configure

# Build
$ make
```

The `configure` script detects macOS vs Linux differences and sets appropriate compiler flags.

## Checking Standards Compliance

### On macOS

```bash
# Compiler standards mode
$ cc -std=c11 -D_POSIX_C_SOURCE=200809L -Wall -pedantic program.c

# Check what POSIX version the system claims
$ getconf POSIX_VERSION
200112

# Check POSIX limits
$ getconf -a | grep POSIX
```

### Useful References

- [POSIX.1-2017 specification](https://pubs.opengroup.org/onlinepubs/9699919799/)
- [Apple's UNIX Conformance documentation](https://developer.apple.com/library/archive/documentation/Porting/Conceptual/PortingUnix/)
- IEEE Std 1003.1 (requires IEEE subscription)

## Summary

macOS's POSIX compliance and UNIX certification mean:

1. **Standard APIs work**: POSIX C functions behave correctly
2. **Basic commands exist**: Core utilities are available
3. **Shell scripts can be portable**: With discipline
4. **Not everything is standard**: macOS and Apple add extensions

When writing portable code:
- Stick to POSIX-defined features when possible
- Test on multiple platforms
- Use tools like `shellcheck` and `autoconf`
- Be aware that BSD and GNU versions of tools differ

The standards provide a foundation, but real-world portability requires attention to detail and testing across target systems.
