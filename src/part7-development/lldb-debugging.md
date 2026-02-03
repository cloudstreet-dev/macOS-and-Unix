# Debugging with LLDB

LLDB is the debugger for macOS, part of the LLVM project. If you're coming from Linux and used to GDB, LLDB will feel familiar but different. This chapter covers practical LLDB usage with command equivalents for GDB users.

## LLDB vs GDB

| Feature | GDB | LLDB |
|---------|-----|------|
| Platform | Linux, others | macOS, iOS, LLVM |
| Project | GNU | LLVM |
| On macOS | Available via Homebrew | Built-in |
| Scripting | Python, Guile | Python |
| Default on macOS | No | Yes |

LLDB is the only debugger that works seamlessly with macOS code signing, entitlements, and System Integrity Protection.

## Getting Started

### Launching LLDB

```bash
# Debug an executable
$ lldb ./myprogram

# Debug with arguments
$ lldb -- ./myprogram arg1 arg2

# Attach to running process
$ lldb -p 12345

# Attach by name
$ lldb -n Safari
```

### Compiling for Debugging

```bash
# Include debug symbols
$ clang -g -O0 program.c -o program

# -g: Generate debug info
# -O0: No optimization (easier debugging)
```

## Essential Commands

### Running

| Task | LLDB | GDB |
|------|------|-----|
| Run program | `run` or `r` | `run` |
| Run with args | `run arg1 arg2` | `run arg1 arg2` |
| Continue | `continue` or `c` | `continue` |
| Step over | `next` or `n` | `next` |
| Step into | `step` or `s` | `step` |
| Step out | `finish` | `finish` |
| Stop | Ctrl+C | Ctrl+C |
| Quit | `quit` or `q` | `quit` |

### Breakpoints

```bash
# Set breakpoint at function
(lldb) breakpoint set --name main
(lldb) b main                        # Shorthand

# Set breakpoint at file:line
(lldb) breakpoint set --file main.c --line 42
(lldb) b main.c:42                   # Shorthand

# Set breakpoint at address
(lldb) breakpoint set --address 0x100003f00

# Conditional breakpoint
(lldb) breakpoint set --name foo --condition 'x > 10'
(lldb) b foo -c 'x > 10'

# List breakpoints
(lldb) breakpoint list
(lldb) br l

# Delete breakpoint
(lldb) breakpoint delete 1
(lldb) br del 1

# Disable/enable breakpoint
(lldb) breakpoint disable 1
(lldb) breakpoint enable 1

# Delete all breakpoints
(lldb) breakpoint delete
```

### Watchpoints

```bash
# Watch variable for write
(lldb) watchpoint set variable myvar

# Watch expression
(lldb) watchpoint set expression -- &myvar

# Watch for read
(lldb) watchpoint set variable -w read myvar

# Watch for read or write
(lldb) watchpoint set variable -w read_write myvar

# List watchpoints
(lldb) watchpoint list

# Delete watchpoint
(lldb) watchpoint delete 1
```

### Examining Data

```bash
# Print variable
(lldb) print myvar
(lldb) p myvar

# Print with format
(lldb) print/x myvar          # Hex
(lldb) print/d myvar          # Decimal
(lldb) print/t myvar          # Binary
(lldb) print/c myvar          # Character

# Print expression
(lldb) print 2 + 2
(lldb) print strlen("hello")

# Print pointer contents
(lldb) print *ptr
(lldb) print ptr[0]

# Print array
(lldb) print myarray
(lldb) parray 10 myarray      # Print 10 elements

# Print struct
(lldb) print mystruct
(lldb) print mystruct.field

# Frame variables (local variables)
(lldb) frame variable
(lldb) fr v

# Specific variable
(lldb) frame variable myvar
(lldb) fr v myvar
```

### Memory Examination

```bash
# Read memory (x command)
(lldb) memory read 0x100003f00
(lldb) x 0x100003f00

# With format and count
(lldb) memory read --size 4 --count 10 --format x 0x100003f00
(lldb) x -s4 -c10 -fx 0x100003f00

# Read as string
(lldb) memory read --format s 0x100003f00
(lldb) x -fs 0x100003f00

# Read variable's memory
(lldb) memory read &myvar

# Common formats: x (hex), d (decimal), s (string), c (char), i (instruction)
```

### Stack Frames

```bash
# Show backtrace
(lldb) thread backtrace
(lldb) bt

# Full backtrace (all threads)
(lldb) thread backtrace all
(lldb) bt all

# Select frame
(lldb) frame select 2
(lldb) f 2

# Frame info
(lldb) frame info

# Up/down frames
(lldb) up
(lldb) down
```

### Threads

```bash
# List threads
(lldb) thread list

# Select thread
(lldb) thread select 2

# Thread backtrace
(lldb) thread backtrace

# Continue specific thread
(lldb) thread continue
```

## GDB to LLDB Command Map

| Task | GDB | LLDB |
|------|-----|------|
| Run | `run` | `run` |
| Break at function | `break main` | `b main` |
| Break at line | `break file:line` | `b file:line` |
| Continue | `continue` | `continue` |
| Step over | `next` | `next` |
| Step into | `step` | `step` |
| Step out | `finish` | `finish` |
| Print | `print var` | `print var` |
| Print hex | `print/x var` | `p/x var` |
| Backtrace | `backtrace` | `bt` |
| List breakpoints | `info breakpoints` | `br list` |
| Delete breakpoint | `delete 1` | `br del 1` |
| Examine memory | `x/10x addr` | `x -c10 -fx addr` |
| Local variables | `info locals` | `fr v` |
| Disassemble | `disas` | `disassemble` |
| Registers | `info registers` | `register read` |
| Attach | `attach pid` | `attach -p pid` |
| Set variable | `set var x=10` | `expr x=10` |

## Advanced Features

### Expressions

```bash
# Evaluate expression
(lldb) expression 2 + 2
(lldb) expr 2 + 2

# Call function
(lldb) expr (int)printf("Hello\n")

# Modify variable
(lldb) expr myvar = 42

# Cast
(lldb) expr (char *)myptr

# Create variable for session
(lldb) expr int $myval = 42
(lldb) print $myval
```

### Disassembly

```bash
# Disassemble current function
(lldb) disassemble
(lldb) di

# Disassemble function by name
(lldb) disassemble --name main
(lldb) di -n main

# Disassemble at address
(lldb) disassemble --start-address 0x100003f00

# Show mixed source and assembly
(lldb) disassemble --mixed
(lldb) di -m

# Disassemble bytes
(lldb) disassemble --bytes
(lldb) di -b
```

### Register Access

```bash
# Read all registers
(lldb) register read

# Read specific register
(lldb) register read rax
(lldb) register read x0    # ARM64

# Write register
(lldb) register write rax 0x42

# Read register in different format
(lldb) register read --format binary rax
```

### Process Control

```bash
# Process info
(lldb) process status

# Kill process
(lldb) process kill

# Detach
(lldb) process detach

# Signal handling
(lldb) process handle SIGINT --stop false --pass true
```

### Source Code

```bash
# Show source
(lldb) source list
(lldb) l

# Show specific lines
(lldb) source list --line 42
(lldb) l -l 42

# Show function
(lldb) source list --name main
```

## macOS-Specific Features

### Debugging System Programs

SIP restricts debugging system processes:

```bash
# This may fail
$ lldb /usr/bin/some_system_tool
error: process exited with status -1 (attach failed (Not allowed to attach to process. ...))
```

For system process debugging, disable SIP (not recommended) or debug copies.

### Code Signing for Debugging

To debug your own signed apps:

```bash
# Check entitlements
$ codesign -d --entitlements - /path/to/app

# May need com.apple.security.get-task-allow entitlement
```

### Debugging Universal Binaries

```bash
# Specify architecture
$ lldb --arch x86_64 ./universal_binary
$ lldb --arch arm64 ./universal_binary
```

### Working with dSYM Files

```bash
# Debug symbols are in dSYM bundles
$ dsymutil myprogram         # Generate dSYM
$ ls myprogram.dSYM/

# LLDB finds dSYM automatically if nearby
# Or specify manually:
(lldb) target symbols add myprogram.dSYM
```

### Crash Log Analysis

```bash
# Symbolicate crash log
$ lldb
(lldb) target create --core /path/to/crashlog

# Or use atos for quick symbolication
$ atos -o myprogram.dSYM/Contents/Resources/DWARF/myprogram -l 0x100000000 0x100003f00
```

## Scripting with Python

### Interactive Python

```bash
(lldb) script
>>> print(lldb.frame.GetFunctionName())
main
>>> exit()
```

### Python Command

```bash
(lldb) script print(lldb.debugger.GetSelectedTarget().GetExecutable())
```

### Custom Commands

Create `~/.lldbinit`:

```python
command script import ~/lldb_scripts/mycommands.py
```

Example script (`mycommands.py`):

```python
import lldb

def hello_command(debugger, command, result, internal_dict):
    print("Hello from LLDB!")

def __lldb_init_module(debugger, internal_dict):
    debugger.HandleCommand('command script add -f mycommands.hello_command hello')
```

## Configuration

### .lldbinit File

Create `~/.lldbinit`:

```bash
# Custom settings
settings set target.x86-disassembly-flavor intel
settings set stop-line-count-after 5
settings set stop-line-count-before 5

# Aliases
command alias bpl breakpoint list
command alias bpc breakpoint clear

# Auto-load scripts
command script import ~/.lldb/custom.py

# Type formatters
type summary add --summary-string "${var.name} (${var.age} years)" Person
```

### Useful Settings

```bash
# Show settings
(lldb) settings list

# Change setting
(lldb) settings set target.run-args arg1 arg2
(lldb) settings set target.env-vars DEBUG=1

# Disassembly flavor
(lldb) settings set target.x86-disassembly-flavor intel

# Auto-confirm
(lldb) settings set auto-confirm true
```

## GUI Mode

LLDB has a curses-based GUI:

```bash
(lldb) gui
```

In GUI mode:
- Tab: Switch panes
- Arrow keys: Navigate
- Enter: Select
- Esc: Back/close
- h: Help

## Integration with Xcode

LLDB is Xcode's built-in debugger:

- Breakpoints set in Xcode use LLDB
- Debug console is LLDB
- Can use all LLDB commands in Xcode's debug console
- Variable inspection uses LLDB

## Common Debugging Scenarios

### Debugging Segfault

```bash
$ lldb ./crashy_program
(lldb) run
Process stopped
* thread #1, stop reason = EXC_BAD_ACCESS (code=1, address=0x0)

(lldb) bt
* frame #0: crashy_program`bad_function at crashy.c:15
  frame #1: crashy_program`main at crashy.c:42

(lldb) frame select 0
(lldb) frame variable
(lldb) print ptr      # See what's null
```

### Finding Memory Leaks

```bash
# Use with MallocStackLogging
$ MallocStackLogging=1 lldb ./program
(lldb) run

# When done, check leaks
$ leaks program_pid
```

### Debugging Deadlocks

```bash
(lldb) process interrupt    # Ctrl+C
(lldb) thread list
(lldb) bt all               # See all thread stacks
```

## Summary

| Category | Key Commands |
|----------|--------------|
| Running | `run`, `continue`, `next`, `step`, `finish` |
| Breakpoints | `b function`, `b file:line`, `br list`, `br del` |
| Inspection | `print var`, `frame variable`, `bt`, `memory read` |
| Threads | `thread list`, `thread select`, `bt all` |
| Memory | `x address`, `memory read` |
| Registers | `register read`, `register write` |
| Control | `process kill`, `process detach`, `quit` |

LLDB is powerful and integrates deeply with macOS. While the command syntax differs from GDB, the concepts are the same. Use the GDB-to-LLDB mapping table while learning, and soon LLDB's syntax will feel natural.
