# Tutor Commands Reference

Complete reference for every Tutor CLI command used in this project.

> Always run with env vars set:
> ```bash
> export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
> export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"
> ```

---

## Service Lifecycle

```bash
# Start all services (subsequent sessions)
tutor dev start

# Start specific services only
tutor dev start lms cms mysql redis

# Stop all services (clean shutdown)
tutor dev stop

# Restart all services
tutor dev reboot

# Restart specific service(s)
tutor dev restart lms
tutor dev restart lms cms
tutor dev restart mysql

# Start in detached mode (no output, runs in background)
tutor dev start -d

# First-time setup (builds images, runs migrations)
tutor dev launch --non-interactive
```

---

## Status and Logs

```bash
# Show container status (like docker ps)
tutor dev dc ps
tutor dev status

# Follow all logs
tutor dev logs --follow

# Follow specific service logs
tutor dev logs --follow lms
tutor dev logs --follow cms
tutor dev logs --follow mysql
tutor dev logs --follow mongodb
tutor dev logs --follow redis
tutor dev logs --follow meilisearch

# Last N lines of logs
tutor dev logs --tail=50 lms
tutor dev logs --tail=100 lms

# List service hostnames/URLs
tutor dev hosts
```

---

## Running Commands in Containers

```bash
# Run a one-off command in a NEW container (doesn't affect running instance)
tutor dev run lms <command>
tutor dev run cms <command>

# Execute command in RUNNING container
tutor dev exec lms <command>
tutor dev exec cms <command>

# Examples
tutor dev run lms bash                          # Open shell in LMS
tutor dev run lms ./manage.py lms shell         # Django shell
tutor dev run lms ./manage.py lms migrate       # Run migrations
tutor dev run lms ./manage.py lms help          # List management commands
tutor dev run lms pip install -e /openedx/xblocks/my-xblock
tutor dev run lms pip list | grep xblock

tutor dev run cms bash
tutor dev run cms ./manage.py cms shell
tutor dev run cms pip install -e /openedx/xblocks/my-xblock
```

---

## Jobs (tutor dev do)

Built-in initialization jobs:

```bash
# List all available jobs
tutor dev do --help

# Run full initialization (migrations, fixtures, search index)
tutor dev do init

# Create a user
tutor dev do createuser --staff --superuser <username> <email>
# Example:
tutor dev do createuser --staff --superuser admin admin@openedx.local

# Open SQL shell as root
tutor dev do sqlshell

# Import the demo course
tutor dev do importdemocourse

# Import demo content libraries
tutor dev do importdemolibraries

# Set a theme
tutor dev do settheme mytheme

# Convert MySQL charset to utf8mb4
tutor dev do convert-mysql-utf8mb4-charset
```

---

## Docker Compose Pass-through

Any Docker Compose command works via `tutor dev dc`:

```bash
# Show all containers with ports
tutor dev dc ps

# View container resource usage
tutor dev dc top

# Direct docker-compose exec
tutor dev dc exec lms bash

# View logs with docker compose syntax
tutor dev dc logs -f --tail=100 lms

# Scale a service (not recommended in dev)
tutor dev dc up --scale lms=2
```

---

## Configuration

```bash
# Print all config values
tutor config printvalue PLATFORM_NAME
tutor config printvalue LMS_HOST
tutor config printvalue MYSQL_ROOT_PASSWORD

# Save config (after manual edits to config.yml)
tutor config save

# Set a config value and save
tutor config save --set PLATFORM_NAME="New Name"

# Regenerate env/ from config.yml (after plugin changes, etc.)
tutor config save
```

---

## Plugin Management

```bash
# List available plugins
tutor plugins list

# Install a plugin
tutor plugins install mfe indigo

# Enable a plugin
tutor plugins enable mfe indigo

# Disable a plugin
tutor plugins disable <plugin-name>

# Show enabled plugins
tutor plugins list  # installed ones show as enabled/disabled
```

---

## File Operations

```bash
# Copy files FROM a container to local
tutor dev copyfrom lms /openedx/edx-platform/lms/static ./local-copy

# Copy files TO a running container
docker cp ./my-xblock/ $(tutor dev dc ps -q lms):/openedx/xblocks/
```

---

## Upgrade

```bash
# Upgrade to latest Tutor (run after pip install --upgrade tutor)
tutor dev upgrade
```

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Start everything | `tutor dev start` |
| Stop everything | `tutor dev stop` |
| Restart LMS | `tutor dev restart lms` |
| View LMS logs | `tutor dev logs --follow lms` |
| Open LMS shell | `tutor dev run lms bash` |
| Django shell | `tutor dev run lms ./manage.py lms shell` |
| Run migrations | `tutor dev run lms ./manage.py lms migrate` |
| Install XBlock | `tutor dev run lms pip install -e /openedx/xblocks/my-xblock` |
| Create user | `tutor dev do createuser --staff --superuser admin admin@openedx.local` |
| SQL shell | `tutor dev do sqlshell` |
| Container status | `tutor dev dc ps` |
| Re-init database | `tutor dev do init` |
