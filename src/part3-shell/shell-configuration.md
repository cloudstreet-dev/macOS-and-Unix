# Shell Configuration on macOS

Getting your shell configured correctly is fundamental to a productive command-line experience. This chapter explains which configuration files are loaded when, how to structure your setup, and macOS-specific considerations.

## Understanding Shell Invocation Modes

Shells can be invoked in different modes, and different configuration files are loaded depending on the mode:

### Login vs Non-Login Shell

**Login shell**: First shell started when you log in. Characteristics:
- Reads "profile" files
- Sets up initial environment
- Terminal.app starts login shells by default

**Non-login shell**: Shells started from other shells or programs:
- Reads "rc" (run commands) files
- Inherits environment from parent
- Scripts typically run as non-login shells

### Interactive vs Non-Interactive

**Interactive**: You're typing commands at a prompt:
- Reads configuration files
- Sets up completion, prompts, aliases

**Non-interactive**: Running a script:
- Generally skips most configuration
- Faster startup

## Zsh Configuration Files

Zsh reads files in this order:

### All Shells

1. `/etc/zshenv` - System-wide, all shells
2. `~/.zshenv` - User, all shells (including scripts)

### Login Shells

3. `/etc/zprofile` - System-wide login
4. `~/.zprofile` - User login

### Interactive Shells

5. `/etc/zshrc` - System-wide interactive
6. `~/.zshrc` - User interactive

### Login Shell Completion

7. `/etc/zlogin` - System-wide, after zshrc
8. `~/.zlogin` - User, after zshrc

### Logout

9. `~/.zlogout` - User logout cleanup
10. `/etc/zlogout` - System-wide logout

### Diagram

```
                    ┌──────────────────┐
                    │  Shell Started   │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │    .zshenv       │ ← All zsh instances
                    └────────┬─────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
      ┌─────┴──────┐   ┌─────┴──────┐  ┌─────┴──────┐
      │ Non-Login  │   │   Login    │  │   Script   │
      │Interactive │   │Interactive │  │    (sh)    │
      └─────┬──────┘   └─────┬──────┘  └────────────┘
            │                │
            │         ┌──────┴──────┐
            │         │  .zprofile  │
            │         └──────┬──────┘
            │                │
      ┌─────┴────────────────┴──────┐
      │          .zshrc             │ ← Interactive shells
      └─────┬────────────────┬──────┘
            │                │
            │         ┌──────┴──────┐
            │         │   .zlogin   │
            │         └─────────────┘
            │
       (shell runs)
```

### Recommended File Usage

| File | Use For |
|------|---------|
| `~/.zshenv` | Environment variables needed by scripts (rarely modified) |
| `~/.zprofile` | Login-specific setup (PATH modifications for login shells) |
| `~/.zshrc` | Interactive configuration (aliases, prompts, completion, key bindings) |
| `~/.zlogin` | Commands that should run after zshrc (rarely used) |
| `~/.zlogout` | Cleanup on logout (rarely used) |

## Bash Configuration Files

For those still using bash:

### Login Shells

1. `/etc/profile` - System-wide
2. First found of: `~/.bash_profile`, `~/.bash_login`, `~/.profile`

### Non-Login Interactive Shells

1. `/etc/bash.bashrc` (not on macOS by default)
2. `~/.bashrc`

### macOS Quirk

Terminal.app always starts **login** shells, but many expect `.bashrc` to be read. Common solution:

```bash
# In ~/.bash_profile
if [ -f ~/.bashrc ]; then
    source ~/.bashrc
fi
```

## Example Zsh Configuration

### ~/.zshenv

Keep minimal—runs for all shells including scripts:

```bash
# ~/.zshenv
# Only put things here that ALL zsh instances need

# Skip global compinit for faster startup
skip_global_compinit=1
```

### ~/.zprofile

Login-specific PATH modifications:

```bash
# ~/.zprofile
# Runs for login shells (Terminal.app)

# Add Homebrew to PATH (Apple Silicon)
eval "$(/opt/homebrew/bin/brew shellenv)"

# Or Intel Mac
# eval "$(/usr/local/bin/brew shellenv)"
```

### ~/.zshrc

The main configuration file:

```bash
# ~/.zshrc
# Interactive shell configuration

#------------------
# History
#------------------
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history

setopt EXTENDED_HISTORY       # Save timestamp
setopt HIST_EXPIRE_DUPS_FIRST # Expire duplicates first
setopt HIST_IGNORE_DUPS       # Don't store duplicates
setopt HIST_IGNORE_SPACE      # Ignore commands starting with space
setopt HIST_VERIFY            # Show expanded history before executing
setopt SHARE_HISTORY          # Share between sessions

#------------------
# Directory Navigation
#------------------
setopt AUTO_CD           # Type directory name to cd
setopt AUTO_PUSHD        # Push directories to stack
setopt PUSHD_IGNORE_DUPS # Ignore duplicate directories
setopt PUSHD_SILENT      # Silent pushd

#------------------
# Completion
#------------------
autoload -Uz compinit
compinit

# Case-insensitive completion
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'

# Menu selection
zstyle ':completion:*' menu select

# Verbose completion
zstyle ':completion:*' verbose yes

# Group completions by category
zstyle ':completion:*' group-name ''

#------------------
# Key Bindings
#------------------
bindkey -e  # Emacs key bindings (default)
# Or: bindkey -v for vi mode

# Better history search
bindkey '^[[A' history-search-backward  # Up arrow
bindkey '^[[B' history-search-forward   # Down arrow

# Word navigation (Option+Arrow)
bindkey "^[[1;3C" forward-word   # Option+Right
bindkey "^[[1;3D" backward-word  # Option+Left

# Delete word
bindkey "^[^?" backward-kill-word  # Option+Backspace

#------------------
# Prompt
#------------------
# Simple prompt with git info
autoload -Uz vcs_info
precmd_vcs_info() { vcs_info }
precmd_functions+=( precmd_vcs_info )
setopt prompt_subst
zstyle ':vcs_info:git:*' formats ' (%b)'
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f%F{yellow}${vcs_info_msg_0_}%f %# '

# Right-side prompt (optional)
# RPROMPT='%F{gray}%T%f'  # Time

#------------------
# Aliases
#------------------
alias ll='ls -la'
alias la='ls -A'
alias l='ls -CF'
alias grep='grep --color=auto'
alias ..='cd ..'
alias ...='cd ../..'

# macOS specific
alias o='open'
alias finder='open -a Finder .'

#------------------
# Functions
#------------------
# Create directory and cd into it
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Extract various archive types
extract() {
    if [ -f "$1" ]; then
        case "$1" in
            *.tar.bz2) tar xjf "$1" ;;
            *.tar.gz)  tar xzf "$1" ;;
            *.bz2)     bunzip2 "$1" ;;
            *.gz)      gunzip "$1" ;;
            *.tar)     tar xf "$1" ;;
            *.tbz2)    tar xjf "$1" ;;
            *.tgz)     tar xzf "$1" ;;
            *.zip)     unzip "$1" ;;
            *.Z)       uncompress "$1" ;;
            *.7z)      7z x "$1" ;;
            *)         echo "'$1' cannot be extracted" ;;
        esac
    else
        echo "'$1' is not a file"
    fi
}

#------------------
# Tool Configuration
#------------------
# FZF (if installed)
[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh

# Node Version Manager (if installed)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"

# Python environment
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
command -v pyenv >/dev/null && eval "$(pyenv init -)"

#------------------
# Local Configuration
#------------------
# Machine-specific settings not in version control
[ -f ~/.zshrc.local ] && source ~/.zshrc.local
```

## macOS-Specific Configuration

### Homebrew Setup

```bash
# In ~/.zprofile for login shells, or ~/.zshrc

# Apple Silicon
if [[ -f /opt/homebrew/bin/brew ]]; then
    eval "$(/opt/homebrew/bin/brew shellenv)"
fi

# Intel
if [[ -f /usr/local/bin/brew ]]; then
    eval "$(/usr/local/bin/brew shellenv)"
fi
```

### macOS Aliases

```bash
# Quick Look
ql() { qlmanage -p "$@" &>/dev/null; }

# Show/hide hidden files
alias showfiles='defaults write com.apple.finder AppleShowAllFiles YES; killall Finder'
alias hidefiles='defaults write com.apple.finder AppleShowAllFiles NO; killall Finder'

# Clipboard
alias pbp='pbpaste'
alias pbc='pbcopy'

# Empty trash
alias emptytrash='rm -rf ~/.Trash/*'

# Lock screen
alias lock='/System/Library/CoreServices/Menu\ Extras/User.menu/Contents/Resources/CGSession -suspend'

# Get macOS software updates
alias update='sudo softwareupdate -i -a'
```

### PATH Configuration

Common PATH additions:

```bash
# In ~/.zprofile or ~/.zshrc

# Standard local binaries
export PATH="/usr/local/bin:$PATH"

# Homebrew (Apple Silicon)
export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:$PATH"

# User binaries
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# Development tools
export PATH="$HOME/go/bin:$PATH"              # Go
export PATH="$HOME/.cargo/bin:$PATH"           # Rust
export PATH="$HOME/.rbenv/bin:$PATH"           # Ruby
```

## Configuration Management

### Version Control

Keep your dotfiles in Git:

```bash
# Initialize dotfiles repo
$ cd ~
$ git init --bare ~/.dotfiles

# Create alias
alias dotfiles='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'

# Add files
$ dotfiles add ~/.zshrc ~/.zprofile
$ dotfiles commit -m "Add shell config"

# Push to remote
$ dotfiles remote add origin git@github.com:username/dotfiles.git
$ dotfiles push -u origin main
```

### Modular Configuration

Split configuration into separate files:

```bash
# In ~/.zshrc
# Load all configuration modules
for config in ~/.config/zsh/*.zsh; do
    source "$config"
done
```

```bash
~/.config/zsh/
├── aliases.zsh
├── completion.zsh
├── functions.zsh
├── history.zsh
├── keybindings.zsh
└── prompt.zsh
```

### Local Overrides

Keep machine-specific settings separate:

```bash
# End of ~/.zshrc
# Load local configuration (not version controlled)
[ -f ~/.zshrc.local ] && source ~/.zshrc.local
```

Add to `.gitignore`:
```
.zshrc.local
```

## Debugging Configuration

### Startup Time

Profile shell startup:

```bash
# Time shell startup
$ time zsh -i -c exit
zsh -i -c exit  0.08s user 0.04s system 94% cpu 0.124 total

# Detailed profiling
$ zsh -xv 2>&1 | head -100
```

### Trace Loading

```bash
# In ~/.zshrc, at the beginning:
zmodload zsh/zprof

# At the end:
zprof
```

### Finding Slow Operations

Common slowdowns:
- `compinit` without caching
- nvm/rbenv initialization
- Plugin managers loading many plugins
- Network operations in prompts

Solutions:
```bash
# Cache compinit
autoload -Uz compinit
if [[ -n ${ZDOTDIR}/.zcompdump(#qN.mh+24) ]]; then
    compinit
else
    compinit -C
fi

# Lazy-load slow tools
nvm() {
    unfunction nvm
    export NVM_DIR="$HOME/.nvm"
    source "$NVM_DIR/nvm.sh"
    nvm "$@"
}
```

## Summary

Shell configuration on macOS:

| Shell | Main Config | Login Config |
|-------|-------------|--------------|
| zsh | `~/.zshrc` | `~/.zprofile` |
| bash | `~/.bashrc` | `~/.bash_profile` |

Best practices:
1. **Keep `.zshenv` minimal** — runs for all shells
2. **Put PATH in `.zprofile`** — for login shells
3. **Put interactive config in `.zshrc`** — aliases, prompts, completion
4. **Use `.zshrc.local`** — for machine-specific settings
5. **Version control your dotfiles** — but not secrets
6. **Profile startup time** — optimize if needed

The configuration files structure your shell experience. Take time to set them up well, and your command-line work becomes significantly more efficient.
