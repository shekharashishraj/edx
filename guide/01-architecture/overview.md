# System Architecture Overview

## Big Picture

ASU Global Connect is a gamified online learning platform built on top of Open edX (Ulmo release). Open edX provides the core LMS engine; we extend it with 6 custom XBlocks that add map-based peer discovery, branching quests, physics simulations, a 3D virtual campus, real-time multiplayer challenges, and a central gamification engine tying them all together.

---

## Full System Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                    ASU Students (Browser / PWA)                  │
└───────────────┬──────────────────────────┬───────────────────────┘
                │ :8000                    │ :8001
      ┌─────────▼────────┐      ┌──────────▼────────┐
      │   LMS (Learner)  │      │  Studio (Authors)  │
      │  local.openedx   │      │ studio.local.open  │
      │  Django 5.2      │      │ Django 5.2         │
      └─────────┬────────┘      └──────────┬─────────┘
                │                          │
      ┌─────────▼──────────────────────────▼─────────┐
      │              XBLOCK LAYER                     │
      │  ┌──────────────┐  ┌──────────────┐           │
      │  │ Gamification │  │  GlobalMap   │           │
      │  │  XP/Badge/LB │  │ Mapbox+PostGIS│          │
      │  └──────────────┘  └──────────────┘           │
      │  ┌──────────────┐  ┌──────────────┐           │
      │  │ QuestEngine  │  │   PhETSim    │           │
      │  │ YAML Quests  │  │ HTML5 Sims   │           │
      │  └──────────────┘  └──────────────┘           │
      │  ┌──────────────┐  ┌──────────────┐           │
      │  │VirtualCampus │  │  Multiplayer │           │
      │  │Mozilla Hubs  │  │  Socket.io   │           │
      │  └──────────────┘  └──────────────┘           │
      └───────────────────────┬───────────────────────┘
                              │
      ┌───────────────────────▼───────────────────────┐
      │               DATA LAYER                      │
      │                                               │
      │  MySQL :3306        MongoDB :27017            │
      │  ─────────────      ─────────────────         │
      │  Users              Course content            │
      │  Enrollments        XBlock state              │
      │  Grades             Modulestore               │
      │  Auth tokens        ORA2 responses            │
      │                                               │
      │  Redis :6379        Meilisearch :7700         │
      │  ─────────────      ─────────────────         │
      │  Session cache      Full-text search          │
      │  Celery queue       Course discovery          │
      │  Leaderboards       Content indexing          │
      └───────────────────────────────────────────────┘
```

---

## Services

### LMS (Learner Management System)
- **URL:** http://local.openedx.io:8000
- **Container:** `tutor_dev-lms-1`
- **Image:** `openedx-dev:21.0.1`
- **Framework:** Django 5.2, Python 3.11
- **Settings module:** `lms.envs.tutor.development`
- **Role:** Everything students see — course catalog, courseware, gradebook, profile, forums

### Studio / CMS (Course Management System)
- **URL:** http://studio.local.openedx.io:8001
- **Container:** `tutor_dev-cms-1`
- **Image:** `openedx-dev:21.0.1`
- **Settings module:** `cms.envs.tutor.development`
- **Role:** Course authoring tool — instructors create sections, upload content, configure XBlocks

### MFE (Micro-Frontend)
- **Container:** `tutor_dev-mfe-1`
- **Image:** `openedx-mfe:21.0.0-indigo`
- **Ports:** 1984–2025 (each MFE app gets its own port)
- **Role:** React-based UI shells — handles login/registration (authn MFE), learner dashboard, etc.
- **Reverse proxy:** Caddy routes `/login`, `/dashboard` etc. to the correct MFE port

### MySQL
- **Container:** `tutor_dev-mysql-1`
- **Image:** `mysql:8.4.0`
- **Port:** 3306 (internal)
- **Database:** `openedx`
- **Role:** All relational data — users, enrollments, permissions, grades, certificates

### MongoDB
- **Container:** `tutor_dev-mongodb-1`
- **Image:** `mongo:7.0.28`
- **Port:** 27017 (internal)
- **Database:** `openedx`
- **Role:** Document store for course content (XBlock definitions, course structure), ORA2 responses

### Redis
- **Container:** `tutor_dev-redis-1`
- **Image:** `redis:7.4.5`
- **Port:** 6379 (internal)
- **Role:** Session cache, Celery task broker, rate limiting, leaderboard sorted sets (future)

### Meilisearch
- **Container:** `tutor_dev-meilisearch-1`
- **Image:** `meilisearch:v1.8.4`
- **Port:** 7700 (internal)
- **Role:** Full-text search for course discovery and content search

### SMTP (Dev Only)
- **Container:** `tutor_dev-smtp-1`
- **Image:** `devture/exim-relay:4.96-r1-0`
- **Port:** 8025 (internal)
- **Role:** Catches all outgoing emails in dev; nothing actually gets sent

### WatchThemes
- **Container:** `tutor_dev-watchthemes-1`
- **Role:** Watches SCSS theme files and recompiles on change (dev-only live reload for styles)

---

## Request Flow

### Typical student page load (e.g., viewing a unit):

```
Browser → local.openedx.io:8000
  → Caddy (reverse proxy)
  → LMS Django app
    → Django Router matches URL
    → View function
      → MySQL: fetch user, enrollment, grade data
      → MongoDB: fetch course structure and XBlock definitions
      → XBlock runtime: render XBlock fragment (HTML + JS + CSS)
      → Redis: check session, cache course overview
    → Django template renders full page
  → HTML response → Browser
  → Browser loads XBlock JS
    → JS calls runtime.handlerUrl() → AJAX POST to LMS
      → XBlock JSON handler
        → MySQL/MongoDB: read/write student state
        → Redis: publish XP event (future)
      → JSON response → JS updates UI
```

### Studio save flow (instructor edits XBlock):
```
Studio UI → AJAX PUT /xblock/<usage_key>
  → CMS Django
    → Validate content
    → MongoDB: write XBlock definition
    → MySQL: log edit audit
  → 200 OK
```

---

## Networking

All services run inside a Docker Compose network named `tutor_dev`. Services reference each other by container name (e.g., LMS connects to MySQL at host `mysql`, not `localhost`).

`local.openedx.io` is an Overhangio-owned domain that resolves to `127.0.0.1` via public DNS — no `/etc/hosts` edits needed.

Port mapping from host:
| Host Port | Container | Service |
|-----------|-----------|---------|
| 8000 | lms:8000 | LMS |
| 8001 | cms:8000 | Studio |
| 1984–2025 | mfe:8002 | MFE apps |
| 7700 | meilisearch:7700 | Search (internal only) |

---

## File Storage

| Path | Purpose |
|------|---------|
| `data/mysql/` | MySQL data files (InnoDB) |
| `data/mongodb/` | MongoDB data files |
| `data/redis/` | Redis AOF persistence |
| `data/meilisearch/` | Search index |
| `data/lms/` | LMS logs, ORA2, modulestore |
| `data/cms/` | Studio logs |
| `data/openedx-media/` | User-uploaded files (images, videos) |
| `data/openedx-media-private/` | Private submissions (ORA2, certificates) |
