# Text Processing: sed, awk, and grep

Text processing is where BSD and GNU tools diverge most significantly. Scripts that work perfectly on Linux often fail on macOS due to subtle differences in `sed`, `awk`, and `grep`. This chapter provides equivalent commands for both platforms and explains when each syntax is required.

## sed: Stream Editor

### In-Place Editing

The most common gotcha when moving from Linux to macOS:

```bash
# GNU sed (Linux)
$ sed -i 's/old/new/g' file.txt

# BSD sed (macOS) - requires backup extension argument
$ sed -i '' 's/old/new/g' file.txt    # No backup created
$ sed -i '.bak' 's/old/new/g' file.txt  # Creates file.txt.bak

# The error you'll see using GNU syntax on BSD:
$ sed -i 's/old/new/g' file.txt
sed: 1: "file.txt": invalid command code f
# BSD interpreted 's/old/new/g' as the backup extension!
```

### Newlines and Special Characters

```bash
# Insert newline - GNU sed
$ echo "hello" | sed 's/$/\n/'
hello
                    # (blank line)

# Insert newline - BSD sed (\n doesn't work)
# Method 1: Use $'...' quoting
$ echo "hello" | sed $'s/$/\\\n/'

# Method 2: Literal newline in the command
$ echo "hello" | sed 's/$/\
/'

# Method 3: Use a variable
$ NL=$'\n'
$ echo "hello" | sed "s/$/$NL/"
```

Tab characters:

```bash
# GNU sed - \t works
$ echo "a b" | sed 's/ /\t/'
a	b

# BSD sed - \t doesn't work in replacement
$ echo "a b" | sed $'s/ /\t/'
a	b

# Or use a literal tab
$ echo "a b" | sed 's/ /	/'    # Actual tab character
```

### Extended Regular Expressions

```bash
# Basic regex (both platforms)
$ echo "hello" | sed 's/l\+/L/'
heLlo

# Extended regex - GNU
$ echo "hello" | sed -E 's/l+/L/'
heLo

# Extended regex - BSD (same -E flag, compatible!)
$ echo "hello" | sed -E 's/l+/L/'
heLo

# Note: GNU also accepts -r for extended regex
$ echo "hello" | sed -r 's/l+/L/'    # GNU only
```

### Case-Insensitive Matching

```bash
# GNU sed supports 'i' flag
$ echo "Hello HELLO hello" | sed 's/hello/hi/gi'
hi hi hi

# BSD sed doesn't support 'i' flag
$ echo "Hello HELLO hello" | sed 's/hello/hi/gi'
sed: 1: "s/hello/hi/gi": bad flag in substitute command: 'i'

# BSD workaround: use character classes
$ echo "Hello HELLO hello" | sed 's/[Hh][Ee][Ll][Ll][Oo]/hi/g'
hi hi hi

# Or use tr for simple case conversion, then sed
$ echo "Hello HELLO hello" | tr '[:upper:]' '[:lower:]' | sed 's/hello/hi/g'
```

### Multiple Commands

```bash
# Both support -e for multiple expressions
$ sed -e 's/a/A/' -e 's/b/B/' file.txt

# Both support semicolons
$ sed 's/a/A/; s/b/B/' file.txt

# Both support newlines in the script
$ sed '
s/a/A/
s/b/B/
' file.txt
```

### Address Ranges

```bash
# Line numbers (compatible)
$ sed '10,20d' file.txt        # Delete lines 10-20
$ sed '5q' file.txt            # Print first 5 lines and quit

# Pattern addresses (compatible)
$ sed '/start/,/end/d' file.txt

# Last line (compatible)
$ sed '$d' file.txt            # Delete last line

# First match only - GNU sed
$ sed '0,/pattern/s/pattern/replace/' file.txt

# First match only - BSD sed (0 address not supported)
$ sed '1,/pattern/s/pattern/replace/' file.txt
# Note: BSD behavior differs if pattern is on line 1
```

### Portable sed Script

```bash
#!/bin/bash
# Works on both BSD and GNU sed

# Detect sed type
if sed --version 2>&1 | grep -q GNU; then
    SED="sed"
    SED_I="sed -i"
else
    SED="sed"
    SED_I="sed -i ''"
fi

# Use with eval for in-place editing
eval $SED_I "'s/old/new/g'" file.txt

# Or use a temp file (most portable)
sed 's/old/new/g' file.txt > file.tmp && mv file.tmp file.txt
```

## awk: Pattern Processing

`awk` on macOS is actually `nawk` (new awk), which is largely compatible with GNU awk. However, differences exist.

### Basic Usage (Compatible)

```bash
# Print columns (compatible)
$ echo "a b c" | awk '{print $2}'
b

# Field separator (compatible)
$ echo "a:b:c" | awk -F: '{print $2}'
b

# Patterns (compatible)
$ awk '/error/ {print}' log.txt

# Variables (compatible)
$ awk -v name="John" 'BEGIN {print "Hello", name}'
Hello John
```

### GNU awk Extensions

```bash
# Case-insensitive matching - GNU awk (gawk)
$ echo -e "Hello\nhello\nHELLO" | gawk 'BEGIN{IGNORECASE=1} /hello/'
Hello
hello
HELLO

# BSD awk doesn't have IGNORECASE
$ echo -e "Hello\nhello\nHELLO" | awk '/[Hh][Ee][Ll][Ll][Oo]/'

# Length function as array length - gawk
$ gawk 'BEGIN {a[1]=1; a[2]=2; print length(a)}'
2

# BSD awk - length(array) may not work
# Use a loop to count
$ awk 'BEGIN {a[1]=1; a[2]=2; for(i in a) n++; print n}'
2
```

### Regular Expression Differences

```bash
# Word boundaries - GNU awk
$ echo "foo foobar" | gawk '{gsub(/\bfoo\b/, "bar"); print}'
bar foobar

# BSD awk doesn't support \b
$ echo "foo foobar" | awk '{gsub(/foo[^a-z]|foo$/, "bar "); print}'
bar foobar

# Interval expressions {n,m} - both support with --posix or -r
$ echo "aaa" | gawk '{print gsub(/a{2,3}/, "X")}'
1

$ echo "aaa" | awk '{print gsub(/a{2,3}/, "X")}'
# May or may not work depending on macOS version
```

### In-Place Editing with awk

```bash
# GNU awk 4.1+ has -i inplace
$ gawk -i inplace '{gsub(/old/, "new")}1' file.txt

# BSD awk has no in-place option
# Use a temp file
$ awk '{gsub(/old/, "new")}1' file.txt > tmp && mv tmp file.txt
```

### Recommended: Install gawk

For consistent behavior, install GNU awk:

```bash
$ brew install gawk

# Use as gawk or add to PATH
$ gawk --version
GNU Awk 5.2.2, API 3.2, PMA Avon 8-g1
```

## grep: Pattern Matching

### Basic Usage (Compatible)

```bash
# Simple pattern (compatible)
$ grep "error" log.txt

# Case insensitive (compatible)
$ grep -i "error" log.txt

# Line numbers (compatible)
$ grep -n "error" log.txt

# Count matches (compatible)
$ grep -c "error" log.txt

# Invert match (compatible)
$ grep -v "debug" log.txt

# Recursive search (compatible)
$ grep -r "TODO" src/

# Only filenames (compatible)
$ grep -l "error" *.log
```

### Extended Regular Expressions

```bash
# Extended regex (compatible with -E)
$ grep -E "error|warning" log.txt
$ grep -E "[0-9]{3}-[0-9]{4}" contacts.txt

# Equivalent using egrep (both platforms)
$ egrep "error|warning" log.txt
```

### Perl-Compatible Regex

The biggest difference - GNU grep's `-P` flag:

```bash
# GNU grep - Perl regex
$ grep -P '\d{3}-\d{4}' contacts.txt
$ grep -P '(?<=@)\w+(?=\.com)' emails.txt    # Lookbehind/ahead

# BSD grep - no -P flag
$ grep -P '\d+'
grep: invalid option -- P

# Alternatives on BSD:

# 1. Use extended regex equivalents
$ grep -E '[0-9]{3}-[0-9]{4}' contacts.txt

# 2. Use perl directly
$ perl -ne 'print if /\d{3}-\d{4}/' contacts.txt

# 3. Install GNU grep
$ brew install grep
$ ggrep -P '\d{3}-\d{4}' contacts.txt
```

### Common Perl Regex Features and BSD Alternatives

```bash
# Digits: \d (Perl) vs [0-9] (POSIX)
$ ggrep -P '\d+'          # GNU
$ grep -E '[0-9]+'        # BSD

# Word characters: \w vs [a-zA-Z0-9_]
$ ggrep -P '\w+'          # GNU
$ grep -E '[a-zA-Z0-9_]+' # BSD

# Word boundaries: \b vs [[:<:]] and [[:>:]]
$ ggrep -P '\bword\b'     # GNU
$ grep -E '[[:<:]]word[[:>:]]'  # BSD

# Non-greedy matching: .*?
$ ggrep -oP '<.*?>'       # GNU - minimal match
# BSD has no equivalent; use different approach
$ grep -oE '<[^>]*>'      # Matches <...> without > inside

# Lookahead/lookbehind
$ ggrep -P '(?<=\$)\d+'   # GNU - digits after $
# BSD has no equivalent
```

### Context Lines

```bash
# Lines before match (compatible)
$ grep -B 3 "error" log.txt

# Lines after match (compatible)
$ grep -A 3 "error" log.txt

# Lines before and after (compatible)
$ grep -C 3 "error" log.txt
```

### Line-Buffered Output

```bash
# Both support --line-buffered for live log watching
$ tail -f log.txt | grep --line-buffered "error"
```

## sort: Sorting Text

### Basic Sorting (Compatible)

```bash
# Alphabetical sort (compatible)
$ sort file.txt

# Reverse sort (compatible)
$ sort -r file.txt

# Numeric sort (compatible)
$ sort -n numbers.txt

# Sort by field (compatible)
$ sort -t: -k2 /etc/passwd
```

### GNU-Only Features

```bash
# Version sort - GNU only
$ printf "1.10\n1.2\n1.1" | sort -V
1.1
1.2
1.10

$ printf "1.10\n1.2\n1.1" | sort -V    # BSD
sort: invalid option -- V

# BSD workaround (complex, not exact equivalent)
$ printf "1.10\n1.2\n1.1" | sort -t. -k1,1n -k2,2n
1.1
1.2
1.10

# Human-readable sizes - GNU only
$ du -h | sort -h
1K    small/
10M   medium/
2G    large/

$ du -h | sort -h    # BSD
sort: invalid option -- h
```

### Stable Sort

```bash
# Both support stable sort
$ sort -s file.txt

# Random sort - both support
$ sort -R file.txt
```

## cut: Extract Fields

### Basic Usage (Compatible)

```bash
# Cut by character position (compatible)
$ echo "hello" | cut -c1-3
hel

# Cut by field (compatible)
$ echo "a:b:c" | cut -d: -f2
b

# Multiple fields (compatible)
$ echo "a:b:c" | cut -d: -f1,3
a:c

# Range of fields (compatible)
$ echo "a:b:c:d:e" | cut -d: -f2-4
b:c:d
```

### GNU Extensions

```bash
# Complement - GNU only
$ echo "a:b:c" | cut -d: --complement -f2
a:c

$ echo "a:b:c" | cut -d: --complement -f2    # BSD
cut: illegal option -- -

# BSD alternative using awk
$ echo "a:b:c" | awk -F: '{print $1":"$3}'
a:c

# Output delimiter - GNU only
$ echo "a:b:c" | cut -d: -f1,3 --output-delimiter=' '
a c

# BSD alternative
$ echo "a:b:c" | cut -d: -f1,3 | tr ':' ' '
a c
```

## tr: Translate Characters

`tr` is mostly compatible between BSD and GNU:

```bash
# Character substitution (compatible)
$ echo "hello" | tr 'a-z' 'A-Z'
HELLO

# Delete characters (compatible)
$ echo "hello 123" | tr -d '0-9'
hello

# Squeeze repeated characters (compatible)
$ echo "heeello" | tr -s 'e'
hello

# Character classes (compatible)
$ echo "Hello123" | tr -d '[:digit:]'
Hello

# Complement (compatible)
$ echo "hello 123" | tr -dc '0-9\n'
123
```

## uniq: Filter Duplicates

Mostly compatible:

```bash
# Remove adjacent duplicates (compatible)
$ sort file.txt | uniq

# Count occurrences (compatible)
$ sort file.txt | uniq -c

# Only duplicates (compatible)
$ sort file.txt | uniq -d

# Only unique (compatible)
$ sort file.txt | uniq -u

# Case insensitive (compatible)
$ sort file.txt | uniq -i
```

## Portable Text Processing Tips

### 1. Use temp files instead of in-place editing

```bash
# Most portable
sed 's/old/new/g' file.txt > file.tmp && mv file.tmp file.txt
```

### 2. Avoid GNU-specific regex

```bash
# Instead of \d, use [0-9]
# Instead of \w, use [a-zA-Z0-9_]
# Instead of \s, use [[:space:]]
# Instead of \b, use [[:<:]] and [[:>:]] (BSD) or word boundary logic
```

### 3. Create wrapper functions

```bash
# Add to ~/.zshrc or script
portable_sed_i() {
    if sed --version 2>&1 | grep -q GNU; then
        sed -i "$@"
    else
        sed -i '' "$@"
    fi
}
```

### 4. Use awk for complex transformations

`awk` is more consistent across platforms than `sed` for complex operations:

```bash
# Instead of complex sed
$ awk '{gsub(/old/, "new"); print}' file.txt > tmp && mv tmp file.txt
```

### 5. Test on both platforms

Before distributing scripts, test on both macOS and Linux:

```bash
# Check for GNU-specific features
$ shellcheck myscript.sh    # Static analysis

# Test in Docker for Linux
$ docker run --rm -v "$PWD:/work" -w /work alpine:latest sh myscript.sh
```

## Summary: BSD vs GNU Text Tools

| Feature | GNU | BSD | Portable Alternative |
|---------|-----|-----|---------------------|
| sed -i | `sed -i` | `sed -i ''` | temp file |
| sed \n | Works | Doesn't work | $'\n' or literal |
| grep -P | Works | Doesn't exist | Use -E with POSIX |
| sort -V | Works | Doesn't exist | Custom solution |
| sort -h | Works | Doesn't exist | Sort raw bytes |
| cut --complement | Works | Doesn't exist | Use awk |
| awk IGNORECASE | Works | Doesn't exist | Character classes |

When in doubt, install GNU tools via Homebrew and use them explicitly, or write POSIX-compliant commands that work everywhere.
