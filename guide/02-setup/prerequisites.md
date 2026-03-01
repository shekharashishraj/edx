# Prerequisites

Everything you need installed before setting up ASU Global Connect.

---

## System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| RAM | 8 GB | 16 GB |
| Disk | 20 GB free | 40 GB free |
| CPU | 4 cores | 8 cores |
| OS | macOS 12+ / Ubuntu 22.04+ | macOS 14+ |

---

## Required Software

### 1. Docker Desktop (macOS)

Open edX runs entirely in Docker containers. You need Docker Desktop with:
- Docker Engine v24+
- Docker Compose v2 (bundled with Docker Desktop)

**Install:** https://www.docker.com/products/docker-desktop/

**Verify:**
```bash
docker --version        # Docker version 26.1.0 or higher
docker compose version  # Docker Compose version v2.x
```

**Important macOS settings** (Docker Desktop → Settings → Resources):
- Memory: 8 GB minimum (Open edX is heavy)
- CPUs: 4 minimum
- Disk image size: 40 GB minimum

### 2. Python 3.10+ (for Tutor CLI)

Tutor itself is a Python CLI tool. Python 3.10 is what's installed on this machine.

**Verify:**
```bash
python3 --version  # Python 3.10.x or higher
```

**Install (macOS):** https://www.python.org/downloads/ or via Homebrew:
```bash
brew install python@3.10
```

### 3. Tutor v21

The Tutor CLI manages the entire Open edX deployment.

**Install:**
```bash
pip3 install "tutor==21.*"
```

**Location on this machine:** `/Users/ashishrajshekhar/Library/Python/3.10/bin/tutor`

**Add to PATH** (add to `~/.zshrc`):
```bash
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
```

**Verify:**
```bash
tutor --version  # Tutor version 21.0.1
```

---

## Shell Environment

Two environment variables must be set in every terminal session:

```bash
# Add to ~/.zshrc to make permanent
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"
```

`TUTOR_ROOT` tells Tutor where your `config.yml` and `data/` live.

Without these, you'll get `tutor: command not found` or Tutor will create a new project in the wrong directory.

---

## Optional but Useful

### Git
For version control and XBlock development.
```bash
git --version  # Should already be installed on macOS
```

### cookiecutter
For scaffolding new XBlocks.
```bash
pip3 install cookiecutter
```

### VS Code + Python Extension
Recommended IDE. Install the Python extension for Django debugging support.

---

## Network Requirements

- `local.openedx.io` resolves to `127.0.0.1` via Overhangio's public DNS — no `/etc/hosts` changes needed
- Port 8000 and 8001 must be free on your machine
- Docker must be running before any `tutor dev` commands

---

## Verify Everything Is Ready

```bash
# Check Docker is running
docker ps

# Check Python
python3 --version

# Check Tutor (with PATH set)
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
tutor --version

# Check TUTOR_ROOT is pointing to the right place
echo $TUTOR_ROOT
ls $TUTOR_ROOT  # should show config.yml, env/, data/, xblocks/
```
