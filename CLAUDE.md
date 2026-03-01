# ASU Global Connect — Claude Code Guide

**Project:** Interactive Learning & Community Platform for ASU Online Students
**Team:** Ashish Raj Shekhar, Dr. Vivek Gupta (CoRAL Lab), Dr. Jessica Hirshorn (CISA)
**Platform:** Open edX (Ulmo/Sumac release) via Tutor v21
**Build Tool:** Claude Code (CLI)
**Goal:** Production pilot live by July 2026 | 500 students | 6 custom XBlocks

---

## Project Structure

```
/Users/ashishrajshekhar/Desktop/2026/edx/     ← TUTOR_ROOT
├── config.yml                                  ← Tutor configuration (single source of truth)
├── env/                                        ← Generated Tutor environment (do not hand-edit)
│   ├── build/openedx/                          ← Dockerfile + edx-platform source
│   └── apps/                                  ← Service configs (lms, cms, mysql, redis...)
├── data/                                       ← Persistent data volumes (DB, media)
├── tutor-launch.log                            ← Log from tutor dev launch
│
├── xblocks/                                    ← All 6 custom XBlocks (create this)
│   ├── gamification-xblock/
│   ├── globalmap-xblock/
│   ├── questengine-xblock/
│   ├── phetsim-xblock/
│   ├── virtualcampus-xblock/
│   └── multiplayer-xblock/
│
└── CLAUDE.md                                   ← This file
```

---

## Environment

### Key Variables
```bash
export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
```
> Add these to your shell profile (`~/.zshrc`) to make permanent.

### Service URLs (dev mode)
| Service | URL |
|---------|-----|
| LMS (learner) | http://local.openedx.io:8000 |
| Studio (CMS) | http://studio.local.openedx.io:8001 |
| LMS Admin | http://local.openedx.io:8000/admin |

> `local.openedx.io` is an Overhangio-owned domain — resolves to 127.0.0.1 automatically. No `/etc/hosts` changes needed.

### Admin Credentials
- **Email:** admin@openedx.local
- **Username:** admin
- **Password:** (set during first `tutor dev launch`, stored in Tutor config)

---

## Tutor Commands Reference

```bash
# First-time setup (already done in Week 1)
tutor dev launch --non-interactive

# Start services (subsequent sessions)
tutor dev start

# Stop services
tutor dev stop

# View logs (all services)
tutor dev logs --follow

# View logs for specific service
tutor dev logs --follow lms
tutor dev logs --follow cms

# Open Django shell in LMS
tutor dev run lms bash
tutor dev run lms ./manage.py lms shell

# Run management command
tutor dev run lms ./manage.py lms <command>

# Restart single service
tutor dev restart lms
```

---

## The 6 Custom XBlocks

| # | XBlock | Package Name | Description | Key Tech |
|---|--------|-------------|-------------|----------|
| 1 | GlobalMap | `globalmap-xblock` | Mapbox GL student map with peer discovery | Mapbox GL JS v3, PostGIS |
| 2 | QuestEngine | `questengine-xblock` | Branching narrative quests with XP rewards | YAML quest definitions |
| 3 | PhETSim | `phetsim-xblock` | HTML5 simulation embeds with grading | postMessage API, iframe |
| 4 | VirtualCampus | `virtualcampus-xblock` | 3D social space via Mozilla Hubs | Mozilla Hubs WebXR |
| 5 | Gamification | `gamification-xblock` | Central XP/badge/leaderboard engine | Redis sorted sets |
| 6 | Multiplayer | `multiplayer-xblock` | Real-time collaborative challenges | Socket.io sidecar |

### XP Rewards Table
| Action | XP | Source XBlock |
|--------|----|--------------|
| Drop map pin | 100 | GlobalMap |
| Complete profile | 50 | GlobalMap |
| Connection accepted | 25 | GlobalMap |
| Quest step completed | 25–100 | QuestEngine |
| Simulation passed | 50–200 | PhETSim |
| Attend virtual event | 75 | VirtualCampus |
| Win multiplayer challenge | 100 | Multiplayer |
| Weekly challenge complete | 50 | Gamification |

### Badges
Globe Trotter, Mentor, Challenger, Cultural Ambassador, Pioneer, Connector, Sim Master, Quest Champion

---

## XBlock Development Workflow

### 1. Create a new XBlock skeleton
```bash
cd /Users/ashishrajshekhar/Desktop/2026/edx/xblocks/
cookiecutter https://github.com/openedx/cookiecutter-xblock
# Enter: package_name = my-xblock, etc.
```

### 2. Install into running dev environment (hot reload)
```bash
cd /Users/ashishrajshekhar/Desktop/2026/edx/xblocks/my-xblock/

# Install into LMS
tutor dev run lms pip install -e /openedx/xblocks/my-xblock

# Install into CMS (Studio)
tutor dev run cms pip install -e /openedx/xblocks/my-xblock

# Then restart the services to pick up the new XBlock
tutor dev restart lms cms
```

> Note: Mount your xblocks dir into the container by adding it to tutor's bind mounts (see below).

### 3. Mount local xblocks directory into containers
Add to `config.yml` or via plugin — bind-mount the xblocks folder:
```bash
tutor config save --set OPENEDX_EXTRA_REQUIREMENTS="[]"
# Then use tutor-plugin or docker-compose override for bind mounts
```

Alternatively for ad-hoc installs:
```bash
# Copy XBlock into running LMS container
docker cp ./my-xblock/ $(tutor dev dc ps -q lms):/openedx/xblocks/
tutor dev run lms pip install -e /openedx/xblocks/my-xblock
```

### 4. Test
```bash
cd /Users/ashishrajshekhar/Desktop/2026/edx/xblocks/my-xblock/
python -m pytest tests/ --cov=my_xblock --cov-report=html
# Coverage target: 80%+
```

### 5. Enable XBlock in LMS/CMS
Add to `INSTALLED_XBLOCKS` in Tutor config or via Studio advanced settings:
- Studio → Course → Advanced Settings → Advanced Module List
- Add: `"my_xblock"`

### 6. Install to production (when ready)
```bash
tutor local run lms pip install git+https://github.com/your-org/my-xblock.git
tutor local run cms pip install git+https://github.com/your-org/my-xblock.git
tutor local restart
```

---

## XBlock Architecture Pattern

Every XBlock in this project follows this structure:

```python
# my_xblock/my_xblock.py
import pkg_resources
from xblock.core import XBlock
from xblock.fields import Integer, String, Dict, Scope
from xblock.fragment import Fragment

class MyXBlock(XBlock):
    # --- Fields ---
    # user_state: per-student data (progress, scores, etc.)
    # settings: instructor-configurable values
    # content: author-defined content

    display_name = String(default="My XBlock", scope=Scope.settings)
    user_data = Dict(default={}, scope=Scope.user_state)

    # --- Views ---
    def student_view(self, context=None):
        """Learner-facing view rendered in LMS."""
        html = self.resource_string("static/html/student.html")
        frag = Fragment(html)
        frag.add_css(self.resource_string("static/css/style.css"))
        frag.add_javascript(self.resource_string("static/js/student.js"))
        frag.initialize_js('MyXBlock')
        return frag

    def studio_view(self, context=None):
        """Instructor-facing edit view in Studio."""
        ...

    # --- Handlers ---
    @XBlock.json_handler
    def save_data(self, data, suffix=''):
        """Called from JS via runtime.handlerUrl()."""
        self.user_data = data
        # Publish event to Gamification XBlock
        self.runtime.publish(self, 'xp_earned', {'xp': 100})
        return {'success': True}

    # --- Helpers ---
    def resource_string(self, path):
        return pkg_resources.resource_string(__name__, path).decode("utf8")

    @staticmethod
    def workbench_scenarios():
        return [("MyXBlock", "<my_xblock/>")]
```

### Publishing Events to Gamification XBlock
```python
# From any XBlock, publish XP events:
self.runtime.publish(self, 'xp_earned', {
    'xp': 100,
    'action': 'drop_map_pin',
    'user_id': self.runtime.user_id,
})
```

---

## Architecture Overview

```
ASU Students (Browser / Mobile PWA)
           │
    ┌──────▼──────────────────────────────────────┐
    │          OPEN edX PLATFORM (Ulmo)            │
    │   LMS (learner) │ Studio │ Insights │ Forum  │
    │                                              │
    │  ┌─────────── CUSTOM XBLOCKS ──────────────┐ │
    │  │ GlobalMap │ QuestEngine │ PhETSim        │ │
    │  │ VirtualCampus │ Gamification │ Multiplayer│ │
    │  └─────────────────────────────────────────┘ │
    │                                              │
    │  ┌────────── INTEGRATIONS ─────────────────┐ │
    │  │ ASU CAS SSO │ Canvas LTI │ Mapbox │ Socket│ │
    │  └─────────────────────────────────────────┘ │
    │                                              │
    │  ┌────────── DATA LAYER ───────────────────┐ │
    │  │ PostgreSQL+PostGIS │ Redis │ S3/MinIO    │ │
    │  └─────────────────────────────────────────┘ │
    └─────────────────────────────────────────────┘
```

---

## Build Phases

| Phase | Weeks | XBlocks | Milestone |
|-------|-------|---------|-----------|
| 0: Environment Setup | 1 | — | Tutor dev env running, CLAUDE.md |
| 1: Gamification + GlobalMap | 2–4 | Gamification, GlobalMap | Core XP engine + interactive map |
| 2: QuestEngine + PhETSim | 5–7 | QuestEngine, PhETSim | Branching quests + sim assessments |
| 3: VirtualCampus + Multiplayer | 8–10 | VirtualCampus, Multiplayer | 3D social + real-time challenges |
| 4: Integration + Deploy | 11–14 | All 6 | ASU theming, CAS SSO, Canvas LTI |
| 5: Pilot Launch | 15–16 | — | 500 students, data collection |

---

## Key Conventions

### Python / Django
- Python 3.11 (edx-platform default)
- Follow Open edX coding standards: `make lint` passes before any commit
- XBlock fields use `Scope.user_state` for student data, `Scope.settings` for instructor config
- Use Django ORM via `XBlockAside` or service lookups — never raw SQL

### JavaScript
- Vanilla JS or React for XBlock frontends
- Access XBlock handlers via: `runtime.handlerUrl(element, 'handler_name')`
- Use `postMessage` for iframe-based XBlocks (PhETSim, VirtualCampus)

### Testing
- 80%+ coverage required for all XBlocks
- Use `unittest.mock` to mock XBlock runtime
- `pytest` with `--cov` for coverage reports

### Git
- Branch naming: `xblock/<name>-<feature>` (e.g., `xblock/globalmap-pin-placement`)
- Commit prefix: `feat:`, `fix:`, `test:`, `docs:`
- Never commit `config.yml` secrets to public repos

### Tutor Config
- All config lives in `config.yml` at TUTOR_ROOT
- Regenerate env after config changes: `tutor config save`
- Custom settings go in a Tutor plugin, not direct file edits in `env/`

---

## ASU CAS SSO (Future — Phase 4)

```python
# settings/production.py
INSTALLED_APPS += ['django_cas_ng']
AUTHENTICATION_BACKENDS = [
    'django_cas_ng.backends.CASBackend',
    'django.contrib.auth.backends.ModelBackend',
]
CAS_SERVER_URL = 'https://weblogin.asu.edu/cas/'
CAS_VERSION = '3'
CAS_REDIRECT_URL = '/'
```

---

## Canvas LTI Integration (Future — Phase 4)

Open edX Ulmo is LTI Advantage Complete certified:
- Dr. Hirshorn adds ASU Global Connect as an LTI tool in Canvas
- PhETSim + QuestEngine grades flow back to Canvas gradebook via LTI AGS
- Students launch from Canvas → Open edX → SSO passes through

---

## Useful Links

- Tutor docs: https://docs.tutor.edly.io
- XBlock API: https://docs.openedx.org/projects/xblock
- Open edX developer docs: https://docs.openedx.org
- cookiecutter-xblock: https://github.com/openedx/cookiecutter-xblock
- edx-platform source: https://github.com/openedx/edx-platform

---

*Built for CoRAL Lab, Arizona State University. March 2026.*
