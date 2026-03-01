# Troubleshooting

Common problems and their fixes.

---

## MySQL Won't Start

**Symptom:** `tutor dev dc ps` shows MySQL as `Restarting (1)`

**Cause:** MySQL was killed mid-operation (Docker force-stopped, machine crashed). InnoDB redo logs are inconsistent.

**Logs show:**
```
[InnoDB] Cannot create redo log files because data files are corrupt
or the database was not shut down cleanly after creating the data files.
```

**Fix:**
```bash
tutor dev stop
rm -rf $TUTOR_ROOT/data/mysql
tutor dev do init
tutor dev start
# Re-create admin user if needed (see installation.md)
```

---

## "Too Many Failed Login Attempts"

**Cause:** LMS rate-limits repeated login failures. Stored in-process memory (not Redis or DB).

**Fix:** Restart the LMS container to clear in-memory state:
```bash
tutor dev restart lms
```

---

## "An Error Has Occurred. Try Refreshing the Page"

**Cause:** Login API returns 500 — usually a missing `UserProfile` for the user.

**Logs show:**
```
django.contrib.auth.models.User.profile.RelatedObjectDoesNotExist: User has no profile.
```

**Fix:** Create the missing UserProfile:
```bash
tutor dev run lms ./manage.py lms shell -c "
from django.contrib.auth import get_user_model
from common.djangoapps.student.models import UserProfile
User = get_user_model()
user = User.objects.get(username='admin')
UserProfile.objects.get_or_create(user=user, defaults={'name': 'Admin'})
print('Done')
"
```

---

## `tutor: command not found`

**Cause:** PATH doesn't include Tutor's install location.

**Fix:**
```bash
export PATH="$PATH:/Users/ashishrajshekhar/Library/Python/3.10/bin"
```

Make permanent by adding to `~/.zshrc`.

---

## Tutor Acts on Wrong Project

**Cause:** `TUTOR_ROOT` not set, so Tutor defaults to `~/.local/share/tutor`.

**Fix:**
```bash
export TUTOR_ROOT="/Users/ashishrajshekhar/Desktop/2026/edx"
```

Verify: `tutor config printvalue LMS_HOST` should print `local.openedx.io`.

---

## LMS Takes Very Long to Respond

**Cause:** Dev server is slow on first request after restart (Django startup + module loading).

**Normal behaviour:** First request after `tutor dev start` can take 30–60 seconds. Subsequent requests are fast.

**Check logs:**
```bash
tutor dev logs --follow lms
```
Wait until you see `Watching for file changes with StatReloader` before opening the browser.

---

## XBlock Not Appearing in Studio

**Cause:** XBlock not installed, not in Advanced Module List, or container not restarted.

**Fix:**
```bash
# Install into both containers
tutor dev run lms pip install -e /openedx/xblocks/my-xblock
tutor dev run cms pip install -e /openedx/xblocks/my-xblock

# Restart to pick up new entry points
tutor dev restart lms cms
```

Then in Studio: Course → Advanced Settings → Advanced Module List → add `"my_xblock"`.

---

## XBlock Handler Returns 403

**Cause:** CSRF token missing or session expired.

**Fix in JS:** Always include the CSRF token in AJAX requests:
```javascript
function getCsrfToken() {
    return document.cookie.split('; ')
        .find(row => row.startsWith('csrftoken='))
        ?.split('=')[1];
}

fetch(handlerUrl, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCsrfToken(),
    },
    body: JSON.stringify(data),
});
```

---

## Containers Keep Restarting

**Check logs for specific service:**
```bash
tutor dev logs --follow mysql      # MySQL issues
tutor dev logs --follow lms        # LMS startup errors
tutor dev logs --follow mongodb    # MongoDB issues
```

**Nuclear option — full restart:**
```bash
tutor dev stop
docker system prune -f             # Remove stopped containers
tutor dev start
```

---

## Port Already in Use

**Symptom:** `bind: address already in use` on port 8000 or 8001

**Find what's using it:**
```bash
lsof -i :8000
lsof -i :8001
```

**Kill it:**
```bash
kill -9 <PID>
```

---

## Checking Service Health

Quick health check for all services:
```bash
# Container status
tutor dev dc ps

# LMS heartbeat
curl http://local.openedx.io:8000/heartbeat

# MySQL (from inside LMS container)
tutor dev run lms ./manage.py lms dbshell --command="SELECT 1;"

# Redis
docker exec tutor_dev-redis-1 redis-cli PING  # → PONG

# MongoDB
docker exec tutor_dev-mongodb-1 mongosh --eval "db.runCommand({ping:1})"
```

---

## Viewing Logs

```bash
# All services (real-time)
tutor dev logs --follow

# Specific service
tutor dev logs --follow lms
tutor dev logs --follow cms
tutor dev logs --follow mysql

# Last N lines
tutor dev logs --tail=50 lms
```
