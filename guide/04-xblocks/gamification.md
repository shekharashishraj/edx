# Gamification XBlock

**Package:** `gamification-xblock`
**Phase:** 1 (Weeks 2–4)
**Status:** Not yet built

---

## Purpose

Central XP/badge/leaderboard engine. Every other XBlock publishes XP events to this one. Gamification listens, tracks, and displays progress.

---

## Student View

Shows:
- Current XP total and level progress bar
- Badge showcase (earned badges)
- Leaderboard (top students in the course)
- Weekly challenge widget

---

## Fields

```python
# Scope.user_state (per-student)
total_xp = Integer(default=0, scope=Scope.user_state)
level = Integer(default=1, scope=Scope.user_state)
badges = List(default=[], scope=Scope.user_state)       # list of badge slugs
xp_history = List(default=[], scope=Scope.user_state)   # last 10 events

# Scope.settings (instructor-configurable)
show_leaderboard = Boolean(default=True, scope=Scope.settings)
leaderboard_size = Integer(default=10, scope=Scope.settings)
```

---

## Handlers

| Handler | Called By | Action |
|---------|-----------|--------|
| `handle_xp_event` | Other XBlocks via publish | Record XP, update level, check badges |
| `get_leaderboard` | Student JS | Return top-N students |
| `get_my_stats` | Student JS | Return current student's XP, level, badges |

---

## Event Listener

```python
@XBlock.json_handler
def handle_xp_event(self, data, suffix=''):
    """Receives xp_earned events from other XBlocks."""
    xp = data.get('xp', 0)
    action = data.get('action', '')
    user_id = data.get('user_id', self.runtime.user_id)

    # Deduplicate one-time actions
    # Update MySQL: gamification_userxp
    # Update Redis: ZADD leaderboard
    # Check badge thresholds
    # Return updated stats
```

---

## Tech Stack

- **Redis sorted sets** — leaderboard storage (O(log N) updates)
- **MySQL** — persistent XP records, badge awards, user totals
- **Celery** — async badge email notifications (optional)

---

## Key Design Decisions

1. **Redis for leaderboard, MySQL for truth** — Redis is fast for ranked queries; MySQL is the source of truth. If Redis is flushed, rebuild from MySQL.
2. **Event-driven** — Gamification never polls other XBlocks. It only reacts to published events.
3. **Idempotent handlers** — Duplicate events (network retries) should not double-award XP.
