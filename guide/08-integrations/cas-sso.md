# ASU CAS SSO Integration

**Phase:** 4 (Weeks 11–14)
**Status:** Planned

Enables ASU students to log in with their ASU credentials (`@asu.edu`) via Arizona State University's Central Authentication Service (CAS).

---

## Overview

CAS (Central Authentication Service) is a standard enterprise SSO protocol. ASU uses CAS v3 at `https://weblogin.asu.edu/cas/`.

**Flow:**
```
Student visits ASU Global Connect
    │
    └── Redirected to https://weblogin.asu.edu/cas/login?service=<return_url>
              │
              └── Student enters ASU credentials (ASURITE ID + password)
                        │
                        └── CAS validates → redirects back with ticket
                                  │
                                  └── Open edX validates ticket with CAS server
                                            │
                                            └── User logged in / created
```

---

## Implementation

### 1. Install django-cas-ng

```bash
# In production via Tutor plugin
tutor local run lms pip install django-cas-ng
```

### 2. Create a Tutor Plugin for CAS Settings

Create `plugins/cas-plugin/plugin.py`:

```python
from tutor import hooks

@hooks.Filters.ENV_PATCHES.add()
def add_cas_settings(items):
    items.append(("lms-common-settings", """
INSTALLED_APPS += ['django_cas_ng']
AUTHENTICATION_BACKENDS = [
    'django_cas_ng.backends.CASBackend',
    'django.contrib.auth.backends.ModelBackend',
]
CAS_SERVER_URL = 'https://weblogin.asu.edu/cas/'
CAS_VERSION = '3'
CAS_REDIRECT_URL = '/'
CAS_STORE_NEXT = True
CAS_FORCE_CHANGE_USERNAME_CASE = 'lower'
MIDDLEWARE += ['django_cas_ng.middleware.CASMiddleware']
"""))
    return items
```

### 3. URL Configuration

Add CAS URLs (via Tutor plugin or Django URL conf):

```python
# In LMS URL config
from django.urls import path
import django_cas_ng.views

urlpatterns += [
    path('cas/login/', django_cas_ng.views.LoginView.as_view(), name='cas_ng_login'),
    path('cas/logout/', django_cas_ng.views.LogoutView.as_view(), name='cas_ng_logout'),
    path('cas/callback/', django_cas_ng.views.CallbackView.as_view(), name='cas_ng_proxy_callback'),
]
```

---

## User Mapping

When a student logs in via CAS for the first time, Open edX creates a new user. The mapping:

| CAS Attribute | Open edX Field |
|--------------|----------------|
| ASURITE ID (uid) | `username` |
| ASU email | `email` |
| Display name | `UserProfile.name` |

Configure attribute mapping in settings:
```python
CAS_APPLY_ATTRIBUTES_TO_USER = True
CAS_RENAME_ATTRIBUTES = {
    'uid': 'username',
    'mail': 'email',
    'displayName': 'name',
}
```

---

## Testing

```bash
# Test CAS redirect
curl -I "https://openedx.asu.edu/cas/login/?next=/dashboard"
# Should redirect to https://weblogin.asu.edu/cas/login?service=...

# Test with a mock CAS server (dev testing)
# Use https://github.com/nicowillis/cas-test-server for dev testing
```

---

## Rollout

1. Test with 5 pilot students using their real ASU credentials
2. Verify user accounts created correctly (username = ASURITE ID)
3. Keep local password login as fallback during transition
4. Full rollout to 500 students at pilot launch
