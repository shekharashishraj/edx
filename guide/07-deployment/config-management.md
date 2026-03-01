# Configuration Management

How ASU Global Connect manages configuration across services and environments.

---

## config.yml — Single Source of Truth

`$TUTOR_ROOT/config.yml` is the only file you should ever hand-edit (besides XBlock code). Everything else is generated from it.

```yaml
# Key entries in config.yml
PLATFORM_NAME: ASU Global Connect
LMS_HOST: local.openedx.io
CMS_HOST: studio.local.openedx.io
CONTACT_EMAIL: ashish@asu.edu
ENABLE_HTTPS: false

# Auto-generated secrets (do not edit manually)
JWT_RSA_PRIVATE_KEY: |
  -----BEGIN RSA PRIVATE KEY-----
  ...
MYSQL_ROOT_PASSWORD: n1zYiaH5
OPENEDX_SECRET_KEY: mkhEuHdhSR15wIWkNSv2Ypp0
OPENEDX_MYSQL_PASSWORD: JVeZjIku

# Plugins
PLUGINS:
  - mfe
  - indigo
```

---

## How Config Propagates

```
config.yml
    │
    └── tutor config save
              │
              ├── env/apps/openedx/config/lms.env.yml
              │     (LMS runtime config: DATABASES, FEATURES, etc.)
              │
              ├── env/apps/openedx/settings/lms/development.py
              │     (Django settings module, imported at startup)
              │
              ├── env/local/docker-compose.yml
              │     (Service definitions, environment variables)
              │
              ├── env/build/openedx/Dockerfile
              │     (Container image build instructions)
              │
              └── env/k8s/*.yml
                    (Kubernetes manifests)
```

**Rule:** After changing `config.yml`, always run:
```bash
tutor config save    # regenerates env/
tutor dev restart    # picks up new config
```

---

## Setting Configuration Values

```bash
# Set a single value and regenerate env
tutor config save --set PLATFORM_NAME="ASU Global Connect v2"

# Set multiple values
tutor config save \
  --set LMS_HOST="openedx.asu.edu" \
  --set CMS_HOST="studio.openedx.asu.edu" \
  --set ENABLE_HTTPS=true

# Read a value
tutor config printvalue LMS_HOST
tutor config printvalue MYSQL_ROOT_PASSWORD
```

---

## Tutor Plugins

Plugins are the correct way to add custom configuration that isn't in the base Tutor:

```bash
# Install a plugin
tutor plugins install mfe indigo

# List plugins
tutor plugins list

# Enable/disable
tutor plugins enable my-plugin
tutor plugins disable my-plugin
```

For project-specific config (XBlock settings, Mapbox API keys, etc.), create a custom Tutor plugin:

```python
# custom-plugin/plugin.py
from tutor import hooks

@hooks.Filters.CONFIG_DEFAULTS.add()
def add_custom_config(items):
    items.append(("MAPBOX_API_KEY", "pk.eyJ1IjoidXNlciJ9..."))
    return items

@hooks.Filters.ENV_PATCHES.add()
def patch_lms_settings(items):
    items.append(("lms-env-features", """
FEATURES:
  ENABLE_MY_CUSTOM_FEATURE: true
"""))
    return items
```

---

## Environment Variables in Containers

Services receive config via environment variables set in docker-compose.yml (generated from config.yml):

```yaml
# In generated env/dev/docker-compose.yml
services:
  lms:
    environment:
      DJANGO_SETTINGS_MODULE: lms.envs.tutor.development
      SERVICE_VARIANT: lms
      # ... other vars set from config.yml
```

LMS startup reads these env vars via `lms.env.yml` which is bind-mounted into the container at `/openedx/config/lms.env.yml`.

---

## Secrets Management

**Current (dev):** Secrets stored in `config.yml` as plaintext. This is acceptable for local dev.

**Production (Phase 4-5):** Move secrets to:
- AWS Secrets Manager / GCP Secret Manager
- Kubernetes Secrets (encrypted at rest)
- Tutor plugin that injects secrets from environment

**Never commit `config.yml` to a public repository** — it contains:
- `JWT_RSA_PRIVATE_KEY` — signs all OAuth tokens
- `MYSQL_ROOT_PASSWORD` — full DB access
- `OPENEDX_SECRET_KEY` — Django session signing
- `MEILISEARCH_MASTER_KEY` — search admin access

---

## Config File Locations (Summary)

| File | Generated? | Purpose |
|------|-----------|---------|
| `config.yml` | No (hand-edited) | Single source of truth |
| `env/apps/openedx/config/lms.env.yml` | Yes | LMS runtime settings |
| `env/apps/openedx/config/cms.env.yml` | Yes | Studio runtime settings |
| `env/apps/openedx/settings/lms/development.py` | Yes | Django dev settings |
| `env/apps/openedx/settings/lms/production.py` | Yes | Django prod settings |
| `env/local/docker-compose.yml` | Yes | Production compose |
| `env/dev/docker-compose.yml` | Yes | Dev compose |
| `env/build/openedx/Dockerfile` | Yes | Image build recipe |
| `env/k8s/*.yml` | Yes | Kubernetes manifests |
