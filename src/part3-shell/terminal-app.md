# Terminal.app Deep Dive

Terminal.app is macOS's built-in terminal emulator. While many power users immediately install iTerm2, Terminal.app is surprisingly capable. Understanding its features helps you work effectively on any Mac, even when you can't install third-party software.

## Finding and Launching Terminal

```bash
# Terminal is in Utilities
/Applications/Utilities/Terminal.app

# Quick access:
# - Spotlight: Cmd+Space, type "Terminal"
# - Finder: Go → Utilities → Terminal
# - Launchpad: Other folder → Terminal
```

### Creating Quick Access

Add Terminal to your Dock by dragging it from Applications/Utilities, or create a keyboard shortcut:

**System Preferences → Keyboard → Shortcuts → Services**
- Enable "New Terminal at Folder"
- Enable "New Terminal Tab at Folder"

Now you can right-click any folder and open Terminal there.

## Terminal Preferences

Access with `Cmd+,` or Terminal → Preferences:

### General Tab

**On startup, open**: Choose default behavior
- New window with profile
- New window with same profile (continues last session's appearance)

**Shells open with**:
- Default login shell (`/bin/zsh`)
- Command (specify custom shell or command)

**New tabs open with**:
- Default Working Directory
- Same Working Directory as current tab

### Profiles Tab

Terminal uses "profiles" for appearance and behavior settings. The default profile is "Basic."

#### Creating Custom Profiles

1. Click `+` to add a new profile
2. Name it (e.g., "Development")
3. Configure settings
4. Set as default with "Default" button

#### Profile Settings

**Text Tab**:
- Font: Choose monospace font (Menlo, SF Mono, etc.)
- Font size: Typically 12-14pt
- Text and bold colors
- Cursor: Block, underline, or vertical bar
- Cursor blink

```bash
# View current terminal size
$ echo "Columns: $COLUMNS, Rows: $LINES"
Columns: 80, Rows: 24
```

**Window Tab**:
- Window size (columns × rows)
- Background color/image
- Blur and transparency
- Title: What appears in title bar

**Shell Tab**:
- Startup command (run on shell start)
- Ask before closing (prevents accidental closure)
- Close if shell exited cleanly

**Keyboard Tab**:
- Use Option as Meta key (important for Emacs, many CLI tools)
- Function key behavior

**Advanced Tab**:
- Terminal type: `xterm-256color` (recommended for color support)
- International encoding: UTF-8
- Scroll behavior

### Recommended Settings

For a good development experience:

1. **Font**: SF Mono or Menlo, 13pt
2. **Window size**: 120×35 or larger
3. **Terminal type**: `xterm-256color`
4. **Option as Meta**: Enable (for Alt key bindings)
5. **Scroll**: Enable "Scroll to bottom on input"

## Working with Windows and Tabs

### Keyboard Shortcuts

| Action | Shortcut |
|--------|----------|
| New window | `Cmd+N` |
| New tab | `Cmd+T` |
| Close tab/window | `Cmd+W` |
| Next tab | `Cmd+Shift+]` or `Ctrl+Tab` |
| Previous tab | `Cmd+Shift+[` or `Ctrl+Shift+Tab` |
| Specific tab | `Cmd+1` through `Cmd+9` |
| Split pane (macOS Catalina+) | Not built-in (use iTerm2) |

### Window Groups

Save and restore window arrangements:

1. Arrange windows as desired
2. Window → Save Windows as Group
3. Name the group
4. Restore: Window → Open Window Group

### Tab Bar

Show tab bar even with single tab:
- View → Show Tab Bar
- Or: `Cmd+Shift+T` toggles tab bar visibility

## Text Selection and Editing

### Selection Shortcuts

| Action | Method |
|--------|--------|
| Select word | Double-click |
| Select line | Triple-click |
| Select URL | Cmd+Double-click |
| Rectangular selection | Hold Option, drag |
| Extend selection | Shift+click |

### Copy and Paste

```bash
# Standard shortcuts
Cmd+C          # Copy
Cmd+V          # Paste
Cmd+Shift+V    # Paste escaped (safe for commands)
```

**Paste Escaped** (`Cmd+Shift+V`): Escapes special characters, preventing accidental command execution when pasting untrusted content.

### Quick Look

Select text, press `Cmd+Y` to Quick Look (preview) URLs or file paths.

## Marks and Navigation

Terminal maintains "marks" at each command prompt:

### Using Marks

| Action | Shortcut |
|--------|----------|
| Jump to previous mark | `Cmd+Up` |
| Jump to next mark | `Cmd+Down` |
| Select between marks | `Cmd+Shift+Up/Down` |
| Bookmark current line | `Cmd+U` |
| Jump to bookmark | `Cmd+Option+U` |

### Searching

| Action | Shortcut |
|--------|----------|
| Find | `Cmd+F` |
| Find next | `Cmd+G` |
| Find previous | `Cmd+Shift+G` |
| Use selection for find | `Cmd+E` |

## Colors and Themes

### Built-in Profiles

Terminal includes several profiles:
- Basic (black on white)
- Grass (green on green)
- Homebrew (green on black, classic)
- Novel (warm, paper-like)
- Ocean (blue theme)
- Pro (semi-transparent dark)
- Red Sands (warm red tones)
- Silver Aerogel (metallic)
- Solid Colors (various)

### Importing Themes

Download `.terminal` files and double-click to import, or:

1. Terminal → Preferences → Profiles
2. Click gear icon → Import
3. Select `.terminal` file

Popular theme sources:
- [GitHub: lysyi3m/macos-terminal-themes](https://github.com/lysyi3m/macos-terminal-themes)
- [Terminal.Sexy](https://terminal.sexy)
- [Gogh Themes](https://gogh-co.github.io/Gogh/)

### Creating Custom Colors

1. Preferences → Profiles → Text
2. Click color swatches to customize
3. For background: Window tab

### 256-Color Support

Ensure "Declare terminal as" is set to `xterm-256color`:

```bash
# Test 256-color support
$ for i in {0..255}; do printf "\e[48;5;${i}m  \e[0m"; done; echo
```

## Shell Integration

macOS Terminal has built-in shell integration (since El Capitan):

### What Shell Integration Provides

- Marks at each prompt (for navigation)
- Current directory tracking (new tabs open in same directory)
- Command exit status indication
- Notification on long-running command completion

### Enabling Shell Integration

Shell integration is automatic with the default zsh configuration. For bash or custom configurations:

```bash
# In .zshrc or .bashrc
# This is usually automatic, but can be explicit:
if [[ "$TERM_PROGRAM" == "Apple_Terminal" ]]; then
    # Terminal is managing shell integration
fi
```

### Directory Tracking

With shell integration, Terminal knows your working directory:

```bash
# New tab opens in same directory
# Cmd+T → new tab at ~/Documents (if that's where you were)

# Title bar shows current directory
```

### Command Completion Notifications

Terminal can notify you when long commands finish:

1. Edit → Notify When Running Process Completes
2. Or: View → Allow Notifications

## Touch Bar Support

On MacBook Pro with Touch Bar:

- Man page viewer
- Color pickers
- Autocomplete suggestions

Customize in System Preferences → Keyboard → Customize Touch Bar.

## Secure Input

For entering passwords securely:

1. Terminal → Secure Keyboard Entry (or `Cmd+Shift+S`)
2. Prevents other applications from reading keystrokes
3. Disable when done to allow normal functionality

```bash
# Icon in title bar indicates secure input mode
```

## Working with Files

### Opening Files

```bash
# Open in default application
$ open document.pdf
$ open image.png

# Open in specific application
$ open -a Preview image.png
$ open -a "Visual Studio Code" file.txt

# Reveal in Finder
$ open -R file.txt
```

### Drag and Drop

- Drag files to Terminal to insert their paths
- Paths are automatically quoted if they contain spaces

### Services

Right-click selected text for Services menu:
- Open man page
- Search in Spotlight
- Look up in Dictionary

## Printing

Print terminal content:

1. Make selection (optional—prints all if nothing selected)
2. File → Print or `Cmd+P`
3. Options: Include timestamp, color, etc.

## Accessibility

### VoiceOver Support

Terminal works with VoiceOver (screen reader):
- `Cmd+F5` to enable VoiceOver
- Reads command output
- Navigate with VoiceOver commands

### Font Smoothing

For better readability:
- System Preferences → Accessibility → Display
- Adjust contrast and transparency

### Window Zoom

- `Cmd++` / `Cmd+-` to zoom
- View → Reset Zoom to default

## Automation

### AppleScript

Terminal is scriptable:

```applescript
tell application "Terminal"
    do script "echo 'Hello from AppleScript'"
    do script "cd ~/Documents" in front window
end tell
```

### Opening URLs

Terminal recognizes URLs:
- `Cmd+Click` to open in browser
- URLs are underlined when hovering

### Custom URL Schemes

Create custom schemes to open Terminal from other apps:

```bash
# URL: x-terminal://open?command=ls%20-la
```

## Troubleshooting

### Reset Terminal

If settings get corrupted:

```bash
# Reset to factory defaults
$ defaults delete com.apple.Terminal
# Restart Terminal
```

### Slow Startup

If Terminal is slow to open:

```bash
# Clear ASL logs
$ sudo rm -rf /var/log/asl/*.asl

# Check shell startup files for slow operations
# Time your shell startup:
$ time zsh -i -c exit
```

### TERM Variable Issues

If applications look wrong:

```bash
# Check TERM
$ echo $TERM
xterm-256color

# Set in Preferences → Profiles → Advanced
# Or in shell config:
export TERM=xterm-256color
```

## Summary

Terminal.app capabilities:

| Feature | Available |
|---------|-----------|
| 256 colors | Yes |
| Unicode/UTF-8 | Yes |
| Tabs | Yes |
| Split panes | No (use iTerm2) |
| Shell integration | Yes |
| Profiles | Yes |
| Touch Bar | Yes |
| Secure input | Yes |
| AppleScript | Yes |

Terminal.app is a solid terminal emulator that handles most needs. For advanced features like split panes, triggers, or extensive customization, consider iTerm2. But for many users, Terminal.app is sufficient and has the advantage of requiring no additional installation.
