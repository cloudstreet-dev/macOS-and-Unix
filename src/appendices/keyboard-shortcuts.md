# Appendix C: Keyboard Shortcuts

This appendix covers keyboard shortcuts for Terminal.app, command-line editing, and common terminal emulators on macOS.

---

## Terminal.app Shortcuts

### Window and Tab Management

| Shortcut | Action |
|----------|--------|
| `Cmd+N` | New window |
| `Cmd+T` | New tab |
| `Cmd+Shift+N` | New window with same command |
| `Cmd+W` | Close tab/window |
| `Cmd+Shift+W` | Close window |
| `Cmd+1-9` | Switch to tab 1-9 |
| `Cmd+Shift+[` | Previous tab |
| `Cmd+Shift+]` | Next tab |
| `Cmd+Left Arrow` | Previous tab |
| `Cmd+Right Arrow` | Next tab |
| `Cmd+Shift+D` | Split pane horizontally |
| `Cmd+D` | Split pane vertically (some configs) |
| `Cmd+Shift+Enter` | Toggle full screen |
| `Cmd+Ctrl+F` | Toggle full screen |

### Text and Display

| Shortcut | Action |
|----------|--------|
| `Cmd++` or `Cmd+=` | Increase font size |
| `Cmd+-` | Decrease font size |
| `Cmd+0` | Reset font size to default |
| `Cmd+K` | Clear screen and scrollback |
| `Cmd+L` | Clear screen (keep scrollback) |
| `Ctrl+L` | Clear screen (shell command) |
| `Cmd+Home` | Scroll to top |
| `Cmd+End` | Scroll to bottom |
| `Page Up` | Scroll up one page |
| `Page Down` | Scroll down one page |
| `Cmd+Up Arrow` | Scroll up one line |
| `Cmd+Down Arrow` | Scroll down one line |

### Selection and Clipboard

| Shortcut | Action |
|----------|--------|
| `Cmd+A` | Select all |
| `Cmd+C` | Copy selection |
| `Cmd+V` | Paste |
| `Cmd+Shift+V` | Paste escaped (for URLs, paths) |
| `Double-click` | Select word |
| `Triple-click` | Select line |
| `Cmd+Click` | Open URL in browser |
| `Option+Click` | Position cursor at click location |
| `Cmd+Drag` | Rectangular selection |

### Search

| Shortcut | Action |
|----------|--------|
| `Cmd+F` | Find |
| `Cmd+G` | Find next |
| `Cmd+Shift+G` | Find previous |
| `Cmd+E` | Use selection for find |
| `Cmd+J` | Jump to selection |

### Marks and Bookmarks

| Shortcut | Action |
|----------|--------|
| `Cmd+U` | Mark current line |
| `Cmd+Shift+U` | Mark line and send return |
| `Cmd+Shift+M` | Insert bookmark |
| `Cmd+Up Arrow` | Jump to previous mark |
| `Cmd+Down Arrow` | Jump to next mark |
| `Cmd+Shift+A` | Select between marks |

---

## Shell Line Editing (Emacs Mode)

Most shells (bash, zsh) use Emacs-style keybindings by default. These work in Terminal.app and other terminal emulators.

### Cursor Movement

| Shortcut | Action |
|----------|--------|
| `Ctrl+A` | Move to beginning of line |
| `Ctrl+E` | Move to end of line |
| `Ctrl+F` | Move forward one character |
| `Ctrl+B` | Move backward one character |
| `Option+F` or `Esc F` | Move forward one word |
| `Option+B` or `Esc B` | Move backward one word |
| `Ctrl+XX` | Toggle between start of line and current position |

> **Note**: On macOS, `Option+F` and `Option+B` may require enabling "Use Option as Meta key" in Terminal preferences, or use `Esc` followed by the letter.

### Deletion

| Shortcut | Action |
|----------|--------|
| `Ctrl+D` | Delete character under cursor (or logout if empty line) |
| `Ctrl+H` | Delete character before cursor (backspace) |
| `Ctrl+W` | Delete word before cursor |
| `Option+D` or `Esc D` | Delete word after cursor |
| `Ctrl+U` | Delete from cursor to beginning of line |
| `Ctrl+K` | Delete from cursor to end of line |
| `Ctrl+Y` | Paste (yank) last deleted text |
| `Option+Y` | Cycle through kill ring |

### Text Manipulation

| Shortcut | Action |
|----------|--------|
| `Ctrl+T` | Transpose characters (swap current and previous) |
| `Option+T` or `Esc T` | Transpose words |
| `Option+U` or `Esc U` | Uppercase word from cursor |
| `Option+L` or `Esc L` | Lowercase word from cursor |
| `Option+C` or `Esc C` | Capitalize word from cursor |

### History Navigation

| Shortcut | Action |
|----------|--------|
| `Ctrl+P` or `Up Arrow` | Previous command in history |
| `Ctrl+N` or `Down Arrow` | Next command in history |
| `Ctrl+R` | Reverse incremental search |
| `Ctrl+S` | Forward incremental search (may need to enable) |
| `Ctrl+G` | Cancel search and restore original line |
| `Option+<` or `Esc <` | First command in history |
| `Option+>` or `Esc >` | Last command in history |
| `Ctrl+O` | Execute command and fetch next from history |
| `!!` | Repeat last command (type and press Enter) |
| `!n` | Repeat command number n from history |
| `!string` | Repeat last command starting with string |
| `!?string` | Repeat last command containing string |

### Process Control

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Interrupt (SIGINT) - cancel current command |
| `Ctrl+Z` | Suspend (SIGTSTP) - pause current command |
| `Ctrl+D` | End of file (EOF) - logout or close input |
| `Ctrl+\` | Quit (SIGQUIT) - forceful termination |
| `Ctrl+S` | Pause output (XOFF) |
| `Ctrl+Q` | Resume output (XON) |

### Completion

| Shortcut | Action |
|----------|--------|
| `Tab` | Auto-complete command, filename, or variable |
| `Tab Tab` | Show all completions |
| `Option+=` or `Esc =` | List possible completions |
| `Option+*` or `Esc *` | Insert all completions |
| `Option+/` or `Esc /` | Complete filename |
| `Ctrl+X /` | List possible filename completions |

### Screen Control

| Shortcut | Action |
|----------|--------|
| `Ctrl+L` | Clear screen, redraw current line at top |
| `Ctrl+S` | Stop output to screen |
| `Ctrl+Q` | Resume output to screen |

---

## Zsh-Specific Shortcuts

Zsh includes additional features beyond standard Emacs bindings:

### Zsh Expansion

| Shortcut | Action |
|----------|--------|
| `Tab` | Complete and show menu if ambiguous |
| `Ctrl+I` | Same as Tab |
| `Shift+Tab` | Reverse through completions |
| `Option+H` or `Esc H` | Run help for current command |
| `Option+?` | Show command help |
| `Ctrl+X A` | Expand alias |
| `Ctrl+X G` | List expansions of current glob |
| `Ctrl+X *` | Expand glob inline |

### Zsh History

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Incremental history search |
| `Ctrl+P` / `Ctrl+N` | Navigate history |
| `Option+P` | History search backward (prefix match) |
| `Option+N` | History search forward (prefix match) |
| `fc` | Edit last command in editor |
| `r` | Re-run last command |
| `r foo=bar` | Re-run last command, replacing foo with bar |

### Zsh Line Editor (ZLE)

| Shortcut | Action |
|----------|--------|
| `Ctrl+X Ctrl+E` | Edit command line in $EDITOR |
| `Option+Q` | Push line to buffer, clear, execute next command, then restore |
| `Option+'` | Quote line |
| `Option+"` | Quote region |
| `Ctrl+X Ctrl+V` | Show zsh version |

---

## Vi Mode

Both bash and zsh support vi-style editing. Enable with:

```bash
# Bash
set -o vi

# Zsh
bindkey -v
```

### Vi Command Mode (Press Escape first)

| Key | Action |
|-----|--------|
| `h` | Move left |
| `l` | Move right |
| `w` | Move forward one word |
| `b` | Move backward one word |
| `e` | Move to end of word |
| `0` | Move to beginning of line |
| `$` | Move to end of line |
| `^` | Move to first non-blank character |
| `x` | Delete character |
| `dw` | Delete word |
| `dd` | Delete line |
| `d$` or `D` | Delete to end of line |
| `d0` | Delete to beginning of line |
| `cw` | Change word |
| `cc` | Change line |
| `c$` or `C` | Change to end of line |
| `yy` | Yank (copy) line |
| `yw` | Yank word |
| `p` | Paste after cursor |
| `P` | Paste before cursor |
| `u` | Undo |
| `Ctrl+R` | Redo |
| `i` | Insert mode at cursor |
| `I` | Insert at beginning of line |
| `a` | Append after cursor |
| `A` | Append at end of line |
| `r` | Replace single character |
| `R` | Replace mode |
| `k` | Previous history |
| `j` | Next history |
| `/` | Search forward in history |
| `?` | Search backward in history |
| `n` | Repeat search |
| `N` | Repeat search in reverse |
| `v` | Edit command in $EDITOR |

### Vi Insert Mode

| Key | Action |
|-----|--------|
| `Escape` | Return to command mode |
| `Ctrl+[` | Return to command mode |
| `Ctrl+C` | Cancel and return to command mode |

---

## iTerm2 Shortcuts

iTerm2 provides additional shortcuts beyond Terminal.app:

### Windows and Tabs

| Shortcut | Action |
|----------|--------|
| `Cmd+N` | New window |
| `Cmd+T` | New tab |
| `Cmd+W` | Close tab |
| `Cmd+Shift+W` | Close window |
| `Cmd+Option+W` | Close all tabs except current |
| `Cmd+1-9` | Switch to tab |
| `Cmd+Left/Right` | Previous/next tab |
| `Cmd+Shift+Enter` | Maximize pane |
| `Cmd+Option+E` | Expose all tabs |

### Panes (Split View)

| Shortcut | Action |
|----------|--------|
| `Cmd+D` | Split vertically |
| `Cmd+Shift+D` | Split horizontally |
| `Cmd+Option+Arrow` | Navigate between panes |
| `Cmd+]` | Next pane |
| `Cmd+[` | Previous pane |
| `Cmd+Shift+Enter` | Toggle pane zoom |
| `Cmd+Option+Shift+H/V` | Move divider |

### Search and Selection

| Shortcut | Action |
|----------|--------|
| `Cmd+F` | Find |
| `Cmd+Shift+H` | Paste history |
| `Cmd+;` | Autocomplete |
| `Cmd+Shift+;` | Open command history |
| `Cmd+Option+/` | Recent directories popup |
| `Cmd+Click` | Open URL/file |
| `Cmd+Option+B` | Instant replay |

### Text

| Shortcut | Action |
|----------|--------|
| `Cmd+K` | Clear buffer |
| `Cmd+Ctrl+K` | Clear scrollback |
| `Cmd+/` | Find cursor |
| `Cmd+Option+;` | Open command history |
| `Cmd+Shift+M` | Set mark |
| `Cmd+Shift+J` | Jump to mark |

### Profiles and Settings

| Shortcut | Action |
|----------|--------|
| `Cmd+I` | Edit session |
| `Cmd+,` | Preferences |
| `Cmd+Option+I` | Toggle broadcast input |
| `Cmd+Shift+O` | Open quickly (fuzzy search tabs) |

---

## Less Pager Shortcuts

When viewing files with `less`:

| Key | Action |
|-----|--------|
| `Space` or `f` | Forward one page |
| `b` | Backward one page |
| `d` | Forward half page |
| `u` | Backward half page |
| `j` or `Down` | Forward one line |
| `k` or `Up` | Backward one line |
| `g` or `Home` | Go to beginning |
| `G` or `End` | Go to end |
| `/<pattern>` | Search forward |
| `?<pattern>` | Search backward |
| `n` | Next search match |
| `N` | Previous search match |
| `&<pattern>` | Show only matching lines |
| `m<letter>` | Mark current position |
| `'<letter>` | Go to mark |
| `F` | Follow mode (like tail -f) |
| `v` | Open in $EDITOR |
| `-N` | Toggle line numbers |
| `-S` | Toggle line wrapping |
| `h` | Help |
| `q` | Quit |

---

## Man Page Shortcuts

Man pages use `less` by default, but some additional keys work:

| Key | Action |
|-----|--------|
| `h` | Help |
| `q` | Quit |
| `Space` | Next page |
| `b` | Previous page |
| `/` | Search |
| `n` | Next match |
| `N` | Previous match |

---

## Vim Quick Reference

For quick edits in vim:

### Normal Mode

| Key | Action |
|-----|--------|
| `i` | Insert before cursor |
| `a` | Insert after cursor |
| `o` | Insert new line below |
| `O` | Insert new line above |
| `x` | Delete character |
| `dd` | Delete line |
| `yy` | Copy line |
| `p` | Paste |
| `u` | Undo |
| `Ctrl+R` | Redo |
| `/` | Search |
| `n` | Next match |
| `:w` | Save |
| `:q` | Quit |
| `:wq` | Save and quit |
| `:q!` | Quit without saving |
| `ZZ` | Save and quit |
| `ZQ` | Quit without saving |

---

## Quick Reference Card

### Essential Shortcuts (Memorize These)

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel/interrupt |
| `Ctrl+D` | Exit/EOF |
| `Ctrl+Z` | Suspend |
| `Ctrl+L` | Clear screen |
| `Ctrl+A` | Start of line |
| `Ctrl+E` | End of line |
| `Ctrl+U` | Delete to start |
| `Ctrl+K` | Delete to end |
| `Ctrl+W` | Delete word |
| `Ctrl+R` | Search history |
| `Tab` | Autocomplete |
| `Up/Down` | History navigation |

### Terminal.app Essentials

| Shortcut | Action |
|----------|--------|
| `Cmd+T` | New tab |
| `Cmd+W` | Close tab |
| `Cmd+1-9` | Switch tab |
| `Cmd+K` | Clear all |
| `Cmd+F` | Find |
| `Cmd++/-` | Font size |

### Process Control

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Kill foreground |
| `Ctrl+Z` | Suspend |
| `bg` | Continue in background |
| `fg` | Bring to foreground |
| `jobs` | List background jobs |
