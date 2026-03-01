# Canvas LTI Integration

**Phase:** 4 (Weeks 11–14)
**Status:** Planned
**Contact:** Dr. Jessica Hirshorn (CISA) — Canvas admin

---

## Overview

Open edX Ulmo is **LTI Advantage Complete certified**. This means it supports LTI 1.3 with all three LTI Advantage services:
- **Names and Roles Provisioning Service (NRPS)** — sync roster from Canvas
- **Assignment and Grade Services (AGS)** — send grades back to Canvas
- **Deep Linking** — embed Open edX content directly in Canvas pages

**Practical meaning:**
- Students launch ASU Global Connect from within Canvas (no separate login needed with CAS SSO)
- PhETSim and QuestEngine grades automatically appear in the Canvas gradebook
- Instructors create LTI-linked assignments in Canvas → students land in specific Open edX units

---

## Architecture

```
Canvas (LMS at ASU)
    │
    ├── Student clicks assignment link
    │         │
    │         └── LTI 1.3 launch request → ASU Global Connect
    │                       │
    │                       ├── Validate LTI JWT
    │                       ├── Log in / create student account
    │                       └── Redirect to course unit
    │
    └── PhETSim completion
              │
              └── Open edX sends grade to Canvas via LTI AGS
                            │
                            └── Grade appears in Canvas gradebook
```

---

## Setup Steps

### 1. Register Open edX as LTI Tool in Canvas

Dr. Hirshorn does this in Canvas Admin:
1. Admin → Developer Keys → LTI Key → Create
2. Set:
   - Redirect URIs: `https://openedx.asu.edu/lti/launch/`
   - JWKS URL: `https://openedx.asu.edu/lti/jwks/`
   - Public JWK URL for Open edX's signing key
3. Enable for the course

### 2. Configure in Open edX LTI Config

```python
# In LMS settings (via Tutor plugin)
LTI_TOOL_CONFIGURATION = {
    'title': 'ASU Global Connect',
    'description': 'Interactive Learning Platform',
    'launch_url': 'https://openedx.asu.edu/lti/',
    'embed_url': '',
    'embed_iframe': True,
    'prefer_embed': False,
    'icon_url': '',
    'course_navigation': {
        'enabled': True,
        'default': 'enabled',
        'text': 'ASU Global Connect',
        'visibility': 'public',
    },
}
```

### 3. Enable AGS in Gradeable XBlocks

PhETSim and QuestEngine XBlocks already publish `grade` events:
```python
self.runtime.publish(self, 'grade', {
    'value': self.score,
    'max_value': 1.0,
})
```

Open edX Ulmo automatically forwards these to Canvas via LTI AGS when the student was launched from an LTI assignment. No extra XBlock code needed.

---

## Grade Passback Flow

```
Student completes PhETSim XBlock
    │
    └── XBlock publishes 'grade' event
              │
              └── Open edX LTI AGS handler
                        │
                        └── POST /api/lti/v1/lti/{launch_id}/grading/
                                  │
                                  └── Canvas AGS endpoint
                                            │
                                            └── Grade updated in Canvas gradebook
```

---

## Testing LTI Integration

```bash
# Test LTI JWKS endpoint
curl https://openedx.asu.edu/lti/jwks/

# Check LTI launch is configured
tutor local run lms ./manage.py lms shell -c "
from lti_consumer.models import LtiConfiguration
print(LtiConfiguration.objects.count(), 'LTI configs')
"
```

---

## Student Experience

With both CAS SSO and Canvas LTI configured:
1. Student is in Canvas, clicks "ASU Global Connect" assignment
2. LTI launches Open edX, CAS SSO handles login silently
3. Student lands directly in the assigned unit (PhETSim, QuestEngine, etc.)
4. Upon completion, grade posts back to Canvas automatically
5. Student returns to Canvas — grade is already there
