# Development Tips

Practical tips for working efficiently in this environment.

---

## Debugging Python (XBlock Backend)

### Using ipdb in containers

The dev image has `ipdb` installed and `PYTHONBREAKPOINT=ipdb.set_trace` set:

```python
# In your XBlock handler:
def save_pin(self, data, suffix=''):
    breakpoint()  # drops into ipdb when this handler is called
    self.pin_data = data
    return {'success': True}
```

Then attach to the running container:
```bash
tutor dev dc attach lms
# Trigger the handler from the browser
# ipdb prompt will appear in this terminal
# Use: n (next), s (step), c (continue), p variable (print)
# Detach: Ctrl+P Ctrl+Q (without killing container)
```

### Logging in XBlocks

```python
import logging
log = logging.getLogger(__name__)

class MyXBlock(XBlock):
    def student_view(self, context=None):
        log.info("student_view called for user %s", self.runtime.user_id)
        log.debug("context: %s", context)
        # ...
```

View in real time:
```bash
tutor dev logs --follow lms
# or filter
tutor dev logs --follow lms 2>&1 | grep "my_xblock"
```

---

## Debugging JavaScript (XBlock Frontend)

### Browser DevTools

Open DevTools → Console while on the LMS page with your XBlock. JavaScript errors appear here.

### XBlock JS pattern for debugging

```javascript
function MyXBlock(runtime, element, initArgs) {
    console.log('MyXBlock init', initArgs);

    var handlerUrl = runtime.handlerUrl(element, 'save_data');

    $(element).find('.my-button').click(function() {
        console.log('Button clicked, calling handler:', handlerUrl);

        $.ajax({
            type: 'POST',
            url: handlerUrl,
            data: JSON.stringify({action: 'click'}),
            success: function(response) {
                console.log('Handler response:', response);
            },
            error: function(xhr, status, error) {
                console.error('Handler error:', error, xhr.responseText);
            }
        });
    });
}
```

---

## Hot Reload

### Python changes

Django dev server watches for file changes and auto-reloads. BUT: the LMS container needs the xblock files to be accessible. With a bind-mount (see below), changes to `.py` files reload automatically.

```bash
# Check if Django reloaded after your change
tutor dev logs --follow lms
# Look for: "Watching for file changes with StatReloader"
```

### JavaScript / CSS / HTML changes

Static files are served directly from the filesystem in dev mode. Changes to `static/js/`, `static/css/`, `static/html/` are immediately visible after a browser hard-refresh (Cmd+Shift+R). No container restart needed.

### Mounting XBlocks for hot reload

To make Python changes in your XBlock auto-reload without re-installing:

```bash
# Install in editable mode (-e flag = symlink install)
tutor dev run lms pip install -e /openedx/xblocks/my-xblock
```

Now edits to `.py` files inside the container (or via bind-mount) trigger auto-reload.

---

## Bind-Mounting XBlocks

To get your local `xblocks/` directory visible inside the containers, add a bind mount. The cleanest way in Tutor is via a plugin or a Docker Compose override.

**Quick ad-hoc method:**
```bash
# Copy XBlock into running container
docker cp $TUTOR_ROOT/xblocks/my-xblock/ tutor_dev-lms-1:/openedx/xblocks/

# Install it
tutor dev exec lms pip install -e /openedx/xblocks/my-xblock
```

**Note:** After container restart, you'd need to re-copy. For persistent mounts, use a Tutor plugin that adds bind mounts to docker-compose.

---

## Useful Django Management Commands

```bash
# Open Django shell
tutor dev run lms ./manage.py lms shell

# List all registered XBlocks
tutor dev run lms ./manage.py lms shell -c "
import pkg_resources
for ep in pkg_resources.iter_entry_points('xblock.v1'):
    print(ep.name, '->', ep.dist)
"

# Run database migrations
tutor dev run lms ./manage.py lms migrate
tutor dev run cms ./manage.py cms migrate

# Show migration status
tutor dev run lms ./manage.py lms showmigrations

# Create a test course (via import)
tutor dev do importdemocourse

# Clear cache
tutor dev run lms ./manage.py lms cache --clear

# Check for Django config errors
tutor dev run lms ./manage.py lms check
```

---

## Checking XBlock Installation

```bash
tutor dev run lms python -c "
import pkg_resources
xblocks = list(pkg_resources.iter_entry_points('xblock.v1'))
print(f'{len(xblocks)} XBlocks registered:')
for ep in xblocks:
    print(f'  {ep.name}')
"
```

---

## Database Shortcuts

```bash
# MySQL — useful queries
tutor dev do sqlshell
# Inside MySQL:
USE openedx;
SHOW TABLES;
SELECT username, email, is_superuser FROM auth_user;
SELECT * FROM student_courseenrollment LIMIT 10;

# MongoDB — inspect course content
docker exec -it tutor_dev-mongodb-1 mongosh openedx
# Inside MongoDB:
db.getCollectionNames()
db["modulestore.split_modulestore.active_versions"].find().limit(3).pretty()

# Redis — inspect keys
docker exec -it tutor_dev-redis-1 redis-cli
# Inside Redis:
KEYS *
INFO keyspace
```

---

## Checking LMS Feature Flags

```bash
tutor dev run lms ./manage.py lms shell -c "
from django.conf import settings
print('ENABLE_OAUTH2_PROVIDER:', settings.FEATURES.get('ENABLE_OAUTH2_PROVIDER'))
print('ENABLE_COMBINED_LOGIN_REGISTRATION:', settings.FEATURES.get('ENABLE_COMBINED_LOGIN_REGISTRATION'))
# Print all features
import json
print(json.dumps(settings.FEATURES, indent=2, default=str))
"
```

---

## Checking Current Config

```bash
# Print a specific config value
tutor config printvalue LMS_HOST
tutor config printvalue PLATFORM_NAME
tutor config printvalue MYSQL_ROOT_PASSWORD

# View full config
cat $TUTOR_ROOT/config.yml
```

---

## Performance: Speeding Up Dev

1. **Don't follow all logs** — `tutor dev logs --follow lms` only, not all services
2. **Use hard refresh** for static file changes: `Cmd+Shift+R`
3. **Django shell** is faster than browser testing for data verification
4. **pytest** is much faster than full browser integration tests for XBlock logic
