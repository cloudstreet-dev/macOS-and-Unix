# Homebrew Best Practices

Homebrew is straightforward to use but benefits from disciplined practices. This chapter covers daily usage patterns, maintenance routines, managing upgrades, and avoiding common pitfalls.

## Daily Usage

### Installing Packages

```bash
# Search for packages
$ brew search postgresql
==> Formulae
postgresql@11    postgresql@14    postgresql@15    postgresql@16

# Get information before installing
$ brew info postgresql@16
==> postgresql@16: stable 16.1 (bottled)
Object-relational database system
...

# Install
$ brew install postgresql@16

# Install specific version
$ brew install postgresql@14

# Install from source (if needed)
$ brew install --build-from-source package
```

### Installing GUI Applications

```bash
# Search casks
$ brew search --cask firefox

# Install cask
$ brew install --cask firefox

# Casks go to /Applications by default
# Verify installation
$ ls /Applications/Firefox.app
```

### Updating and Upgrading

```bash
# Update Homebrew itself and formula definitions
$ brew update

# See what's outdated
$ brew outdated

# Upgrade all packages
$ brew upgrade

# Upgrade specific package
$ brew upgrade wget

# Upgrade casks
$ brew upgrade --cask

# Pin a package (prevent upgrades)
$ brew pin postgresql@16
$ brew list --pinned
$ brew unpin postgresql@16
```

### Uninstalling

```bash
# Uninstall a package
$ brew uninstall wget

# Uninstall with dependencies check
$ brew uninstall --zap firefox  # For casks: removes preferences too

# Remove unused dependencies
$ brew autoremove
```

## Maintenance Routine

### Regular Maintenance Commands

Run these periodically (weekly or monthly):

```bash
# Update everything
$ brew update && brew upgrade

# Clean up old versions
$ brew cleanup

# Check for problems
$ brew doctor

# Remove unused dependencies
$ brew autoremove
```

### Cleanup Details

```bash
# Preview what cleanup would remove
$ brew cleanup --dry-run

# Clean up older than default (120 days)
$ brew cleanup --prune=30  # 30 days

# Clean specific package
$ brew cleanup wget

# See disk usage before/after
$ du -sh $(brew --cache)
1.2G    /Users/david/Library/Caches/Homebrew
$ brew cleanup
$ du -sh $(brew --cache)
256M    /Users/david/Library/Caches/Homebrew
```

### Dealing with Doctor Warnings

```bash
$ brew doctor
Please note that these warnings are just used to help the Homebrew maintainers
with debugging if you file an issue. If everything you use Homebrew for is
working fine: please don't worry or file an issue; just ignore this. Thanks!

Warning: Some installed formulae are deprecated or disabled.
...
```

Common warnings and fixes:

**Unbrewed files in Homebrew directories**:
```bash
Warning: Unbrewed dylibs were found in /opt/homebrew/lib
# Usually safe to ignore, or remove manually if you know they're orphaned
```

**Outdated Xcode Command Line Tools**:
```bash
Warning: A newer Command Line Tools release is available.
$ softwareupdate --list
$ softwareupdate -i "Command Line Tools for Xcode-15.0"
# Or just update Xcode from App Store
```

**Broken symlinks**:
```bash
Warning: Broken symlinks were found
$ brew doctor --list-checks | xargs brew
# Or manually remove listed broken symlinks
```

## Managing Multiple Versions

### Version-Specific Packages

```bash
# Install specific versions
$ brew install python@3.11
$ brew install python@3.12

# Both are installed but one is linked
$ brew list | grep python
python@3.11
python@3.12

# Switch versions
$ brew unlink python@3.11
$ brew link python@3.12

# Check which is active
$ python3 --version
```

### Using Unlinked Versions

Access unlinked versions directly:

```bash
# Direct path
$ /opt/homebrew/opt/python@3.11/bin/python3 --version
Python 3.11.7

# Or add to PATH for session
$ export PATH="/opt/homebrew/opt/python@3.11/bin:$PATH"
```

## Dealing with Dependencies

### Understanding Dependency Problems

```bash
# See why a package is installed
$ brew uses --installed openssl@3
curl
git
python@3.11
wget

# See what a package needs
$ brew deps wget
gettext
libidn2
openssl@3

# Full dependency tree
$ brew deps --tree --installed
```

### Dependency Conflicts

When upgrades break dependencies:

```bash
# Reinstall package and dependencies
$ brew reinstall wget

# Rebuild all dependents of a package
$ brew reinstall $(brew uses --installed openssl@3)
```

## Services Management

### Using brew services

```bash
# List all services
$ brew services list

# Start service (runs at login)
$ brew services start postgresql@16

# Run once (doesn't persist across reboot)
$ brew services run postgresql@16

# Stop service
$ brew services stop postgresql@16

# Restart service
$ brew services restart postgresql@16

# View service status
$ brew services info postgresql@16
```

### Service Files

```bash
# Location of service plists
$ ls ~/Library/LaunchAgents/homebrew.*
homebrew.mxcl.postgresql@16.plist

# View service configuration
$ cat ~/Library/LaunchAgents/homebrew.mxcl.postgresql@16.plist

# Manual service management
$ launchctl list | grep homebrew
```

## Bundler: Reproducible Environments

### Creating a Brewfile

```bash
# Generate Brewfile from current installations
$ brew bundle dump

# Creates Brewfile:
$ cat Brewfile
tap "homebrew/bundle"
tap "homebrew/cask"
tap "homebrew/core"
brew "git"
brew "node"
brew "python@3.11"
cask "visual-studio-code"
cask "docker"
```

### Installing from Brewfile

```bash
# Install everything in Brewfile
$ brew bundle

# Check what would be installed
$ brew bundle check

# List what's in Brewfile but not installed
$ brew bundle list
```

### Brewfile Best Practices

```ruby
# Brewfile with options and documentation

# Taps
tap "homebrew/bundle"
tap "homebrew/cask-fonts"

# Development tools
brew "git"
brew "gh"  # GitHub CLI
brew "neovim"

# Languages
brew "python@3.11"
brew "node"
brew "go"

# Databases
brew "postgresql@16", restart_service: true
brew "redis"

# Applications
cask "visual-studio-code"
cask "docker"
cask "iterm2"

# Fonts
cask "font-fira-code"

# Mac App Store apps (requires mas)
# mas "Xcode", id: 497799835
```

## Troubleshooting

### Common Issues

**"Error: No available formula with the name..."**
```bash
$ brew install nonexistent
Error: No available formula with the name "nonexistent"

# Solution: Update and search
$ brew update
$ brew search partial-name
```

**"Error: Cannot install in Homebrew on ARM..."**
```bash
# This happens when mixing architectures
# Ensure you're using the right Homebrew
$ which brew
/opt/homebrew/bin/brew  # Should be this on Apple Silicon
```

**Permission errors**:
```bash
$ brew install something
Error: Permission denied @ rb_sysopen

# Fix permissions
$ sudo chown -R $(whoami) $(brew --prefix)/*
```

**Bottle not available**:
```bash
# Install from source when bottle unavailable
$ brew install --build-from-source package
```

### Resetting Homebrew

When things are really broken:

```bash
# Nuclear option: Uninstall Homebrew
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"

# Then reinstall
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Less nuclear:

```bash
# Reset to clean state
$ brew update-reset

# Fix all symlinks
$ brew link --overwrite --dry-run $(brew list --formula)
$ brew link --overwrite $(brew list --formula)
```

## Security Considerations

### Verifying Packages

```bash
# Homebrew verifies checksums automatically
# View checksum
$ brew info --json wget | jq '.[0].versions.bottle_sha256'

# Audit a formula
$ brew audit wget
```

### Avoiding Untrusted Taps

```bash
# List your taps
$ brew tap

# Only use trusted taps
# Official: homebrew/core, homebrew/cask
# Verify third-party taps before adding
```

### Cask Security

```bash
# Some casks require passwords - be cautious
# Check what a cask does before installing
$ brew cat --cask suspicious-app

# Review artifacts and scripts
```

## Performance Tips

### Faster Updates

```bash
# Use parallel downloads
$ export HOMEBREW_PARALLEL_FETCH=4

# Skip analytics
$ brew analytics off
```

### Reducing Disk Usage

```bash
# Regular cleanup
$ brew cleanup --prune=all

# Check what's using space
$ brew list --formula | xargs -I{} sh -c 'echo -n "{}: " && du -sh $(brew --cellar)/{} | cut -f1'

# Remove old versions aggressively
$ brew cleanup --prune=0
```

### Cache Management

```bash
# View cache
$ ls ~/Library/Caches/Homebrew/

# Clear cache
$ rm -rf ~/Library/Caches/Homebrew/*

# Or selectively
$ brew cleanup --prune=all
```

## CI/CD Usage

### GitHub Actions Example

```yaml
- name: Install Homebrew dependencies
  run: |
    brew update
    brew install cmake ninja

# Or with Brewfile
- name: Install from Brewfile
  run: brew bundle
```

### Caching in CI

```yaml
- name: Cache Homebrew
  uses: actions/cache@v3
  with:
    path: |
      ~/Library/Caches/Homebrew
      /opt/homebrew/Cellar
    key: ${{ runner.os }}-brew-${{ hashFiles('Brewfile.lock.json') }}
```

## Summary

Best practices checklist:

| Practice | Command |
|----------|---------|
| Update regularly | `brew update && brew upgrade` |
| Clean up | `brew cleanup` |
| Check health | `brew doctor` |
| Remove orphans | `brew autoremove` |
| Pin critical packages | `brew pin package` |
| Use Brewfile | `brew bundle dump` |
| Audit before adding taps | Research tap source |

Golden rules:
1. **Update before installing** — `brew update` first
2. **Use bottles when possible** — Much faster than source
3. **Run cleanup regularly** — Saves disk space
4. **Use Brewfile for teams** — Reproducible environments
5. **Don't sudo brew** — Never run Homebrew with sudo
6. **Check doctor warnings** — Not all need fixing, but awareness helps
