# PhETSim XBlock

**Package:** `phetsim-xblock`
**Phase:** 2 (Weeks 5–7)
**Status:** Not yet built

---

## Purpose

Embeds University of Colorado PhET Interactive Simulations (HTML5) inside Open edX units. Grades student performance via the postMessage API. Connects to Canvas gradebook via LTI (Phase 4).

---

## Student View

- Fullscreen iframe containing the PhET simulation
- Grade indicator (shows score after submission)
- Submit button (captures current sim state)
- Instructions panel (instructor-authored)

---

## Fields

```python
# Scope.settings (instructor)
sim_url = String(
    default='https://phet.colorado.edu/sims/html/wave-on-a-string/latest/wave-on-a-string_en.html',
    scope=Scope.settings
)
sim_name = String(default='Wave on a String', scope=Scope.settings)
grading_mode = String(default='completion', scope=Scope.settings)  # 'completion' | 'score'
min_interaction_seconds = Integer(default=60, scope=Scope.settings)  # time-on-task requirement
xp_reward = Integer(default=100, scope=Scope.settings)

# Scope.user_state (per-student)
grade = Float(default=None, scope=Scope.user_state)
time_spent = Integer(default=0, scope=Scope.user_state)   # seconds
submitted = Boolean(default=False, scope=Scope.user_state)
sim_state = Dict(default={}, scope=Scope.user_state)      # last known state from postMessage
```

---

## Handlers

| Handler | Called By | Action |
|---------|-----------|--------|
| `receive_state` | Student JS (postMessage relay) | Store sim state, track time |
| `submit` | Student JS | Finalize grade, publish XP |
| `get_grade` | Student JS | Return current grade |

---

## postMessage Protocol

PhET sims send messages to the parent window. The XBlock JS listens and relays to the handler:

```javascript
// In student.js
window.addEventListener('message', function(event) {
    // Only accept messages from the PhET iframe
    if (!event.origin.includes('phet.colorado.edu')) return;

    var data = event.data;
    console.log('PhET message:', data);

    // Relay to XBlock handler
    fetch(runtime.handlerUrl(element, 'receive_state'), {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRFToken': getCsrfToken(),
        },
        body: JSON.stringify({
            sim_state: data,
            timestamp: Date.now(),
        }),
    });
});
```

---

## Grading Modes

| Mode | Logic |
|------|-------|
| `completion` | Grade = 1.0 if student spent ≥ `min_interaction_seconds` in sim |
| `score` | Grade = score reported by sim via postMessage (if sim supports it) |

Most PhET sims don't natively report a score — use `completion` mode as default.

---

## XP Events Published

```python
# On successful submission
self.runtime.publish(self, 'xp_earned', {
    'xp': self.xp_reward,       # 50-200 XP depending on sim difficulty
    'action': 'simulation_passed',
    'metadata': {'sim_name': self.sim_name}
})

# Grade reported to Open edX gradebook
self.runtime.publish(self, 'grade', {
    'value': self.grade,
    'max_value': 1.0,
})
```

---

## LTI Integration (Phase 4)

When Canvas LTI is enabled, grades flow back automatically:
- Student completes PhETSim XBlock → grade stored in MySQL
- Open edX LTI AGS (Assignment and Grade Services) → sends grade to Canvas
- Grade appears in Canvas gradebook

No extra code needed in the XBlock — Open edX handles LTI AGS automatically when grade events are published.

---

## Security Notes

- Iframe `sandbox` attribute must allow `allow-scripts allow-same-origin` for PhET sims to function
- CORS: PhET sims are served from `phet.colorado.edu`. postMessage origin is validated before processing.
