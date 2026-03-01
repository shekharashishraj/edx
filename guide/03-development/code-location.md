The code is **inside the Docker container**, baked into the image during the build. Here's the full picture:

---

## Where the Code Lives

### Backend (Django / Python)
```
Inside container: /openedx/edx-platform/
```
Cloned from: `https://github.com/openedx/edx-platform` branch `release/ulmo.1`

Key backend folders:
```
/openedx/edx-platform/
├── lms/                ← LMS backend (views, APIs, URLs)
│   ├── djangoapps/     ← Django apps (grades, certificates, courseware, etc.)
│   └── envs/           ← Django settings files
├── cms/                ← Studio/CMS backend
│   └── djangoapps/
├── openedx/            ← Shared platform code
│   └── core/djangoapps/  ← Auth, search, user management, LTI
├── xmodule/            ← XBlock base classes and built-in XBlocks
├── common/             ← Shared models (student, course_modes, entitlements)
├── manage.py           ← Django entrypoint
└── requirements/       ← Python dependencies
```

### Frontend (JS/React/SCSS)
```
Inside container: /openedx/edx-platform/
```

Key frontend folders:
```
/openedx/edx-platform/
├── lms/static/         ← LMS static assets (legacy Backbone/jQuery JS)
├── cms/static/         ← Studio static assets
├── themes/             ← Theme SCSS/HTML overrides
├── node_modules/       ← npm packages
├── webpack.*.config.js ← Webpack build config
└── package.json        ← npm dependencies
```

The **MFE (React apps)** — login page, learner dashboard — are a separate image:
```
Container: tutor_dev-mfe-1 (image: openedx-mfe:21.0.0-indigo)
Source:    https://github.com/openedx/frontend-app-authn   (login)
           https://github.com/openedx/frontend-app-learner-dashboard
```

---

## How to Explore or Edit It

```bash
# Open a shell inside the LMS container
tutor dev run lms bash

# Then navigate the source
ls /openedx/edx-platform/lms/djangoapps/
ls /openedx/edx-platform/openedx/core/djangoapps/user_authn/

# Or read a specific file directly
docker exec tutor_dev-lms-1 cat /openedx/edx-platform/lms/djangoapps/courseware/views/views.py
```

---

## Your Custom Code Location

| What | Where |
|------|-------|
| Open edX platform (LMS + Studio) | Inside Docker image at `/openedx/edx-platform/` |
| MFE React apps (login, dashboard) | Inside MFE Docker image |
| Your custom XBlocks | `$TUTOR_ROOT/xblocks/` (local) → installed into containers |
| Platform settings (dev) | `$TUTOR_ROOT/env/apps/openedx/settings/` (local, bind-mounted) |
| Platform config | `$TUTOR_ROOT/env/apps/openedx/config/lms.env.yml` (local, bind-mounted) |

The settings files in `env/` **are** bind-mounted into the container — so they're the one part of the backend you can edit locally and have take effect after a restart.