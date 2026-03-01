# QuestEngine XBlock

**Package:** `questengine-xblock`
**Phase:** 2 (Weeks 5–7)
**Status:** Not yet built

---

## Purpose

Branching narrative quest system. Students follow choose-your-own-adventure style quests tied to course content. Each choice has consequences; completing quests earns XP. Drives engagement and exploration of course material.

---

## Student View

- Quest narrative text with choice buttons
- Progress indicator (step X of Y)
- XP earned indicator per completed step
- Quest history (completed quests)
- Active quest badge

---

## Quest Definition Format (YAML)

Quests are authored in YAML files stored as XBlock content:

```yaml
# quest-001-global-explorer.yml
id: global_explorer
title: "The Global Explorer"
description: "Discover the world of ASU Global Connect"
total_xp: 300

steps:
  - id: start
    text: |
      Welcome to ASU Global Connect! You've just joined thousands of
      students from around the world. What would you like to do first?
    choices:
      - text: "Explore the world map"
        next_step: explore_map
        xp: 25
      - text: "Learn about the community"
        next_step: learn_community
        xp: 25

  - id: explore_map
    text: |
      Great choice! The GlobalMap shows where all your peers are located.
      You can drop a pin at your location to let others find you.
    choices:
      - text: "Drop my pin now"
        next_step: pin_dropped
        xp: 50
        trigger_action: drop_map_pin   # links to GlobalMap XBlock action
      - text: "Maybe later"
        next_step: skip_pin
        xp: 10

  - id: pin_dropped
    text: "You've marked your location! Other students can now find you."
    choices:
      - text: "Continue the quest"
        next_step: final
        xp: 75
    is_terminal: false

  - id: final
    text: "Congratulations! You've completed the Global Explorer quest."
    is_terminal: true
    completion_xp: 150
```

---

## Fields

```python
# Scope.user_state (per-student)
active_quest_id = String(default='', scope=Scope.user_state)
current_step_id = String(default='', scope=Scope.user_state)
completed_quests = List(default=[], scope=Scope.user_state)
step_history = List(default=[], scope=Scope.user_state)  # breadcrumb
total_xp_from_quests = Integer(default=0, scope=Scope.user_state)

# Scope.content (author-defined)
quest_yaml = String(default='', scope=Scope.content)  # YAML quest definition
```

---

## Handlers

| Handler | Called By | Action |
|---------|-----------|--------|
| `start_quest` | Student JS | Initialize quest, set step to 'start' |
| `make_choice` | Student JS | Advance to next step, award step XP |
| `get_state` | Student JS | Return current step and history |
| `list_quests` | Student JS | Return all available quests |

---

## XP Events Published

```python
# Per step completion
self.runtime.publish(self, 'xp_earned', {
    'xp': step.xp,
    'action': 'quest_step_completed',
    'metadata': {'quest_id': quest.id, 'step_id': step.id}
})

# On quest completion
self.runtime.publish(self, 'xp_earned', {
    'xp': quest.completion_xp,
    'action': 'quest_completed',
    'metadata': {'quest_id': quest.id}
})
```

---

## Key Design Decisions

1. **YAML in content field** — Quest definitions are stored in the XBlock's content field (MongoDB), authored in Studio. No separate file system dependency.
2. **Stateless steps** — Steps are stateless descriptions. State (current step, history) is in `Scope.user_state`.
3. **Trigger actions** — Steps can trigger actions in other XBlocks (e.g., `trigger_action: drop_map_pin`). The XBlock JS watches for these and calls the appropriate XBlock.
