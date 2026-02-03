# SSH Configuration and Usage

SSH (Secure Shell) is fundamental to modern development workflows. Whether you're deploying to servers, managing cloud infrastructure, or pushing code to GitHub, SSH handles the secure communication. macOS includes a full OpenSSH implementation, and understanding its configuration can dramatically improve your productivity.

## SSH Basics

### How SSH Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    SSH Connection                                │
│                                                                  │
│  ┌─────────────┐           Encrypted Channel          ┌─────────────┐
│  │  SSH Client │ ◄─────────────────────────────────► │  SSH Server │
│  │   (macOS)   │                                      │   (Remote)  │
│  └─────────────┘                                      └─────────────┘
│                                                                  │
│  Authentication Methods:                                         │
│  1. Public Key (recommended)                                     │
│  2. Password (fallback)                                          │
│  3. Keyboard-interactive                                         │
│  4. Certificate-based                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Basic SSH Connection

```bash
# Connect to a remote host
$ ssh user@hostname

# Connect on non-standard port
$ ssh -p 2222 user@hostname

# Connect with specific identity (key)
$ ssh -i ~/.ssh/specific_key user@hostname

# Run a command remotely
$ ssh user@hostname "ls -la /var/log"

# Run command and stay connected
$ ssh -t user@hostname "cd /app && bash"
```

## SSH Key Management

### Generating SSH Keys

```bash
# Generate Ed25519 key (recommended for modern systems)
$ ssh-keygen -t ed25519 -C "your_email@example.com"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/user/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:

# Generate RSA key (broader compatibility)
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Generate key with specific filename
$ ssh-keygen -t ed25519 -f ~/.ssh/github_key -C "github key"

# View public key
$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG... your_email@example.com
```

### Key Types Comparison

| Type | Security | Compatibility | Recommendation |
|------|----------|---------------|----------------|
| Ed25519 | Excellent | Modern systems | Recommended |
| RSA 4096 | Very good | Universal | Legacy/compatibility |
| ECDSA | Good | Wide | Alternative |
| DSA | Deprecated | Old systems | Avoid |

### Installing Keys on Remote Servers

```bash
# Using ssh-copy-id (easiest)
$ ssh-copy-id user@hostname
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/Users/user/.ssh/id_ed25519.pub"
user@hostname's password:
Number of key(s) added: 1

# Specify a particular key
$ ssh-copy-id -i ~/.ssh/specific_key.pub user@hostname

# Manual method (if ssh-copy-id unavailable)
$ cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Copy key to GitHub via API
$ cat ~/.ssh/id_ed25519.pub | pbcopy
# Then paste in GitHub Settings → SSH keys
```

### Key Permissions

SSH is strict about file permissions:

```bash
# Set correct permissions
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/id_ed25519
$ chmod 644 ~/.ssh/id_ed25519.pub
$ chmod 600 ~/.ssh/config
$ chmod 600 ~/.ssh/authorized_keys

# Fix permissions recursively
$ chmod 700 ~/.ssh && chmod 600 ~/.ssh/*

# Verify permissions
$ ls -la ~/.ssh
drwx------   8 user  staff   256 Jan  1 12:00 .
-rw-------   1 user  staff   464 Jan  1 12:00 id_ed25519
-rw-r--r--   1 user  staff   100 Jan  1 12:00 id_ed25519.pub
-rw-------   1 user  staff   500 Jan  1 12:00 config
```

## SSH Configuration File

The SSH config file (`~/.ssh/config`) is the key to efficient SSH usage.

### Basic Configuration

```bash
# ~/.ssh/config

# Default settings for all hosts
Host *
    AddKeysToAgent yes
    UseKeychain yes  # macOS-specific: store passphrase in Keychain
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Work server
Host work
    HostName work-server.example.com
    User admin
    Port 22
    IdentityFile ~/.ssh/work_key

# Personal server
Host personal
    HostName my-server.example.com
    User myuser
    IdentityFile ~/.ssh/personal_key
    ForwardAgent yes

# Connect with: ssh work
#          or: ssh personal
```

### Pattern Matching

```bash
# Match multiple hosts with patterns
Host dev-*
    User developer
    IdentityFile ~/.ssh/dev_key

Host prod-*
    User deployer
    IdentityFile ~/.ssh/prod_key
    ForwardAgent no

# Specific server overrides pattern
Host dev-db
    HostName dev-database.example.com
    LocalForward 5432 localhost:5432

# Matches: ssh dev-web, ssh dev-api, ssh prod-web, etc.
```

### SSH Config Options Reference

| Option | Description | Example |
|--------|-------------|---------|
| HostName | Real hostname/IP | `HostName 192.168.1.100` |
| User | Login username | `User admin` |
| Port | SSH port | `Port 2222` |
| IdentityFile | Path to private key | `IdentityFile ~/.ssh/key` |
| IdentitiesOnly | Only use specified keys | `IdentitiesOnly yes` |
| ForwardAgent | Forward ssh-agent | `ForwardAgent yes` |
| ProxyJump | Jump through host | `ProxyJump bastion` |
| LocalForward | Forward local port | `LocalForward 8080 localhost:80` |
| RemoteForward | Forward remote port | `RemoteForward 9090 localhost:9090` |
| DynamicForward | SOCKS proxy | `DynamicForward 1080` |
| ServerAliveInterval | Keep-alive interval | `ServerAliveInterval 60` |
| Compression | Enable compression | `Compression yes` |
| StrictHostKeyChecking | Host key checking | `StrictHostKeyChecking ask` |

## SSH Agent

The SSH agent holds your decrypted private keys in memory, so you don't have to enter your passphrase repeatedly.

### macOS SSH Agent

macOS has an integrated ssh-agent that works with Keychain:

```bash
# Start ssh-agent (usually automatic on macOS)
$ eval "$(ssh-agent -s)"
Agent pid 12345

# Add key to agent
$ ssh-add ~/.ssh/id_ed25519
Enter passphrase for /Users/user/.ssh/id_ed25519:
Identity added: /Users/user/.ssh/id_ed25519

# Add key and store passphrase in Keychain (macOS-specific)
$ ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# List keys in agent
$ ssh-add -l
256 SHA256:abc123... user@email.com (ED25519)

# Remove all keys from agent
$ ssh-add -D
All identities removed.

# Remove specific key
$ ssh-add -d ~/.ssh/id_ed25519
```

### Automatic Key Loading

Configure in `~/.ssh/config`:

```bash
Host *
    AddKeysToAgent yes
    UseKeychain yes  # macOS: store passphrase in Keychain
    IdentityFile ~/.ssh/id_ed25519
```

With this configuration:
1. First connection prompts for passphrase
2. Passphrase is stored in macOS Keychain
3. Subsequent connections use stored passphrase

### Agent Forwarding

Forward your local SSH agent to remote servers:

```bash
# Enable in config
Host server
    ForwardAgent yes

# Or on command line
$ ssh -A user@server

# On the remote server, you can now SSH to other servers
# using your local keys
user@server$ ssh git@github.com
# Works without having keys on the server
```

**Security Warning**: Only enable agent forwarding to trusted servers. A compromised server could use your forwarded agent.

### Managing Multiple Keys

```bash
# ~/.ssh/config - different keys for different services

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes

Host gitlab.com
    HostName gitlab.com
    User git
    IdentityFile ~/.ssh/gitlab_key
    IdentitiesOnly yes

Host bitbucket.org
    HostName bitbucket.org
    User git
    IdentityFile ~/.ssh/bitbucket_key
    IdentitiesOnly yes

# Work servers use work key
Host *.work.example.com
    IdentityFile ~/.ssh/work_key

# Personal servers use personal key
Host *.personal.example.com
    IdentityFile ~/.ssh/personal_key
```

## ProxyJump: Jumping Through Bastion Hosts

Many networks require connecting through a bastion (jump) host:

```
┌─────────────┐        ┌─────────────┐        ┌─────────────┐
│  Your Mac   │──SSH──▶│   Bastion   │──SSH──▶│ Target Host │
│             │        │  (Public)   │        │  (Private)  │
└─────────────┘        └─────────────┘        └─────────────┘
```

### Using ProxyJump

```bash
# Command line
$ ssh -J bastion.example.com user@internal-server

# Multiple jumps
$ ssh -J jump1.example.com,jump2.example.com user@target

# In config file
Host bastion
    HostName bastion.example.com
    User admin

Host internal-*
    ProxyJump bastion
    User developer

Host internal-web
    HostName 10.0.1.10

Host internal-db
    HostName 10.0.1.20
    LocalForward 5432 localhost:5432

# Now connect directly:
$ ssh internal-web
# Automatically jumps through bastion
```

### Legacy ProxyCommand

For older SSH versions or complex scenarios:

```bash
Host internal-*
    ProxyCommand ssh -W %h:%p bastion.example.com

# Using netcat through bastion
Host legacy-internal
    ProxyCommand ssh bastion.example.com nc %h %p
```

## Connection Multiplexing

Multiplexing shares a single TCP connection for multiple SSH sessions, dramatically speeding up subsequent connections.

### Enabling Multiplexing

```bash
# ~/.ssh/config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600  # Keep connection for 10 minutes

# Create socket directory
$ mkdir -p ~/.ssh/sockets
$ chmod 700 ~/.ssh/sockets
```

### How It Works

```bash
# First connection: establishes TCP connection + SSH handshake
$ ssh server
# Takes ~500ms for key exchange, etc.

# Second connection: reuses existing connection
$ ssh server
# Takes ~50ms - almost instant

# View active control sockets
$ ls ~/.ssh/sockets
user@server-22

# Check control socket status
$ ssh -O check server
Master running (pid=12345)

# Close control socket
$ ssh -O exit server
Exit request sent.
```

### Multiplexing Options

| Option | Description |
|--------|-------------|
| ControlMaster auto | Create master if none exists |
| ControlMaster yes | Always create master |
| ControlMaster no | Don't use multiplexing |
| ControlPersist 600 | Keep master for 600 seconds |
| ControlPersist yes | Keep master indefinitely |

## Port Forwarding (Tunneling)

### Local Port Forwarding

Forward a local port to access a remote service:

```bash
# Access remote MySQL through local port
$ ssh -L 3306:localhost:3306 user@server
# Now connect to localhost:3306 to reach server's MySQL

# Access internal service through bastion
$ ssh -L 8080:internal-app:80 bastion
# localhost:8080 → bastion → internal-app:80

# In config file
Host db-tunnel
    HostName db-server.example.com
    User admin
    LocalForward 5432 localhost:5432
    LocalForward 6379 localhost:6379

# Multiple forwards
$ ssh -L 5432:localhost:5432 -L 6379:localhost:6379 server
```

### Remote Port Forwarding

Expose a local service to the remote server:

```bash
# Make local web server accessible on remote
$ ssh -R 8080:localhost:3000 server
# server:8080 → your machine:3000

# Allow external connections to forwarded port
$ ssh -R 0.0.0.0:8080:localhost:3000 server
# Requires GatewayPorts yes on server

# In config
Host expose-local
    HostName server.example.com
    RemoteForward 8080 localhost:3000
```

### Dynamic Port Forwarding (SOCKS Proxy)

```bash
# Create SOCKS5 proxy
$ ssh -D 1080 server

# Configure applications to use localhost:1080 as SOCKS proxy
# All traffic through that proxy goes via the SSH connection

# In config
Host proxy
    HostName server.example.com
    DynamicForward 1080

# Use with curl
$ curl --socks5 localhost:1080 https://example.com
```

## Practical SSH Configurations

### Developer Setup

```bash
# ~/.ssh/config - Complete developer configuration

# Global settings
Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

# GitHub
Host github.com
    User git
    IdentityFile ~/.ssh/github_ed25519

# GitLab
Host gitlab.com
    User git
    IdentityFile ~/.ssh/gitlab_ed25519

# Work infrastructure
Host bastion
    HostName bastion.work.example.com
    User myuser
    IdentityFile ~/.ssh/work_key

Host work-*
    ProxyJump bastion
    User myuser
    IdentityFile ~/.ssh/work_key

Host work-web
    HostName 10.0.1.10

Host work-api
    HostName 10.0.1.20

Host work-db
    HostName 10.0.1.30
    LocalForward 5432 localhost:5432
    LocalForward 6379 localhost:6379

# Personal servers
Host vps
    HostName my-vps.example.com
    User root
    IdentityFile ~/.ssh/personal_key
    ForwardAgent yes

# Raspberry Pi
Host pi
    HostName raspberrypi.local
    User pi
    IdentityFile ~/.ssh/pi_key
```

### CI/CD Access

```bash
# Service accounts with restricted access
Host deploy-*
    User deploy
    IdentityFile ~/.ssh/deploy_key
    IdentitiesOnly yes
    ForwardAgent no
    RequestTTY no

Host deploy-prod
    HostName prod.example.com
    # No port forwarding
    PermitLocalCommand no
```

## SSH Security Best Practices

### Key Security

```bash
# Use strong key types
$ ssh-keygen -t ed25519  # Recommended
$ ssh-keygen -t rsa -b 4096  # If Ed25519 not supported

# Always use passphrases on keys
# Store passphrases in macOS Keychain

# Rotate keys periodically
# Keep separate keys for different purposes
```

### Configuration Security

```bash
# Disable password authentication (on servers you control)
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes

# On client, be cautious with host key checking
Host trusted-server
    StrictHostKeyChecking yes  # Fail if key changes

Host new-servers
    StrictHostKeyChecking ask  # Prompt user

# Never use:
# StrictHostKeyChecking no  # DANGEROUS
```

### Audit SSH Usage

```bash
# View SSH authentication logs (macOS)
$ log show --predicate 'subsystem == "com.openssh.sshd"' --last 1h

# View all SSH attempts
$ log show --predicate 'processImagePath contains "ssh"' --last 1h

# Check authorized keys
$ cat ~/.ssh/authorized_keys

# List fingerprints of authorized keys
$ while read -r line; do echo "$line" | ssh-keygen -lf -; done < ~/.ssh/authorized_keys
```

## Troubleshooting SSH

### Verbose Output

```bash
# Increase verbosity
$ ssh -v user@host    # Verbose
$ ssh -vv user@host   # More verbose
$ ssh -vvv user@host  # Maximum verbosity

# Common issues revealed:
# - Key not being offered
# - Permission problems
# - Authentication method issues
```

### Common Issues

```bash
# Permission denied (publickey)
# Check:
$ ssh-add -l  # Is key loaded in agent?
$ ls -la ~/.ssh/  # Are permissions correct?
$ ssh -vv user@host  # What's actually happening?

# Host key verification failed
# The server's key changed (or MITM attack)
$ ssh-keygen -R hostname  # Remove old key
$ ssh user@hostname  # Accept new key

# Connection refused
# Check if SSH is running on server
$ nc -zv hostname 22

# Timeout issues
# Add keep-alive settings
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Too many authentication failures
# You're trying too many keys
Host specific-server
    IdentitiesOnly yes
    IdentityFile ~/.ssh/specific_key
```

### Debug SSH Config

```bash
# See effective configuration for a host
$ ssh -G hostname
user developer
hostname internal.example.com
port 22
identityfile /Users/user/.ssh/work_key
proxyjump bastion.example.com

# Test connection without connecting
$ ssh -T git@github.com
Hi username! You've successfully authenticated...
```

## Summary

| Task | Command/Config |
|------|----------------|
| Generate key | `ssh-keygen -t ed25519` |
| Add to agent | `ssh-add ~/.ssh/key` |
| Copy key | `ssh-copy-id user@host` |
| Quick connect | Configure in `~/.ssh/config` |
| Jump host | `ProxyJump bastion` |
| Port forward | `LocalForward 5432 localhost:5432` |
| Multiplexing | `ControlMaster auto` |
| Debug | `ssh -vvv user@host` |

Key practices:
1. Use Ed25519 keys with strong passphrases
2. Configure `~/.ssh/config` for all regular hosts
3. Use SSH agent with Keychain integration
4. Enable multiplexing for faster connections
5. Use ProxyJump for bastion access
6. Keep separate keys for different purposes
7. Regularly audit authorized_keys
