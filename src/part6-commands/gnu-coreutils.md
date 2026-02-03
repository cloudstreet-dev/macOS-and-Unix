# Installing and Using GNU Coreutils

If you're frustrated by BSD tool limitations or need scripts to behave the same way on macOS and Linux, you can install GNU coreutils. This gives you the familiar GNU versions while keeping BSD tools available as a fallback.

## Installation via Homebrew

Install individual GNU tools or the complete coreutils package:

```bash
# Install GNU coreutils (includes ls, cp, mv, rm, date, etc.)
$ brew install coreutils

# Install individual GNU tools
$ brew install gnu-sed    # GNU sed
$ brew install grep       # GNU grep (note: Homebrew grep IS GNU grep)
$ brew install gawk       # GNU awk
$ brew install findutils  # GNU find, xargs, locate
$ brew install gnu-tar    # GNU tar
$ brew install gnu-which  # GNU which

# Install everything you might need
$ brew install coreutils gnu-sed gawk findutils gnu-tar grep
```

## Understanding the g-prefix

To avoid breaking system scripts that depend on BSD tools, Homebrew installs GNU commands with a `g` prefix:

```bash
# BSD tools (system default)
$ which ls
/bin/ls

$ which sed
/usr/bin/sed

# GNU tools (g-prefixed)
$ which gls
/opt/homebrew/bin/gls

$ which gsed
/opt/homebrew/bin/gsed
```

Use GNU tools by their g-prefixed names:

```bash
# GNU versions
$ gls --color=auto
$ gsed -i 's/old/new/' file.txt
$ ggrep -P '\d+' file.txt
$ gawk 'BEGIN {print "hello"}'
$ gfind . -name "*.txt"
$ gdate -d "yesterday"
$ gtar --zstd -cvf archive.tar.zst directory/
```

## Making GNU Tools the Default

You can add GNU tool paths to your PATH so they're used instead of BSD tools. Homebrew's coreutils installs unprefixed versions in a specific directory:

```bash
# Check where unprefixed versions are installed
$ ls /opt/homebrew/opt/coreutils/libexec/gnubin/
base32  cat     chgrp   chmod   chown   chroot  cksum   comm    cp
csplit  cut     date    dd      df      dir     dircolors  dirname
du      echo    env     expand  expr    factor  false   fmt     fold
...

# Same for other GNU packages
$ ls /opt/homebrew/opt/gnu-sed/libexec/gnubin/
sed

$ ls /opt/homebrew/opt/findutils/libexec/gnubin/
find  locate  updatedb  xargs
```

Add to your shell configuration (`~/.zshrc` or `~/.bash_profile`):

```bash
# Add GNU coreutils to PATH
if [ -d "/opt/homebrew/opt/coreutils/libexec/gnubin" ]; then
    export PATH="/opt/homebrew/opt/coreutils/libexec/gnubin:$PATH"
fi

# Add GNU sed
if [ -d "/opt/homebrew/opt/gnu-sed/libexec/gnubin" ]; then
    export PATH="/opt/homebrew/opt/gnu-sed/libexec/gnubin:$PATH"
fi

# Add GNU findutils
if [ -d "/opt/homebrew/opt/findutils/libexec/gnubin" ]; then
    export PATH="/opt/homebrew/opt/findutils/libexec/gnubin:$PATH"
fi

# Add GNU tar
if [ -d "/opt/homebrew/opt/gnu-tar/libexec/gnubin" ]; then
    export PATH="/opt/homebrew/opt/gnu-tar/libexec/gnubin:$PATH"
fi

# Add grep (Homebrew's grep is already GNU grep, no gnubin needed)
if [ -d "/opt/homebrew/opt/grep/libexec/gnubin" ]; then
    export PATH="/opt/homebrew/opt/grep/libexec/gnubin:$PATH"
fi

# Add man pages for GNU tools
if [ -d "/opt/homebrew/opt/coreutils/libexec/gnuman" ]; then
    export MANPATH="/opt/homebrew/opt/coreutils/libexec/gnuman:$MANPATH"
fi
```

Reload your shell:

```bash
$ source ~/.zshrc

# Verify GNU tools are now default
$ which ls
/opt/homebrew/opt/coreutils/libexec/gnubin/ls

$ ls --version
ls (GNU coreutils) 9.4
```

## One-Line PATH Setup

For a comprehensive setup, add this to your shell configuration:

```bash
# GNU tools PATH setup
for gnubin in /opt/homebrew/opt/*/libexec/gnubin; do
    export PATH="$gnubin:$PATH"
done
```

This automatically adds all installed GNU tools to your PATH.

## Maintaining Access to BSD Tools

Even with GNU tools in PATH, you can still access BSD tools:

```bash
# Explicit path to BSD tools
$ /bin/ls
$ /usr/bin/sed
$ /usr/bin/grep
$ /usr/bin/awk
$ /usr/bin/find

# Create aliases for easy access
alias bls='/bin/ls'
alias bsed='/usr/bin/sed'
alias bgrep='/usr/bin/grep'
alias bawk='/usr/bin/awk'
```

## Selective GNU Tool Usage

You might want only specific GNU tools as defaults. Here's a targeted approach:

```bash
# ~/.zshrc - Only add specific GNU tools

# GNU sed (most commonly needed)
alias sed='gsed'

# GNU grep with Perl regex
alias grep='ggrep'

# GNU awk
alias awk='gawk'

# GNU date
alias date='gdate'

# Keep ls as BSD (some prefer its output)
# alias ls='gls --color=auto'
```

Or use a function that tries GNU first, falls back to BSD:

```bash
# Try GNU version, fall back to BSD
sed() {
    if command -v gsed &> /dev/null; then
        gsed "$@"
    else
        /usr/bin/sed "$@"
    fi
}
```

## Verifying Your Setup

Check which versions are active:

```bash
# Create a test script
$ cat << 'EOF' > check_tools.sh
#!/bin/bash
echo "=== Tool Versions ==="
echo -n "ls:   "; ls --version 2>&1 | head -1 || echo "BSD (no --version)"
echo -n "sed:  "; sed --version 2>&1 | head -1 || echo "BSD (no --version)"
echo -n "grep: "; grep --version 2>&1 | head -1 || echo "BSD (no --version)"
echo -n "awk:  "; awk --version 2>&1 | head -1 || echo "BSD (no --version)"
echo -n "find: "; find --version 2>&1 | head -1 || echo "BSD (no --version)"
echo -n "date: "; date --version 2>&1 | head -1 || echo "BSD (no --version)"
echo -n "tar:  "; tar --version 2>&1 | head -1 || echo "BSD (no --version)"

echo ""
echo "=== Tool Paths ==="
which ls sed grep awk find date tar
EOF

$ chmod +x check_tools.sh
$ ./check_tools.sh
```

## Using Both Versions in Scripts

Sometimes you need to use both versions. Here's how to be explicit:

```bash
#!/bin/bash
# Script that uses both BSD and GNU tools

# Define tool paths explicitly
GNU_SED="/opt/homebrew/opt/gnu-sed/libexec/gnubin/sed"
BSD_SED="/usr/bin/sed"

# Use GNU sed for complex regex
$GNU_SED -i 's/\bword\b/replacement/g' file.txt

# Use BSD sed if script needs to be portable
# (with BSD syntax)
$BSD_SED -i '' 's/simple/replace/' file.txt
```

Or detect and adapt:

```bash
#!/bin/bash

# Detect sed type and set options accordingly
if sed --version 2>&1 | grep -q GNU; then
    SED_INPLACE="-i"
else
    SED_INPLACE="-i ''"
fi

# Now use: eval sed $SED_INPLACE 's/old/new/' file.txt
```

## Potential Issues

### Script Compatibility

Some macOS system scripts expect BSD behavior. If you make GNU tools the default:

```bash
# Potential issue: macOS scripts expecting BSD tools
# Solution: Use full paths in system scripts or
# don't add GNU tools to global PATH for root
```

### Homebrew's grep vs System grep

Note that `brew install grep` installs GNU grep:

```bash
$ brew install grep
$ /opt/homebrew/bin/grep --version
grep (GNU grep) 3.11

# The g-prefixed version
$ /opt/homebrew/bin/ggrep --version
grep (GNU grep) 3.11

# They're the same
$ ls -la /opt/homebrew/bin/ggrep
lrwxr-xr-x  1 user  staff  28 Jan 15 10:00 /opt/homebrew/bin/ggrep -> ../Cellar/grep/3.11/bin/ggrep
```

### Intel vs Apple Silicon Paths

Paths differ by architecture:

```bash
# Apple Silicon
/opt/homebrew/opt/coreutils/libexec/gnubin

# Intel
/usr/local/opt/coreutils/libexec/gnubin

# Portable approach in shell config
HOMEBREW_PREFIX=$(brew --prefix 2>/dev/null || echo "/opt/homebrew")
export PATH="$HOMEBREW_PREFIX/opt/coreutils/libexec/gnubin:$PATH"
```

## Complete Setup Example

Here's a complete `~/.zshrc` snippet for a Linux-like experience:

```bash
# ~/.zshrc - GNU tools configuration

# Homebrew prefix (works on both Intel and Apple Silicon)
HOMEBREW_PREFIX="${HOMEBREW_PREFIX:-$(brew --prefix 2>/dev/null)}"

if [ -n "$HOMEBREW_PREFIX" ]; then
    # Add GNU tools to PATH (order matters - first takes precedence)
    for pkg in coreutils gnu-sed gnu-tar findutils grep; do
        gnubin="$HOMEBREW_PREFIX/opt/$pkg/libexec/gnubin"
        if [ -d "$gnubin" ]; then
            export PATH="$gnubin:$PATH"
        fi
    done

    # Add GNU man pages
    for pkg in coreutils gnu-sed gnu-tar findutils grep; do
        gnuman="$HOMEBREW_PREFIX/opt/$pkg/libexec/gnuman"
        if [ -d "$gnuman" ]; then
            export MANPATH="$gnuman:${MANPATH:-}"
        fi
    done
fi

# Aliases for GNU tools that don't use gnubin
alias awk='gawk'

# Enable colors for GNU ls
alias ls='ls --color=auto'
alias ll='ls -la --color=auto'

# Keep BSD tools accessible
alias bsd-ls='/bin/ls'
alias bsd-sed='/usr/bin/sed'
alias bsd-grep='/usr/bin/grep'
```

## Troubleshooting

### "illegal option" Errors

You're using GNU syntax with BSD tools:

```bash
$ ls --color
ls: illegal option -- -
# Solution: Check which ls is in PATH
$ which ls
/bin/ls    # BSD version
# Either use: gls --color
# Or add GNU coreutils to PATH
```

### Man Pages Show Wrong Version

```bash
$ man ls    # Shows BSD man page even though GNU ls is used
# Add GNU man paths
export MANPATH="/opt/homebrew/opt/coreutils/libexec/gnuman:$MANPATH"
```

### Homebrew Commands Not Found

```bash
$ gsed
zsh: command not found: gsed
# GNU sed not installed
$ brew install gnu-sed
```

## Summary

| GNU Package | Commands Included | g-prefix Examples |
|-------------|-------------------|-------------------|
| coreutils | ls, cp, mv, rm, date, cat, chmod, etc. | gls, gcp, gmv, grm, gdate |
| gnu-sed | sed | gsed |
| grep | grep, egrep, fgrep | ggrep |
| gawk | awk | gawk |
| findutils | find, xargs, locate | gfind, gxargs, glocate |
| gnu-tar | tar | gtar |
| gnu-which | which | gwhich |

Installing GNU tools gives you:
- Familiar Linux behavior on macOS
- Perl-compatible regex in grep (`-P`)
- Easier in-place editing with sed
- Version sorting in sort (`-V`)
- Human-readable sort (`-h`)
- And many more GNU extensions

The trade-off is managing PATH carefully and being aware of which version you're using, especially when writing scripts meant for others.
