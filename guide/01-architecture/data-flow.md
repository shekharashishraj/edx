# Data Flow

How data moves through ASU Global Connect from user action to storage and back.

---

## 1. Student Login Flow

```
Browser (authn MFE at :8000/login)
  │
  ├─ POST /api/user/v2/account/login_session/
  │    Body: { email_or_username, password }
  │
  └─ LMS Django View (login.py)
       │
       ├─ MySQL: lookup User by username/email
       ├─ MySQL: verify password hash (PBKDF2)
       ├─ MySQL: fetch UserProfile (name, etc.)
       ├─ Redis: write session key → user_id mapping
       ├─ MySQL: write LoginFailures (reset on success)
       │
       └─ Response: Set-Cookie: sessionid=<token>
            │
            └─ Browser: stores session cookie
                 → Subsequent requests are authenticated
```

**Key models:**
- `auth_user` (MySQL) — Django's base User model
- `auth_userprofile` (MySQL) — Open edX UserProfile (name, bio, location)
- `student_loginfailures` (MySQL) — rate limit tracker
- Redis `session:<session_key>` — session data

---

## 2. Course Page Load Flow

```
Browser → GET /courses/course-v1:ASU+CS101+2026/courseware/
  │
  └─ LMS Django
       │
       ├─ MySQL: verify enrollment (StudentCourseEnrollment)
       ├─ MongoDB: fetch course structure (SplitModulestore)
       │    Collection: modulestore.split_modulestore.active_versions
       │
       ├─ XBlock Runtime
       │    ├─ MongoDB: fetch XBlock definitions for each block
       │    ├─ MySQL: fetch user_state (per-student XBlock data)
       │    └─ render student_view() for each block
       │         → returns Fragment(html, css, js)
       │
       ├─ Redis: cache course overview (TTL 1 hour)
       │
       └─ Django template: assemble page HTML
            → include XBlock fragments
            → include runtime.js (XBlock JS runtime)
```

---

## 3. XBlock Student Interaction (e.g., Map Pin Drop)

```
Browser: user drops pin on GlobalMap
  │
  └─ JS: fetch(runtime.handlerUrl(element, 'save_pin'), {
             method: 'POST',
             body: JSON.stringify({ lat: 33.42, lng: -111.93 })
         })
         │
         └─ LMS: POST /handler/globalmap/<usage_key>/save_pin/
              │
              └─ GlobalMap.save_pin() XBlock handler
                   │
                   ├─ MongoDB: write self.pin_data = { lat, lng }
                   │    (Scope.user_state field → per-student storage)
                   │
                   ├─ self.runtime.publish(self, 'xp_earned', {
                   │      'xp': 100, 'action': 'drop_map_pin'
                   │  })
                   │    └─ Event bus → Gamification XBlock listener
                   │         ├─ MySQL: UPDATE user_xp SET total = total + 100
                   │         └─ Redis: ZADD leaderboard score user_id
                   │
                   └─ return { 'success': True, 'total_xp': 350 }
                        │
                        └─ JS: update UI with new pin + XP notification
```

---

## 4. XP Event Flow (Cross-XBlock Communication)

The Gamification XBlock acts as the central event listener:

```
Any XBlock
  └─ self.runtime.publish(self, 'xp_earned', {
         'xp': 100,
         'action': 'drop_map_pin',
         'user_id': self.runtime.user_id,
     })
     │
     └─ Open edX Event Bus
          │
          └─ GamificationXBlock.handle_xp_event()
               │
               ├─ MySQL: read current user XP total
               ├─ MySQL: write new XP total
               ├─ MySQL: check badge thresholds → award badges
               ├─ Redis ZADD: update sorted set leaderboard
               └─ MySQL: write XP audit log entry
```

---

## 5. Course Content Save (Studio)

```
Instructor: edits XBlock in Studio
  │
  └─ PUT /xblock/<usage_key>
       Body: { data: { ... }, metadata: { display_name: "..." } }
       │
       └─ CMS Django
            │
            ├─ MongoDB: update XBlock definition
            │    Collection: modulestore.split_modulestore.definitions
            │
            ├─ MongoDB: update course structure version
            │    Collection: modulestore.split_modulestore.structures
            │
            ├─ MySQL: log edit to CourseEditLog
            ├─ Celery: enqueue course_publish task (async)
            │    └─ Redis broker: task payload written
            │         └─ Celery worker: picks up task
            │              ├─ Rebuild course blocks cache
            │              └─ Update Meilisearch search index
            │
            └─ 200 OK
```

---

## 6. Search Query Flow

```
Student: types in course search bar
  │
  └─ MFE Learner Dashboard → GET /api/v1/search/?q=physics
       │
       └─ LMS Search API
            │
            └─ Meilisearch :7700
                 └─ Full-text search across indexed course titles, descriptions
                 └─ Returns ranked results
                 └─ LMS serializes → JSON response
                      │
                      └─ MFE renders search results
```

---

## 7. Celery Async Task Flow

Long-running operations (grade recalc, certificate generation, email) run async:

```
LMS View: triggers async work
  │
  └─ celery_task.delay(args)
       │
       └─ Redis Broker: stores serialized task payload
            │
            └─ Celery Worker (LMS container)
                 └─ picks up task from Redis queue
                 └─ executes task (e.g., send_email, recalculate_grades)
                 └─ writes result to MySQL
                 └─ (optionally) writes result to Redis result backend
```

---

## 8. Data Ownership Summary

| Data Type | Database | Key Model / Collection |
|-----------|----------|----------------------|
| User accounts | MySQL | `auth_user` |
| User profiles | MySQL | `student_userprofile` |
| Enrollments | MySQL | `student_courseenrollment` |
| Grades | MySQL | `grades_persistentcoursegrade` |
| XP totals | MySQL | Custom (Gamification XBlock) |
| Badges | MySQL | Custom (Gamification XBlock) |
| Certificates | MySQL | `certificates_generatedcertificate` |
| Auth tokens | MySQL | `oauth2_accesstoken` |
| Course structure | MongoDB | `modulestore.split_modulestore.structures` |
| XBlock definitions | MongoDB | `modulestore.split_modulestore.definitions` |
| XBlock student state | MongoDB | `modulestore.xblock_django_*` |
| ORA2 responses | MongoDB | `openassessment_submission` |
| Sessions | Redis | `session:<key>` |
| Task queue | Redis | `celery` default queue |
| Leaderboard | Redis | sorted set `xp:leaderboard` (future) |
| Search index | Meilisearch | `course_index` |
| Uploaded media | Filesystem | `data/openedx-media/` |
