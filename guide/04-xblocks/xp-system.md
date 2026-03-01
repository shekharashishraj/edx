# XP / Badge / Leaderboard System

The Gamification XBlock is the central hub for all engagement mechanics in ASU Global Connect.

---

## XP Rewards Table

| Action | XP | Source XBlock | Trigger |
|--------|-----|--------------|---------|
| Drop map pin | 100 | GlobalMap | First pin placed |
| Complete profile | 50 | GlobalMap | Bio + location filled |
| Connection accepted | 25 | GlobalMap | Peer accepts connection |
| Quest step completed | 25–100 | QuestEngine | Per step (varies by difficulty) |
| Quest completed | 150 | QuestEngine | Final step of any quest |
| Simulation passed | 50–200 | PhETSim | Graded simulation completed |
| Attend virtual event | 75 | VirtualCampus | 15+ min in Hubs session |
| Win multiplayer challenge | 100 | Multiplayer | First place finish |
| Weekly challenge complete | 50 | Gamification | Weekly bonus quest |

---

## XP Levels

| Level | XP Required | Title |
|-------|-------------|-------|
| 1 | 0 | Newcomer |
| 2 | 250 | Explorer |
| 3 | 750 | Contributor |
| 4 | 1,500 | Collaborator |
| 5 | 3,000 | Leader |
| 6 | 6,000 | Champion |
| 7 | 10,000 | Legend |

---

## Badges

| Badge | Trigger | Icon |
|-------|---------|------|
| Globe Trotter | Drop 5 pins in 3+ continents | Globe |
| Mentor | Accept 10 peer connections | Star |
| Challenger | Win 3 multiplayer challenges | Shield |
| Cultural Ambassador | Connect with students from 5+ countries | Flag |
| Pioneer | First 50 students on platform | Rocket |
| Connector | 20+ accepted connections | Chain |
| Sim Master | Complete all PhETSim activities | Flask |
| Quest Champion | Complete all quests in a course | Crown |

---

## Event Flow

Every XBlock publishes XP events using the same protocol:

```python
# In any XBlock handler:
self.runtime.publish(self, 'xp_earned', {
    'xp': 100,
    'action': 'drop_map_pin',      # string identifier for this action
    'user_id': self.runtime.user_id,
    'course_id': str(self.scope_ids.usage_id.course_key),
    'metadata': {                   # optional extra data
        'location': 'Asia',
    }
})
```

The Gamification XBlock listens for `xp_earned` events and:
1. Adds XP to the student's total
2. Checks if new level reached → notifies student
3. Checks if any badge thresholds crossed → awards badge
4. Updates the Redis leaderboard sorted set
5. Logs the event to the XP audit table

---

## Leaderboard Implementation

Redis sorted sets are the data structure for leaderboards:

```python
# Gamification XBlock internal storage
import redis

r = redis.Redis(host='redis', port=6379)

# When a student earns XP:
r.zadd('xp:leaderboard:course:{course_id}', {str(user_id): total_xp})

# Get top 10:
top_10 = r.zrevrange('xp:leaderboard:course:{course_id}', 0, 9, withscores=True)

# Get a student's rank:
rank = r.zrevrank('xp:leaderboard:course:{course_id}', str(user_id))
```

Redis sorted sets give O(log N) updates and O(log N + K) range queries — fast enough for real-time leaderboard updates.

---

## Database Design (MySQL)

### xp_records table
```sql
CREATE TABLE gamification_xprecord (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL,           -- FK to auth_user.id
    course_id   VARCHAR(255) NOT NULL,
    action      VARCHAR(100) NOT NULL,  -- e.g. 'drop_map_pin'
    xp          INT NOT NULL,
    timestamp   DATETIME NOT NULL,
    metadata    JSON,                   -- extra action data
    INDEX (user_id),
    INDEX (course_id),
    INDEX (timestamp)
);
```

### user_xp table (materialized total)
```sql
CREATE TABLE gamification_userxp (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL UNIQUE,    -- FK to auth_user.id
    course_id   VARCHAR(255) NOT NULL,
    total_xp    INT NOT NULL DEFAULT 0,
    level       INT NOT NULL DEFAULT 1,
    UNIQUE KEY (user_id, course_id)
);
```

### badges table
```sql
CREATE TABLE gamification_badge (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL,
    badge_slug  VARCHAR(100) NOT NULL,  -- e.g. 'globe_trotter'
    awarded_at  DATETIME NOT NULL,
    course_id   VARCHAR(255) NOT NULL,
    UNIQUE KEY (user_id, badge_slug, course_id)
);
```

---

## Deduplication

Each (user_id, action, course_id) combination is tracked to prevent XP farming:

```python
# In Gamification XBlock handler
from .models import XPRecord

@XBlock.json_handler
def handle_xp_event(self, data, suffix=''):
    action = data['action']
    user_id = data['user_id']

    # Actions marked as one-time (e.g. 'drop_map_pin')
    ONE_TIME_ACTIONS = {'drop_map_pin', 'complete_profile'}

    if action in ONE_TIME_ACTIONS:
        already_earned = XPRecord.objects.filter(
            user_id=user_id,
            action=action,
            course_id=self.course_id
        ).exists()
        if already_earned:
            return {'skipped': True, 'reason': 'already_earned'}

    # Award XP
    XPRecord.objects.create(
        user_id=user_id,
        action=action,
        xp=data['xp'],
        course_id=self.course_id,
    )
    # Update total ...
```
