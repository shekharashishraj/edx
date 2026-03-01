# Redis — Cache, Queue & Leaderboards

Redis is the fastest component in the stack. It's used for session caching, Celery task brokering, rate limiting, and (in Phase 1) XP leaderboards.

---

## Access

```bash
docker exec -it tutor_dev-redis-1 redis-cli
```

```bash
# Basic commands
PING                    # → PONG
KEYS *                  # list all keys
INFO keyspace           # summary of DB usage
DBSIZE                  # total number of keys
```

---

## What Open edX Stores in Redis

### Sessions

Django stores HTTP session data in Redis:

```
Key format:   django_session:<session_key>
Value:        pickled Python dict (session data)
TTL:          configured SESSION_COOKIE_AGE (default: 2 weeks)
```

```bash
# In redis-cli
KEYS django_session:*
TTL django_session:<key>
```

### Celery Task Queue

Celery uses Redis as its message broker. When LMS code does `my_task.delay(args)`, the task payload is written to Redis:

```
Key:          celery             (default queue)
Type:         List (LPUSH/BRPOP pattern)
```

Active queues in Open edX:
- `default` — general tasks
- `lms.bulk_email` — bulk email sending
- `edx.core.answer_problem` — grade recalculation
- `edx.certificate` — certificate generation

```bash
# In redis-cli
LLEN celery             # number of pending tasks in default queue
LRANGE celery 0 9       # peek at first 10 tasks
```

### Rate Limiting

Django's rate limiting (for login attempts, API calls) stores counters in Redis:

```
Key format:   rl:<endpoint>:<user_or_ip>
Type:         String (counter) with TTL
```

```bash
KEYS rl:*               # see all rate limit keys
DEL rl:*                # clear all rate limits (emergency fix)
```

### Cache Keys (General)

Django's cache framework uses Redis for general-purpose caching:

```
Key format:   :1:<cache_key>
Value:        pickled Python object
TTL:          varies (seconds to hours)
```

Common cached items:
- Course overviews (catalog page)
- User permissions
- Feature flags
- Meilisearch query results

```bash
KEYS :1:*               # see all general cache keys
```

---

## Custom Usage: Leaderboards (Phase 1)

The Gamification XBlock will use Redis sorted sets for real-time leaderboards:

### Sorted Set Design

```bash
# Data structure: ZADD key score member
# score = XP total, member = user_id

# Add/update a student's XP
ZADD xp:leaderboard:course-v1:ASU+CS101+2026 350 "user:42"

# Get top 10 (highest XP first)
ZREVRANGE xp:leaderboard:course-v1:ASU+CS101+2026 0 9 WITHSCORES

# Get a student's rank (0-indexed from top)
ZREVRANK xp:leaderboard:course-v1:ASU+CS101+2026 "user:42"

# Get a student's score
ZSCORE xp:leaderboard:course-v1:ASU+CS101+2026 "user:42"

# Get students within a score range
ZRANGEBYSCORE xp:leaderboard:course-v1:ASU+CS101+2026 500 1000 WITHSCORES
```

### In Python (XBlock)

```python
import redis

r = redis.Redis(host='redis', port=6379, decode_responses=True)

LEADERBOARD_KEY = f'xp:leaderboard:{course_id}'

def update_leaderboard(user_id, new_total_xp):
    r.zadd(LEADERBOARD_KEY, {f'user:{user_id}': new_total_xp})

def get_top_n(n=10):
    return r.zrevrange(LEADERBOARD_KEY, 0, n-1, withscores=True)

def get_rank(user_id):
    rank = r.zrevrank(LEADERBOARD_KEY, f'user:{user_id}')
    return rank + 1 if rank is not None else None  # 1-indexed
```

### Rebuilding from MySQL (if Redis is flushed)

```python
from gamification.models import UserXP

def rebuild_leaderboard(course_id):
    """Rebuild Redis leaderboard from MySQL source of truth."""
    r = redis.Redis(host='redis', port=6379)
    key = f'xp:leaderboard:{course_id}'
    r.delete(key)

    for record in UserXP.objects.filter(course_id=course_id):
        r.zadd(key, {f'user:{record.user_id}': record.total_xp})
```

---

## Persistence Configuration

Redis is configured with AOF (Append-Only File) persistence in dev:

```
data/redis/appendonlydir/   ← AOF files here
```

This means Redis data survives container restarts. In production, AOF + RDB snapshots provide full durability.

---

## Quick Debug Commands

```bash
# Check Redis health
docker exec tutor_dev-redis-1 redis-cli PING

# Count all keys
docker exec tutor_dev-redis-1 redis-cli DBSIZE

# Memory usage
docker exec tutor_dev-redis-1 redis-cli INFO memory | grep used_memory_human

# Clear all keys (DANGEROUS - only in dev!)
docker exec tutor_dev-redis-1 redis-cli FLUSHALL
```
