# Security Best Practices

Securing macOS requires a layered approach that combines Apple's built-in security features with good operational practices. This chapter provides a comprehensive hardening checklist, introduces security auditing tools, and outlines monitoring strategies for maintaining security over time.

## Security Hardening Checklist

### Essential Security (All Macs)

```bash
#!/bin/bash
# essential-security-check.sh - Verify essential security settings

echo "=== Essential Security Checklist ==="

# 1. FileVault
echo -n "[1/10] FileVault: "
if fdesetup status | grep -q "On"; then
    echo "ENABLED"
else
    echo "DISABLED - Enable in System Preferences > Security & Privacy > FileVault"
fi

# 2. System Integrity Protection
echo -n "[2/10] SIP: "
if csrutil status | grep -q "enabled"; then
    echo "ENABLED"
else
    echo "DISABLED - Enable via Recovery Mode"
fi

# 3. Gatekeeper
echo -n "[3/10] Gatekeeper: "
if spctl --status | grep -q "enabled"; then
    echo "ENABLED"
else
    echo "DISABLED - Run: sudo spctl --master-enable"
fi

# 4. Firewall
echo -n "[4/10] Firewall: "
FW=$(/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate 2>/dev/null)
if echo "$FW" | grep -q "enabled"; then
    echo "ENABLED"
else
    echo "DISABLED - Enable in System Preferences > Security & Privacy > Firewall"
fi

# 5. Auto-login disabled
echo -n "[5/10] Auto-login: "
if defaults read /Library/Preferences/com.apple.loginwindow autoLoginUser 2>/dev/null; then
    echo "ENABLED - SECURITY RISK"
else
    echo "DISABLED (good)"
fi

# 6. Password after sleep
echo -n "[6/10] Password on wake: "
SCREEN_SAVER=$(defaults read com.apple.screensaver askForPassword 2>/dev/null)
if [[ "$SCREEN_SAVER" == "1" ]]; then
    echo "ENABLED"
else
    echo "DISABLED - Enable in System Preferences > Security & Privacy"
fi

# 7. Remote login
echo -n "[7/10] SSH: "
if sudo systemsetup -getremotelogin 2>/dev/null | grep -q "Off"; then
    echo "DISABLED (good unless needed)"
else
    echo "ENABLED - Verify this is intentional"
fi

# 8. Screen sharing
echo -n "[8/10] Screen Sharing: "
if launchctl list | grep -q screensharing; then
    echo "ENABLED - Verify this is intentional"
else
    echo "DISABLED (good)"
fi

# 9. Find My Mac
echo -n "[9/10] Find My Mac: "
if defaults read ~/Library/Preferences/com.apple.finder.plist FXEnableFindMyMac 2>/dev/null | grep -q "1"; then
    echo "ENABLED"
else
    echo "Check in System Preferences > Apple ID > iCloud"
fi

# 10. Software updates
echo -n "[10/10] Auto Updates: "
if defaults read /Library/Preferences/com.apple.SoftwareUpdate AutomaticCheckEnabled 2>/dev/null | grep -q "1"; then
    echo "ENABLED"
else
    echo "DISABLED - Enable automatic updates"
fi

echo "=== Checklist Complete ==="
```

### Network Security

```bash
#!/bin/bash
# network-security-check.sh - Audit network security settings

echo "=== Network Security Audit ==="

# Firewall details
echo "--- Firewall Configuration ---"
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
/usr/libexec/ApplicationFirewall/socketfilterfw --getblockall
/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode

# Listening services
echo
echo "--- Listening Services ---"
sudo lsof -iTCP -sTCP:LISTEN -n -P | awk 'NR==1 || $9 !~ /127\.0\.0\.1|::1/' | head -20

# Sharing services
echo
echo "--- Sharing Services ---"
echo -n "Remote Login (SSH): "
sudo systemsetup -getremotelogin 2>/dev/null

echo -n "Screen Sharing: "
sudo launchctl list | grep -q com.apple.screensharing && echo "Enabled" || echo "Disabled"

echo -n "File Sharing (SMB): "
sudo launchctl list | grep -q com.apple.smbd && echo "Enabled" || echo "Disabled"

echo -n "Remote Apple Events: "
sudo systemsetup -getremoteappleevents 2>/dev/null

# Network interfaces
echo
echo "--- Active Interfaces ---"
ifconfig | grep -E "^[a-z]|inet " | grep -v "127.0.0.1"

echo "=== Network Audit Complete ==="
```

### Application Security

```bash
#!/bin/bash
# app-security-check.sh - Audit application security

echo "=== Application Security Audit ==="

# Check for unsigned applications
echo "--- Unsigned Applications in /Applications ---"
for app in /Applications/*.app; do
    if ! codesign -v "$app" 2>/dev/null; then
        echo "UNSIGNED: $app"
    fi
done

# Check Gatekeeper assessments
echo
echo "--- Gatekeeper Assessment Failures ---"
for app in /Applications/*.app; do
    result=$(spctl --assess -v "$app" 2>&1)
    if echo "$result" | grep -q "rejected"; then
        echo "REJECTED: $app"
    fi
done

# Check for quarantined files
echo
echo "--- Quarantined Files in Downloads ---"
find ~/Downloads -xattr -print0 2>/dev/null | xargs -0 -I {} sh -c 'xattr -l "{}" 2>/dev/null | grep -q quarantine && echo "{}"' | head -20

# Browser extensions (common locations)
echo
echo "--- Installed Browser Extensions ---"
ls ~/Library/Application\ Support/Google/Chrome/Default/Extensions/ 2>/dev/null | wc -l | xargs echo "Chrome extensions:"
ls ~/Library/Safari/Extensions/ 2>/dev/null | wc -l | xargs echo "Safari extensions:"

echo "=== Application Audit Complete ==="
```

## Security Configuration Scripts

### Enable Firewall

```bash
#!/bin/bash
# enable-firewall.sh - Configure application firewall

# Enable firewall
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# Enable stealth mode (don't respond to pings)
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on

# Block all incoming (except essential services)
# WARNING: This may break some apps
# sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setblockall on

# Allow signed apps automatically
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setallowsigned on
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setallowsignedapp on

# Log firewall events
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setloggingmode on

echo "Firewall configured. Restart may be required."
```

### Secure SSH Configuration

```bash
#!/bin/bash
# secure-ssh.sh - Harden SSH configuration

# Backup original config
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Create secure configuration
sudo tee /etc/ssh/sshd_config.d/hardening.conf << 'EOF'
# SSH Hardening Configuration

# Disable root login
PermitRootLogin no

# Use SSH protocol 2 only (default in modern OpenSSH)
Protocol 2

# Disable password authentication (use keys)
PasswordAuthentication no
ChallengeResponseAuthentication no

# Limit authentication attempts
MaxAuthTries 3

# Set idle timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# Disable X11 forwarding
X11Forwarding no

# Disable TCP forwarding (if not needed)
AllowTcpForwarding no

# Only allow specific users (customize as needed)
# AllowUsers admin

# Use strong ciphers
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256
EOF

# Restart SSH
sudo launchctl kickstart -k system/com.openssh.sshd

echo "SSH hardened. Test connection before closing current session!"
```

### Secure Safari Settings

```bash
#!/bin/bash
# secure-safari.sh - Configure Safari security settings

# Disable auto-open of safe downloads
defaults write com.apple.Safari AutoOpenSafeDownloads -bool false

# Enable fraud warnings
defaults write com.apple.Safari WarnAboutFraudulentWebsites -bool true

# Disable plugins
defaults write com.apple.Safari WebKitPluginsEnabled -bool false
defaults write com.apple.Safari com.apple.Safari.ContentPageGroupIdentifier.WebKit2PluginsEnabled -bool false

# Enable Do Not Track
defaults write com.apple.Safari SendDoNotTrackHTTPHeader -bool true

# Block pop-ups
defaults write com.apple.Safari WebKitJavaScriptCanOpenWindowsAutomatically -bool false
defaults write com.apple.Safari com.apple.Safari.ContentPageGroupIdentifier.WebKit2JavaScriptCanOpenWindowsAutomatically -bool false

# Disable auto-fill
defaults write com.apple.Safari AutoFillFromAddressBook -bool false
defaults write com.apple.Safari AutoFillPasswords -bool false
defaults write com.apple.Safari AutoFillCreditCardData -bool false

echo "Safari security settings configured. Restart Safari to apply."
```

## Security Monitoring

### System Log Monitoring

```bash
#!/bin/bash
# security-monitor.sh - Monitor security-related events

echo "=== Security Event Monitor ==="
echo "Press Ctrl+C to stop"
echo

# Monitor security-related logs
log stream --predicate '
    subsystem == "com.apple.securityd" OR
    subsystem == "com.apple.TCC" OR
    subsystem == "com.apple.sandbox" OR
    subsystem == "com.apple.authd" OR
    eventMessage CONTAINS[c] "authentication" OR
    eventMessage CONTAINS[c] "password" OR
    eventMessage CONTAINS[c] "denied" OR
    eventMessage CONTAINS[c] "violation"
' --style compact
```

### Failed Login Monitoring

```bash
#!/bin/bash
# failed-logins.sh - Check for failed login attempts

echo "=== Failed Login Attempts (Last 24 Hours) ==="

# Authentication failures
log show --predicate 'process == "authd" AND eventMessage CONTAINS "failed"' --last 24h

# SSH failures (if SSH is enabled)
log show --predicate 'process == "sshd" AND eventMessage CONTAINS "Failed"' --last 24h | tail -20
```

### File Integrity Monitoring

```bash
#!/bin/bash
# integrity-monitor.sh - Basic file integrity monitoring

BASELINE_FILE="$HOME/.security/baseline.txt"
ALERT_LOG="$HOME/.security/alerts.log"

mkdir -p "$HOME/.security"

# Critical paths to monitor
PATHS=(
    "/etc/hosts"
    "/etc/passwd"
    "/etc/sudoers"
    "/Library/LaunchDaemons"
    "/Library/LaunchAgents"
    "$HOME/Library/LaunchAgents"
)

generate_baseline() {
    echo "Generating baseline..."
    for path in "${PATHS[@]}"; do
        if [[ -e "$path" ]]; then
            if [[ -d "$path" ]]; then
                find "$path" -type f -exec shasum -a 256 {} \; 2>/dev/null
            else
                shasum -a 256 "$path" 2>/dev/null
            fi
        fi
    done > "$BASELINE_FILE"
    echo "Baseline saved to $BASELINE_FILE"
}

check_integrity() {
    if [[ ! -f "$BASELINE_FILE" ]]; then
        echo "No baseline found. Run with 'baseline' first."
        exit 1
    fi

    echo "Checking file integrity..."
    CHANGES=0

    while IFS= read -r line; do
        hash=$(echo "$line" | awk '{print $1}')
        file=$(echo "$line" | awk '{print $2}')

        if [[ -f "$file" ]]; then
            current_hash=$(shasum -a 256 "$file" | awk '{print $1}')
            if [[ "$hash" != "$current_hash" ]]; then
                echo "MODIFIED: $file"
                echo "$(date): MODIFIED: $file" >> "$ALERT_LOG"
                ((CHANGES++))
            fi
        else
            echo "MISSING: $file"
            echo "$(date): MISSING: $file" >> "$ALERT_LOG"
            ((CHANGES++))
        fi
    done < "$BASELINE_FILE"

    # Check for new files
    for path in "${PATHS[@]}"; do
        if [[ -d "$path" ]]; then
            while IFS= read -r file; do
                if ! grep -q "$file" "$BASELINE_FILE" 2>/dev/null; then
                    echo "NEW: $file"
                    echo "$(date): NEW: $file" >> "$ALERT_LOG"
                    ((CHANGES++))
                fi
            done < <(find "$path" -type f 2>/dev/null)
        fi
    done

    if [[ $CHANGES -eq 0 ]]; then
        echo "No changes detected."
    else
        echo "$CHANGES change(s) detected!"
    fi
}

case "$1" in
    baseline)
        generate_baseline
        ;;
    check)
        check_integrity
        ;;
    *)
        echo "Usage: $0 {baseline|check}"
        exit 1
        ;;
esac
```

## Security Tools

### Built-in Tools

| Tool | Purpose | Example |
|------|---------|---------|
| `codesign` | Verify code signatures | `codesign -v /Applications/App.app` |
| `spctl` | Gatekeeper control | `spctl --assess -v /path/to/app` |
| `csrutil` | SIP management | `csrutil status` |
| `fdesetup` | FileVault management | `fdesetup status` |
| `security` | Keychain management | `security find-generic-password` |
| `log` | System log access | `log show --predicate 'subsystem == "com.apple.securityd"'` |
| `tccutil` | TCC permission reset | `tccutil reset Camera` |
| `xattr` | Extended attributes | `xattr -d com.apple.quarantine file` |

### Third-Party Security Tools

```bash
# Install common security tools via Homebrew

# Network security
brew install nmap              # Network scanner
brew install wireshark         # Packet analyzer
brew install mtr               # Network diagnostics

# System security
brew install lynis             # Security auditing
brew install osquery           # System instrumentation
brew install rkhunter          # Rootkit hunter

# Password/credential tools
brew install pwgen             # Password generator
brew install 1password-cli     # 1Password CLI

# Malware scanning
brew install clamav            # Antivirus
```

### Running Lynis Security Audit

```bash
# Install Lynis
$ brew install lynis

# Run audit (comprehensive security scan)
$ sudo lynis audit system

# Quick audit
$ sudo lynis audit system --quick

# View report
$ cat /var/log/lynis-report.dat
```

## Privacy Hardening

### Disable Telemetry

```bash
#!/bin/bash
# disable-telemetry.sh - Reduce data sent to Apple

# Disable Siri analytics
defaults write com.apple.assistant.support "Siri Data Sharing Opt-In Status" -int 2

# Disable analytics sharing
defaults write com.apple.CrashReporter DialogType -string "none"

# Disable personalized ads
defaults write com.apple.AdLib allowApplePersonalizedAdvertising -bool false

# Disable location-based suggestions
defaults write com.apple.assistant.support "Allow Location Suggestions" -bool false

echo "Telemetry settings updated. Restart for full effect."
```

### Secure DNS

```bash
#!/bin/bash
# Configure encrypted DNS (DNS over HTTPS)

# Option 1: Use a configuration profile (recommended)
# Create a .mobileconfig file with DNS settings

# Option 2: Use networksetup
# Set DNS servers (example: Cloudflare)
networksetup -setdnsservers Wi-Fi 1.1.1.1 1.0.0.1

# Verify
networksetup -getdnsservers Wi-Fi

# Flush DNS cache
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

## Incident Response

### Collecting Evidence

```bash
#!/bin/bash
# collect-evidence.sh - Collect system state for incident analysis

EVIDENCE_DIR="$HOME/evidence-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$EVIDENCE_DIR"

echo "Collecting system evidence..."

# System info
system_profiler > "$EVIDENCE_DIR/system_profiler.txt"

# Running processes
ps aux > "$EVIDENCE_DIR/processes.txt"

# Network connections
netstat -an > "$EVIDENCE_DIR/netstat.txt"
lsof -i > "$EVIDENCE_DIR/network_connections.txt"

# Open files
lsof > "$EVIDENCE_DIR/open_files.txt"

# Login history
last > "$EVIDENCE_DIR/last_logins.txt"

# Installed apps
ls -la /Applications > "$EVIDENCE_DIR/applications.txt"

# Launch items
ls -la /Library/LaunchDaemons > "$EVIDENCE_DIR/launch_daemons.txt"
ls -la /Library/LaunchAgents > "$EVIDENCE_DIR/launch_agents_system.txt"
ls -la ~/Library/LaunchAgents > "$EVIDENCE_DIR/launch_agents_user.txt"

# Recent logs
log show --last 1h > "$EVIDENCE_DIR/recent_logs.txt"

# Kernel extensions
kextstat > "$EVIDENCE_DIR/kexts.txt"

# Browser history (if accessible)
cp ~/Library/Safari/History.db "$EVIDENCE_DIR/" 2>/dev/null
cp ~/Library/Application\ Support/Google/Chrome/Default/History "$EVIDENCE_DIR/chrome_history" 2>/dev/null

# Create archive
tar -czf "$EVIDENCE_DIR.tar.gz" "$EVIDENCE_DIR"
rm -rf "$EVIDENCE_DIR"

echo "Evidence collected to $EVIDENCE_DIR.tar.gz"
echo "Preserve this file for analysis"
```

### Malware Removal Checklist

```bash
#!/bin/bash
# malware-check.sh - Basic malware detection steps

echo "=== Malware Detection Checklist ==="

# 1. Check for suspicious launch items
echo "--- Checking Launch Items ---"
echo "System LaunchDaemons:"
ls /Library/LaunchDaemons/ | grep -v "com.apple"

echo "System LaunchAgents:"
ls /Library/LaunchAgents/ | grep -v "com.apple"

echo "User LaunchAgents:"
ls ~/Library/LaunchAgents/ 2>/dev/null

# 2. Check login items
echo
echo "--- Login Items ---"
osascript -e 'tell application "System Events" to get the name of every login item'

# 3. Check browser extensions
echo
echo "--- Browser Extensions ---"
ls ~/Library/Safari/Extensions/ 2>/dev/null
ls ~/Library/Application\ Support/Google/Chrome/Default/Extensions/ 2>/dev/null | head -10

# 4. Check for suspicious processes
echo
echo "--- Suspicious Processes ---"
ps aux | grep -E "^[^root]" | grep -v "^$USER" | grep -v "_"

# 5. Check network connections
echo
echo "--- Unusual Network Connections ---"
lsof -i | grep ESTABLISHED | grep -v -E "Safari|Chrome|firefox|Mail|Messages" | head -10

# 6. Check kernel extensions
echo
echo "--- Third-party Kernel Extensions ---"
kextstat | grep -v com.apple

# 7. Check profiles
echo
echo "--- Configuration Profiles ---"
profiles -P 2>/dev/null

echo
echo "=== Review output for suspicious items ==="
echo "If malware is found, consider professional help or factory reset"
```

## Summary

Security best practices for macOS:

### Essential Security

1. **Enable FileVault** - Encrypt your disk
2. **Keep SIP enabled** - Don't disable without good reason
3. **Enable Gatekeeper** - Block unsigned apps
4. **Turn on Firewall** - Control network access
5. **Disable auto-login** - Require authentication
6. **Enable automatic updates** - Stay patched
7. **Use strong passwords** - With a password manager
8. **Enable Find My Mac** - For lost/stolen devices

### Monitoring Checklist

- [ ] Review security logs weekly
- [ ] Audit TCC permissions monthly
- [ ] Check for unsigned apps quarterly
- [ ] Update security baseline after changes
- [ ] Test backups regularly
- [ ] Review user accounts quarterly

### Quick Security Commands

```bash
# Check overall security posture
$ fdesetup status && csrutil status && spctl --status

# Review recent security events
$ log show --predicate 'subsystem == "com.apple.securityd"' --last 1h

# Audit network services
$ lsof -iTCP -sTCP:LISTEN -n -P

# Check for unsigned apps
$ for app in /Applications/*.app; do codesign -v "$app" 2>/dev/null || echo "Unsigned: $app"; done

# Reset suspicious app's permissions
$ tccutil reset All com.suspicious.app
```

Security is an ongoing process, not a one-time configuration. Regular audits, updates, and vigilance are essential for maintaining a secure macOS environment.
