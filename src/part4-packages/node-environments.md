# Managing Node.js Environments

Node.js development often requires different versions for different projects—an older LTS for production, the latest for new features, or specific versions matching your CI environment. Version managers like nvm, fnm, and volta solve this problem, while package managers like npm, yarn, and pnpm handle dependencies.

## Node.js Installation Options

| Method | Pros | Cons |
|--------|------|------|
| nvm | Most popular, well-supported | Shell startup overhead |
| fnm | Fast, Rust-based | Newer, less documentation |
| volta | Also manages npm/yarn | Newer tool |
| Homebrew | Simple | Single version only |
| Official installer | Direct from source | Manual version management |

## nvm: Node Version Manager

nvm is the most widely used Node.js version manager.

### Installation

```bash
# Install nvm
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Or with Homebrew (note: Homebrew version has caveats)
$ brew install nvm
```

### Shell Configuration

Add to `~/.zshrc`:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # Load nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # Completion
```

Restart shell or `source ~/.zshrc`.

### Using nvm

```bash
# List available versions
$ nvm ls-remote
        v18.18.0   (LTS: Hydrogen)
        v18.19.0   (Latest LTS: Hydrogen)
        v20.10.0   (LTS: Iron)
        v21.5.0
...

# Install a version
$ nvm install 20  # Installs latest v20.x
$ nvm install 18.19.0  # Specific version
$ nvm install --lts  # Latest LTS

# List installed versions
$ nvm ls
->     v20.10.0
       v18.19.0
default -> 20 (-> v20.10.0)
...

# Switch versions
$ nvm use 18
Now using node v18.19.0 (npm v10.2.3)

# Set default
$ nvm alias default 20

# Use in current directory (.nvmrc)
$ echo "20" > .nvmrc
$ nvm use
Found '/path/to/project/.nvmrc' with version <20>
Now using node v20.10.0 (npm v10.2.3)
```

### Auto-switching

Automatically switch when entering directories:

```bash
# Add to ~/.zshrc
autoload -U add-zsh-hook
load-nvmrc() {
  local nvmrc_path="$(nvm_find_nvmrc)"
  if [ -n "$nvmrc_path" ]; then
    local nvmrc_node_version=$(nvm version "$(cat "${nvmrc_path}")")
    if [ "$nvmrc_node_version" = "N/A" ]; then
      nvm install
    elif [ "$nvmrc_node_version" != "$(nvm version)" ]; then
      nvm use
    fi
  elif [ -n "$(PWD=$OLDPWD nvm_find_nvmrc)" ] && [ "$(nvm version)" != "$(nvm version default)" ]; then
    echo "Reverting to nvm default version"
    nvm use default
  fi
}
add-zsh-hook chpwd load-nvmrc
load-nvmrc
```

### nvm Performance

nvm can slow shell startup. Mitigate with lazy loading:

```bash
# Lazy load nvm (in ~/.zshrc)
export NVM_DIR="$HOME/.nvm"

nvm() {
  unset -f nvm node npm npx
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  nvm "$@"
}

node() {
  unset -f nvm node npm npx
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  node "$@"
}

npm() {
  unset -f nvm node npm npx
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  npm "$@"
}

npx() {
  unset -f nvm node npm npx
  [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
  npx "$@"
}
```

## fnm: Fast Node Manager

fnm is a Rust-based alternative that's significantly faster.

### Installation

```bash
# Via Homebrew
$ brew install fnm

# Or curl
$ curl -fsSL https://fnm.vercel.app/install | bash
```

### Configuration

```bash
# Add to ~/.zshrc
eval "$(fnm env --use-on-cd)"
```

### Usage

```bash
# Install Node
$ fnm install 20
$ fnm install --lts

# List installed
$ fnm list
* v20.10.0 default
  v18.19.0

# Use version
$ fnm use 18

# Set default
$ fnm default 20

# Auto-switching via .nvmrc or .node-version
$ fnm use  # Reads from .nvmrc
```

### fnm vs nvm

| Feature | fnm | nvm |
|---------|-----|-----|
| Speed | Fast (Rust) | Slower (shell) |
| .nvmrc support | Yes | Yes |
| Completions | Yes | Yes |
| Windows support | Yes | No (use nvm-windows) |
| Maturity | Newer | Established |

## volta: Toolchain Manager

Volta manages Node.js plus npm/yarn versions per project.

### Installation

```bash
$ curl https://get.volta.sh | bash
```

### Usage

```bash
# Install Node
$ volta install node@20
$ volta install node@lts

# Install global tools
$ volta install yarn
$ volta install typescript

# Pin version in project
$ volta pin node@18
# Updates package.json with volta field

# Now anyone with volta uses same versions
```

### How volta Differs

Volta pins tool versions in `package.json`:

```json
{
  "name": "myproject",
  "volta": {
    "node": "18.19.0",
    "npm": "10.2.3"
  }
}
```

This ensures consistent tooling across team members.

## Package Managers

### npm (default)

npm comes with Node.js:

```bash
# Check version
$ npm --version

# Install dependencies
$ npm install

# Add package
$ npm install lodash

# Add dev dependency
$ npm install --save-dev jest

# Global install
$ npm install -g typescript

# Update packages
$ npm update

# Audit security
$ npm audit
$ npm audit fix
```

### yarn

Yarn offers performance improvements and workspaces:

```bash
# Install yarn
$ npm install -g yarn
# Or with corepack (Node 16.10+)
$ corepack enable
$ corepack prepare yarn@stable --activate

# Install dependencies
$ yarn install

# Add package
$ yarn add lodash
$ yarn add --dev jest

# Update
$ yarn upgrade

# Workspaces (monorepo)
$ yarn workspaces list
```

### pnpm

pnpm saves disk space via hard links:

```bash
# Install pnpm
$ npm install -g pnpm
# Or with corepack
$ corepack enable
$ corepack prepare pnpm@latest --activate

# Install dependencies
$ pnpm install

# Add package
$ pnpm add lodash
$ pnpm add -D jest

# Disk savings
$ pnpm store path
$ pnpm store prune
```

### Choosing a Package Manager

| Manager | Best For |
|---------|----------|
| npm | Default, wide compatibility |
| yarn | Workspaces, performance |
| pnpm | Disk efficiency, strict |

## Global Packages

### Where They Go

```bash
# npm global location
$ npm root -g
/Users/david/.nvm/versions/node/v20.10.0/lib/node_modules

# yarn global location
$ yarn global dir
/Users/david/.config/yarn/global
```

### Managing Globals

```bash
# List globals
$ npm list -g --depth=0
$ yarn global list

# Install global
$ npm install -g typescript
$ yarn global add typescript

# Update globals
$ npm update -g
$ yarn global upgrade
```

### Globals with Version Managers

Global packages are per Node version:

```bash
# Install in Node 20
$ nvm use 20
$ npm install -g typescript

# Switch to Node 18 - typescript not available
$ nvm use 18
$ which tsc
tsc not found

# Need to install again
$ npm install -g typescript
```

For shared globals, use `npx` instead:

```bash
# Run without installing
$ npx typescript --version
$ npx create-react-app myapp
```

## Project Configuration

### .nvmrc / .node-version

```bash
# Create version file
$ echo "20" > .nvmrc
# Or specific
$ echo "20.10.0" > .nvmrc

# Works with nvm, fnm, and others
$ nvm use
```

### package.json engines

```json
{
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  }
}
```

Enforced with `npm install --engine-strict`.

### Corepack

Node 16.10+ includes Corepack for package manager management:

```bash
# Enable corepack
$ corepack enable

# Specify in package.json
{
  "packageManager": "yarn@4.0.2"
}

# Now yarn is auto-installed at correct version
$ yarn --version
4.0.2
```

## Troubleshooting

### "node: command not found" After nvm install

```bash
# Check nvm is loaded
$ command -v nvm

# Reload shell config
$ source ~/.zshrc

# Set default
$ nvm alias default 20
$ nvm use default
```

### Permission Errors

```bash
# Never use sudo with nvm-installed node
# If you have permission issues, nvm isn't set up right
$ which node
/Users/david/.nvm/versions/node/v20.10.0/bin/node  # Good
/usr/local/bin/node  # Bad - not nvm
```

### Wrong Node in Scripts

```bash
# Scripts may use different PATH
# Add to script or use full path
#!/usr/bin/env node

# Or ensure PATH includes nvm
export PATH="$NVM_DIR/versions/node/$(nvm version)/bin:$PATH"
```

### node-gyp Build Errors

```bash
# Install build tools
$ xcode-select --install

# For Python dependency
$ brew install python
```

## Summary

Node.js management strategy:

| Tool | Recommendation |
|------|----------------|
| Version manager | fnm (fast) or nvm (established) |
| Package manager | npm (default), yarn/pnpm for specific needs |
| Project config | .nvmrc + engines in package.json |

Key commands (nvm):

| Task | Command |
|------|---------|
| Install nvm | `curl -o- https://raw.githubusercontent.com/.../nvm/.../install.sh \| bash` |
| Install Node | `nvm install 20` |
| Use version | `nvm use 20` |
| Set default | `nvm alias default 20` |
| Project version | `echo "20" > .nvmrc` |
| List versions | `nvm ls` |

Best practices:
1. **Use a version manager** (nvm or fnm)
2. **Include .nvmrc in projects**
3. **Set engines in package.json**
4. **Don't sudo npm install -g**
5. **Use npx for one-off commands**
