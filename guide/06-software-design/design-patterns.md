# Software Design Patterns

Patterns used consistently across ASU Global Connect.

---

## 1. XBlock Plugin Pattern

All 6 custom XBlocks follow the same structure, making them interchangeable and easy to onboard into.

**Pattern:** Python entry points for discovery, Fragment-based rendering, JSON handlers for AJAX, XBlock fields for storage.

```
XBlock Class
  ├── Fields (Scope-declared storage)
  ├── student_view() → Fragment
  ├── studio_view() → Fragment
  └── @json_handler methods
```

Every XBlock is independently installable, independently testable, and independently deployable. They don't import each other — they communicate only via the event bus.

---

## 2. Event-Driven XP Collection

XBlocks never update the leaderboard or badge system directly. They publish events; Gamification subscribes and reacts.

**Pattern:** Observer / Publish-Subscribe

```
Publisher (any XBlock)
  └── runtime.publish('xp_earned', {xp, action, user_id})
                │
         Event Bus (Open edX runtime)
                │
      Subscriber (Gamification XBlock)
        └── handle_xp_event()
              ├── Update MySQL totals
              ├── Update Redis leaderboard
              └── Check badge thresholds
```

**Why this matters:**
- Adding a new XBlock never requires modifying Gamification
- Gamification can be replaced/upgraded without touching other XBlocks
- Events can be logged, audited, and replayed

---

## 3. Scope-Based State Management

XBlock fields declare their storage scope at class definition time. The runtime handles persistence automatically — no manual DB reads/writes.

```python
# Reading state: just access the field
current_xp = self.total_xp       # runtime fetches from MongoDB

# Writing state: just assign
self.total_xp = self.total_xp + 100  # runtime persists automatically
```

**Why this matters:**
- No boilerplate DB code in business logic
- Scope mismatch bugs are caught at definition time, not runtime
- The platform handles multi-tenancy (same XBlock type, different student = different state)

---

## 4. Fragment-Based Rendering

XBlock views return `Fragment` objects, not full HTML pages. The LMS assembles fragments from multiple XBlocks into a single unit page.

```python
def student_view(self, context=None):
    frag = Fragment(self.resource_string("static/html/student.html"))
    frag.add_css(self.resource_string("static/css/style.css"))
    frag.add_javascript(self.resource_string("static/js/student.js"))
    frag.initialize_js('MyXBlock', {'data': 'for_js'})
    return frag
```

The LMS:
1. Collects fragments from all XBlocks in a unit
2. Injects them into the unit template
3. Calls each XBlock's JS initializer after page load

**Why:** Multiple XBlocks coexist on one page without stepping on each other's CSS/JS.

---

## 5. Sidecar Service Pattern (Multiplayer)

The Multiplayer XBlock needs WebSockets — something Django's synchronous request-response model doesn't handle well. Solution: a separate Socket.io server runs as a sidecar container.

```
LMS Django (sync HTTP)  ←→  Socket.io Server (async WS)
                                      ↕
                               Browser Clients
```

The LMS issues auth tokens; the sidecar validates them. Game state is authoritative in MySQL; Socket.io is just a message relay.

**Why:** Keeps Django clean (no Channels dependency), horizontally scalable separately.

---

## 6. postMessage Bridge Pattern (PhETSim, VirtualCampus)

For third-party iframes (PhET, Mozilla Hubs), direct JS calls are blocked by browser same-origin policy. Solution: postMessage as a bridge.

```
XBlock JS (parent frame)
  ├── Sets up window.addEventListener('message', ...)
  ├── Creates iframe (PhET sim / Hubs room)
  └── iframe sends postMessage events
        └── XBlock JS receives message
              └── Calls XBlock handler via AJAX
                    └── Django handler saves state / awards XP
```

**Why:** postMessage is the only browser-native way for cross-origin iframe communication without server-side proxying.

---

## 7. Single Source of Truth Pattern

`config.yml` is the single source of truth for all platform configuration. The `env/` directory is entirely generated from it.

```
config.yml
    └── tutor config save
              └── generates env/
                    ├── docker-compose.yml
                    ├── lms.env.yml
                    ├── settings/lms/development.py
                    └── ... (all config files)
```

**Never hand-edit files in `env/`** — they'll be overwritten. All changes go through `config.yml` or Tutor plugins.

---

## 8. Read-Heavy Caching Pattern

Course overview pages, user permission checks, and feature flags are cached in Redis with appropriate TTLs:

```python
from django.core.cache import cache

def get_course_overview(course_id):
    cache_key = f'course_overview:{course_id}'
    overview = cache.get(cache_key)
    if overview is None:
        overview = CourseOverview.objects.get(id=course_id)
        cache.set(cache_key, overview, timeout=3600)  # 1 hour TTL
    return overview
```

**Pattern:** Cache-aside (lazy loading). Read from cache; on miss, read from DB and populate cache.

---

## 9. Deduplication via Unique Constraints

One-time XP actions (drop first pin, complete profile) are deduplicated using database unique constraints rather than application-level checks:

```sql
-- Badge table has a unique constraint
UNIQUE KEY uk_user_badge_course (user_id, badge_slug, course_id)
```

```python
try:
    Badge.objects.create(user=user, slug='globe_trotter', course_id=course_id)
    # Success: first time earning this badge
except IntegrityError:
    pass  # Already has the badge — silently skip
```

**Why:** Application-level duplicate checks have race conditions. Database unique constraints are atomic.

---

## 10. Django ORM — Never Raw SQL

All database access goes through the Django ORM:

```python
# Good
enrollments = CourseEnrollment.objects.filter(user=user, is_active=True)

# Bad (never do this)
cursor.execute("SELECT * FROM student_courseenrollment WHERE user_id = %s", [user_id])
```

**Why:** Raw SQL bypasses Django's security (SQL injection protection), type coercion, and migration tracking. ORM queries are also testable with mocked databases.
