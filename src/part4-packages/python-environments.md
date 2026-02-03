# Managing Python Environments

Python on macOS is notoriously confusing. There's the system Python (don't use it), Homebrew Python (convenient), pyenv (version management), and the official python.org installers. Add virtual environments and you have a recipe for confusion. This chapter untangles the mess.

## Python Versions on macOS

### System Python

macOS historically included Python, but:
- **macOS 12.3+**: No Python pre-installed (finally!)
- **Older macOS**: Python 2.7 at `/usr/bin/python` (deprecated)
- **Never modify system Python**: It's for macOS internals

```bash
# Check if system Python exists
$ /usr/bin/python --version
zsh: /usr/bin/python: bad interpreter: No such file or directory

# Modern macOS has no /usr/bin/python
# But may have /usr/bin/python3 from Xcode CLT
$ /usr/bin/python3 --version
Python 3.9.6
```

### Which Python Should You Use?

| Source | Recommendation |
|--------|----------------|
| System Python | **Never** — don't modify |
| Xcode CLT Python | For basic scripting only |
| Homebrew | Good for single-version needs |
| pyenv | **Recommended** — version management |
| python.org | Alternative to pyenv |

## Homebrew Python

### Installation

```bash
# Install latest Python
$ brew install python

# Or specific version
$ brew install python@3.11
$ brew install python@3.12

# Check installation
$ brew info python
```

### Location and Linking

```bash
# Where is it?
$ which python3
/opt/homebrew/bin/python3

$ ls -la /opt/homebrew/bin/python3*
lrwxr-xr-x  1 user  admin  42 Jan 15 10:00 /opt/homebrew/bin/python3 -> ../Cellar/python@3.12/3.12.0/bin/python3
lrwxr-xr-x  1 user  admin  49 Jan 15 10:00 /opt/homebrew/bin/python3.12 -> ../Cellar/python@3.12/3.12.0/bin/python3.12

# Actual location
$ /opt/homebrew/opt/python@3.12/bin/python3 --version
```

### Managing Multiple Homebrew Pythons

```bash
# Install multiple versions
$ brew install python@3.11 python@3.12

# By default, one is linked
$ python3 --version
Python 3.12.0

# Use specific version directly
$ python3.11 --version
$ python3.12 --version

# Or switch linked version
$ brew unlink python@3.12
$ brew link python@3.11
```

### Homebrew pip

```bash
# pip is version-specific
$ pip3 --version
pip 23.3.1 from /opt/homebrew/lib/python3.12/site-packages/pip (python 3.12)

# Install packages
$ pip3 install requests

# Where packages go
$ python3 -c "import site; print(site.getsitepackages())"
['/opt/homebrew/lib/python3.12/site-packages']
```

## pyenv: Version Management

pyenv lets you install and switch between Python versions easily.

### Installing pyenv

```bash
# Install with Homebrew
$ brew install pyenv pyenv-virtualenv

# Or from source
$ curl https://pyenv.run | bash
```

### Shell Configuration

Add to `~/.zprofile` and `~/.zshrc`:

```bash
# ~/.zprofile
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"

# ~/.zshrc
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

Restart your shell or `source ~/.zshrc`.

### Using pyenv

```bash
# List available versions
$ pyenv install --list | grep "^  3\."
  3.10.13
  3.11.7
  3.12.0
  ...

# Install a version
$ pyenv install 3.11.7
Downloading Python-3.11.7.tar.xz...
Installing Python-3.11.7...
Installed Python-3.11.7 to /Users/david/.pyenv/versions/3.11.7

# List installed versions
$ pyenv versions
  system
  3.11.7
  3.12.0
* 3.12.0 (set by /Users/david/.pyenv/version)

# Set global default
$ pyenv global 3.11.7

# Set local version (per directory)
$ cd myproject
$ pyenv local 3.11.7
# Creates .python-version file

# Set for current shell only
$ pyenv shell 3.10.13
```

### pyenv Build Dependencies

pyenv compiles Python from source. Install dependencies first:

```bash
$ brew install openssl readline sqlite3 xz zlib tcl-tk

# Set build flags (in ~/.zshrc)
export LDFLAGS="-L$(brew --prefix openssl@3)/lib -L$(brew --prefix readline)/lib -L$(brew --prefix sqlite3)/lib -L$(brew --prefix zlib)/lib"
export CPPFLAGS="-I$(brew --prefix openssl@3)/include -I$(brew --prefix readline)/include -I$(brew --prefix sqlite3)/include -I$(brew --prefix zlib)/include"
```

## Virtual Environments

Virtual environments isolate project dependencies.

### Built-in venv

```bash
# Create virtual environment
$ python3 -m venv myenv

# Activate
$ source myenv/bin/activate
(myenv) $

# Now pip installs to this env
(myenv) $ pip install requests
(myenv) $ which python
/path/to/myenv/bin/python

# Deactivate
(myenv) $ deactivate
$
```

### pyenv-virtualenv

Combined version and environment management:

```bash
# Create virtualenv with specific Python
$ pyenv virtualenv 3.11.7 myproject-env

# List virtualenvs
$ pyenv virtualenvs

# Activate
$ pyenv activate myproject-env
(myproject-env) $

# Or set local (auto-activates in directory)
$ pyenv local myproject-env

# Deactivate
$ pyenv deactivate

# Delete virtualenv
$ pyenv virtualenv-delete myproject-env
```

### Poetry

Modern dependency management:

```bash
# Install Poetry
$ curl -sSL https://install.python-poetry.org | python3 -
# Or
$ brew install poetry

# New project
$ poetry new myproject
$ cd myproject

# Add dependencies
$ poetry add requests

# Install from existing pyproject.toml
$ poetry install

# Run in environment
$ poetry run python script.py

# Activate shell
$ poetry shell
```

### pipenv

Alternative to Poetry:

```bash
# Install
$ brew install pipenv

# Create environment and install packages
$ pipenv install requests

# Activate
$ pipenv shell

# Run command
$ pipenv run python script.py
```

## Recommended Setup

### For Most Developers

1. **Install pyenv**:
```bash
$ brew install pyenv pyenv-virtualenv
# Add shell configuration
```

2. **Install Python versions**:
```bash
$ pyenv install 3.11.7
$ pyenv install 3.12.0
$ pyenv global 3.11.7
```

3. **Use virtual environments per project**:
```bash
$ cd myproject
$ pyenv virtualenv 3.11.7 myproject
$ pyenv local myproject
```

### For Data Science

Consider Conda/Miniconda for scientific packages:

```bash
# Install Miniconda
$ brew install --cask miniconda

# Initialize
$ conda init zsh

# Create environment
$ conda create -n datascience python=3.11 numpy pandas scikit-learn

# Activate
$ conda activate datascience
```

## Troubleshooting

### "python: command not found"

```bash
# Check what's available
$ which python3
$ pyenv which python

# Create alias if needed (in ~/.zshrc)
alias python=python3
alias pip=pip3
```

### Wrong Python Version

```bash
# Check which Python
$ which python3
$ python3 --version

# Check pyenv
$ pyenv version
$ pyenv which python

# Common issue: Homebrew Python overriding pyenv
# Ensure pyenv is early in PATH
$ echo $PATH | tr ':' '\n' | head
```

### pip Install Permission Errors

```bash
# Never use sudo pip!
# Use virtual environment instead
$ python3 -m venv myenv
$ source myenv/bin/activate
$ pip install package
```

### SSL Certificate Errors

```bash
# Ensure certificates are installed
$ /Applications/Python\ 3.x/Install\ Certificates.command
# Or
$ pip install certifi
$ python -c "import certifi; print(certifi.where())"
```

### pyenv install Fails

```bash
# Install build dependencies
$ brew install openssl readline sqlite3 xz zlib

# Set compiler flags
$ CFLAGS="-I$(brew --prefix openssl)/include -I$(brew --prefix readline)/include" \
  LDFLAGS="-L$(brew --prefix openssl)/lib -L$(brew --prefix readline)/lib" \
  pyenv install 3.11.7
```

## Summary

Python management strategy:

| Scenario | Recommendation |
|----------|----------------|
| Single Python version | Homebrew `python@3.x` |
| Multiple versions | pyenv |
| Project isolation | venv or pyenv-virtualenv |
| Dependency management | Poetry or pip + requirements.txt |
| Data science | Conda/Miniconda |

Key commands:

| Task | Command |
|------|---------|
| Install Python (Homebrew) | `brew install python@3.11` |
| Install Python (pyenv) | `pyenv install 3.11.7` |
| Set global version | `pyenv global 3.11.7` |
| Set project version | `pyenv local 3.11.7` |
| Create virtualenv | `python3 -m venv env` |
| Activate virtualenv | `source env/bin/activate` |
| Install packages | `pip install package` |

Golden rules:
1. **Never modify system Python**
2. **Always use virtual environments**
3. **Pick one version manager (pyenv recommended)**
4. **Document Python version in projects** (.python-version or pyproject.toml)
