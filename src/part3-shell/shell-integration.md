# Shell Integration with macOS Services

The command line on macOS doesn't exist in isolation—it can interact with Finder, Spotlight, Keychain, and other system services. This integration enables workflows that combine the precision of shell commands with the convenience of macOS features.

## Finder Integration

### Opening Finder from Terminal

```bash
# Open current directory
$ open .

# Open specific directory
$ open ~/Documents

# Reveal file in Finder
$ open -R /path/to/file.txt
```

### Getting Finder Selection

Get the currently selected files in Finder:

```bash
# Using AppleScript
$ osascript -e 'tell application "Finder" to get selection as alias list'

# More useful: get paths
$ osascript -e 'tell application "Finder"
    set sel to selection as alias list
    set output to ""
    repeat with f in sel
        set output to output & POSIX path of f & "\n"
    end repeat
    return output
end tell'
```

Create a shell function:

```bash
# Add to ~/.zshrc
finder_selection() {
    osascript -e 'tell application "Finder"
        set sel to selection as alias list
        set output to ""
        repeat with f in sel
            set output to output & POSIX path of f & "\n"
        end repeat
        return output
    end tell' 2>/dev/null | sed '/^$/d'
}

# Usage
$ finder_selection
/Users/david/Documents/file1.txt
/Users/david/Documents/file2.txt
```

### Creating Finder Aliases

```bash
# Create Finder alias (different from symlink)
$ osascript -e 'tell application "Finder" to make new alias file at desktop to POSIX file "/path/to/original"'
```

### Tags and Comments

```bash
# Get tags (using mdls)
$ mdls -name kMDItemUserTags file.txt
kMDItemUserTags = (
    Red,
    Important
)

# Set tags (using xattr - complex, better to use tools)
# Install 'tag' for easier tag management
$ brew install tag

# Add tag
$ tag -a Red file.txt

# Remove tag
$ tag -r Red file.txt

# List tags
$ tag -l file.txt

# Find files with tag
$ tag -f Red
$ mdfind "kMDItemUserTags == 'Red'"
```

### Finder Comments

```bash
# Set Finder comment
$ osascript -e 'tell application "Finder" to set comment of (POSIX file "/path/to/file" as alias) to "My comment"'

# Get Finder comment
$ mdls -name kMDItemFinderComment file.txt
```

## Keychain Access

### security Command

Access Keychain from the command line:

```bash
# Find a password
$ security find-generic-password -a "account_name" -s "service_name" -w
# -w prints just the password

# Find internet password
$ security find-internet-password -a "username" -s "server.com" -w

# Add a password
$ security add-generic-password -a "account_name" -s "service_name" -w "password123"

# Delete a password
$ security delete-generic-password -a "account_name" -s "service_name"
```

### Practical Keychain Usage

Store and retrieve secrets safely:

```bash
# Store API key
$ security add-generic-password -a "$USER" -s "my_api_key" -w "secret123"

# Retrieve in scripts
API_KEY=$(security find-generic-password -a "$USER" -s "my_api_key" -w 2>/dev/null)

# Use in environment (in ~/.zshrc.local - not version controlled)
export GITHUB_TOKEN=$(security find-generic-password -a "$USER" -s "github_token" -w 2>/dev/null)
```

### SSH Keys in Keychain

macOS can store SSH passphrases in Keychain:

```bash
# Add SSH key to agent with Keychain storage
$ ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# Configure SSH to use Keychain (~/.ssh/config)
Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519
```

## AppleScript and JavaScript for Automation

### osascript Command

Run AppleScript from the shell:

```bash
# Basic dialog
$ osascript -e 'display dialog "Hello" buttons {"OK"}'

# Get input
$ osascript -e 'text returned of (display dialog "Enter name:" default answer "")'

# Alert
$ osascript -e 'display alert "Warning" message "Something happened"'

# Choose from list
$ osascript -e 'choose from list {"Option 1", "Option 2", "Option 3"}'
```

### JavaScript for Automation (JXA)

Modern alternative to AppleScript:

```bash
# Using JXA
$ osascript -l JavaScript -e 'Application("Finder").selection().map(f => f.url())'

# Multi-line JXA
$ osascript -l JavaScript << 'EOF'
const app = Application.currentApplication();
app.includeStandardAdditions = true;
const result = app.displayDialog("Enter your name:", {
    defaultAnswer: "",
    withTitle: "Input"
});
result.textReturned;
EOF
```

### Controlling Applications

```bash
# Control iTunes/Music
$ osascript -e 'tell application "Music" to playpause'
$ osascript -e 'tell application "Music" to next track'
$ osascript -e 'tell application "Music" to get name of current track'

# Control Safari
$ osascript -e 'tell application "Safari" to get URL of current tab of window 1'

# Control System Events
$ osascript -e 'tell application "System Events" to get name of every process'
```

## Automator and Shortcuts

### Running Automator Workflows

```bash
# Run workflow
$ automator /path/to/workflow.workflow

# Run workflow with input
$ automator -i "input text" /path/to/workflow.workflow
```

### Shortcuts (macOS Monterey+)

```bash
# List shortcuts
$ shortcuts list

# Run a shortcut
$ shortcuts run "My Shortcut"

# Run with input
$ echo "input text" | shortcuts run "My Shortcut"

# View shortcut
$ shortcuts view "My Shortcut"
```

## Quick Look

### Preview Files

```bash
# Quick Look from command line
$ qlmanage -p file.pdf

# Multiple files
$ qlmanage -p *.png

# Generate thumbnail
$ qlmanage -t -s 256 file.pdf -o /tmp/thumbnails/

# Generate preview
$ qlmanage -p -s 1024 file.pdf -o /tmp/previews/
```

### Quick Look Generator Info

```bash
# List Quick Look generators
$ qlmanage -m

# Debug Quick Look for a file
$ qlmanage -d4 -p file.pdf
```

## Services Menu Integration

Create shell-accessible services:

### Creating a Service

1. Open Automator
2. Create new "Quick Action"
3. Add "Run Shell Script" action
4. Write your script
5. Save with descriptive name

The service appears in:
- Right-click context menu
- Application menu → Services

### Running Services from Shell

```bash
# Services aren't directly callable, but you can:
# 1. Use Automator to create a workflow
# 2. Run the workflow with automator command
$ automator /path/to/service.workflow
```

## Notification Center

### Built-in Notification

```bash
# Using osascript
$ osascript -e 'display notification "Message" with title "Title" subtitle "Subtitle" sound name "Glass"'
```

### terminal-notifier

More powerful notification tool:

```bash
# Install
$ brew install terminal-notifier

# Basic notification
$ terminal-notifier -title "Build" -message "Complete"

# With actions
$ terminal-notifier -title "Alert" -message "Click me" \
    -open "https://apple.com"

# Execute command on click
$ terminal-notifier -title "Alert" -message "Click to run" \
    -execute "echo 'clicked' >> /tmp/log"

# With buttons (macOS 10.9+)
$ terminal-notifier -title "Question" -message "Choose:" \
    -actions "Yes,No" -closeLabel "Maybe"

# Notify when command completes
$ long_command; terminal-notifier -message "Done" -title "Task Complete"
```

## Calendar and Reminders

### Calendar Events

```bash
# List calendars
$ osascript -e 'tell application "Calendar" to get name of every calendar'

# Create event
$ osascript << 'EOF'
tell application "Calendar"
    tell calendar "Work"
        make new event with properties {
            summary: "Meeting",
            start date: (current date) + 1 * days,
            end date: (current date) + 1 * days + 1 * hours
        }
    end tell
end tell
EOF
```

### Reminders

```bash
# Create reminder
$ osascript -e 'tell application "Reminders" to make new reminder with properties {name: "Buy milk"}'

# Create with due date
$ osascript << 'EOF'
tell application "Reminders"
    tell list "Tasks"
        make new reminder with properties {
            name: "Important task",
            due date: (current date) + 2 * days
        }
    end tell
end tell
EOF
```

## Location Services

### Core Location

```bash
# Get current location (requires permission)
# Create and compile a Swift snippet or use:
$ osascript -e 'tell application "System Events" to get the current location'
# Note: Often restricted

# Alternative: Use whereami tool
$ brew install whereami
$ whereami
Latitude: 37.7749
Longitude: -122.4194
```

## Integration Scripts

### Complete Example: Project Opener

Create a script that integrates multiple services:

```bash
#!/bin/zsh
# project-open: Open a project with all necessary tools

project_dir="$1"

if [[ ! -d "$project_dir" ]]; then
    terminal-notifier -title "Error" -message "Directory not found"
    exit 1
fi

cd "$project_dir"

# Open in VS Code
open -a "Visual Studio Code" .

# Open Finder
open .

# Open terminal tab
osascript -e "tell application \"Terminal\"
    do script \"cd '$project_dir' && git status\"
end tell"

# Notify
terminal-notifier -title "Project Opened" \
    -message "$(basename $project_dir)" \
    -open "file://$project_dir"

# Play sound
afplay /System/Library/Sounds/Glass.aiff
```

### Clipboard Workflow

```bash
#!/bin/zsh
# Process clipboard contents

# Get clipboard
content=$(pbpaste)

# Transform (example: convert to uppercase)
transformed=$(echo "$content" | tr '[:lower:]' '[:upper:]')

# Put back
echo "$transformed" | pbcopy

# Notify
osascript -e 'display notification "Clipboard transformed" with title "Done"'
```

### File Watcher with Actions

```bash
#!/bin/zsh
# Watch directory and act on changes

watch_dir="$HOME/Downloads"

fswatch -0 "$watch_dir" | while read -d "" event; do
    filename=$(basename "$event")

    case "$filename" in
        *.pdf)
            # Move PDFs to Documents
            mv "$event" ~/Documents/PDFs/
            terminal-notifier -message "PDF filed: $filename"
            ;;
        *.zip)
            # Extract ZIPs
            unzip -d "${event%.zip}" "$event"
            terminal-notifier -message "Extracted: $filename"
            ;;
    esac
done
```

## Summary

macOS shell integration enables powerful workflows:

| Service | Command/Method |
|---------|----------------|
| Finder | `open`, osascript |
| Keychain | `security` |
| AppleScript | `osascript` |
| Notifications | osascript, terminal-notifier |
| Quick Look | `qlmanage` |
| Spotlight | `mdfind`, `mdls` |
| Shortcuts | `shortcuts` |
| Calendar | osascript |
| Reminders | osascript |

Key integration patterns:
1. Use `osascript` for GUI application control
2. Use `security` for credential management
3. Use `terminal-notifier` for user feedback
4. Combine tools in shell scripts for workflows

The command line on macOS isn't separate from the graphical system—it's deeply integrated, enabling automation that bridges both worlds.
