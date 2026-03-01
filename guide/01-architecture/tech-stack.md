# Technology Stack

Everything used in ASU Global Connect, what version, and why it was chosen.

---

## Platform Foundation

### Open edX — Ulmo Release
- **What it is:** Open-source LMS used by MIT, Harvard, edX.org. Handles courses, users, grades, forums, certificates.
- **Release:** `ulmo` (branch `release/ulmo.1`) — Tutor v21 default
- **Why:** Most battle-tested open-source LMS. LTI Advantage Complete certified. Extensible via XBlocks.
- **Docs:** https://docs.openedx.org

### Tutor v21.0.1
- **What it is:** Docker-based Open edX distribution and management tool
- **Why:** Eliminates weeks of manual Open edX setup. Single `tutor dev launch` gets a full environment running. Handles config, Docker Compose orchestration, migrations, asset builds.
- **Config:** `config.yml` at TUTOR_ROOT — single source of truth
- **Docs:** https://docs.tutor.edly.io

### Django 5.2
- **What it is:** Python web framework powering both LMS and Studio
- **Why:** Open edX is built on Django. Not a choice — it's the platform.
- **ORM:** Used for all MySQL interactions (never raw SQL)
- **Settings files:** `env/apps/openedx/settings/lms/` and `env/apps/openedx/settings/cms/`

---

## Languages

| Language | Used For | Version |
|----------|----------|---------|
| Python | XBlock backend logic, Django views, management commands | 3.11.8 |
| JavaScript | XBlock frontend (student.js), MFE React apps | ES2020+ |
| SCSS/CSS | XBlock styles, Indigo theme customization | — |
| YAML | Quest definitions (QuestEngine), Tutor config | — |
| SQL | Direct DB inspection only (never in app code) | MySQL 8.4 dialect |

---

## Infrastructure

### Docker v26.1.0
- All services run in containers via Docker Compose
- Two compose files overlay: `env/local/docker-compose.yml` (base) + `env/dev/docker-compose.yml` (dev overrides)
- Dev image includes `ipdb`, `ipython`, `npm watch` for hot reloading

### Docker Compose
- **Project name:** `tutor_dev`
- **Network:** All services on the same Docker bridge network, addressed by container name
- **Volumes:** Bind-mounted to `data/` for persistence across container restarts

---

## Databases

### MySQL 8.4.0
- **Role:** Primary relational database
- **Hosts:** Users, enrollments, grades, auth tokens, certificates, forum data
- **Connection:** Django ORM only, `ENGINE: django.db.backends.mysql`
- **Database name:** `openedx`
- **Why MySQL:** Open edX has used MySQL since day one. Full Django ORM support.

### MongoDB 7.0.28
- **Role:** Document store for course content and XBlock state
- **Hosts:** XBlock definitions, course structure (modulestore), ORA2 responses, split modulestore
- **Connection:** `xmodule.contentstore.mongo.MongoContentStore`
- **Database name:** `openedx`
- **Why MongoDB:** Course content is hierarchical JSON — perfect for documents. Course structure changes frequently; schema-free storage is ideal.

### Redis 7.4.5
- **Role:** In-memory cache + task queue + leaderboard store (future)
- **Uses:**
  - Django session cache
  - Celery task broker (async jobs like email, grade recalculation)
  - Rate limiting counters
  - Future: sorted sets for XP leaderboards (Gamification XBlock)
- **Why Redis:** Ultra-fast. Celery ships with Redis support. Sorted sets are perfect for leaderboards.

### Meilisearch v1.8.4
- **Role:** Full-text search engine
- **Uses:** Course discovery search, content search
- **Why Meilisearch:** Open edX Ulmo migrated from Elasticsearch to Meilisearch. Faster, easier to operate, no JVM.

---

## XBlock-Specific Technologies

### Mapbox GL JS v3 (GlobalMap XBlock)
- **What:** JavaScript library for interactive vector maps
- **Use:** Renders the student location map, drop-pin interface, peer discovery layer
- **Why Mapbox:** Best-in-class vector maps, 3D capability, generous free tier, PostGIS integration

### PostGIS (GlobalMap XBlock)
- **What:** Geospatial extension for PostgreSQL (or MySQL via geometry types)
- **Use:** Stores student location pins, enables proximity queries ("who's near me?")
- **Why:** Standard for geospatial SQL. Supports point-in-polygon, distance queries natively.

### Mozilla Hubs WebXR (VirtualCampus XBlock)
- **What:** Open-source 3D social VR platform, runs in browser
- **Use:** Hosts the ASU virtual campus space students can walk around in
- **Why:** No download required (pure browser WebGL/WebXR). Open source. Self-hostable.

### Socket.io (Multiplayer XBlock)
- **What:** Real-time bidirectional event library (WebSocket wrapper)
- **Use:** Powers real-time collaborative challenge sessions between students
- **Architecture:** Runs as a sidecar container alongside the LMS
- **Why Socket.io:** Handles WebSocket fallbacks, rooms/namespaces, reconnection logic automatically.

### PhET Interactive Simulations (PhETSim XBlock)
- **What:** University of Colorado HTML5 physics/chemistry/math simulations
- **Use:** Embed PhET sims in courseware; grade via postMessage API
- **Why:** Gold standard for STEM simulations. 100% free. HTML5 = no Flash/Java.

### postMessage API (PhETSim + VirtualCampus XBlocks)
- **What:** Browser API for secure cross-origin iframe communication
- **Use:** PhET sims send grade events to parent XBlock; VirtualCampus sends attendance events
- **Pattern:** `window.addEventListener('message', handler)` in XBlock JS

---

## Open edX Plugins Enabled

### MFE Plugin
- Adds React-based Micro-Frontend apps for login, registration, learner dashboard
- Each MFE is a separate React app served by Caddy on its own port
- **Authn MFE** handles all login/registration flows (the login page you see)

### Indigo Theme
- Default Open edX visual theme included with Tutor
- Customizable via SCSS variables
- Theme files: `env/build/openedx/themes/`

---

## Build Tools

### uv (Python package installer)
- Ultra-fast drop-in replacement for pip
- Used in the Dockerfile to install all Python dependencies
- Much faster than pip for the ~400 packages Open edX needs

### pyenv
- Manages Python version (3.11.8) inside the Docker image
- Ensures exact Python version reproducibility across environments

### npm + Webpack
- Builds all JavaScript/CSS static assets for edx-platform
- `npm run build-dev` creates development assets with source maps
- `watchthemes` container runs `npm run watch-sass` for live SCSS recompilation

### Caddy
- Reverse proxy that routes traffic to LMS, CMS, MFE apps
- Handles URL rewriting (e.g., `/login` → authn MFE)
- Configured via `env/plugins/mfe/apps/mfe/Caddyfile`

---

## Development Tools

### cookiecutter-xblock
- Scaffold generator for new XBlocks
- Generates the standard file structure: `setup.py`, XBlock class, static files, tests
- URL: https://github.com/openedx/cookiecutter-xblock

### pytest
- Test runner for all XBlocks
- Target: 80%+ coverage per XBlock
- Run: `python -m pytest tests/ --cov=my_xblock --cov-report=html`

### ipdb / ipython
- Available in dev containers for interactive debugging
- `PYTHONBREAKPOINT=ipdb.set_trace` set in Dockerfile

---

## Future Tech (Phase 4)

### django-cas-ng
- ASU CAS SSO integration
- Single sign-on via `https://weblogin.asu.edu/cas/`

### LTI Advantage (Canvas Integration)
- Open edX Ulmo is LTI Advantage Complete certified
- Grades from PhETSim + QuestEngine flow to Canvas gradebook via LTI AGS (Assignment and Grade Services)
