# Third-Party Integrations

External services used by the 6 custom XBlocks.

---

## Mapbox (GlobalMap XBlock)

**What:** Vector map rendering and geospatial APIs
**Used by:** GlobalMap XBlock
**Tier needed:** Free (Mapbox has a generous free tier for educational use)

### Setup

1. Create account at https://mapbox.com
2. Generate a public access token (starts with `pk.`)
3. Store in XBlock settings (instructor-configurable in Studio)

### Integration Points

```javascript
// In globalmap student.js
mapboxgl.accessToken = initArgs.mapbox_token;  // passed from Python

var map = new mapboxgl.Map({
    container: 'globalmap-container',
    style: 'mapbox://styles/mapbox/light-v11',
    center: [0, 20],
    zoom: 1.5,
    projection: 'globe',  // 3D globe view
});
```

### XBlock Field

```python
mapbox_token = String(
    default='',
    scope=Scope.settings,
    help='Mapbox public access token (pk.eyJ...)'
)
```

### Privacy

- Student pins are rounded to 1 decimal degree (~11 km) before storage
- Mapbox receives anonymous tile requests (no user data sent to Mapbox)
- Actual coordinates never sent to Mapbox

---

## Mozilla Hubs (VirtualCampus XBlock)

**What:** Browser-based 3D social VR platform
**Used by:** VirtualCampus XBlock
**Option:** Hubs Cloud (managed) or self-hosted

### Setup

**Option A — Hubs Cloud:**
1. Sign up at https://hubs.mozilla.com
2. Create a room
3. Copy the room URL (e.g., `https://hubs.mozilla.com/abc123`)
4. Set as XBlock `hub_url` in Studio

**Option B — Self-hosted (Production):**
```bash
# Deploy Hubs Community Edition
docker pull mozillareality/hubs-stack:latest
# Configure at https://hubs.mozilla.com/docs/hubs-cloud-aws.html
```

### Integration

```html
<!-- In virtualcampus student.html -->
<iframe
    src="{self.hub_url}?embed_token={embed_token}"
    allow="microphone; camera; vr; speaker"
    style="width: 100%; height: 600px; border: none;"
></iframe>
```

### postMessage Events from Hubs

```javascript
window.addEventListener('message', (event) => {
    if (!event.data?.type) return;
    switch(event.data.type) {
        case 'entered_room':
            startAttendanceTracking();
            break;
        case 'exited_room':
            stopAttendanceTracking();
            break;
    }
});
```

---

## PhET Interactive Simulations

**What:** Free HTML5 interactive science/math simulations from University of Colorado
**Used by:** PhETSim XBlock
**Cost:** Free (open source, https://phet.colorado.edu)
**License:** Creative Commons

### Available Simulations

All available at `https://phet.colorado.edu/sims/html/<sim-name>/latest/<sim-name>_en.html`

Relevant for ASU Global Connect:
| Simulation | Subject | URL Slug |
|-----------|---------|----------|
| Wave on a String | Physics | `wave-on-a-string` |
| Gravity and Orbits | Physics | `gravity-and-orbits` |
| Molecule Shapes | Chemistry | `molecule-shapes` |
| Fraction Basics | Math | `fraction-basics` |
| Build an Atom | Chemistry | `build-an-atom` |

### Embedding

```html
<!-- In phetsim student.html -->
<iframe
    id="phet-sim"
    src="{self.sim_url}"
    style="width: 100%; height: 600px; border: none;"
    allow="fullscreen"
></iframe>
```

### No API Key Needed

PhET sims are publicly accessible HTML5 files. No API key or authentication required.

---

## Socket.io (Multiplayer XBlock)

**What:** Real-time bidirectional event communication library
**Used by:** Multiplayer XBlock
**Deployment:** Node.js sidecar container

### Package

```json
{
  "dependencies": {
    "socket.io": "^4.7.0",
    "axios": "^1.6.0",
    "express": "^4.18.0"
  }
}
```

### Sidecar Container

The Socket.io server runs as a separate Docker container. In Tutor, this requires a plugin to add it to docker-compose:

```yaml
# In Tutor plugin's docker-compose addition:
services:
  multiplayer:
    image: asu-multiplayer-server:latest
    ports:
      - "3000:3000"
    environment:
      LMS_URL: "http://lms:8000"
      JWT_SECRET: "${OPENEDX_SECRET_KEY}"
    depends_on:
      - lms
      - mysql
```

### Client Connection

```javascript
// In multiplayer student.js
const socket = io('http://local.openedx.io:3000', {
    auth: { token: initArgs.socket_token },
    transports: ['websocket'],
});

socket.on('connect', () => console.log('Connected to multiplayer server'));
socket.on('challenge_start', (data) => startChallenge(data));
socket.on('player_answered', (data) => updateLiveLeaderboard(data));
```

---

## PostGIS (GlobalMap XBlock — Phase 4)

**What:** Geospatial extension for relational databases
**Used by:** GlobalMap XBlock (proximity queries)
**Status:** Phase 4 addition

PostGIS enables SQL queries like "find all students within 50km of me":

```sql
-- Enable PostGIS (or use MySQL spatial types)
-- MySQL 8 has built-in spatial support

SELECT user_id, ST_Distance_Sphere(
    POINT(student_lng, student_lat),
    POINT(-111.93, 33.42)  -- Tempe, AZ
) AS distance_meters
FROM globalmap_pins
WHERE ST_Distance_Sphere(
    POINT(student_lng, student_lat),
    POINT(-111.93, 33.42)
) < 50000  -- 50 km radius
ORDER BY distance_meters
LIMIT 20;
```

In Django, use GeoDjango:
```python
from django.contrib.gis.db import models
from django.contrib.gis.geos import Point
from django.contrib.gis.measure import Distance

class StudentPin(models.Model):
    user_id = models.IntegerField()
    location = models.PointField()  # GeoDjango spatial field

# Query: students within 50km
nearby = StudentPin.objects.filter(
    location__distance_lte=(my_location, Distance(km=50))
)
```
