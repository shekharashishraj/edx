# Event System

How XBlocks communicate with each other through the Open edX event bus.

---

## Overview

XBlocks in Open edX communicate via a publish-subscribe event system built into the XBlock runtime. This is how all 6 custom XBlocks coordinate without directly importing each other.

---

## Publishing an Event

Any XBlock can publish an event:

```python
self.runtime.publish(self, event_type, event_data)
```

| Argument | Type | Description |
|----------|------|-------------|
| `self` | XBlock | The publishing XBlock (used to identify source) |
| `event_type` | str | Event name string, e.g. `'xp_earned'`, `'grade'` |
| `event_data` | dict | Arbitrary JSON-serializable data |

---

## Standard Events in This Project

### `xp_earned` — XP Award
Published by every XBlock when a student earns XP.

```python
self.runtime.publish(self, 'xp_earned', {
    'xp': 100,                          # XP amount (int)
    'action': 'drop_map_pin',           # action identifier (str)
    'user_id': self.runtime.user_id,    # student's user ID (int)
    'course_id': str(self.course_id),   # course identifier (str)
    'metadata': {                        # optional extra data (dict)
        'location': 'Tempe, AZ',
    }
})
```

### `grade` — Grade Submission
Published by gradeable XBlocks (PhETSim, QuestEngine) to report a grade to Open edX.

```python
self.runtime.publish(self, 'grade', {
    'value': 0.85,      # student's score (float 0.0–1.0)
    'max_value': 1.0,   # max possible score (always 1.0)
})
```

This automatically flows into the LMS gradebook and (in Phase 4) to Canvas via LTI AGS.

### `completion` — Activity Completion
Marks an XBlock as "completed" for the course completion tracker.

```python
self.runtime.publish(self, 'completion', {
    'completion': 1.0,  # 1.0 = fully completed
})
```

---

## Listening for Events (Gamification XBlock)

The Gamification XBlock registers as a listener for `xp_earned` events:

```python
class GamificationXBlock(XBlock):

    def student_view(self, context=None):
        # Register event handlers in the runtime
        # This varies by Open edX version — in Ulmo use XBlockAside or service
        ...

    @XBlock.json_handler
    def handle_xp_event(self, data, suffix=''):
        """
        This handler is called by other XBlocks' published 'xp_earned' events.
        In Open edX, this requires registering as an aside or via a service.
        """
        xp = data.get('xp', 0)
        action = data.get('action', '')
        user_id = data.get('user_id', self.runtime.user_id)

        self._award_xp(user_id, xp, action)
        return {'received': True}
```

---

## XP Event Flow Diagram

```
GlobalMapXBlock.save_pin()
    │
    └── runtime.publish('xp_earned', {xp:100, action:'drop_map_pin'})
                │
         Open edX Event Bus
                │
    ┌───────────▼────────────────────┐
    │  GamificationXBlock            │
    │  handle_xp_event()             │
    │    ├── MySQL: add 100 to total │
    │    ├── MySQL: check level up   │
    │    ├── MySQL: check badges     │
    │    └── Redis: update sorted set│
    └────────────────────────────────┘
```

---

## Event Deduplication

Some events should only fire once per student (e.g., "drop first pin"). Use the audit log to deduplicate:

```python
def _award_xp(self, user_id, xp, action):
    ONE_TIME_ACTIONS = {
        'drop_map_pin',
        'complete_profile',
        'attend_virtual_event',
    }

    if action in ONE_TIME_ACTIONS:
        # Check if this user already earned XP for this action in this course
        already_earned = XPRecord.objects.filter(
            user_id=user_id,
            action=action,
            course_id=self.course_id,
        ).exists()

        if already_earned:
            return  # Don't double-award

    # Create audit record
    XPRecord.objects.create(
        user_id=user_id,
        action=action,
        xp=xp,
        course_id=self.course_id,
    )

    # Update total
    user_xp, _ = UserXP.objects.get_or_create(
        user_id=user_id, course_id=self.course_id,
        defaults={'total_xp': 0, 'level': 1}
    )
    user_xp.total_xp += xp
    user_xp.level = self._calculate_level(user_xp.total_xp)
    user_xp.save()

    # Update Redis leaderboard
    redis_client.zadd(
        f'xp:leaderboard:{self.course_id}',
        {f'user:{user_id}': user_xp.total_xp}
    )
```

---

## Open edX Built-in Events

Open edX also publishes its own events that XBlocks can listen to (via `openedx-events` library):

| Event | When |
|-------|------|
| `STUDENT_REGISTRATION_COMPLETED` | New user registers |
| `COURSE_ENROLLMENT_CREATED` | Student enrolls in course |
| `COURSE_GRADE_CHANGED` | Grade recalculated |
| `SESSION_LOGIN_COMPLETED` | User logs in |

These use a different system (`openedx_events.tooling`) and are available in Open edX Ulmo. XBlocks can subscribe to these for advanced integrations (e.g., award a "Pioneer" badge to the first 50 enrollments).
