# Environments

ASU Global Connect runs in three distinct environments, each with different Docker Compose configurations.

---

## Environment Comparison

| Feature | Dev | Local (Prod-like) | Kubernetes |
|---------|-----|-------------------|------------|
| Command prefix | `tutor dev` | `tutor local` | `tutor k8s` |
| Django settings | `development.py` | `production.py` | `production.py` |
| Server | Django runserver | uWSGI | uWSGI |
| Hot reload | Yes | No | No |
| Debug mode | Yes | No | No |
| Compose file | `env/dev/docker-compose.yml` | `env/local/docker-compose.yml` | `env/k8s/` |
| HTTPS | No | Optional | Yes |
| Status | In use | Ready | Future |

---

## Dev Environment (Current)

**Use for:** All daily development work, XBlock building, debugging.

```bash
# Start
tutor dev start

# Service URLs
LMS:    http://local.openedx.io:8000
Studio: http://studio.local.openedx.io:8001
Admin:  http://local.openedx.io:8000/admin

# Django settings
env/apps/openedx/settings/lms/development.py
env/apps/openedx/settings/cms/development.py
```

**Key differences from production:**
- Django's built-in dev server (`runserver`) with auto-reload
- `DEBUG = True` (full error pages, no production error suppression)
- `watchthemes` container auto-recompiles SCSS
- `ipdb` available for breakpoint debugging
- Emails caught by local SMTP (nothing actually sent)

---

## Local (Prod-like) Environment

**Use for:** Staging testing, production config verification, before deploying to cloud.

```bash
# Start
tutor local start

# Stop
tutor local stop

# Initialize (first time)
tutor local launch
```

**Key differences from dev:**
- uWSGI production server (multi-worker, no auto-reload)
- `DEBUG = False`
- Full Caddy reverse proxy configuration
- Stricter security settings (CSRF, CORS, HSTS ready)
- Same Docker Compose network but different container configs

---

## Kubernetes Environment (Phase 5)

**Use for:** Production pilot (500 students, July 2026).

```bash
# Deploy to Kubernetes cluster
tutor k8s start

# Management commands in K8s
tutor k8s run lms ./manage.py lms migrate
```

Tutor generates all Kubernetes manifests in `env/k8s/`:
```
env/k8s/
├── namespace.yml       → openedx namespace
├── deployments.yml     → LMS, CMS, MFE, Meilisearch deployments
├── services.yml        → ClusterIP services for inter-pod communication
├── volumes.yml         → PersistentVolumeClaims for data
├── jobs.yml            → Init jobs (migrations, etc.)
└── override.yml        → Custom overrides
```

**K8s considerations:**
- Managed MySQL (RDS) replaces containerized MySQL for reliability
- Managed MongoDB (Atlas) replaces containerized MongoDB
- Redis Cluster for high availability
- S3/MinIO for media storage (not local filesystem)
- Load balancer + Ingress for HTTP/HTTPS routing

---

## Config Differences by Environment

Settings are layered:

```python
# development.py
from lms.envs.devstack import *  # base development settings
# Debug overrides...

# production.py
from lms.envs.production import *  # base production settings
# Production security settings...
```

Key settings that differ:

| Setting | Dev | Production |
|---------|-----|------------|
| `DEBUG` | `True` | `False` |
| `ALLOWED_HOSTS` | `['*']` | `['local.openedx.io']` |
| `EMAIL_BACKEND` | SMTP (local) | SES / Mailgun |
| `DEFAULT_FILE_STORAGE` | Local filesystem | S3 |
| `SESSION_COOKIE_SECURE` | `False` | `True` |
| `CSRF_COOKIE_SECURE` | `False` | `True` |

---

## Switching Environments

You use the same `config.yml` for all environments — Tutor generates different compose files/settings based on which subcommand you use (`dev`, `local`, `k8s`).

Never run `tutor dev` and `tutor local` at the same time — they use overlapping ports.
