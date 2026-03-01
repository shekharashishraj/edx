# VirtualCampus XBlock

**Package:** `virtualcampus-xblock`
**Phase:** 3 (Weeks 8–10)
**Status:** Not yet built

---

## Purpose

Embeds a Mozilla Hubs 3D social space inside the LMS. Students attend virtual office hours, study groups, and social events in a browser-based 3D environment. Attendance is tracked and rewarded with XP.

---

## Student View

- Embedded Hubs room (full iframe)
- Event schedule widget (upcoming events)
- Attendance timer (shows time spent in room)
- XP award notification when 15-minute threshold crossed

---

## Fields

```python
# Scope.settings (instructor)
hub_url = String(default='', scope=Scope.settings)  # Mozilla Hubs room URL
event_name = String(default='Virtual Office Hours', scope=Scope.settings)
min_attendance_minutes = Integer(default=15, scope=Scope.settings)
xp_reward = Integer(default=75, scope=Scope.settings)

# Scope.user_state (per-student)
attended = Boolean(default=False, scope=Scope.user_state)
total_minutes = Integer(default=0, scope=Scope.user_state)
attendance_xp_awarded = Boolean(default=False, scope=Scope.user_state)
```

---

## Handlers

| Handler | Called By | Action |
|---------|-----------|--------|
| `heartbeat` | Student JS (every 60s) | Track time in room, award XP at threshold |
| `get_attendance` | Student JS | Return attendance status and minutes |

---

## Attendance Tracking

The JS sends a heartbeat every 60 seconds while the student is in the Hubs room:

```javascript
// In student.js
var heartbeatInterval = null;
var heartbeatUrl = runtime.handlerUrl(element, 'heartbeat');

function startHeartbeat() {
    heartbeatInterval = setInterval(function() {
        fetch(heartbeatUrl, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRFToken': getCsrfToken(),
            },
            body: JSON.stringify({ minutes: 1 }),
        })
        .then(r => r.json())
        .then(data => {
            if (data.xp_awarded) {
                showXpNotification(data.xp);
            }
        });
    }, 60000);  // every 60 seconds
}

// Start when iframe loads
document.querySelector('.hubs-iframe').addEventListener('load', startHeartbeat);

// Stop when user leaves
window.addEventListener('beforeunload', function() {
    clearInterval(heartbeatInterval);
});
```

---

## postMessage Events from Hubs

Mozilla Hubs can emit postMessage events (join, leave, user count). The XBlock can optionally use these:

```javascript
window.addEventListener('message', function(event) {
    if (!event.origin.includes('hubs.mozilla.com')) return;
    if (event.data.type === 'entered_room') {
        startHeartbeat();
    }
    if (event.data.type === 'exited_room') {
        clearInterval(heartbeatInterval);
    }
});
```

---

## XP Events Published

```python
# After min_attendance_minutes (one-time per event)
self.runtime.publish(self, 'xp_earned', {
    'xp': self.xp_reward,
    'action': 'attend_virtual_event',
    'metadata': {'event_name': self.event_name}
})
```

---

## Self-Hosted Hubs (Phase 4+)

For production, host a private Hubs instance:
```bash
# Hubs Cloud (AWS or GCP)
# https://hubs.mozilla.com/cloud
# Private rooms with ASU SSO login
```
