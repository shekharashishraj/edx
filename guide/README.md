# ASU Global Connect — Documentation Guide

**Project:** Interactive Learning & Community Platform for ASU Online Students
**Platform:** Open edX Ulmo (Tutor v21.0.1)
**Team:** Ashish Raj Shekhar · Dr. Vivek Gupta (CoRAL Lab) · Dr. Jessica Hirshorn (CISA)
**Pilot Target:** 500 students · July 2026

---

## How to Use This Guide

This guide covers everything about the ASU Global Connect repository — from first-time setup to deep architectural decisions. Start with the section that matches your goal.

---

## Table of Contents

### 01 — Architecture
| File | What it covers |
|------|----------------|
| [overview.md](01-architecture/overview.md) | Full system diagram, services, how everything connects |
| [tech-stack.md](01-architecture/tech-stack.md) | Every technology used, versions, and why it was chosen |
| [data-flow.md](01-architecture/data-flow.md) | How data moves through the system — request to DB and back |

### 02 — Setup
| File | What it covers |
|------|----------------|
| [prerequisites.md](02-setup/prerequisites.md) | What you need before starting (Docker, Python, Tutor) |
| [installation.md](02-setup/installation.md) | Step-by-step install from zero to running LMS |
| [troubleshooting.md](02-setup/troubleshooting.md) | Common problems and how to fix them |

### 03 — Development
| File | What it covers |
|------|----------------|
| [workflow.md](03-development/workflow.md) | Day-to-day development workflow |
| [tutor-commands.md](03-development/tutor-commands.md) | Every Tutor CLI command you will use |
| [dev-tips.md](03-development/dev-tips.md) | Hot reload, debugging, shell access, log reading |

### 04 — XBlocks
| File | What it covers |
|------|----------------|
| [overview.md](04-xblocks/overview.md) | What XBlocks are and how Open edX uses them |
| [architecture.md](04-xblocks/architecture.md) | Base XBlock code pattern every block in this project follows |
| [development-guide.md](04-xblocks/development-guide.md) | Complete guide to building a new XBlock |
| [xp-system.md](04-xblocks/xp-system.md) | XP/badge/leaderboard system design |
| [gamification.md](04-xblocks/gamification.md) | Gamification XBlock spec |
| [globalmap.md](04-xblocks/globalmap.md) | GlobalMap XBlock spec |
| [questengine.md](04-xblocks/questengine.md) | QuestEngine XBlock spec |
| [phetsim.md](04-xblocks/phetsim.md) | PhETSim XBlock spec |
| [virtualcampus.md](04-xblocks/virtualcampus.md) | VirtualCampus XBlock spec |
| [multiplayer.md](04-xblocks/multiplayer.md) | Multiplayer XBlock spec |

### 05 — Database
| File | What it covers |
|------|----------------|
| [overview.md](05-database/overview.md) | Why three databases, who owns what data |
| [mysql.md](05-database/mysql.md) | Key tables, Django ORM models, schema design |
| [mongodb.md](05-database/mongodb.md) | Collections, XBlock state storage, course content |
| [redis.md](05-database/redis.md) | Cache keys, Celery queues, session storage, leaderboards |

### 06 — Software Design
| File | What it covers |
|------|----------------|
| [design-patterns.md](06-software-design/design-patterns.md) | Patterns used across the project |
| [event-system.md](06-software-design/event-system.md) | XBlock event publishing and the XP event bus |
| [api-design.md](06-software-design/api-design.md) | REST handlers, JSON handlers, AJAX patterns |

### 07 — Deployment
| File | What it covers |
|------|----------------|
| [environments.md](07-deployment/environments.md) | Dev vs. Local (prod) vs. Kubernetes environments |
| [config-management.md](07-deployment/config-management.md) | config.yml, Tutor plugins, env/ generation |
| [production.md](07-deployment/production.md) | Steps to deploy to production |

### 08 — Integrations
| File | What it covers |
|------|----------------|
| [cas-sso.md](08-integrations/cas-sso.md) | ASU CAS SSO (Phase 4) |
| [canvas-lti.md](08-integrations/canvas-lti.md) | Canvas LTI Advantage integration (Phase 4) |
| [third-party.md](08-integrations/third-party.md) | Mapbox, Mozilla Hubs, Socket.io, PhET |

---

## Quick Reference

```bash
# Start dev environment
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"
tutor dev start

# Service URLs
LMS:    http://local.openedx.io:8000
Studio: http://studio.local.openedx.io:8001
Admin:  http://local.openedx.io:8000/admin

# Login
Username: admin
Email:    admin@openedx.local
Password: admin1234
```

## Build Phases at a Glance

| Phase | Weeks | XBlocks | Status |
|-------|-------|---------|--------|
| 0: Environment Setup | 1 | — | Done |
| 1: Core Engine | 2–4 | Gamification, GlobalMap | In Progress |
| 2: Learning Tools | 5–7 | QuestEngine, PhETSim | Upcoming |
| 3: Social Features | 8–10 | VirtualCampus, Multiplayer | Upcoming |
| 4: Integration | 11–14 | All 6 | Upcoming |
| 5: Pilot Launch | 15–16 | — | July 2026 |
