# Installation Guide

Step-by-step setup from zero to a running Open edX dev environment.

> **Note:** Week 1 setup is already complete on this machine. This document is for reproducibility and onboarding new team members.

---

## Phase 0: First-Time Setup (Already Done)

### Step 1 — Install Tutor

```bash
pip3 install "tutor==21.*"
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
```

### Step 2 — Set TUTOR_ROOT

```bash
export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"
mkdir -p $TUTOR_ROOT
cd $TUTOR_ROOT
```

### Step 3 — Configure Platform

```bash
tutor config save \
  --set PLATFORM_NAME="ASU Global Connect" \
  --set LMS_HOST="local.openedx.io" \
  --set CMS_HOST="studio.local.openedx.io" \
  --set CONTACT_EMAIL="ashish@asu.edu" \
  --set ENABLE_HTTPS=false
```

This writes `config.yml`.

### Step 4 — Install Required Plugins

```bash
tutor plugins install mfe indigo
tutor plugins enable mfe indigo
tutor config save
```

### Step 5 — Launch (First Time Only)

This builds Docker images from scratch (~20–40 min depending on network):

```bash
tutor dev launch --non-interactive
```

What this does:
1. Generates `env/` from `config.yml` (templates → actual configs)
2. Pulls base Docker images (Ubuntu, MySQL, MongoDB, Redis)
3. Builds the `openedx-dev:21.0.1` image (checkouts edx-platform, installs ~400 Python packages, compiles JS/CSS assets)
4. Starts all containers
5. Runs `tutor dev do init` (Django migrations, creates default data)

### Step 6 — Create Admin User

```bash
# Interactive (prompts for password):
tutor dev do createuser --staff --superuser admin admin@openedx.local

# OR non-interactive via Django shell:
tutor dev run lms ./manage.py lms shell -c "
from django.contrib.auth import get_user_model
from common.djangoapps.student.models import UserProfile
User = get_user_model()
u = User.objects.create_superuser('admin', 'admin@openedx.local', 'admin1234')
UserProfile.objects.create(user=u, name='Admin')
"
```

### Step 7 — Create XBlock Directories

```bash
mkdir -p $TUTOR_ROOT/xblocks/{gamification,globalmap,questengine,phetsim,virtualcampus,multiplayer}-xblock
```

---

## Starting a Dev Session (Every Day)

After the one-time setup, starting is fast:

```bash
# 1. Set environment
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"

# 2. Make sure Docker Desktop is running, then:
tutor dev start

# 3. Verify everything is up
tutor dev dc ps

# 4. Check LMS responds
curl -I http://local.openedx.io:8000/heartbeat
```

---

## Stopping

```bash
tutor dev stop
```

This stops all containers but keeps data in `data/`. Next `tutor dev start` picks up where you left off.

---

## After Stopping Uncleanly (Docker killed, machine crashed)

If MySQL crashes with redo log errors:

```bash
# Stop everything
tutor dev stop

# Remove corrupted MySQL data (safe for dev — data recreatable)
rm -rf $TUTOR_ROOT/data/mysql

# Reinitialize
tutor dev do init

# Start
tutor dev start
```

See [troubleshooting.md](troubleshooting.md) for more scenarios.

---

## Verifying the Installation

After `tutor dev start`:

```bash
# All containers should show "Up"
tutor dev dc ps

# LMS heartbeat
curl http://local.openedx.io:8000/heartbeat  # → {"status": "ok"}

# Studio responds
curl -I http://studio.local.openedx.io:8001  # → 200 OK
```

Open in browser:
- LMS: http://local.openedx.io:8000
- Studio: http://studio.local.openedx.io:8001
- Admin: http://local.openedx.io:8000/admin

Login: `admin` / `admin1234`

---

## Reinstalling From Scratch

If you need a completely clean slate:

```bash
tutor dev stop
rm -rf $TUTOR_ROOT/data/
rm -rf $TUTOR_ROOT/env/
tutor dev launch --non-interactive
# Then re-create admin user
```

> **Warning:** This deletes all data including courses, users, and enrollments.
