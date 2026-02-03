# Appendix A: Command Reference

This appendix provides a quick reference to essential commands for macOS. Commands are organized by category for easy lookup. For detailed information on any command, use `man command-name`.

## Legend

- `$` - Run as regular user
- `#` - Requires `sudo` (administrator privileges)
- `[optional]` - Optional parameter
- `<required>` - Required parameter

---

## File and Directory Operations

### Navigation and Listing

| Command | Description | Example |
|---------|-------------|---------|
| `pwd` | Print working directory | `$ pwd` |
| `cd <dir>` | Change directory | `$ cd ~/Documents` |
| `cd -` | Return to previous directory | `$ cd -` |
| `ls` | List directory contents | `$ ls -la` |
| `ls -la` | Long format with hidden files | `$ ls -la ~/` |
| `ls -lah` | Human-readable sizes | `$ ls -lah` |
| `ls -lt` | Sort by modification time | `$ ls -lt` |
| `tree` | Directory tree (install via Homebrew) | `$ tree -L 2` |

### File Operations

| Command | Description | Example |
|---------|-------------|---------|
| `cp <src> <dst>` | Copy file | `$ cp file.txt backup.txt` |
| `cp -r <src> <dst>` | Copy directory recursively | `$ cp -r dir1 dir2` |
| `cp -a <src> <dst>` | Archive mode (preserve attributes) | `$ cp -a /source /dest` |
| `mv <src> <dst>` | Move/rename file | `$ mv old.txt new.txt` |
| `rm <file>` | Remove file | `$ rm unwanted.txt` |
| `rm -r <dir>` | Remove directory recursively | `$ rm -r old_dir` |
| `rm -rf <dir>` | Force remove (dangerous!) | `$ rm -rf temp/` |
| `mkdir <dir>` | Create directory | `$ mkdir newdir` |
| `mkdir -p <path>` | Create nested directories | `$ mkdir -p a/b/c` |
| `rmdir <dir>` | Remove empty directory | `$ rmdir emptydir` |
| `touch <file>` | Create file or update timestamp | `$ touch newfile.txt` |
| `ln -s <target> <link>` | Create symbolic link | `$ ln -s /path/to/file link` |
| `ditto <src> <dst>` | Copy preserving macOS metadata | `$ ditto src dst` |

### File Information

| Command | Description | Example |
|---------|-------------|---------|
| `file <file>` | Determine file type | `$ file document.pdf` |
| `stat <file>` | Display file status | `$ stat file.txt` |
| `du -sh <path>` | Disk usage summary | `$ du -sh ~/Downloads` |
| `du -h -d 1` | Disk usage one level deep | `$ du -h -d 1 ~/` |
| `wc -l <file>` | Count lines in file | `$ wc -l file.txt` |
| `mdls <file>` | View Spotlight metadata | `$ mdls photo.jpg` |

### Permissions

| Command | Description | Example |
|---------|-------------|---------|
| `chmod <mode> <file>` | Change permissions | `$ chmod 755 script.sh` |
| `chmod +x <file>` | Make executable | `$ chmod +x run.sh` |
| `chmod -R <mode> <dir>` | Recursive permissions | `$ chmod -R 644 docs/` |
| `chown <user> <file>` | Change owner | `# chown admin file.txt` |
| `chown -R <user>:<group>` | Recursive ownership | `# chown -R www:www html/` |
| `chgrp <group> <file>` | Change group | `$ chgrp staff file.txt` |
| `ls -le` | List with ACLs | `$ ls -le` |
| `chmod +a <acl>` | Add ACL entry | `$ chmod +a "user:joe:allow read" file` |

---

## Text Processing

### Viewing Files

| Command | Description | Example |
|---------|-------------|---------|
| `cat <file>` | Display file contents | `$ cat file.txt` |
| `cat -n <file>` | Display with line numbers | `$ cat -n script.sh` |
| `head <file>` | First 10 lines | `$ head file.txt` |
| `head -n 20 <file>` | First N lines | `$ head -n 20 log.txt` |
| `tail <file>` | Last 10 lines | `$ tail file.txt` |
| `tail -n 50 <file>` | Last N lines | `$ tail -n 50 log.txt` |
| `tail -f <file>` | Follow file (live updates) | `$ tail -f /var/log/system.log` |
| `less <file>` | Page through file | `$ less largefile.txt` |
| `more <file>` | Simple pager | `$ more file.txt` |

### Searching

| Command | Description | Example |
|---------|-------------|---------|
| `grep <pattern> <file>` | Search for pattern | `$ grep "error" log.txt` |
| `grep -i <pattern>` | Case-insensitive search | `$ grep -i "warning" file` |
| `grep -r <pattern> <dir>` | Recursive search | `$ grep -r "TODO" src/` |
| `grep -n <pattern>` | Show line numbers | `$ grep -n "func" code.py` |
| `grep -v <pattern>` | Invert match (exclude) | `$ grep -v "debug" log` |
| `grep -E <regex>` | Extended regex | `$ grep -E "err(or)?s?" log` |
| `grep -l <pattern>` | List matching files only | `$ grep -l "main" *.c` |
| `grep -c <pattern>` | Count matches | `$ grep -c "error" log.txt` |
| `mdfind <query>` | Spotlight search | `$ mdfind "project report"` |
| `mdfind -name <name>` | Search by filename | `$ mdfind -name "config.yaml"` |

### Text Manipulation

| Command | Description | Example |
|---------|-------------|---------|
| `sort <file>` | Sort lines | `$ sort names.txt` |
| `sort -n <file>` | Numeric sort | `$ sort -n numbers.txt` |
| `sort -r <file>` | Reverse sort | `$ sort -r file.txt` |
| `sort -u <file>` | Sort and remove duplicates | `$ sort -u list.txt` |
| `uniq` | Remove adjacent duplicates | `$ sort file \| uniq` |
| `uniq -c` | Count occurrences | `$ sort file \| uniq -c` |
| `cut -d: -f1` | Extract field | `$ cut -d: -f1 /etc/passwd` |
| `cut -c1-10` | Extract characters | `$ cut -c1-10 file.txt` |
| `tr 'a-z' 'A-Z'` | Translate characters | `$ echo "hi" \| tr 'a-z' 'A-Z'` |
| `tr -d '\r'` | Delete characters | `$ tr -d '\r' < file.txt` |
| `sed 's/old/new/'` | Substitute first match | `$ sed 's/foo/bar/' file` |
| `sed 's/old/new/g'` | Substitute all matches | `$ sed 's/foo/bar/g' file` |
| `awk '{print $1}'` | Print first field | `$ awk '{print $1}' file` |
| `awk -F: '{print $1}'` | Custom delimiter | `$ awk -F: '{print $1}' passwd` |

### Comparison

| Command | Description | Example |
|---------|-------------|---------|
| `diff <file1> <file2>` | Compare files | `$ diff old.txt new.txt` |
| `diff -u <f1> <f2>` | Unified diff format | `$ diff -u old new` |
| `diff -r <dir1> <dir2>` | Compare directories | `$ diff -r src1 src2` |
| `comm <f1> <f2>` | Compare sorted files | `$ comm list1 list2` |
| `cmp <f1> <f2>` | Byte-by-byte compare | `$ cmp file1 file2` |

---

## Finding Files

| Command | Description | Example |
|---------|-------------|---------|
| `find <path> -name <pattern>` | Find by name | `$ find . -name "*.txt"` |
| `find <path> -type f` | Find files only | `$ find . -type f` |
| `find <path> -type d` | Find directories only | `$ find /var -type d` |
| `find <path> -mtime -7` | Modified in last 7 days | `$ find . -mtime -7` |
| `find <path> -size +100M` | Files larger than 100MB | `$ find . -size +100M` |
| `find <path> -empty` | Find empty files/dirs | `$ find . -empty` |
| `find <path> -exec cmd {} \;` | Execute command on results | `$ find . -name "*.log" -exec rm {} \;` |
| `locate <pattern>` | Search locate database | `$ locate nginx.conf` |
| `which <command>` | Show command path | `$ which python` |
| `whereis <command>` | Locate binary/man/source | `$ whereis ls` |
| `type <command>` | Describe command type | `$ type cd` |

---

## Archiving and Compression

| Command | Description | Example |
|---------|-------------|---------|
| `tar -cvf <archive> <files>` | Create tar archive | `$ tar -cvf backup.tar dir/` |
| `tar -xvf <archive>` | Extract tar archive | `$ tar -xvf backup.tar` |
| `tar -czvf <archive> <files>` | Create gzipped tar | `$ tar -czvf backup.tar.gz dir/` |
| `tar -xzvf <archive>` | Extract gzipped tar | `$ tar -xzvf backup.tar.gz` |
| `tar -cjvf <archive> <files>` | Create bzip2 tar | `$ tar -cjvf backup.tar.bz2 dir/` |
| `tar -xjvf <archive>` | Extract bzip2 tar | `$ tar -xjvf backup.tar.bz2` |
| `tar -tvf <archive>` | List tar contents | `$ tar -tvf backup.tar` |
| `gzip <file>` | Compress with gzip | `$ gzip file.txt` |
| `gunzip <file.gz>` | Decompress gzip | `$ gunzip file.txt.gz` |
| `zip -r <archive> <dir>` | Create zip archive | `$ zip -r archive.zip dir/` |
| `unzip <archive>` | Extract zip archive | `$ unzip archive.zip` |
| `unzip -l <archive>` | List zip contents | `$ unzip -l archive.zip` |

---

## Process Management

### Viewing Processes

| Command | Description | Example |
|---------|-------------|---------|
| `ps aux` | List all processes | `$ ps aux` |
| `ps aux \| grep <name>` | Find specific process | `$ ps aux \| grep nginx` |
| `pgrep <name>` | Get PIDs by name | `$ pgrep -l Safari` |
| `top` | Interactive process viewer | `$ top` |
| `htop` | Enhanced top (install via Homebrew) | `$ htop` |
| `Activity Monitor` | GUI process manager | Open from Spotlight |

### Process Control

| Command | Description | Example |
|---------|-------------|---------|
| `kill <PID>` | Terminate process | `$ kill 1234` |
| `kill -9 <PID>` | Force kill process | `$ kill -9 1234` |
| `killall <name>` | Kill by name | `$ killall Safari` |
| `pkill <pattern>` | Kill by pattern | `$ pkill -f "python script"` |
| `<command> &` | Run in background | `$ long_task &` |
| `jobs` | List background jobs | `$ jobs` |
| `fg` | Bring to foreground | `$ fg %1` |
| `bg` | Continue in background | `$ bg %1` |
| `nohup <cmd> &` | Run immune to hangups | `$ nohup ./script.sh &` |
| `Ctrl+Z` | Suspend current process | |
| `Ctrl+C` | Interrupt current process | |

### Resource Usage

| Command | Description | Example |
|---------|-------------|---------|
| `vm_stat` | Virtual memory statistics | `$ vm_stat` |
| `iostat` | I/O statistics | `$ iostat` |
| `fs_usage` | Filesystem activity | `# fs_usage -f filesys` |
| `sample <PID> <secs>` | Sample process | `$ sample Safari 5` |

---

## System Information

| Command | Description | Example |
|---------|-------------|---------|
| `uname -a` | System information | `$ uname -a` |
| `sw_vers` | macOS version | `$ sw_vers` |
| `system_profiler` | Detailed system info | `$ system_profiler SPHardwareDataType` |
| `hostname` | Show hostname | `$ hostname` |
| `whoami` | Current username | `$ whoami` |
| `id` | User ID and groups | `$ id` |
| `uptime` | System uptime | `$ uptime` |
| `date` | Current date/time | `$ date` |
| `cal` | Calendar | `$ cal` |
| `df -h` | Disk free space | `$ df -h` |
| `diskutil list` | List disks and partitions | `$ diskutil list` |
| `diskutil info <disk>` | Disk information | `$ diskutil info disk0` |
| `sysctl -a` | All kernel parameters | `$ sysctl -a` |
| `sysctl hw.memsize` | Total RAM | `$ sysctl hw.memsize` |
| `sysctl machdep.cpu` | CPU information | `$ sysctl machdep.cpu` |
| `ioreg -l` | IOKit registry | `$ ioreg -l` |
| `nvram -p` | NVRAM variables | `$ nvram -p` |

---

## Network Commands

### Network Information

| Command | Description | Example |
|---------|-------------|---------|
| `ifconfig` | Network interface config | `$ ifconfig` |
| `ifconfig en0` | Specific interface | `$ ifconfig en0` |
| `ipconfig getifaddr en0` | Get IP address | `$ ipconfig getifaddr en0` |
| `networksetup -listallnetworkservices` | List services | `$ networksetup -listallnetworkservices` |
| `networksetup -getinfo "Wi-Fi"` | Service info | `$ networksetup -getinfo "Wi-Fi"` |
| `scutil --dns` | DNS configuration | `$ scutil --dns` |
| `scutil --proxy` | Proxy configuration | `$ scutil --proxy` |
| `netstat -an` | Network connections | `$ netstat -an` |
| `netstat -rn` | Routing table | `$ netstat -rn` |
| `lsof -i` | Open network connections | `$ lsof -i` |
| `lsof -i :80` | Connections on port 80 | `$ lsof -i :80` |

### Connectivity Testing

| Command | Description | Example |
|---------|-------------|---------|
| `ping <host>` | Test connectivity | `$ ping apple.com` |
| `ping -c 5 <host>` | Send 5 pings | `$ ping -c 5 google.com` |
| `traceroute <host>` | Trace route to host | `$ traceroute apple.com` |
| `mtr <host>` | Combined ping/traceroute | `$ mtr google.com` |
| `curl <url>` | Transfer URL | `$ curl https://api.example.com` |
| `curl -I <url>` | Fetch headers only | `$ curl -I https://apple.com` |
| `curl -o <file> <url>` | Download to file | `$ curl -o file.zip https://url` |
| `wget <url>` | Download file (Homebrew) | `$ wget https://example.com/file` |
| `nc -zv <host> <port>` | Test port connectivity | `$ nc -zv google.com 443` |
| `nslookup <host>` | DNS lookup | `$ nslookup apple.com` |
| `dig <host>` | DNS lookup (detailed) | `$ dig apple.com` |
| `host <host>` | DNS lookup (simple) | `$ host apple.com` |
| `arp -a` | ARP table | `$ arp -a` |

### Wi-Fi

| Command | Description | Example |
|---------|-------------|---------|
| `networksetup -setairportpower en0 on` | Turn Wi-Fi on | `$ networksetup -setairportpower en0 on` |
| `networksetup -setairportpower en0 off` | Turn Wi-Fi off | `$ networksetup -setairportpower en0 off` |
| `networksetup -getairportnetwork en0` | Current network | `$ networksetup -getairportnetwork en0` |

---

## Disk and Volume Management

| Command | Description | Example |
|---------|-------------|---------|
| `diskutil list` | List all disks | `$ diskutil list` |
| `diskutil info <disk>` | Disk information | `$ diskutil info disk0` |
| `diskutil mount <vol>` | Mount volume | `$ diskutil mount disk2s1` |
| `diskutil unmount <vol>` | Unmount volume | `$ diskutil unmount disk2s1` |
| `diskutil eject <disk>` | Eject disk | `$ diskutil eject disk2` |
| `diskutil verifyVolume <vol>` | Verify filesystem | `# diskutil verifyVolume /` |
| `diskutil repairVolume <vol>` | Repair filesystem | `# diskutil repairVolume disk1` |
| `diskutil apfs list` | List APFS volumes | `$ diskutil apfs list` |
| `hdiutil attach <dmg>` | Mount disk image | `$ hdiutil attach image.dmg` |
| `hdiutil detach <vol>` | Detach disk image | `$ hdiutil detach /Volumes/Image` |
| `hdiutil create` | Create disk image | `$ hdiutil create -size 1g -fs APFS -volname "MyDisk" disk.dmg` |
| `tmutil` | Time Machine utility | `$ tmutil listbackups` |

---

## User Management

| Command | Description | Example |
|---------|-------------|---------|
| `whoami` | Current user | `$ whoami` |
| `id` | User/group IDs | `$ id` |
| `groups` | Group memberships | `$ groups` |
| `dscl . -list /Users` | List all users | `$ dscl . -list /Users` |
| `dscl . -list /Groups` | List all groups | `$ dscl . -list /Groups` |
| `dscl . -read /Users/<user>` | User details | `$ dscl . -read /Users/admin` |
| `dscacheutil -q user` | Query user cache | `$ dscacheutil -q user` |
| `sysadminctl -addUser` | Add user | `# sysadminctl -addUser joe -password pass` |
| `sysadminctl -deleteUser` | Delete user | `# sysadminctl -deleteUser joe` |
| `sudo -s` | Root shell | `$ sudo -s` |
| `su - <user>` | Switch user | `$ su - otheruser` |

---

## Service and Process Management

### launchctl

| Command | Description | Example |
|---------|-------------|---------|
| `launchctl list` | List loaded services | `$ launchctl list` |
| `launchctl list \| grep <name>` | Find service | `$ launchctl list \| grep apache` |
| `launchctl load <plist>` | Load service (legacy) | `$ launchctl load ~/Library/LaunchAgents/job.plist` |
| `launchctl unload <plist>` | Unload service (legacy) | `$ launchctl unload ~/Library/LaunchAgents/job.plist` |
| `launchctl bootstrap gui/<uid> <plist>` | Bootstrap service | `$ launchctl bootstrap gui/501 job.plist` |
| `launchctl bootout gui/<uid> <plist>` | Remove service | `$ launchctl bootout gui/501 job.plist` |
| `launchctl kickstart` | Start service | `$ launchctl kickstart gui/501/com.example.agent` |
| `launchctl kill <sig> <service>` | Send signal | `# launchctl kill SIGTERM system/com.example.daemon` |
| `launchctl print system` | Print system domain | `$ launchctl print system` |
| `launchctl print gui/501` | Print user domain | `$ launchctl print gui/501` |

### Homebrew Services

| Command | Description | Example |
|---------|-------------|---------|
| `brew services list` | List services | `$ brew services list` |
| `brew services start <name>` | Start service | `$ brew services start postgresql` |
| `brew services stop <name>` | Stop service | `$ brew services stop postgresql` |
| `brew services restart <name>` | Restart service | `$ brew services restart nginx` |
| `brew services run <name>` | Run once (no restart) | `$ brew services run redis` |

---

## Package Management (Homebrew)

| Command | Description | Example |
|---------|-------------|---------|
| `brew install <pkg>` | Install package | `$ brew install wget` |
| `brew uninstall <pkg>` | Uninstall package | `$ brew uninstall wget` |
| `brew upgrade` | Upgrade all packages | `$ brew upgrade` |
| `brew upgrade <pkg>` | Upgrade specific package | `$ brew upgrade node` |
| `brew update` | Update Homebrew | `$ brew update` |
| `brew search <term>` | Search packages | `$ brew search python` |
| `brew info <pkg>` | Package information | `$ brew info python` |
| `brew list` | List installed packages | `$ brew list` |
| `brew list --cask` | List installed casks | `$ brew list --cask` |
| `brew outdated` | Show outdated packages | `$ brew outdated` |
| `brew cleanup` | Remove old versions | `$ brew cleanup` |
| `brew doctor` | Diagnose issues | `$ brew doctor` |
| `brew deps <pkg>` | Show dependencies | `$ brew deps node` |
| `brew uses --installed <pkg>` | What uses this package | `$ brew uses --installed openssl` |
| `brew install --cask <app>` | Install GUI app | `$ brew install --cask firefox` |

---

## macOS-Specific Commands

### Clipboard

| Command | Description | Example |
|---------|-------------|---------|
| `pbcopy` | Copy to clipboard | `$ cat file.txt \| pbcopy` |
| `pbpaste` | Paste from clipboard | `$ pbpaste > output.txt` |
| `pbcopy < file.txt` | Copy file contents | `$ pbcopy < ~/.ssh/id_rsa.pub` |

### Launching and Opening

| Command | Description | Example |
|---------|-------------|---------|
| `open <file>` | Open with default app | `$ open document.pdf` |
| `open -a <app> <file>` | Open with specific app | `$ open -a "Visual Studio Code" .` |
| `open .` | Open current dir in Finder | `$ open .` |
| `open <url>` | Open URL in browser | `$ open https://apple.com` |
| `open -R <file>` | Reveal in Finder | `$ open -R file.txt` |

### System

| Command | Description | Example |
|---------|-------------|---------|
| `say <text>` | Text-to-speech | `$ say "Hello world"` |
| `screencapture` | Take screenshot | `$ screencapture screen.png` |
| `screencapture -c` | Screenshot to clipboard | `$ screencapture -c` |
| `pmset -g` | Power management status | `$ pmset -g` |
| `pmset -g batt` | Battery status | `$ pmset -g batt` |
| `caffeinate` | Prevent sleep | `$ caffeinate -t 3600` |
| `softwareupdate -l` | List software updates | `$ softwareupdate -l` |
| `softwareupdate -ia` | Install all updates | `# softwareupdate -ia` |
| `defaults read` | Read preferences | `$ defaults read com.apple.finder` |
| `defaults write` | Write preferences | `$ defaults write com.apple.finder ShowHiddenFiles -bool true` |
| `osascript -e` | Run AppleScript | `$ osascript -e 'display notification "Done"'` |

### Spotlight

| Command | Description | Example |
|---------|-------------|---------|
| `mdfind <query>` | Spotlight search | `$ mdfind "meeting notes"` |
| `mdfind -name <name>` | Search by name | `$ mdfind -name "config.yaml"` |
| `mdfind -onlyin <dir> <query>` | Search in directory | `$ mdfind -onlyin ~/Documents "report"` |
| `mdls <file>` | File metadata | `$ mdls photo.jpg` |
| `mdutil -s /` | Spotlight status | `$ mdutil -s /` |
| `mdutil -E /` | Rebuild Spotlight index | `# mdutil -E /` |

---

## Logging and Debugging

| Command | Description | Example |
|---------|-------------|---------|
| `log show` | View unified log | `$ log show --last 1h` |
| `log show --predicate` | Filter logs | `$ log show --predicate 'process == "Safari"'` |
| `log stream` | Real-time log stream | `$ log stream --level debug` |
| `console` | Open Console.app | `$ open -a Console` |
| `syslog` | Legacy system log (deprecated) | `$ syslog` |
| `dmesg` | Kernel messages | `$ dmesg` |
| `system_profiler SPLogsDataType` | System logs info | `$ system_profiler SPLogsDataType` |

---

## Security Commands

| Command | Description | Example |
|---------|-------------|---------|
| `csrutil status` | SIP status | `$ csrutil status` |
| `spctl --status` | Gatekeeper status | `$ spctl --status` |
| `spctl -a -v <app>` | Verify app signature | `$ spctl -a -v /Applications/Safari.app` |
| `codesign -dv <app>` | Code signature details | `$ codesign -dv /Applications/Safari.app` |
| `codesign -s <identity> <file>` | Sign code | `$ codesign -s "Developer ID" myapp` |
| `security list-keychains` | List keychains | `$ security list-keychains` |
| `security find-identity -v` | List signing identities | `$ security find-identity -v` |
| `security find-generic-password` | Find password | `$ security find-generic-password -s "service"` |
| `fdesetup status` | FileVault status | `$ fdesetup status` |
| `sudo spctl --master-disable` | Disable Gatekeeper | `# spctl --master-disable` |

---

## Remote Access

| Command | Description | Example |
|---------|-------------|---------|
| `ssh <user>@<host>` | SSH connection | `$ ssh admin@server.local` |
| `ssh -i <key> <user>@<host>` | SSH with key | `$ ssh -i ~/.ssh/mykey user@host` |
| `ssh -L <local>:<remote>` | SSH tunnel | `$ ssh -L 8080:localhost:80 user@host` |
| `scp <src> <user>@<host>:<dst>` | Secure copy | `$ scp file.txt user@host:/path/` |
| `scp -r <dir> <user>@<host>:<dst>` | Recursive copy | `$ scp -r dir/ user@host:/path/` |
| `rsync -avz <src> <dst>` | Sync files | `$ rsync -avz dir/ user@host:/path/` |
| `sftp <user>@<host>` | SFTP session | `$ sftp admin@server.local` |

---

## Shell Built-ins and History

| Command | Description | Example |
|---------|-------------|---------|
| `history` | Show command history | `$ history` |
| `history \| grep <term>` | Search history | `$ history \| grep git` |
| `!!` | Repeat last command | `$ !!` |
| `!<n>` | Run command N from history | `$ !42` |
| `!<string>` | Run last command starting with | `$ !git` |
| `Ctrl+R` | Reverse search history | |
| `alias` | Show aliases | `$ alias` |
| `alias <name>=<cmd>` | Create alias | `$ alias ll='ls -la'` |
| `unalias <name>` | Remove alias | `$ unalias ll` |
| `export <var>=<val>` | Set environment variable | `$ export PATH="/usr/local/bin:$PATH"` |
| `env` | Show environment | `$ env` |
| `printenv` | Print environment variables | `$ printenv` |
| `echo $<var>` | Print variable | `$ echo $HOME` |

---

## Getting Help

| Command | Description | Example |
|---------|-------------|---------|
| `man <command>` | Manual page | `$ man ls` |
| `man -k <term>` | Search manual pages | `$ man -k network` |
| `apropos <term>` | Same as man -k | `$ apropos copy` |
| `<command> --help` | Command help | `$ git --help` |
| `<command> -h` | Short help | `$ grep -h` |
| `whatis <command>` | One-line description | `$ whatis curl` |
| `info <command>` | GNU info pages | `$ info bash` |

---

## Quick Reference: Keyboard Shortcuts in Commands

| Shortcut | Action |
|----------|--------|
| `Ctrl+C` | Cancel/interrupt |
| `Ctrl+D` | EOF/logout |
| `Ctrl+Z` | Suspend process |
| `Ctrl+L` | Clear screen |
| `Ctrl+A` | Beginning of line |
| `Ctrl+E` | End of line |
| `Ctrl+U` | Clear line before cursor |
| `Ctrl+K` | Clear line after cursor |
| `Ctrl+W` | Delete word before cursor |
| `Ctrl+R` | Reverse search history |
| `Tab` | Auto-complete |
| `Tab Tab` | Show completions |
