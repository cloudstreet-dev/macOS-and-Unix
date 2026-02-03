# Managing Ruby Environments

macOS includes a system Ruby, but like system Python, you shouldn't rely on it for development. Ruby version managers solve the problem of needing different Ruby versions for different projects. This chapter covers rbenv (recommended), rvm, and chruby—plus managing gems effectively.

## System Ruby

macOS includes Ruby, but it's outdated and restricted:

```bash
$ /usr/bin/ruby --version
ruby 2.6.10p210 (2022-04-12 revision 67958) [universal.arm64e-darwin23]

# System Ruby is:
# - Old (2.6.x)
# - Requires sudo for gem install
# - Modified by macOS updates
# - Not suitable for development
```

**Rule**: Never use system Ruby for development.

## rbenv: Recommended Approach

rbenv is lightweight and follows Unix philosophy—it does one thing well.

### Installation

```bash
# Install rbenv and ruby-build
$ brew install rbenv ruby-build

# Verify installation
$ rbenv --version
rbenv 1.2.0
```

### Shell Configuration

Add to `~/.zshrc`:

```bash
# Initialize rbenv
eval "$(rbenv init - zsh)"
```

Restart shell or `source ~/.zshrc`.

### Installing Ruby Versions

```bash
# List available versions
$ rbenv install -l
3.2.2
3.3.0
jruby-9.4.5.0
truffleruby-23.1.2
...

# Install a version
$ rbenv install 3.2.2
Downloading ruby-3.2.2.tar.gz...
Installing ruby-3.2.2...
Installed ruby-3.2.2 to /Users/david/.rbenv/versions/3.2.2

# List installed versions
$ rbenv versions
  system
* 3.2.2 (set by /Users/david/.rbenv/version)
```

### Switching Versions

```bash
# Set global default
$ rbenv global 3.2.2

# Set local version (per directory)
$ cd myproject
$ rbenv local 3.1.4
# Creates .ruby-version file

# Set for current shell
$ rbenv shell 3.0.6

# Check current version
$ rbenv version
3.2.2 (set by /Users/david/.rbenv/version)

$ ruby --version
ruby 3.2.2 (2023-03-30 revision e51014f9c0) [arm64-darwin23]
```

### How rbenv Works

rbenv uses shims—wrapper scripts that intercept Ruby commands:

```bash
# Shims directory
$ ls ~/.rbenv/shims
bundle  erb  gem  irb  rake  rdoc  ri  ruby

# Shim intercepts and redirects
$ which ruby
/Users/david/.rbenv/shims/ruby

# Actual Ruby
$ rbenv which ruby
/Users/david/.rbenv/versions/3.2.2/bin/ruby
```

### Rehashing

After installing gems with executables, rehash:

```bash
$ rbenv rehash
# Creates shims for new executables

# Modern rbenv auto-rehashes, but manual rehash if needed
```

## chruby: Minimal Alternative

chruby is even simpler than rbenv—no shims, just environment modification.

### Installation

```bash
$ brew install chruby ruby-install
```

### Configuration

Add to `~/.zshrc`:

```bash
source /opt/homebrew/opt/chruby/share/chruby/chruby.sh
source /opt/homebrew/opt/chruby/share/chruby/auto.sh  # Auto-switching
```

### Usage

```bash
# Install Ruby
$ ruby-install ruby 3.2.2

# List Rubies
$ chruby
   ruby-3.2.2

# Switch
$ chruby ruby-3.2.2

# Auto-switching reads .ruby-version
$ echo "ruby-3.2.2" > .ruby-version
$ cd .  # Triggers auto-switch
```

## rvm: Full-Featured Alternative

RVM (Ruby Version Manager) is feature-rich but more complex.

### Installation

```bash
# Install rvm
$ \curl -sSL https://get.rvm.io | bash -s stable

# Restart shell or source
$ source ~/.rvm/scripts/rvm
```

### Usage

```bash
# Install Ruby
$ rvm install 3.2.2

# List versions
$ rvm list

# Switch versions
$ rvm use 3.2.2

# Set default
$ rvm --default use 3.2.2

# Create gemset (isolated gem environment)
$ rvm gemset create myproject
$ rvm use 3.2.2@myproject

# Project-specific .rvmrc
$ echo "rvm use 3.2.2@myproject" > .rvmrc
```

### rvm vs rbenv

| Feature | rbenv | rvm |
|---------|-------|-----|
| Philosophy | Minimal | Full-featured |
| Gemsets | No (use bundler) | Yes |
| Shell modification | Shims | PATH modification |
| Complexity | Low | Higher |
| Recommendation | **Preferred** | If you need gemsets |

## Gem Management

### Understanding Gems

```bash
# Gem environment
$ gem env
RubyGems Environment:
  - RUBYGEMS VERSION: 3.4.10
  - RUBY VERSION: 3.2.2
  - INSTALLATION DIRECTORY: /Users/david/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0
  - USER INSTALLATION DIRECTORY: /Users/david/.gem/ruby/3.2.0
  - GEM PATHS:
     - /Users/david/.rbenv/versions/3.2.2/lib/ruby/gems/3.2.0
     - /Users/david/.gem/ruby/3.2.0

# Install gem
$ gem install rails

# List installed gems
$ gem list

# Uninstall
$ gem uninstall rails
```

### Bundler: Project Dependencies

Bundler manages gems per-project:

```bash
# Install bundler
$ gem install bundler

# Create Gemfile
$ bundle init

# Edit Gemfile
$ cat Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 7.0'
gem 'puma'

# Install dependencies
$ bundle install

# Run command with bundled gems
$ bundle exec rails server

# Update gems
$ bundle update

# Lock to specific versions
$ bundle lock
```

### Gemfile Best Practices

```ruby
# Gemfile
source 'https://rubygems.org'

# Specify Ruby version
ruby '3.2.2'

# Specify gem versions
gem 'rails', '~> 7.0.8'
gem 'pg', '~> 1.5'

# Development-only gems
group :development do
  gem 'rubocop'
  gem 'pry'
end

group :test do
  gem 'rspec'
end
```

### Gem Configuration

```bash
# ~/.gemrc
gem: --no-document

# Skip documentation for faster installs
$ gem install rails --no-document
```

## Rails Development Setup

Complete setup for Rails development:

```bash
# 1. Install rbenv
$ brew install rbenv ruby-build

# 2. Configure shell
$ echo 'eval "$(rbenv init - zsh)"' >> ~/.zshrc
$ source ~/.zshrc

# 3. Install Ruby
$ rbenv install 3.2.2
$ rbenv global 3.2.2

# 4. Install Bundler
$ gem install bundler

# 5. Install Rails
$ gem install rails

# 6. Verify
$ rails --version
Rails 7.1.2

# 7. Create project
$ rails new myapp
$ cd myapp
$ bundle install
$ bin/rails server
```

## Troubleshooting

### "gem install" Permission Denied

```bash
# Don't use sudo!
# Ensure you're using rbenv Ruby, not system
$ which ruby
/Users/david/.rbenv/shims/ruby  # Good

$ which ruby
/usr/bin/ruby  # Bad - system Ruby

# Fix: Check rbenv setup
$ rbenv versions
$ rbenv global 3.2.2
```

### Command Not Found After gem install

```bash
# Rehash to create shims
$ rbenv rehash

# Or check if gem's bin is in PATH
$ gem env | grep "EXECUTABLE DIRECTORY"
```

### Build Failures

```bash
# Install build dependencies
$ brew install openssl readline libyaml

# Set build flags
$ RUBY_CONFIGURE_OPTS="--with-openssl-dir=$(brew --prefix openssl@3)" rbenv install 3.2.2
```

### SSL Certificate Errors

```bash
# Update certificates
$ brew install openssl
$ rbenv install 3.2.2
# openssl from Homebrew is used automatically

# Or update system certificates
$ security find-certificate -a -p /Library/Keychains/System.keychain > certs.pem
```

### Wrong Ruby Version in Project

```bash
# Check .ruby-version
$ cat .ruby-version
3.1.4

# Install if missing
$ rbenv install 3.1.4

# cd out and back in
$ cd .. && cd myproject
$ ruby --version
```

## Summary

Ruby version management:

| Tool | Best For |
|------|----------|
| rbenv | Most users (recommended) |
| chruby | Minimalists |
| rvm | Users needing gemsets |
| Homebrew ruby | Single version, casual use |

Key commands (rbenv):

| Task | Command |
|------|---------|
| Install rbenv | `brew install rbenv ruby-build` |
| Install Ruby | `rbenv install 3.2.2` |
| Set global | `rbenv global 3.2.2` |
| Set local | `rbenv local 3.2.2` |
| List versions | `rbenv versions` |
| Install gem | `gem install name` |
| Project gems | `bundle install` |

Best practices:
1. **Never use system Ruby**
2. **Use rbenv for version management**
3. **Use Bundler for project dependencies**
4. **Commit Gemfile.lock to version control**
5. **Specify Ruby version in .ruby-version**
