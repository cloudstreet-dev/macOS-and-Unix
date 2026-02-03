# iTerm2 and Alternative Terminals

While Terminal.app is capable, many power users prefer third-party terminal emulators. iTerm2 is the most popular choice, offering features like split panes, extensive customization, and powerful automation. This chapter covers iTerm2 in depth and surveys alternatives.

## iTerm2

iTerm2 is a free, open-source terminal emulator for macOS with an active community and extensive feature set.

### Installation

```bash
# Via Homebrew (recommended)
$ brew install --cask iterm2

# Or download from: https://iterm2.com
```

### Key Features

#### Split Panes

Create multiple panes in a single tab:

| Action | Shortcut |
|--------|----------|
| Split vertically | `Cmd+D` |
| Split horizontally | `Cmd+Shift+D` |
| Navigate panes | `Cmd+Option+Arrow` |
| Maximize pane | `Cmd+Shift+Enter` |
| Close pane | `Cmd+W` |

#### Hotkey Window

A terminal that slides down from the top (like Quake consoles):

1. Preferences → Keys → Hotkey
2. Create a Dedicated Hotkey Window
3. Assign a hotkey (e.g., `Option+Space`)
4. Configure to show/hide with hotkey

#### Search with Regex

`Cmd+F` opens find bar with regex support:

```
# Search for IP addresses
\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}

# Search for email-like patterns
\S+@\S+\.\S+
```

#### Shell Integration

iTerm2's shell integration provides:

- Current directory in tab title
- Command history with timestamps
- Marks at prompts
- Captured command output
- Alerts on long command completion
- File download from remote servers

Install:
```bash
# Automatic installation
$ curl -L https://iterm2.com/shell_integration/install_shell_integration.sh | bash

# Or install for specific shell
$ curl -L https://iterm2.com/shell_integration/zsh -o ~/.iterm2_shell_integration.zsh
$ echo 'source ~/.iterm2_shell_integration.zsh' >> ~/.zshrc
```

After installation, right-click to:
- Select output of last command
- Download files from remote hosts
- Open Recent Directories

#### Triggers

Automatically perform actions when text matches patterns:

1. Preferences → Profiles → Advanced → Triggers
2. Add regex patterns
3. Choose actions: highlight, alert, run command, etc.

Examples:
- Highlight errors in red
- Alert on build completion
- Run command when pattern matches

#### Instant Replay

Scroll back through terminal history with time-based replay:

1. Press `Cmd+Option+B`
2. Use arrow keys to move through history
3. Press Escape to return to live view

#### Password Manager

Store frequently-used passwords:

1. Preferences → Profiles → Advanced → Triggers
2. Or: Window → Password Manager (`Cmd+Option+F`)
3. Store username/password pairs
4. Click to paste (use with Secure Keyboard Entry)

#### Image Display

iTerm2 can display images inline:

```bash
# Using imgcat (part of shell integration)
$ imgcat image.png

# For remote servers, install imgcat:
$ curl -L https://iterm2.com/utilities/imgcat -o ~/bin/imgcat
$ chmod +x ~/bin/imgcat
```

#### Broadcast Input

Type in multiple panes simultaneously:

1. Shell → Broadcast Input
2. Options: To All Panes, To All Tabs, etc.
3. Type once, appears everywhere

Useful for running the same command on multiple servers.

#### Profiles

Create different profiles for different contexts:

- Development (large font, dark theme)
- SSH (different colors per server)
- Presentation (huge font, high contrast)

Switch with `Cmd+O` (profile picker).

### Configuration

#### Appearance

Preferences → Appearance:
- Theme: Dark, Light, Minimal, etc.
- Tab position: Top, Left, Bottom
- Status bar: Enable/configure

#### Keys

Preferences → Keys → Key Bindings:
- Global hotkey
- Custom key mappings
- Presets: Natural Text Editing (makes Option+Arrow work like other apps)

**Natural Text Editing Preset**:
1. Preferences → Profiles → Keys
2. Key Mappings → Presets → Natural Text Editing

This enables:
- `Option+Left/Right` - Move by word
- `Option+Backspace` - Delete word
- `Cmd+Left/Right` - Move to line start/end

#### Status Bar

Preferences → Profiles → Session → Status bar:

Components:
- Current directory
- Git branch
- CPU/Memory usage
- Battery
- Custom text

### Scripting

iTerm2 supports Python scripting:

```python
#!/usr/bin/env python3
# Requires: pip install iterm2

import iterm2

async def main(connection):
    app = await iterm2.async_get_app(connection)
    window = app.current_terminal_window
    if window:
        await window.current_tab.current_session.async_send_text('echo Hello\n')

iterm2.run_until_complete(main)
```

### Recommended Settings

**For productivity**:
1. Enable hotkey window (`Option+Space`)
2. Install shell integration
3. Enable Natural Text Editing
4. Configure triggers for errors/completions
5. Set up profiles for different contexts

## Alternative Terminal Emulators

### Alacritty

GPU-accelerated, fast, cross-platform terminal:

```bash
$ brew install --cask alacritty
```

**Characteristics**:
- Very fast (GPU rendering)
- Configuration via TOML file
- No tabs (use tmux)
- Cross-platform (macOS, Linux, Windows)
- Minimal features by design

Configuration (`~/.config/alacritty/alacritty.toml`):

```toml
[font]
normal = { family = "SF Mono", style = "Regular" }
size = 13.0

[colors.primary]
background = "#1d1f21"
foreground = "#c5c8c6"

[window]
dimensions = { columns = 120, lines = 35 }
padding = { x = 5, y = 5 }
```

**Best for**: Users who want speed and use tmux for session management.

### Kitty

Feature-rich, GPU-accelerated terminal:

```bash
$ brew install --cask kitty
```

**Characteristics**:
- GPU rendering (fast)
- Native tabs and splits
- Image display
- Ligature support
- Remote file access
- Extensive customization

Configuration (`~/.config/kitty/kitty.conf`):

```conf
font_family SF Mono
font_size 13.0
background #1a1a1a
foreground #d8d8d8
enable_audio_bell no
```

**Best for**: Users wanting features without sacrificing performance.

### Warp

Modern terminal with AI features:

```bash
$ brew install --cask warp
```

**Characteristics**:
- Block-based command interface
- AI command suggestions
- Modern IDE-like editing
- Shared workflows
- Requires account (free tier available)

**Best for**: Users who want a modern take on terminals with AI assistance.

### Hyper

Web-technology-based terminal:

```bash
$ brew install --cask hyper
```

**Characteristics**:
- Built on Electron (HTML/CSS/JS)
- Highly themeable
- Plugin ecosystem
- Cross-platform
- Higher memory usage than native apps

Configuration (`~/.hyper.js`):

```javascript
module.exports = {
    config: {
        fontSize: 13,
        fontFamily: '"SF Mono", monospace',
        cursorShape: 'BLOCK',
        shell: '/bin/zsh',
    },
    plugins: ['hyper-dracula'],
};
```

**Best for**: Web developers who want to customize with JS/CSS.

### WezTerm

GPU-accelerated, cross-platform, Lua-configured:

```bash
$ brew install --cask wezterm
```

**Characteristics**:
- GPU rendering
- Lua configuration
- Multiplexing built-in
- Cross-platform
- Active development

Configuration (`~/.wezterm.lua`):

```lua
local wezterm = require 'wezterm'
return {
    font = wezterm.font 'SF Mono',
    font_size = 13.0,
    color_scheme = 'Catppuccin Mocha',
    enable_tab_bar = true,
}
```

**Best for**: Users who want customization with a real programming language.

## Feature Comparison

| Feature | Terminal.app | iTerm2 | Alacritty | Kitty | Warp |
|---------|-------------|--------|-----------|-------|------|
| Split panes | No | Yes | No | Yes | Yes |
| GPU rendering | No | Optional | Yes | Yes | Yes |
| Tabs | Yes | Yes | No | Yes | Yes |
| Shell integration | Basic | Advanced | No | Yes | Advanced |
| Profiles | Yes | Yes | No | Yes | Yes |
| Images inline | No | Yes | Yes | Yes | Yes |
| Hotkey window | No | Yes | No | No | No |
| Cross-platform | No | No | Yes | Yes | No |
| Configuration | GUI | GUI | TOML | conf | GUI |
| Price | Free | Free | Free | Free | Freemium |

## tmux: Terminal Multiplexer

Regardless of which terminal you choose, tmux provides powerful session management:

```bash
$ brew install tmux
```

**tmux provides**:
- Sessions that persist after disconnection
- Split panes (works in any terminal)
- Multiple windows
- Session sharing
- Scriptable

Basic tmux usage:

```bash
# Start new session
$ tmux new -s work

# Detach
Ctrl+b, d

# List sessions
$ tmux ls

# Reattach
$ tmux attach -t work

# Split panes
Ctrl+b, %    # Vertical
Ctrl+b, "    # Horizontal

# Navigate panes
Ctrl+b, arrow
```

**Recommendation**: If you use Alacritty, tmux is essential. For iTerm2 or Kitty users, tmux is optional but useful for remote work.

## Choosing a Terminal

**Use Terminal.app if**:
- You want no additional software
- You're working on shared/managed Macs
- Your needs are basic

**Use iTerm2 if**:
- You want split panes and hotkey window
- You need advanced features (triggers, shell integration)
- You prefer GUI configuration
- You're a power user

**Use Alacritty/Kitty if**:
- You want maximum performance
- You prefer config-file configuration
- You use tmux
- You work cross-platform

**Use Warp if**:
- You want AI assistance
- You prefer modern IDE-like interfaces
- You're comfortable with an account requirement

## Summary

The terminal emulator choice is personal, but common paths:

1. **Start with Terminal.app** - Learn the basics
2. **Graduate to iTerm2** - Most macOS power users land here
3. **Consider alternatives** - If you have specific needs (speed, cross-platform, etc.)

iTerm2's combination of features, stability, and active development makes it the default recommendation for macOS power users. But any terminal that supports UTF-8 and 256 colors will work well for most tasks—the choice is about workflow optimization, not capability.
