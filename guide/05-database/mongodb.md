# MongoDB — Collections & Course Content

MongoDB stores all course content, XBlock state, and student quiz responses. It uses Open edX's "Split Modulestore" — a version-controlled course content store.

---

## Access

```bash
docker exec -it tutor_dev-mongodb-1 mongosh openedx
```

```javascript
// List collections
db.getCollectionNames()

// Count documents in a collection
db["modulestore.split_modulestore.active_versions"].countDocuments()
```

---

## Key Collections

### Split Modulestore (Course Content)

The modulestore is Open edX's course content database. It stores XBlock definitions, course structure, and content in a version-controlled, immutable-snapshot model.

```
modulestore.split_modulestore.active_versions
  → Maps course_id to a "version" pointer
  → Like a git HEAD: which snapshot is "live"

modulestore.split_modulestore.structures
  → A snapshot (version) of a course's block tree
  → Immutable: saving a course creates a NEW structure, not modifying the old one
  → Contains: blocks array [{id, fields, block_type}]

modulestore.split_modulestore.definitions
  → The actual content of each XBlock
  → {_id, block_type, fields: {display_name, data, ...}}
  → Shared across course versions (referenced by ID)
```

### Viewing Course Structure

```javascript
// Find your course's active version
db["modulestore.split_modulestore.active_versions"].find({}).pretty()

// Get a structure (snapshot) - use the version pointer from above
db["modulestore.split_modulestore.structures"].findOne(
    {_id: ObjectId("<version_id>")}
)

// Get an XBlock definition
db["modulestore.split_modulestore.definitions"].findOne(
    {block_type: "problem"}
)
```

### XBlock Student State

```
xblock_django_xblockasidedefinition   → per-definition XBlock aside data
xblock_django_xblockasidestudentstate → per-student XBlock aside state
xblock_studentmodule                  → per-student, per-XBlock state (legacy)
```

The primary student state for XBlocks is stored via Django ORM in MySQL (`courseware_studentmodule`), but some XBlock runtimes use MongoDB. Check which your XBlock uses.

### ORA2 (Open Response Assessment)

```
openassessment_submission         → Student essay/project submissions
openassessment_assessment         → Peer assessments
openassessment_assessmentpart     → Individual rubric criterion scores
openassessment_studenttrainingworkflow → Training step data
```

```javascript
// View recent submissions
db.openassessment_submission.find({}).sort({created_at: -1}).limit(5).pretty()
```

---

## How Course Content Is Stored

A course is a tree:

```
Course Block
└── Chapter (Section)
      └── Sequential (Subsection)
            └── Vertical (Unit)
                  ├── VideoXBlock
                  ├── ProblemXBlock
                  └── GlobalMapXBlock   ← our custom XBlock
```

Each node in the tree is a "block" with:
- `block_type`: e.g., `course`, `chapter`, `sequential`, `vertical`, `globalmap`
- `block_id`: unique ID within the course
- `fields`: the XBlock's `Scope.content` and `Scope.settings` data
- `children`: list of child block IDs

In MongoDB:
```javascript
// A simplified structure document
{
  "_id": ObjectId("..."),
  "blocks": {
    "i4x://ASU/CS101/course/2026": {
      "block_type": "course",
      "fields": {"display_name": "Intro to CS"},
      "children": ["i4x://ASU/CS101/chapter/week1"]
    },
    "i4x://ASU/CS101/chapter/week1": {
      "block_type": "chapter",
      "fields": {"display_name": "Week 1"},
      "children": ["i4x://ASU/CS101/sequential/intro"]
    }
    // ...
  }
}
```

---

## XBlock Content Storage

When an instructor edits an XBlock in Studio and saves, Open edX:

1. Creates a new **definition** document in `modulestore.split_modulestore.definitions`
2. Creates a new **structure** (snapshot) referencing the new definition
3. Updates `active_versions` to point to the new structure

This gives full version history — you can roll back a course to any previous state.

---

## Important Notes

- **Never write directly to MongoDB** in XBlock code — use XBlock fields (`Scope.user_state`, etc.) which are saved via the XBlock runtime
- **Never drop MongoDB collections** without a backup — course content lives here
- The `openedx` database has 20+ collections; most are internal Open edX plumbing

---

## Useful Debug Queries

```javascript
// How many courses exist?
db["modulestore.split_modulestore.active_versions"].countDocuments()

// What XBlock types are defined?
db["modulestore.split_modulestore.definitions"].distinct("block_type")

// Find all GlobalMap XBlock definitions
db["modulestore.split_modulestore.definitions"].find(
    {block_type: "globalmap"}
).pretty()

// Count ORA2 submissions
db.openassessment_submission.countDocuments()
```
