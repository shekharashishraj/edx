# API Design

How XBlocks expose endpoints to their JavaScript frontends, and how the Open edX REST API is structured.

---

## XBlock Handler API Pattern

XBlock handlers are the primary API surface for custom XBlocks. They're POST-only AJAX endpoints.

### Handler URL Generation

JavaScript gets the handler URL from the XBlock runtime:

```javascript
var handlerUrl = runtime.handlerUrl(element, 'handler_name');
// → "/api/xblock/v2/xblocks/<usage_key>/handler/handler_name/"
```

### Making a Handler Call

```javascript
// Using fetch (modern)
async function callHandler(handlerName, data) {
    const url = runtime.handlerUrl(element, handlerName);
    const response = await fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRFToken': getCsrfToken(),
        },
        body: JSON.stringify(data),
    });
    if (!response.ok) throw new Error(`Handler error: ${response.status}`);
    return response.json();
}

// Usage
const result = await callHandler('save_pin', {lat: 33.42, lng: -111.93});
console.log(result.xp);  // 100

// Using jQuery (also works, Open edX bundles jQuery)
$.ajax({
    type: 'POST',
    url: runtime.handlerUrl(element, 'save_pin'),
    data: JSON.stringify({lat: 33.42, lng: -111.93}),
    contentType: 'application/json',
    success: function(response) { console.log(response.xp); },
});
```

### CSRF Token Helper

```javascript
function getCsrfToken() {
    const match = document.cookie.match(/csrftoken=([^;]+)/);
    return match ? match[1] : '';
}
```

### Handler Implementation (Python)

```python
@XBlock.json_handler
def save_pin(self, data, suffix=''):
    """
    POST /handler/save_pin/
    Body: { "lat": 33.42, "lng": -111.93 }
    Returns: { "success": true, "xp": 100 }
    """
    lat = data.get('lat')
    lng = data.get('lng')

    if not lat or not lng:
        return {'error': 'Missing lat/lng', 'success': False}

    self.pin_location = {'lat': lat, 'lng': lng}

    # Award XP
    self.runtime.publish(self, 'xp_earned', {'xp': 100, 'action': 'drop_map_pin'})

    return {
        'success': True,
        'xp': 100,
        'pin': {'lat': lat, 'lng': lng},
    }
```

---

## Handler Return Values

Always return a Python dict — it gets JSON-serialized:

| Return | Meaning |
|--------|---------|
| `{'success': True}` | Generic success |
| `{'error': 'message', 'success': False}` | Application error (still HTTP 200) |
| Raise exception | HTTP 500 (avoid — use explicit error dicts) |

**Note:** HTTP status code is always 200 for `@XBlock.json_handler`. Errors are communicated via the returned dict's `error` key.

---

## Open edX REST API

Open edX LMS exposes REST APIs consumed by the MFE and external tools:

### Key Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/user/v2/account/login_session/` | POST | Login |
| `/api/user/v1/me` | GET | Current user info |
| `/api/enrollment/v1/enrollment` | GET/POST | Course enrollments |
| `/api/course_home/v1/outline/{course_id}` | GET | Course outline |
| `/api/grades/v1/courses/{course_id}/` | GET | Course grades |
| `/api/courseware/v1/course/{course_id}/` | GET | Courseware access |

### Authentication

API calls from the MFE use session cookies (set at login). External API calls use OAuth2 tokens:

```bash
# Get OAuth2 token
curl -X POST http://local.openedx.io:8000/oauth2/access_token/ \
  -d "client_id=<client_id>&client_secret=<secret>&grant_type=client_credentials"

# Use token
curl -H "Authorization: Bearer <token>" \
  http://local.openedx.io:8000/api/user/v1/me
```

---

## XBlock Data API (Reading XBlock State)

```python
# In LMS views, read XBlock state for a student
from lms.djangoapps.courseware.module_render import get_module_for_descriptor

xblock = get_module_for_descriptor(user, request, descriptor, field_data_cache, course_key)
student_state = xblock.user_data  # reads Scope.user_state fields
```

---

## Studio REST API (XBlock CRUD)

Studio exposes endpoints for XBlock management:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/xblock/` | POST | Create new XBlock |
| `/xblock/{usage_key}` | GET | Get XBlock data |
| `/xblock/{usage_key}` | PUT | Update XBlock settings |
| `/xblock/{usage_key}` | DELETE | Delete XBlock |

Studio uses these internally when instructors edit course content. Custom XBlocks don't need to use them directly.

---

## Error Handling in Handlers

```python
@XBlock.json_handler
def submit_answer(self, data, suffix=''):
    try:
        answer = data['answer']  # KeyError if missing
    except KeyError:
        return {'error': 'Missing required field: answer', 'success': False}

    if self.attempts >= self.max_attempts:
        return {
            'error': 'No attempts remaining',
            'success': False,
            'attempts': self.attempts,
        }

    try:
        result = self._process_answer(answer)
    except Exception as e:
        log.exception("Error processing answer for user %s", self.runtime.user_id)
        return {'error': 'Internal error. Please try again.', 'success': False}

    return {'success': True, 'result': result}
```

**Rules:**
1. Never let exceptions bubble up from handlers (they become HTTP 500)
2. Log exceptions with `log.exception()` before catching
3. Return informative error messages to help debug
4. Always include a `success` boolean in the response
