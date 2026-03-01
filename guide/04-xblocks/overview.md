# XBlocks — Overview

## What Is an XBlock?

An XBlock is Open edX's plugin system for course components. Everything students interact with inside a course — videos, quizzes, text, simulations — is an XBlock. Open edX ships with built-in XBlocks (VideoXBlock, ProblemXBlock, HTMLXBlock). This project adds 6 custom XBlocks on top.

Think of XBlocks as self-contained widgets that:
- Render their own HTML/CSS/JS
- Store their own student state (answers, progress, pins)
- Expose their own REST handlers (AJAX endpoints)
- Publish events to other XBlocks or the platform

---

## How Open edX Uses XBlocks

```
Course Structure (MongoDB)
  └── Section
        └── Subsection
              └── Unit
                    ├── VideoXBlock (built-in)
                    ├── ProblemXBlock (built-in)
                    └── GlobalMapXBlock  ← our custom XBlock
```

When a student loads a unit:
1. LMS fetches the unit's XBlock tree from MongoDB
2. LMS calls `student_view()` on each XBlock
3. Each XBlock returns an HTML fragment + CSS + JS
4. LMS assembles the page with all fragments
5. LMS includes `runtime.js` — a small JS library that lets XBlock JS talk back to the server

---

## The 6 Custom XBlocks in This Project

| # | XBlock | Package | Build Phase | Primary Role |
|---|--------|---------|-------------|-------------|
| 1 | **Gamification** | `gamification-xblock` | Phase 1 (Weeks 2-4) | Central XP engine, badges, leaderboard |
| 2 | **GlobalMap** | `globalmap-xblock` | Phase 1 (Weeks 2-4) | Interactive world map, peer discovery |
| 3 | **QuestEngine** | `questengine-xblock` | Phase 2 (Weeks 5-7) | Branching narrative quests |
| 4 | **PhETSim** | `phetsim-xblock` | Phase 2 (Weeks 5-7) | HTML5 simulation embeds with grading |
| 5 | **VirtualCampus** | `virtualcampus-xblock` | Phase 3 (Weeks 8-10) | 3D social space via Mozilla Hubs |
| 6 | **Multiplayer** | `multiplayer-xblock` | Phase 3 (Weeks 8-10) | Real-time collaborative challenges |

---

## XBlock Scopes (Where Data Lives)

XBlock fields use a `Scope` to declare where their data is stored:

| Scope | Who sees it | Where stored | Example use |
|-------|------------|--------------|-------------|
| `Scope.user_state` | Per-student, per-XBlock instance | MongoDB | Student's map pin, quiz score |
| `Scope.user_state_summary` | Aggregated across students | MongoDB | Class average score |
| `Scope.preferences` | Per-student, across all instances of this XBlock type | MongoDB | Student's preferred language |
| `Scope.settings` | Instructor-configured, shared for all students | MongoDB | XBlock title, configuration |
| `Scope.content` | Author-defined course content | MongoDB | Question text, correct answer |
| `Scope.user_info` | Per-student global info | MySQL | Username (read-only) |

---

## XBlock Views

Every XBlock defines views — methods that return rendered HTML fragments:

| View | Called By | Purpose |
|------|-----------|---------|
| `student_view()` | LMS | What students see |
| `studio_view()` | Studio | Instructor edit UI |
| `author_view()` | Studio preview | Read-only author preview |
| `public_view()` | LMS (unauthenticated) | Preview without login |

---

## XBlock Handlers

Handlers are server-side methods callable from JavaScript via AJAX:

```python
@XBlock.json_handler
def save_pin(self, data, suffix=''):
    # data = parsed JSON from JS
    self.pin_location = data.get('lat_lng')
    return {'success': True, 'xp': 100}
```

Called from JS:
```javascript
var url = runtime.handlerUrl(element, 'save_pin');
fetch(url, { method: 'POST', body: JSON.stringify({lat_lng: [33.4, -111.9]}) });
```

---

## XBlock Entry Points

XBlocks are registered with Python's entry point system. In `setup.py`:

```python
entry_points={
    'xblock.v1': [
        'globalmap = globalmap_xblock.globalmap:GlobalMapXBlock',
    ]
}
```

After installing (`pip install -e .`), Open edX discovers the XBlock via this entry point.

---

## Event Publishing

XBlocks communicate with each other (especially to Gamification) via events:

```python
# Any XBlock → publish XP event
self.runtime.publish(self, 'xp_earned', {
    'xp': 100,
    'action': 'drop_map_pin',
    'user_id': self.runtime.user_id,
})
```

The Gamification XBlock listens for `xp_earned` events and updates the student's XP total.

---

## XBlock Directory Structure

```
xblocks/
├── gamification-xblock/      ← Phase 1
├── globalmap-xblock/         ← Phase 1
├── questengine-xblock/       ← Phase 2
├── phetsim-xblock/           ← Phase 2
├── virtualcampus-xblock/     ← Phase 3
└── multiplayer-xblock/       ← Phase 3
```

Each XBlock follows the same internal structure — see [architecture.md](architecture.md).
