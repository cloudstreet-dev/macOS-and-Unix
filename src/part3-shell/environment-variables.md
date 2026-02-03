# Environment Variables and PATH

Environment variables on macOS present unique challenges. Between Homebrew, GUI applications, multiple shell contexts, and macOS-specific mechanisms like `launchd`, getting your PATH and environment right requires understanding how each context inherits (or doesn't inherit) environment settings.

## Environment Variable Basics

Environment variables are name-value pairs available to processes:

```bash
# View all environment variables
$ env
# or
$ printenv

# View specific variable
$ echo $HOME
/Users/david

# Set variable for current session
$ export MY_VAR="value"

# Set for a single command
$ MY_VAR="value" some_command
```

### Important Built-in Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `PATH` | Executable search path | `/usr/bin:/bin` |
| `HOME` | User home directory | `/Users/david` |
| `USER` | Current username | `david` |
| `SHELL` | Default shell | `/bin/zsh` |
| `TERM` | Terminal type | `xterm-256color` |
| `LANG` | Locale setting | `en_US.UTF-8` |
| `EDITOR` | Default text editor | `vim` |
| `TMPDIR` | Temporary directory | `/var/folders/.../T/` |
| `PWD` | Current working directory | `/Users/david/project` |

### macOS-Specific Variables

| Variable | Purpose |
|----------|---------|
| `TERM_PROGRAM` | Terminal app name (`Apple_Terminal`, `iTerm.app`) |
| `TERM_SESSION_ID` | Unique session identifier |
| `__CFBundleIdentifier` | Current app bundle ID |
| `COMMAND_MODE` | Unix compatibility mode |
| `SECURITYSESSIONID` | Security session |

## Understanding PATH

PATH tells the shell where to find executables:

```bash
$ echo $PATH
/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

# Shell searches directories left to right
# First match wins
```

### Default macOS PATH

A fresh macOS installation has a basic PATH:

```
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

### PATH Sources

macOS PATH comes from multiple places:

1. **System-wide PATH** (`/etc/paths`):
```bash
$ cat /etc/paths
/usr/local/bin
/usr/bin
/bin
/usr/sbin
/sbin
```

2. **PATH additions** (`/etc/paths.d/*`):
```bash
$ ls /etc/paths.d
100-rvictl
MacGPG2

$ cat /etc/paths.d/MacGPG2
/usr/local/MacGPG2/bin
```

3. **Shell configuration** (`~/.zprofile`, `~/.zshrc`):
```bash
export PATH="/opt/homebrew/bin:$PATH"
```

4. **path_helper utility** (`/usr/libexec/path_helper`):
```bash
# Combines /etc/paths and /etc/paths.d
$ /usr/libexec/path_helper -s
PATH="/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/MacGPG2/bin"; export PATH;
```

### Setting PATH Correctly

For zsh (default shell):

```bash
# In ~/.zprofile (for login shells)
# Add to BEGINNING of PATH (searched first)
export PATH="/opt/homebrew/bin:$PATH"

# Add to END of PATH (searched last)
export PATH="$PATH:$HOME/bin"
```

### Common PATH Additions

```bash
# Homebrew (Apple Silicon)
export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:$PATH"

# Homebrew (Intel)
export PATH="/usr/local/bin:/usr/local/sbin:$PATH"

# User binaries
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# Go
export PATH="$HOME/go/bin:$PATH"

# Rust
export PATH="$HOME/.cargo/bin:$PATH"

# Python (pyenv)
export PATH="$HOME/.pyenv/shims:$PATH"

# Node (nvm adds its own path)
```

## Environment for GUI Applications

GUI applications **do not** inherit your shell environment. This is a common source of confusion.

### The Problem

```bash
# Terminal: This works
$ echo $MY_VAR
my_value

# But GUI apps don't see it
# VS Code launched from Dock won't have MY_VAR
```

### Solutions

#### 1. Launch from Terminal

```bash
# Open VS Code with terminal's environment
$ code .

# Open any app
$ open -a "Visual Studio Code"
```

#### 2. Use launchctl setenv

Set environment variables system-wide:

```bash
# Set for current boot
$ launchctl setenv MY_VAR "my_value"

# Verify
$ launchctl getenv MY_VAR
my_value
```

**Note**: This persists only until logout. For persistence, use Launch Agents.

#### 3. Launch Agent for Persistent Environment

Create `~/Library/LaunchAgents/environment.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>my.environment</string>
    <key>ProgramArguments</key>
    <array>
        <string>sh</string>
        <string>-c</string>
        <string>
            launchctl setenv MY_VAR "my_value"
            launchctl setenv PATH "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"
        </string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

Load it:
```bash
$ launchctl load ~/Library/LaunchAgents/environment.plist
```

#### 4. Per-Application Settings

Some apps have their own environment settings:
- VS Code: `terminal.integrated.env.osx` in settings
- IntelliJ: Run configurations → Environment variables
- Xcode: Scheme → Run → Arguments → Environment Variables

### Checking GUI App Environment

```bash
# See what environment an app sees
$ open -a TextEdit
# In TextEdit, open: /dev/fd/1 shows stdout
# Or check in Activity Monitor → Process → Environment
```

## SSH and Remote Sessions

SSH sessions need environment setup too:

### ~/.ssh/environment

If `PermitUserEnvironment` is enabled on the server:

```bash
# ~/.ssh/environment
MY_VAR=value
PATH=/usr/local/bin:/usr/bin:/bin
```

### SendEnv and AcceptEnv

Configure which variables are sent over SSH:

```bash
# ~/.ssh/config
Host *
    SendEnv LANG LC_*
    SendEnv MY_*
```

Server must have `AcceptEnv` configured to receive them.

### Remote PATH Issues

Remote servers won't have your local Homebrew:

```bash
# Script that works locally may fail remotely
$ ssh server 'brew list'  # Fails - brew not in default PATH

# Solutions:
# 1. Use full path
$ ssh server '/opt/homebrew/bin/brew list'

# 2. Source profile
$ ssh server 'source ~/.profile && brew list'

# 3. Interactive shell
$ ssh -t server 'bash -l -c "brew list"'
```

## Common Issues and Solutions

### "Command Not Found" After Installing

```bash
$ brew install something
$ something
zsh: command not found: something
```

**Solutions**:

```bash
# Refresh PATH
$ source ~/.zprofile
# Or
$ source ~/.zshrc
# Or start a new terminal

# Check if Homebrew PATH is set
$ echo $PATH | grep homebrew
# Should see /opt/homebrew/bin

# If not, add to ~/.zprofile:
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### PATH Order Issues

```bash
# System Python instead of Homebrew Python
$ which python3
/usr/bin/python3  # Wrong!

# Check PATH order
$ echo $PATH | tr ':' '\n'
```

**Fix**: Ensure Homebrew path comes before system paths:

```bash
export PATH="/opt/homebrew/bin:$PATH"
```

### Subshell Doesn't Have Variables

```bash
$ export MY_VAR="value"
$ bash -c 'echo $MY_VAR'  # Works
$ zsh script.zsh          # May not work if script starts fresh
```

**Solution**: Use `-l` for login shell or source configuration:

```bash
$ bash -l -c 'echo $MY_VAR'
```

### Scripts Don't Have PATH

Scripts run with `#!/bin/sh` start with minimal environment:

```bash
#!/bin/sh
brew list  # May fail - brew not in PATH
```

**Solutions**:

```bash
# Use full path in scripts
#!/bin/sh
/opt/homebrew/bin/brew list

# Or set PATH in script
#!/bin/sh
export PATH="/opt/homebrew/bin:$PATH"
brew list

# Or use env with explicit path
#!/usr/bin/env -P /opt/homebrew/bin python3
```

## Environment File Patterns

### Per-Directory Environment (.env files)

Many tools support `.env` files:

```bash
# .env file
DATABASE_URL=postgres://localhost/mydb
API_KEY=secret123
```

Load manually:
```bash
$ export $(grep -v '^#' .env | xargs)
```

Or use tools like `direnv`:
```bash
$ brew install direnv
$ echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc

# Now .envrc files load automatically
```

### direnv for Automatic Environment

```bash
# Install
$ brew install direnv

# Add to ~/.zshrc
eval "$(direnv hook zsh)"

# Create .envrc in project
$ cd myproject
$ echo 'export MY_VAR=value' > .envrc
$ direnv allow

# Variables load when you enter directory
$ cd myproject
direnv: loading .envrc
direnv: export +MY_VAR

$ cd ..
direnv: unloading
```

## Summary

Environment variable contexts on macOS:

| Context | Config Location | Inheritance |
|---------|-----------------|-------------|
| Login shell | `~/.zprofile` | From system |
| Interactive shell | `~/.zshrc` | From login shell |
| Scripts | Minimal | Must be explicit |
| GUI apps | None | launchctl setenv |
| SSH | `~/.ssh/environment` | Explicit |
| Subprocesses | - | From parent |

Key points:
1. **PATH order matters** — first match wins
2. **GUI apps don't get shell environment** — use launchctl or launch from terminal
3. **Homebrew needs explicit PATH setup** — add in `.zprofile`
4. **Scripts have minimal environment** — use full paths or set PATH explicitly
5. **Use direnv for project-specific environment** — automatic loading/unloading

Getting environment configuration right eliminates entire categories of "works on my machine" problems.
