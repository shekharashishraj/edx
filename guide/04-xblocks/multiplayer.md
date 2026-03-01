# Multiplayer XBlock

**Package:** `multiplayer-xblock`
**Phase:** 3 (Weeks 8–10)
**Status:** Not yet built

---

## Purpose

Real-time collaborative challenge system. Students compete and collaborate in live quiz/puzzle sessions. Uses Socket.io for real-time bidirectional communication. Winners earn XP.

---

## Student View

- Challenge lobby (waiting room with current participants)
- Live challenge interface (questions appear in real-time)
- Live leaderboard (updates as answers come in)
- Results screen with XP awarded

---

## Architecture

The Multiplayer XBlock uses a **sidecar** pattern — a separate Socket.io server runs alongside the LMS:

```
Browser (student A)  ────┐
Browser (student B)  ────┤──→ Socket.io Server (:3000)
Browser (student C)  ────┘          │
                                    │  REST API
                              LMS Django (:8000)
                                    │
                                  MySQL (challenge state)
```

The Socket.io server is stateless — it brokers messages. Canonical state lives in MySQL.

---

## Fields

```python
# Scope.settings (instructor)
challenge_type = String(default='quiz', scope=Scope.settings)  # 'quiz' | 'puzzle' | 'debate'
challenge_data = Dict(default={}, scope=Scope.settings)  # questions, answers
max_players = Integer(default=20, scope=Scope.settings)
duration_seconds = Integer(default=300, scope=Scope.settings)  # 5 min default

# Scope.user_state (per-student)
sessions_played = Integer(default=0, scope=Scope.user_state)
sessions_won = Integer(default=0, scope=Scope.user_state)
total_xp_from_multiplayer = Integer(default=0, scope=Scope.user_state)
```

---

## Handlers

| Handler | Called By | Action |
|---------|-----------|--------|
| `get_socket_token` | Student JS | Return auth token for Socket.io connection |
| `join_challenge` | Student JS | Register in session, return session ID |
| `record_result` | Socket.io server (webhook) | Record win/loss, award XP |
| `get_history` | Student JS | Return player's challenge history |

---

## Socket.io Server (Sidecar)

The Socket.io server is a small Node.js process:

```javascript
// multiplayer-server/server.js
const io = require('socket.io')(3000);
const axios = require('axios');

io.on('connection', (socket) => {
    socket.on('join_room', async ({ session_id, token }) => {
        // Validate token via LMS API
        const valid = await validateToken(token);
        if (!valid) { socket.disconnect(); return; }

        socket.join(session_id);
        io.to(session_id).emit('player_joined', { count: roomSize(session_id) });
    });

    socket.on('submit_answer', ({ session_id, answer, timestamp }) => {
        // Broadcast to all in room
        io.to(session_id).emit('answer_submitted', { answer, timestamp });
        // Check if all answered → emit results
    });
});
```

---

## XP Events Published

```python
# Winner
self.runtime.publish(self, 'xp_earned', {
    'xp': 100,
    'action': 'win_multiplayer_challenge',
    'metadata': {'challenge_type': self.challenge_type}
})

# Participation (just for playing)
self.runtime.publish(self, 'xp_earned', {
    'xp': 20,
    'action': 'participate_multiplayer',
})
```

---

## Key Design Decisions

1. **Sidecar pattern** — Socket.io server is separate from LMS so WebSocket traffic doesn't block Django request handling
2. **Token auth** — LMS issues short-lived JWT tokens for Socket.io connections; server validates via LMS REST API
3. **Webhook for results** — Socket.io server calls LMS `/api/multiplayer/results/` webhook when a session ends; LMS records results and awards XP
4. **No persistent WebSocket state** — Socket.io server can restart without losing data since all state is in MySQL
