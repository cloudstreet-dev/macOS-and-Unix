# Understanding Property Lists (plists)

Property lists are macOS's standard format for structured data. launchd configurations, application preferences, and system settings all use plists. Understanding how to read, create, and modify plists is essential for macOS system administration.

## Plist Formats

Property lists can be stored in multiple formats:

| Format | Extension | Human Readable | Performance |
|--------|-----------|----------------|-------------|
| XML | .plist | Yes | Slower |
| Binary | .plist | No | Faster |
| JSON | .json | Yes | Medium |

```bash
# Check plist format
$ file /Library/LaunchDaemons/com.apple.example.plist
/Library/LaunchDaemons/com.apple.example.plist: XML 1.0 document text, ASCII text

$ file /System/Library/LaunchDaemons/com.apple.mds.plist
/System/Library/LaunchDaemons/com.apple.mds.plist: Apple binary property list
```

## XML Plist Structure

### Basic Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.myjob</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myscript</string>
        <string>--flag</string>
        <string>argument</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>300</integer>
</dict>
</plist>
```

### Data Types

| Type | XML Element | Example |
|------|-------------|---------|
| String | `<string>` | `<string>hello</string>` |
| Integer | `<integer>` | `<integer>42</integer>` |
| Real | `<real>` | `<real>3.14</real>` |
| Boolean | `<true/>` or `<false/>` | `<true/>` |
| Date | `<date>` | `<date>2024-01-15T10:30:00Z</date>` |
| Data (binary) | `<data>` | `<data>SGVsbG8=</data>` |
| Array | `<array>` | See below |
| Dictionary | `<dict>` | See below |

### Arrays

```xml
<key>MyArray</key>
<array>
    <string>item1</string>
    <string>item2</string>
    <integer>42</integer>
</array>
```

### Nested Dictionaries

```xml
<key>ParentDict</key>
<dict>
    <key>ChildKey</key>
    <string>value</string>
    <key>NestedDict</key>
    <dict>
        <key>DeepKey</key>
        <string>deep value</string>
    </dict>
</dict>
```

## Command-Line Tools

### plutil

Validate and convert plists:

```bash
# Validate syntax
$ plutil /path/to/file.plist
/path/to/file.plist: OK

# Convert to XML
$ plutil -convert xml1 binary.plist

# Convert to binary
$ plutil -convert binary1 file.plist

# Convert to JSON
$ plutil -convert json file.plist -o file.json

# Pretty print (plutil -p)
$ plutil -p file.plist
{
  "Label" => "com.example.myjob"
  "ProgramArguments" => [
    0 => "/usr/local/bin/myscript"
    1 => "--flag"
  ]
  "RunAtLoad" => 1
}
```

### defaults

Read and write preference plists:

```bash
# Read all values
$ defaults read com.apple.finder

# Read specific key
$ defaults read com.apple.finder ShowExternalHardDrivesOnDesktop
1

# Write value
$ defaults write com.apple.finder ShowExternalHardDrivesOnDesktop -bool false

# Delete key
$ defaults delete com.apple.finder ShowExternalHardDrivesOnDesktop

# Read arbitrary plist file
$ defaults read /path/to/file.plist

# Type specifiers: -string, -int, -float, -bool, -data, -array, -dict
$ defaults write com.example.app count -int 42
$ defaults write com.example.app enabled -bool true
```

### PlistBuddy

More powerful plist editing:

```bash
# Read value
$ /usr/libexec/PlistBuddy -c "Print :Label" file.plist
com.example.myjob

# Write value
$ /usr/libexec/PlistBuddy -c "Set :StartInterval 600" file.plist

# Add new key
$ /usr/libexec/PlistBuddy -c "Add :NewKey string 'value'" file.plist

# Add array item
$ /usr/libexec/PlistBuddy -c "Add :ProgramArguments: string '--newarg'" file.plist

# Delete key
$ /usr/libexec/PlistBuddy -c "Delete :UnwantedKey" file.plist

# Nested access
$ /usr/libexec/PlistBuddy -c "Print :EnvironmentVariables:PATH" file.plist

# Interactive mode
$ /usr/libexec/PlistBuddy file.plist
Command: Print
Command: Set :Label com.example.newname
Command: Save
Command: Exit
```

## Creating Launch Agent Plists

### Minimal Agent

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.minimal</string>
    <key>Program</key>
    <string>/usr/local/bin/myprogram</string>
</dict>
</plist>
```

### Agent with Arguments

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.withargs</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myprogram</string>
        <string>--config</string>
        <string>/etc/myconfig.conf</string>
        <string>--verbose</string>
    </array>
</dict>
</plist>
```

### Full-Featured Agent

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.full</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/myprogram</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/david</string>
    </dict>

    <key>WorkingDirectory</key>
    <string>/Users/david/project</string>

    <key>StandardOutPath</key>
    <string>/tmp/myprogram.stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/myprogram.stderr.log</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>ThrottleInterval</key>
    <integer>10</integer>
</dict>
</plist>
```

## Validation and Testing

### Validate Syntax

```bash
# Check plist syntax
$ plutil -lint file.plist
file.plist: OK

# Detailed check
$ launchctl print gui/$(id -u)/com.example.myjob 2>&1 | head
```

### Common Errors

**Bad permissions**:
```bash
# Agents must be owned by user
$ ls -la ~/Library/LaunchAgents/com.example.plist
-rw-r--r--  1 david  staff  500 Jan 15 10:00 com.example.plist

# Daemons must be owned by root
$ ls -la /Library/LaunchDaemons/com.example.plist
-rw-r--r--  1 root  wheel  500 Jan 15 10:00 com.example.plist
```

**Missing Label**:
```
launchctl: Error unloading: com.example.myjob: No such process
```

**Invalid Program Path**:
```bash
# Check path exists and is executable
$ ls -la /usr/local/bin/myprogram
$ file /usr/local/bin/myprogram
```

## Editing Tips

### Using Xcode

Xcode provides a plist editor:
1. Open plist file in Xcode
2. Visual editor shows keys and values
3. Right-click to add/remove entries

### Using Text Editor

For XML plists:
```bash
$ nano ~/Library/LaunchAgents/com.example.plist
# or
$ code ~/Library/LaunchAgents/com.example.plist
```

### Convert Binary to Edit

```bash
# Convert to XML for editing
$ plutil -convert xml1 /path/to/file.plist

# Edit...

# Convert back to binary (optional, for performance)
$ plutil -convert binary1 /path/to/file.plist
```

## Summary

Plist essentials:

| Tool | Use Case |
|------|----------|
| `plutil` | Validate, convert formats |
| `defaults` | Read/write app preferences |
| `PlistBuddy` | Complex edits, scripting |

Key points:
- XML format is human-readable
- Binary format is faster
- Always validate after editing
- Check permissions for launchd plists
- Use proper data types
