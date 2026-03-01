# GlobalMap XBlock

**Package:** `globalmap-xblock`
**Phase:** 1 (Weeks 2–4)
**Status:** Not yet built

---

## Purpose

An interactive world map showing where ASU Global Connect students are located. Students drop a pin at their location, discover peers nearby, and send connection requests. Drives community and cross-cultural connection.

---

## Student View

- Full-width interactive Mapbox GL map
- "Drop my pin" button — places the student's location
- Other students' pins visible on map (privacy: city-level precision only)
- Click a pin → see student name, program, interests
- "Connect" button → send connection request
- My connections list (sidebar)

---

## Fields

```python
# Scope.user_state (per-student)
pin_location = Dict(default={}, scope=Scope.user_state)
# e.g. {'lat': 33.42, 'lng': -111.93, 'city': 'Tempe', 'country': 'US'}

profile_complete = Boolean(default=False, scope=Scope.user_state)
connection_ids = List(default=[], scope=Scope.user_state)  # accepted connection user_ids
pending_requests = List(default=[], scope=Scope.user_state)

# Scope.settings (instructor)
map_style = String(default='mapbox://styles/mapbox/light-v11', scope=Scope.settings)
allow_connections = Boolean(default=True, scope=Scope.settings)
privacy_level = String(default='city', scope=Scope.settings)  # 'exact', 'city', 'country'
```

---

## Handlers

| Handler | Called By | Action |
|---------|-----------|--------|
| `save_pin` | Student JS | Save student location, award 100 XP |
| `get_all_pins` | Student JS | Return all student pins (anonymized) |
| `send_connection` | Student JS | Send connection request |
| `accept_connection` | Student JS | Accept pending request, award 25 XP |
| `get_connections` | Student JS | Return student's connection list |
| `save_profile` | Student JS | Save bio/interests, award 50 XP on first complete |

---

## XP Events Published

```python
# Pin dropped (one-time)
self.runtime.publish(self, 'xp_earned', {'xp': 100, 'action': 'drop_map_pin'})

# Profile completed (one-time)
self.runtime.publish(self, 'xp_earned', {'xp': 50, 'action': 'complete_profile'})

# Connection accepted (per connection, up to some cap)
self.runtime.publish(self, 'xp_earned', {'xp': 25, 'action': 'connection_accepted'})
```

---

## Tech Stack

- **Mapbox GL JS v3** — interactive vector map rendering
- **PostGIS** (optional, Phase 4) — geospatial proximity queries for "students near me"
- Privacy: Pins snapped to city centroid, never exact coordinates

---

## Privacy Design

Student coordinates are truncated to city-level before storage:
```python
def _anonymize_location(lat, lng):
    """Round to 1 decimal place (~11km precision) for city-level privacy."""
    return round(lat, 1), round(lng, 1)
```

Students control their visibility via a "Show my pin" toggle.

---

## Mapbox Integration

```javascript
mapboxgl.accessToken = 'pk.eyJ1IjoidHVrb3IiLCJhIjoi...';  // from XBlock settings

var map = new mapboxgl.Map({
    container: 'map',
    style: 'mapbox://styles/mapbox/light-v11',
    center: [0, 20],
    zoom: 2
});

// Load all pins
fetch(runtime.handlerUrl(element, 'get_all_pins'))
    .then(r => r.json())
    .then(data => {
        data.pins.forEach(pin => {
            new mapboxgl.Marker()
                .setLngLat([pin.lng, pin.lat])
                .setPopup(new mapboxgl.Popup().setHTML(pin.popup_html))
                .addTo(map);
        });
    });
```
