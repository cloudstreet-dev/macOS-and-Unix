# The Great Shell Transition: Bash to Zsh

In macOS Catalina (2019), Apple changed the default shell from bash to zsh. This was one of the most significant changes for command-line users in macOS history. Understanding why this happened, what's different, and how to work with both shells is essential knowledge.

## Why Apple Switched

### The Licensing Issue

The primary reason was licensing. Bash 3.2 was the last version released under GPLv2. Starting with Bash 4.0 (released in 2009), bash is licensed under GPLv3.

Apple has consistently avoided GPLv3 software because:
- GPLv3 includes provisions about hardware restrictions (Tivoization)
- GPLv3 has patent licensing requirements
- Apple's business model conflicts with some GPLv3 terms

This left Apple shipping decade-old Bash 3.2:

```bash
# macOS's built-in bash (still available)
$ /bin/bash --version
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin23)
Copyright (C) 2007 Free Software Foundation, Inc.

# Modern bash (via Homebrew)
$ /opt/homebrew/bin/bash --version
GNU bash, version 5.2.26(1)-release (aarch64-apple-darwin23.2.0)
```

### Zsh's Advantages

Beyond licensing, zsh offers genuine improvements:
- Better completion system
- More flexible customization
- Improved array handling
- Better globbing and pattern matching
- More active development community
- Broad plugin ecosystem (Oh My Zsh, etc.)

## What Changed

### Default Shell

New user accounts use zsh. Existing accounts retain their shell:

```bash
# Check your current shell
$ echo $SHELL
/bin/zsh

# Check what shell is running
$ echo $0
-zsh

# See all configured shells
$ cat /etc/shells
```

### Configuration Files

Zsh uses different configuration files than bash:

| Bash | Zsh | Purpose |
|------|-----|---------|
| `~/.bash_profile` | `~/.zprofile` | Login shell setup |
| `~/.bashrc` | `~/.zshrc` | Interactive shell config |
| `~/.bash_login` | `~/.zlogin` | Login shell (after profile) |
| `~/.bash_logout` | `~/.zlogout` | Logout cleanup |
| - | `~/.zshenv` | All shells (including scripts) |

### Startup Warnings

If you have bash configurations but use zsh, you may see:

```
The default interactive shell is now zsh.
To update your account to use zsh, please run `chsh -s /bin/zsh`.
For more details, please visit https://support.apple.com/kb/HT208050.
```

To silence this warning:

```bash
# In ~/.bash_profile or ~/.bashrc
export BASH_SILENCE_DEPRECATION_WARNING=1
```

## Staying with Bash

You can continue using bash:

```bash
# Change default shell to bash
$ chsh -s /bin/bash

# Or install modern bash and use that
$ brew install bash
$ sudo sh -c 'echo /opt/homebrew/bin/bash >> /etc/shells'
$ chsh -s /opt/homebrew/bin/bash
```

Note: Using Homebrew's bash gives you version 5.x features.

## Key Differences

### Array Indexing

This is a common gotcha:

```bash
# Bash: Arrays are 0-indexed
arr=(one two three)
echo ${arr[0]}  # "one"

# Zsh: Arrays are 1-indexed by default
arr=(one two three)
echo ${arr[1]}  # "one"
echo ${arr[0]}  # Empty!
```

For bash compatibility in zsh:

```bash
# In .zshrc - make arrays 0-indexed
setopt KSH_ARRAYS

# Or use zsh-native indexing
echo $arr[1]  # "one"
```

### Word Splitting

```bash
# Bash: Variables split on whitespace
var="one two three"
for word in $var; do echo $word; done
# Output: one, two, three (three lines)

# Zsh: Variables don't split by default
var="one two three"
for word in $var; do echo $word; done
# Output: one two three (one line!)

# Zsh: Use explicit splitting
for word in ${=var}; do echo $word; done
# Or set option
setopt SH_WORD_SPLIT
```

### Glob Patterns

```bash
# Bash: Unmatched globs pass through as literal
$ ls *.nonexistent
ls: *.nonexistent: No such file or directory

# Zsh: Unmatched globs are errors by default
$ ls *.nonexistent
zsh: no matches found: *.nonexistent

# Zsh: Bash-like behavior
setopt NULL_GLOB  # Unmatched patterns expand to nothing
# or
setopt NO_NOMATCH  # Unmatched patterns pass through literally
```

### History

```bash
# Zsh has better history features
# These are often set by default or by Oh My Zsh

# Share history between sessions
setopt SHARE_HISTORY

# Append rather than overwrite
setopt APPEND_HISTORY

# Add timestamps
setopt EXTENDED_HISTORY

# Don't store duplicates
setopt HIST_IGNORE_DUPS
```

### Prompts

```bash
# Bash PS1
export PS1='\u@\h:\w\$ '

# Zsh PROMPT (different escape sequences)
export PROMPT='%n@%m:%~%# '

# Zsh equivalents:
# %n = username (\u in bash)
# %m = hostname (\h in bash)
# %~ = current directory with ~ abbreviation (\w in bash)
# %# = # for root, % for users (\$ in bash)
```

### Completion

Zsh's completion system is more powerful:

```bash
# Enable completion system
autoload -Uz compinit
compinit

# Case-insensitive completion
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'

# Menu selection
zstyle ':completion:*' menu select

# Colored completion
zstyle ':completion:*:default' list-colors ${(s.:.)LS_COLORS}
```

## Migration Guide

### Step 1: Check Current Setup

```bash
# What shell files do you have?
ls -la ~/.bash* ~/.zsh* ~/.profile 2>/dev/null
```

### Step 2: Create Zsh Configuration

Create `~/.zshrc` with your settings:

```bash
# ~/.zshrc

# History configuration
HISTSIZE=10000
SAVEHIST=10000
HISTFILE=~/.zsh_history
setopt SHARE_HISTORY
setopt HIST_IGNORE_DUPS

# Directory navigation
setopt AUTO_CD
setopt AUTO_PUSHD

# Completion
autoload -Uz compinit
compinit

# Prompt (simple example)
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f %# '

# Your aliases
alias ll='ls -la'
alias la='ls -A'

# Your PATH additions
export PATH="/opt/homebrew/bin:$PATH"
```

### Step 3: Migrate Aliases and Functions

Most aliases work identically:

```bash
# These work in both bash and zsh
alias grep='grep --color=auto'
alias ll='ls -la'
alias ..='cd ..'
```

Functions may need adjustment:

```bash
# Bash function
function_name() {
    local var="$1"
    # ...
}

# Works in zsh too, but zsh allows:
function function_name {
    local var="$1"
    # ...
}
```

### Step 4: Test Scripts

Scripts should specify their interpreter:

```bash
#!/bin/bash
# This script will run in bash regardless of user's default shell

#!/bin/zsh
# This script will run in zsh

#!/bin/sh
# This script runs in POSIX sh (which is bash on macOS)
```

## Bash Compatibility Mode

For scripts that need to work in both:

```bash
# In zsh, enable bash-like behavior
emulate -L bash
# or individual options:
setopt BASH_REMATCH
setopt KSH_ARRAYS
setopt SH_WORD_SPLIT
```

Or create a compatibility shim in `.zshrc`:

```bash
# Source bash configuration if it exists
if [ -f ~/.bashrc ]; then
    emulate -L bash
    source ~/.bashrc
    emulate -L zsh
fi
```

## Oh My Zsh

Many users adopt Oh My Zsh for easier zsh configuration:

```bash
# Install Oh My Zsh
$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Oh My Zsh provides:
- Sensible defaults
- Themes (prompt customization)
- Plugins (git, docker, etc.)
- Easier configuration

```bash
# Example .zshrc with Oh My Zsh
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(git docker brew macos)
source $ZSH/oh-my-zsh.sh
```

### Alternative Frameworks

- **Prezto**: Faster than Oh My Zsh
- **Zinit**: Modern plugin manager
- **Antibody**: Lightweight plugin manager
- **Plain zsh**: No framework needed

## Checking Shell Compatibility

### Test Scripts

```bash
# Test script in different shells
$ bash script.sh
$ zsh script.sh
$ sh script.sh
```

### ShellCheck

```bash
# Install shellcheck
$ brew install shellcheck

# Check a script
$ shellcheck script.sh
```

### Portable Scripts

For maximum portability, use POSIX sh:

```bash
#!/bin/sh
# POSIX-compliant script

# Use [ ] not [[ ]]
if [ "$var" = "value" ]; then
    echo "Match"
fi

# Use $(command) not `command`
result=$(date +%Y)

# Avoid bash/zsh-specific features
```

## Summary

The bash-to-zsh transition:

| Aspect | Recommendation |
|--------|----------------|
| New users | Use zsh (default), it's excellent |
| Existing bash users | Migrate gradually, zsh is compatible with most setups |
| Scripts | Specify interpreter explicitly (`#!/bin/bash` or `#!/bin/zsh`) |
| Portability | Use `#!/bin/sh` for maximum compatibility |
| Bash 4+ features needed | Install modern bash via Homebrew |

Key zsh differences to remember:
- Arrays are 1-indexed (not 0)
- Variables don't word-split by default
- Unmatched globs are errors
- Different prompt escapes
- More powerful completion and globbing

Zsh is a capable shell that rewards learning its features. Whether you migrate fully or maintain dual proficiency, understanding both shells makes you effective on macOS.
