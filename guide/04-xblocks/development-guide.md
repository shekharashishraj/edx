# XBlock Development Guide

Step-by-step: how to build a new XBlock from scratch.

---

## Step 1 — Scaffold with cookiecutter

```bash
cd $TUTOR_ROOT/xblocks/

pip3 install cookiecutter
cookiecutter https://github.com/openedx/cookiecutter-xblock
```

When prompted:
```
author_name: ASU CoRAL Lab
author_email: ashish@asu.edu
github_username: asu-coral-lab
project_name: GlobalMap XBlock
project_short_description: Mapbox GL student map with peer discovery
repo_name: globalmap-xblock
package_name: globalmap_xblock
xblock_class_name: GlobalMapXBlock
```

This creates:
```
globalmap-xblock/
├── setup.py
├── globalmap_xblock/
│   ├── globalmap_xblock.py
│   ├── static/html/student.html
│   ├── static/js/student.js
│   ├── static/css/style.css
│   └── tests/test_xblock.py
└── README.md
```

---

## Step 2 — Implement the XBlock

Edit `globalmap_xblock/globalmap_xblock.py`. Follow the pattern in [architecture.md](architecture.md).

Key decisions for each XBlock:
- What fields does it need? (`Scope.user_state` vs `Scope.settings`)
- What does `student_view()` render?
- What handlers does JS need to call?
- What XP events does it publish?

---

## Step 3 — Install into Dev Containers

```bash
# Copy into running container (or use bind mount)
docker cp $TUTOR_ROOT/xblocks/globalmap-xblock/ tutor_dev-lms-1:/openedx/xblocks/
docker cp $TUTOR_ROOT/xblocks/globalmap-xblock/ tutor_dev-cms-1:/openedx/xblocks/

# Install in editable mode
tutor dev exec lms pip install -e /openedx/xblocks/globalmap-xblock
tutor dev exec cms pip install -e /openedx/xblocks/globalmap-xblock

# Restart to pick up new entry points
tutor dev restart lms cms
```

---

## Step 4 — Verify Installation

```bash
tutor dev run lms python -c "
import pkg_resources
eps = list(pkg_resources.iter_entry_points('xblock.v1'))
names = [ep.name for ep in eps]
print('globalmap' in names, '← should be True')
"
```

---

## Step 5 — Enable in Studio

1. Go to http://studio.local.openedx.io:8001
2. Open your course
3. Settings → Advanced Settings
4. Find "Advanced Module List"
5. Add `"globalmap"` (the entry point name from setup.py)
6. Save

---

## Step 6 — Add to a Unit

1. Studio → Course Outline → Section → Subsection → Unit
2. Click "Add Component" → "Advanced"
3. Select "GlobalMap XBlock"
4. Click "Edit" to configure

---

## Step 7 — Test in Browser

1. Studio → "Preview" button (top right)
2. Or: LMS → navigate to the unit

---

## Step 8 — Write Tests

```bash
cd $TUTOR_ROOT/xblocks/globalmap-xblock/

# Run tests
python -m pytest tests/ -v

# With coverage
python -m pytest tests/ --cov=globalmap_xblock --cov-report=html --cov-report=term

# Check coverage report
open htmlcov/index.html
```

Target: **80%+ coverage**.

---

## Step 9 — Iterate

For Python changes:
```bash
tutor dev restart lms cms
# Django auto-reloads on .py file changes when using editable install
```

For JS/CSS/HTML changes:
- Hard refresh browser: `Cmd+Shift+R`
- No container restart needed

---

## Common Pitfalls

### Missing UserProfile error on login
If you create users directly with `create_superuser`, they won't have a UserProfile. Always create it:
```python
UserProfile.objects.create(user=user, name='Display Name')
```

### Handler returns 500
Check LMS logs: `tutor dev logs --follow lms`
Most common: Python exception in the handler. The full traceback is in logs.

### XBlock doesn't appear in Studio
Check it's in the Advanced Module List AND that `tutor dev restart lms cms` was run after installing.

### Static files not updating
Hard refresh: `Cmd+Shift+R`. Static files are served directly; no restart needed.

### CSRF error (403) in AJAX
Add the CSRF token to fetch/$.ajax calls:
```javascript
headers: { 'X-CSRFToken': document.cookie.match(/csrftoken=([^;]+)/)?.[1] }
```

---

## XBlock Workbench (Optional Local Testing)

For testing XBlocks outside of Open edX:
```bash
pip install xblock-sdk
xblock-workbench
# Open http://localhost:8000
# Your XBlock appears if workbench_scenarios() is defined
```
