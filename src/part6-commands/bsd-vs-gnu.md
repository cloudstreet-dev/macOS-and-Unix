# BSD vs GNU: The Command Divide

The same command name doesn't mean the same behavior. macOS uses BSD-derived tools, while Linux uses GNU tools. These two families share common ancestry but have diverged significantly. Understanding these differences is essential for anyone working across both platforms.

## Historical Context

The split dates back to the 1980s:

- **BSD tools**: Developed at UC Berkeley, focused on simplicity and POSIX compliance
- **GNU tools**: Created by the Free Software Foundation, adding many extensions

macOS inherited BSD tools through its NeXTSTEP lineage, while Linux adopted GNU tools because they were freely available and feature-rich.

## The Major Differences

### sed: In-Place Editing

The most commonly encountered difference:

```bash
# GNU sed (Linux)
$ sed -i 's/old/new/g' file.txt

# BSD sed (macOS) - requires backup extension argument
$ sed -i '' 's/old/new/g' file.txt    # No backup
$ sed -i '.bak' 's/old/new/g' file.txt  # Creates file.txt.bak
```

BSD sed's `-i` flag requires an argument specifying the backup file extension. An empty string `''` means no backup. Omitting this argument entirely causes the substitution pattern to be interpreted as the backup extension—hence the confusing error messages.

More sed differences:

```bash
# Insert a newline (GNU sed)
$ echo "hello" | sed 's/$/\n/'

# Insert a newline (BSD sed) - \n doesn't work
$ echo "hello" | sed 's/$/\'$'\n''/'   # Using $'...' quoting
# Or use a literal newline:
$ echo "hello" | sed 's/$/\
/'

# Case-insensitive matching (GNU sed)
$ sed 's/foo/bar/gi' file.txt

# BSD sed doesn't support the 'i' flag
# Use tr or awk instead, or install GNU sed
```

### grep: Regular Expression Flavors

```bash
# Perl-compatible regex (GNU grep only)
$ grep -P '\d{3}-\d{4}' file.txt    # Match phone numbers
grep: invalid option -- P

# BSD grep doesn't have -P flag
# Use extended regex instead
$ grep -E '[0-9]{3}-[0-9]{4}' file.txt
```

The `-P` flag for Perl-compatible regular expressions is a GNU extension. BSD grep supports `-E` (extended) and `-G` (basic) regex only.

Other grep differences:

```bash
# GNU grep colors matches by default (usually via alias)
$ grep --color=auto pattern file

# BSD grep requires explicit color flag
$ grep --color pattern file    # Works, but not default

# Line buffering (GNU)
$ tail -f log.txt | grep --line-buffered error

# BSD grep has --line-buffered too (one area of compatibility)
$ tail -f log.txt | grep --line-buffered error
```

### ls: Display Options

```bash
# Colored output (GNU)
$ ls --color=auto

# Colored output (BSD/macOS)
$ ls -G

# Human-readable sizes (both support -h)
$ ls -lh

# Sort by modification time (both support -t)
$ ls -lt

# GNU-specific options that BSD lacks:
$ ls --group-directories-first  # BSD: illegal option
$ ls --time-style=long-iso      # BSD: illegal option
```

### date: Format and Parsing

Date handling differs dramatically:

```bash
# Parse a date string (GNU)
$ date -d "2024-01-15" +%s
1705276800

# Parse a date string (BSD)
$ date -j -f "%Y-%m-%d" "2024-01-15" +%s
1705276800

# Show date N days ago (GNU)
$ date -d "7 days ago"
Mon Jan  8 10:00:00 PST 2024

# Show date N days ago (BSD)
$ date -v-7d
Mon Jan  8 10:00:00 PST 2024

# Show date N days from now (GNU)
$ date -d "+7 days"

# Show date N days from now (BSD)
$ date -v+7d

# Relative dates with GNU
$ date -d "next friday"
$ date -d "last month"

# BSD doesn't support natural language dates
```

BSD date uses `-j` (don't set the date) with `-f` (input format), while GNU date uses `-d` for parsing date strings.

### cp, mv, rm: Progress and Verbosity

```bash
# Show progress (GNU cp)
$ cp --progress large_file.iso /destination/

# BSD cp has no --progress flag
# Use rsync instead on macOS
$ rsync --progress large_file.iso /destination/

# Verbose mode (both support -v)
$ cp -v source dest
$ mv -v old new
$ rm -v file

# Interactive mode (both support -i)
$ rm -i file
remove file? y
```

### find: Expression Differences

```bash
# Delete found files (GNU)
$ find . -name "*.tmp" -delete

# BSD also supports -delete (this is compatible)
$ find . -name "*.tmp" -delete

# Regex matching (GNU - default is Emacs regex)
$ find . -regex ".*\.txt"

# BSD requires explicit regex type
$ find . -E -regex ".*\.txt"    # -E for extended regex

# Time-based search differs:
# GNU: -mtime uses 24-hour periods
# BSD: -mtime also uses 24-hour periods (compatible)

# But modification time in minutes:
# Both support -mmin (compatible)
$ find . -mmin -60    # Modified in last 60 minutes
```

### xargs: Null Delimiter

```bash
# Handle filenames with spaces (GNU)
$ find . -name "*.txt" -print0 | xargs -0 rm

# BSD also supports -0 (compatible)
$ find . -name "*.txt" -print0 | xargs -0 rm

# Replace string (GNU)
$ echo "file.txt" | xargs -I {} cp {} {}.bak

# BSD also supports -I (compatible)
$ echo "file.txt" | xargs -I {} cp {} {}.bak

# Parallel execution (GNU)
$ find . -name "*.jpg" -print0 | xargs -0 -P 4 convert

# BSD xargs doesn't have -P
# Use parallel or GNU xargs
```

### sort: Numeric and Version Sorting

```bash
# Version sort (GNU only)
$ printf "1.10\n1.2\n1.1" | sort -V
1.1
1.2
1.10

# BSD sort doesn't have -V
$ printf "1.10\n1.2\n1.1" | sort -V
sort: invalid option -- V

# Human numeric sort (GNU only)
$ du -h * | sort -h
1K    small.txt
10M   medium.txt
2G    large.txt

# BSD sort doesn't have -h
# Workaround: use plain numeric sort on raw bytes
$ du -k * | sort -n
```

### cut: Character vs Byte

```bash
# Cut by character (mostly compatible)
$ echo "hello" | cut -c1-3
hel

# Cut by field (compatible)
$ echo "a:b:c" | cut -d: -f2
b

# Complement (GNU only)
$ echo "a:b:c" | cut -d: --complement -f2
a:c

# BSD cut doesn't have --complement
```

### head and tail: Line Counts

```bash
# First N lines (compatible)
$ head -n 10 file.txt
$ head -10 file.txt      # Shorthand works on both

# Last N lines (compatible)
$ tail -n 10 file.txt
$ tail -10 file.txt

# All but last N lines (GNU)
$ head -n -5 file.txt    # All but last 5

# BSD head doesn't support negative counts
$ head -n -5 file.txt
head: illegal line count -- -5

# Workaround for BSD
$ tail -r file.txt | tail -n +6 | tail -r
```

### tar: Option Syntax

```bash
# Extract archive (both, but syntax preference differs)
$ tar -xvf archive.tar.gz     # Works on both
$ tar xvf archive.tar.gz      # Also works on both

# Create archive (compatible)
$ tar -cvf archive.tar directory/

# Compression selection (mostly compatible)
$ tar -czvf archive.tar.gz directory/    # gzip
$ tar -cjvf archive.tar.bz2 directory/   # bzip2

# Auto-detect compression on extract
$ tar -xf archive.tar.gz    # Both auto-detect

# GNU tar has more compression options
$ tar --zstd -cvf archive.tar.zst directory/  # GNU only
```

### stat: Completely Different

The `stat` command is almost entirely incompatible:

```bash
# File size (GNU)
$ stat -c %s file.txt
1024

# File size (BSD/macOS)
$ stat -f %z file.txt
1024

# Modification time (GNU)
$ stat -c %Y file.txt
1705276800

# Modification time (BSD)
$ stat -f %m file.txt
1705276800

# Human-readable (GNU)
$ stat file.txt
  File: file.txt
  Size: 1024            Blocks: 8          IO Block: 4096   regular file
...

# Human-readable (BSD)
$ stat file.txt
16777220 12345678 -rw-r--r-- 1 user staff 0 1024 "Jan 15 10:00:00 2024" ...
```

Portable alternative:

```bash
# Get file size portably
$ wc -c < file.txt
1024

# Or using ls
$ ls -l file.txt | awk '{print $5}'
1024
```

### readlink: Getting Real Paths

```bash
# Canonical path (GNU)
$ readlink -f /usr/local/bin/python
/opt/homebrew/Cellar/python@3.11/3.11.5/bin/python3.11

# BSD readlink doesn't have -f
$ readlink -f symlink
readlink: illegal option -- f

# BSD alternative
$ realpath symlink    # If available (macOS 12.3+)
$ python -c "import os; print(os.path.realpath('symlink'))"

# Or use this function
canonicalize() {
    cd -P "$(dirname "$1")" && echo "$(pwd)/$(basename "$1")"
}
```

### mktemp: Temporary Files

```bash
# Create temp file (GNU)
$ mktemp
/tmp/tmp.Xa3B2c1D

# BSD mktemp requires template
$ mktemp
/var/folders/.../tmp.XXXXXXXX    # Actually works on modern macOS

# But explicit template is more portable
$ mktemp /tmp/myapp.XXXXXX
/tmp/myapp.a1b2c3

# Create temp directory (both support -d)
$ mktemp -d
```

## Quick Reference Table

| Command | GNU (Linux) | BSD (macOS) |
|---------|-------------|-------------|
| In-place sed | `sed -i 's/a/b/'` | `sed -i '' 's/a/b/'` |
| Perl regex grep | `grep -P '\d+'` | Not available |
| Colored ls | `ls --color` | `ls -G` |
| Date parsing | `date -d "string"` | `date -j -f "fmt" "str"` |
| Relative date | `date -d "+7 days"` | `date -v+7d` |
| File stats | `stat -c %s file` | `stat -f %z file` |
| Version sort | `sort -V` | Not available |
| Human size sort | `sort -h` | Not available |
| Canonical path | `readlink -f` | `realpath` (macOS 12.3+) |
| xargs parallel | `xargs -P 4` | Not available |

## Recommendations

1. **For scripts**: Use POSIX-compatible syntax when possible
2. **For interactive use**: Install GNU coreutils via Homebrew
3. **For maximum portability**: Test on both platforms
4. **For macOS-only**: BSD tools are fine, just learn their syntax

The next chapter covers macOS-specific commands that have no GNU equivalent, followed by how to install GNU tools alongside BSD tools.
