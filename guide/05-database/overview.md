# Database Overview

ASU Global Connect uses three databases, each optimized for a different type of data.

---

## Why Three Databases?

| Database | Type | Best For | Used For |
|----------|------|----------|---------|
| MySQL | Relational SQL | Structured data, ACID transactions, complex queries | Users, enrollments, grades, XP records |
| MongoDB | Document NoSQL | Flexible schemas, hierarchical data, frequent changes | Course content, XBlock state, quiz responses |
| Redis | In-memory key-value | Speed-critical reads/writes, ephemeral data | Sessions, task queue, leaderboards |

Open edX was designed with this split from the beginning:
- **MySQL** → "who are you, what have you done" (identity + progress)
- **MongoDB** → "what does the course look like" (content)
- **Redis** → "what's happening right now" (sessions + tasks)

---

## Data Location Summary

```
data/mysql/     → MySQL 8.4 InnoDB files
data/mongodb/   → MongoDB 7.0 WiredTiger files
data/redis/     → Redis 7.4 AOF persistence
```

All three are bind-mounted into their respective containers. Data persists across container restarts.

---

## Database Names

| Database | Name |
|----------|------|
| MySQL | `openedx` |
| MongoDB | `openedx` |
| Redis | `0` (default DB) |

---

## Connections

| Database | Host (inside Docker) | Port |
|----------|---------------------|------|
| MySQL | `mysql` | 3306 |
| MongoDB | `mongodb` | 27017 |
| Redis | `redis` | 6379 |

In `env/apps/openedx/config/lms.env.yml`:
```yaml
DATABASES:
  default:
    ENGINE: django.db.backends.mysql
    NAME: openedx
    USER: openedx
    PASSWORD: JVeZjIku     # from config.yml
    HOST: mysql
    PORT: 3306
```

---

## Access Tools

```bash
# MySQL
tutor dev do sqlshell                    # MySQL root shell
tutor dev run lms ./manage.py lms dbshell  # via Django

# MongoDB
docker exec -it tutor_dev-mongodb-1 mongosh openedx

# Redis
docker exec -it tutor_dev-redis-1 redis-cli
```

---

## Backup Strategy (Production)

```bash
# MySQL dump
tutor local run mysql bash -c \
  "mysqldump -uroot -p<password> openedx > /tmp/backup.sql"

# MongoDB dump
tutor local run mongodb bash -c \
  "mongodump --db openedx --out /tmp/mongodump"

# Redis: AOF persistence file is at data/redis/appendonlydir/
# Copy that directory for backup
```

---

## See Also

- [mysql.md](mysql.md) — Key tables, Django models, schema details
- [mongodb.md](mongodb.md) — Collections, course content structure, XBlock state
- [redis.md](redis.md) — Cache keys, Celery queues, leaderboard sorted sets
