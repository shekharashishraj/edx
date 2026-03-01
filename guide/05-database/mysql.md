# MySQL — Schema & Key Tables

MySQL stores all relational data: users, enrollments, grades, auth tokens, and custom XBlock tables (XP records, badges).

---

## Access

```bash
# Root shell
tutor dev do sqlshell

# Via Django
tutor dev run lms ./manage.py lms dbshell
```

```sql
USE openedx;
SHOW TABLES;  -- 200+ tables
```

---

## Key Open edX Tables

### Users & Authentication

```sql
-- Core user table (Django built-in)
auth_user
  id, username, email, password, is_staff, is_superuser, is_active, date_joined

-- Open edX user profile (extended user data)
auth_userprofile
  id, user_id (FK auth_user), name, bio, location, country, language, gender,
  year_of_birth, goals, meta (JSON)

-- Login failure tracking (rate limiting)
student_loginfailures
  id, user_id, failure_count, lockout_until

-- OAuth2 access tokens
oauth2_accesstoken
  id, user_id, token, expires, scope

-- Third-party auth (SSO) links
social_auth_usersocialauth
  id, user_id, provider, uid, extra_data (JSON)
```

### Enrollments & Courses

```sql
-- Course enrollments
student_courseenrollment
  id, user_id, course_id, created, is_active, mode
  -- mode: 'audit', 'verified', 'honor', 'professional'
  -- course_id format: 'course-v1:ASU+CS101+2026'

-- Course overviews (cached course metadata)
course_overviews_courseoverview
  id (= course_id), display_name, start, end, enrollment_start, enrollment_end,
  course_image_url, short_description

-- Course access roles (staff, instructor, etc.)
student_courseaccessrole
  id, user_id, course_id, role, org
```

### Grades

```sql
-- Per-course persistent grades
grades_persistentcoursegrade
  id, user_id, course_id, course_edited_timestamp, letter_grade, percent_grade,
  passed_timestamp

-- Per-subsection grades
grades_persistentsubsectiongrade
  id, user_id, course_id, usage_key, earned_all, possible_all, earned_graded,
  possible_graded, first_attempted

-- Grade overrides (manual instructor override)
grades_persistentsubsectiongradeoverride
  id, grade_id, system, earned_all_override, possible_all_override
```

### Certificates

```sql
-- Generated certificates
certificates_generatedcertificate
  id, user_id, course_id, grade, status, name, created_date, modified_date
  -- status: 'downloadable', 'notpassing', 'unavailable'
```

### Forum / Discussion

```sql
-- Forum roles
django_comment_client_role
  id, name, course_id

django_comment_client_permission
  id, name
```

---

## Custom XBlock Tables (To Be Created)

These tables will be created when Gamification XBlock is built:

### XP Records
```sql
CREATE TABLE gamification_xprecord (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL REFERENCES auth_user(id),
    course_id   VARCHAR(255) NOT NULL,
    action      VARCHAR(100) NOT NULL,
    xp          INT NOT NULL,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    metadata    JSON,
    INDEX idx_user_course (user_id, course_id),
    INDEX idx_action (action),
    INDEX idx_created (created_at)
);
```

### User XP Totals (materialized)
```sql
CREATE TABLE gamification_userxp (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL REFERENCES auth_user(id),
    course_id   VARCHAR(255) NOT NULL,
    total_xp    INT NOT NULL DEFAULT 0,
    level       TINYINT NOT NULL DEFAULT 1,
    updated_at  DATETIME NOT NULL ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_course (user_id, course_id)
);
```

### Badges
```sql
CREATE TABLE gamification_badge (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL REFERENCES auth_user(id),
    badge_slug  VARCHAR(100) NOT NULL,
    course_id   VARCHAR(255) NOT NULL,
    awarded_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_badge_course (user_id, badge_slug, course_id)
);
```

---

## Django ORM Usage (Never Raw SQL)

All queries go through Django ORM:

```python
from django.contrib.auth import get_user_model
from common.djangoapps.student.models import CourseEnrollment, UserProfile

User = get_user_model()

# Get a user
user = User.objects.get(username='admin')

# Get all enrollments for a user
enrollments = CourseEnrollment.objects.filter(user=user, is_active=True)

# Get user profile
profile = user.profile  # reverse FK, raises DoesNotExist if missing

# Count active students in a course
count = CourseEnrollment.objects.filter(
    course_id='course-v1:ASU+CS101+2026',
    is_active=True
).count()
```

---

## Useful Queries (for debugging in dbshell)

```sql
-- List all users
SELECT id, username, email, is_superuser FROM auth_user;

-- Check enrollments
SELECT u.username, e.course_id, e.mode, e.is_active
FROM student_courseenrollment e
JOIN auth_user u ON e.user_id = u.id
ORDER BY e.created DESC LIMIT 20;

-- Check grades
SELECT u.username, g.course_id, g.percent_grade, g.letter_grade
FROM grades_persistentcoursegrade g
JOIN auth_user u ON g.user_id = u.id;

-- Check login failures
SELECT u.username, f.failure_count, f.lockout_until
FROM student_loginfailures f
JOIN auth_user u ON f.user_id = u.id;

-- Clear login failures (fix lockout)
DELETE FROM student_loginfailures;
```
