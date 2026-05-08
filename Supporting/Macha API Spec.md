# MACHA — API Technical Specification
*For implementation reference. Contains all endpoint definitions, request/response shapes, error codes, and backend behavior rules.*

---

## Conventions

- Base path: `/api/v1`
- All request/response bodies: `application/json` unless file upload
- File uploads: `multipart/form-data`
- All timestamps: ISO 8601 — `2025-01-15T14:30:00`
- All errors return: `{ "error": { "code": "ERROR_CODE", "message": "Human readable description" } }`
- All operations confined to `/data/` directory — no exceptions

---

## Design Principles

- The agent never constructs or receives file paths. The backend derives all paths from title + subject + category internally.
- The agent only ever holds IDs and titles.
- Subject creation only happens through `POST /setup/semester` or `POST /subjects`. Saves never auto-create subjects.
- Title uniqueness is enforced at the DB level per `UNIQUE (subject_id, category, title)`.
- No backend cap on returned file lists — agent enforces the 5-file threshold.
- Failures are always loud — no silent swallowing of errors.
- `file_type` value `"file"` is a catch-all for all binary formats (pdf, docx, pptx, zip, etc.). No further enumeration — the backend stores and serves it as-is regardless of the actual format.
- `resource_type` is intentionally free-form — it is a plain string constructed by the agent from natural language (e.g. `"notes"`, `"previous_year_paper"`, `"lab_manual"`). The backend stores and returns it as-is. No enumeration, no CHECK constraint, no validation against a fixed list. Adding such a constraint would break the system.

---

## Area 1 — Setup

### `POST /setup/semester`

One-time initialization. Creates semester folder, subject folders with all four fixed subcategories, and inserts subject rows into DB. Idempotent — re-running skips existing subjects without error.

**Request:**
```json
{
  "semester": "SEM_3",
  "subjects": ["DBMS", "OS", "DAA", "CN"]
}
```

**Response `200`:**
```json
{
  "semester": "SEM_3",
  "created": ["DBMS", "OS", "DAA", "CN"],
  "skipped": []
}
```

**Backend behavior:**
- Creates `/data/SEM_3/` if it doesn't exist
- For each subject in `created`: creates `/data/SEM_3/<Subject>/` with four fixed subcategories: `material/`, `questions/`, `info/`, `miscellaneous/`
- Inserts row into `subjects` table: `(name, semester, archived=false)`
- All operations atomic per subject — folder creation and DB insert succeed or fail together
- Subjects already in DB for that semester go into `skipped`, no error

---

### `POST /subjects`

Creates a single subject mid-semester. Same atomic guarantees as bulk setup.

**Request:**
```json
{
  "semester": "SEM_3",
  "name": "Machine Learning"
}
```

**Response `201`:**
```json
{
  "subject_id": 12,
  "name": "Machine Learning",
  "semester": "SEM_3"
}
```

**Errors:**
```
409 Conflict — { "error": { "code": "SUBJECT_EXISTS", "message": "..." } }
```

**Backend behavior:**
- Creates `/data/SEM_3/Machine Learning/` with all four subcategories atomically
- Inserts row into `subjects` table
- Returns `409` if `(name, semester)` already exists in DB

---

### `PATCH /subjects/archive`

Archives all subjects for a given semester. Called during new semester transition after explicit student confirmation.

**Request:**
```json
{
  "semester": "SEM_3"
}
```

**Response `200`:**
```json
{
  "semester": "SEM_3",
  "archived_count": 4
}
```

**Errors:**
```
404 — no subjects found for that semester
400 — { "error": { "code": "ALREADY_ARCHIVED", "message": "All subjects for this semester are already archived" } }
```

**Backend behavior:**
- Sets `archived = TRUE` on all subject rows where `semester = ?`
- Does not touch directory structure — folders remain on disk for past semester retrieval
- Idempotent per subject — already-archived subjects are included in `archived_count` without error

---

## Area 2 — Academic Resources

### `POST /resources`

Saves a new academic resource. Does not auto-create subjects.

**Request** (`multipart/form-data`):
```
semester       string   required
subject_name   string   required
category       string   required   — "material" | "questions" | "info" | "miscellaneous"
title          string   required   — agent-constructed, e.g. "dbms_notes_unit_1_part1"
resource_type  string   required   — e.g. "notes", "previous_year_paper", "lab_manual"
file_type      string   required   — "file" | "image" | "text" | "link"
file           binary   cond.      — required if file_type is "file" or "image"
content        string   cond.      — required if file_type is "text" or "link"
```

**Response `201`:**
```json
{
  "resource_id": 42,
  "subject_id": 7
}
```

**Errors:**
```
404 — { "error": { "code": "SUBJECT_NOT_FOUND", "message": "..." } }
409 — { "error": { "code": "TITLE_EXISTS", "message": "..." } }
```

**Backend behavior:**
- Looks up `subject_id` from `(subject_name, semester)` — returns `SUBJECT_NOT_FOUND` if missing
- Checks `UNIQUE (subject_id, category, title)` — returns `TITLE_EXISTS` on collision
- Derives filename and absolute path from title + subject + category internally
- Writes file to disk, inserts DB row, returns `resource_id`
- Never constructs or exposes file paths to the agent

---

### `GET /resources/:id`

Returns JSON metadata for a resource. Always JSON regardless of file_type.

**Response `200`:**
```json
{
  "id": 42,
  "subject_id": 7,
  "subject_archived": false,
  "title": "dbms_notes_unit_1_part1",
  "resource_type": "notes",
  "category": "material",
  "file_type": "file",
  "content": null,
  "created_at": "2025-01-10T09:00:00"
}
```

**Notes:**
- `subject_archived: true` when parent subject has `archived = TRUE` in DB
- `content` is populated for `file_type: "text"` and `file_type: "link"`, null otherwise

---

### `GET /resources/:id/file`

Streams binary for file and image types.

**Response `200`:** Binary stream with appropriate `Content-Type`

**Errors:**
```
404 — file_type is "text" or "link" (no file on disk)
404 — resource ID not found
```

---

### `PUT /resources/:id`

Replaces file, content, or title. At least one field required.

**Request** (`multipart/form-data`):
```
title      string   optional
file       binary   optional
content    string   optional
file_type  string   optional
```

**Response `200`:**
```json
{ "resource_id": 42, "updated": true }
```

**Errors:**
```
400 — no fields provided
404 — resource ID not found
```

---

### `DELETE /resources/:id`

Deletes file from disk and DB row.

**Response `200`:**
```json
{ "resource_id": 42, "deleted": true }
```

**Errors:**
```
404 — resource ID not found
500 — { "error": { "code": "DISK_DELETE_FAILED", "message": "..." } }
500 — { "error": { "code": "DB_DELETE_FAILED", "message": "..." } }
```

Distinct error codes for disk vs DB failure — agent needs to know if there is a mismatch between disk and DB state.

---

## Area 3 — Miscellaneous Content

### `POST /miscellaneous`

Saves non-academic content. Category is created on first use.

**Request** (`multipart/form-data`):
```
title      string   required   — agent-decided, free-form descriptive
category   string   required   — e.g. "events", "scholarships"
file_type  string   required   — "file" | "image" | "text" | "link"
file       binary   cond.
content    string   cond.
```

**Response `201`:**
```json
{
  "misc_id": 15,
  "category_created": true
}
```

`category_created: true` when the category didn't exist before — agent updates its known-categories list in memory.

**Errors:**
```
409 — { "error": { "code": "TITLE_EXISTS", "message": "..." } }
```

**Backend behavior:**
- Creates `/data/MISCELLANEOUS/<category>/` if it doesn't exist
- Inserts row into `miscellaneous` table
- Returns `TITLE_EXISTS` on title collision within same category

---

### `GET /miscellaneous/:id`

Returns JSON metadata.

**Response `200`:**
```json
{
  "id": 15,
  "title": "techfest_registration_2025",
  "category": "events",
  "file_type": "link",
  "content": "https://drive.google.com/...",
  "created_at": "2025-01-10T09:00:00"
}
```

---

### `GET /miscellaneous/:id/file`

Streams binary for file and image types. Returns `404` if `file_type` is `text` or `link`.

---

### `PUT /miscellaneous/:id`

Replaces title, file, or content. At least one field required.

**Response `200`:**
```json
{ "misc_id": 15, "updated": true }
```

---

### `DELETE /miscellaneous/:id`

Deletes from disk and DB. Cascades to any linked deadline row via FK.

**Response `200`:**
```json
{ "misc_id": 15, "deleted": true }
```

---

## Area 4 — Deadlines

### `POST /deadlines`

Creates a deadline. Automatically creates two reminders: 2 hours before and 1 hour before `due_date`. Reminders that fall in the past are silently skipped.

**Request:**
```json
{
  "title": "CN Assignment Submission",
  "description": "Upload to classroom link before it closes",
  "due_date": "2025-01-17T22:00:00",
  "subject_id": 7
}
```

Or for miscellaneous:
```json
{
  "title": "Hackathon Registration Closes",
  "due_date": "2025-01-20T23:59:00",
  "misc_id": 15
}
```

**Response `201`:**
```json
{
  "deadline_id": 3,
  "reminders_created": [
    "2025-01-17T20:00:00",
    "2025-01-17T21:00:00"
  ]
}
```

`reminders_created` lists only the reminder times that were actually created (i.e. are in the future). Agent uses this to confirm to the student exactly when reminders will fire.

**Errors:**
```
400 — both subject_id and misc_id provided
400 — neither subject_id nor misc_id provided
404 — subject_id or misc_id not found
400 — due_date is in the past
```

**Backend behavior:**
- Enforces CHECK constraint: exactly one of `subject_id` / `misc_id`
- Inserts deadline row
- Calculates 2-hour and 1-hour reminder times from `due_date`
- Inserts only reminder rows where `reminder_time > NOW()`

---

### `GET /deadlines`

Queries deadlines. Used for both narrowing pulls (agent finding a specific deadline to update) and student-facing "what's due" queries.

**Query params:**
```
subject_id    int        optional
misc_id       int        optional
from          datetime   optional   defaults to NOW()
to            datetime   optional
completed     boolean    optional   defaults to false
```

**Response `200`:**
```json
{
  "deadlines": [
    {
      "id": 3,
      "title": "CN Assignment Submission",
      "description": "Upload to classroom link",
      "due_date": "2025-01-17T22:00:00",
      "is_completed": false,
      "subject_id": 7,
      "misc_id": null
    }
  ]
}
```

---

### `PATCH /deadlines/:id`

Partial update. Typically used for rescheduled or postponed events. Only send fields that changed.

**Request:**
```json
{
  "due_date": "2025-01-19T22:00:00"
}
```

**Response `200`:**
```json
{ "deadline_id": 3, "updated": true }
```

**Backend behavior:**
- Updates only provided fields
- If `due_date` changes, deletes existing reminders and recreates them at new times (skipping any that fall in the past)

---

### `PATCH /deadlines/:id/complete`

Marks a deadline as completed.

**Response `200`:**
```json
{ "deadline_id": 3, "is_completed": true }
```

---

### `DELETE /deadlines/:id`

Deletes deadline. Cascades to all linked reminders via FK `ON DELETE CASCADE`.

**Response `200`:**
```json
{ "deadline_id": 3, "deleted": true }
```

---

## Area 5 — Reminders

### `POST /reminders`

Creates an additional reminder for an existing deadline. Only called when the student explicitly requests one beyond the defaults. Agent validates that `reminder_time` is in the future before calling.

**Request:**
```json
{
  "deadline_id": 3,
  "reminder_time": "2025-01-16T20:00:00"
}
```

**Response `201`:**
```json
{ "reminder_id": 8 }
```

**Errors:**
```
400 — reminder_time is after due_date
404 — deadline_id not found
```

---

### `GET /reminders/due`

Heartbeat endpoint. Returns all reminders where `reminder_time <= as_of` and parent deadline is not completed.

**Query params:**
```
as_of   datetime   optional   defaults to NOW()
```

**Response `200`:**
```json
{
  "reminders": [
    {
      "reminder_id": 8,
      "deadline_id": 3,
      "deadline_title": "CN Assignment Submission",
      "due_date": "2025-01-17T22:00:00",
      "reminder_time": "2025-01-17T20:00:00"
    }
  ]
}
```

`deadline_title` and `due_date` included so agent can compose the WhatsApp message without a second lookup.

---

### `DELETE /reminders/:id`

Called immediately after agent fires the WhatsApp message. Row is permanently deleted — will not appear in future heartbeat cycles.

**Response `200`:**
```json
{ "reminder_id": 8, "deleted": true }
```

---

## Area 6 — Discovery

Read-only narrowing pull endpoints. Agent calls these before every save and retrieve to ground decisions in what actually exists.

### `GET /subjects`

First narrowing pull. List all subjects for a semester.

**Query params:**
```
semester   string   required
```

**Response `200`:**
```json
{
  "subjects": [
    { "id": 7, "name": "DBMS", "archived": false },
    { "id": 8, "name": "OS", "archived": false },
    { "id": 3, "name": "Math", "archived": true }
  ]
}
```

---

### `GET /subjects/:id/resource-types`

Second narrowing pull. List all distinct resource types and categories under a subject.

**Response `200`:**
```json
{
  "subject_id": 7,
  "resource_types": [
    { "resource_type": "notes", "category": "material" },
    { "resource_type": "previous_year_paper", "category": "questions" },
    { "resource_type": "lab_manual", "category": "material" }
  ]
}
```

Backed by `SELECT DISTINCT resource_type, category FROM resources WHERE subject_id = ?`.

---

### `GET /subjects/:id/resources`

Third narrowing pull. List all titles and IDs under a subject + category.

**Query params:**
```
category   string   required
```

**Response `200`:**
```json
{
  "subject_id": 7,
  "category": "material",
  "resources": [
    { "id": 42, "title": "dbms_notes_unit_1_part1", "file_type": "file" },
    { "id": 43, "title": "dbms_notes_unit_1_part2", "file_type": "file" },
    { "id": 44, "title": "dbms_drive_link_unit_2", "file_type": "link" }
  ]
}
```

`file_type` included so agent knows whether to call `/file` or use content from metadata endpoint — avoids a redundant call.

---

### `GET /miscellaneous/categories`

Lists all existing miscellaneous categories. Agent checks this before deciding to create a new one.

**Response `200`:**
```json
{
  "categories": ["events", "scholarships", "timetable_changes", "internships"]
}
```

Backed by `SELECT DISTINCT category FROM miscellaneous`.

---

### `GET /miscellaneous`

Lists titles and IDs within a miscellaneous category.

**Query params:**
```
category   string   required
```

**Response `200`:**
```json
{
  "category": "events",
  "items": [
    { "id": 15, "title": "techfest_registration_2025", "file_type": "link" },
    { "id": 16, "title": "blood_donation_drive_jan", "file_type": "image" }
  ]
}
```

---

## Area 7 — Storage

### `GET /storage/status`

Returns current disk usage. Agent checks this before accepting any new file.

**Response `200`:**
```json
{
  "available_gb": 4.2,
  "status": "warning"
}
```

`status` values:
- `"ok"` — above 5GB available
- `"warning"` — below 5GB available
- `"critical"` — below 3GB available

Agent behavior on each status is defined in `RESTRICTIONS.md`.

---

## Complete Endpoint Index

```
POST   /setup/semester
POST   /subjects
PATCH  /subjects/archive

POST   /resources
GET    /resources/:id
GET    /resources/:id/file
PUT    /resources/:id
DELETE /resources/:id

POST   /miscellaneous
GET    /miscellaneous/:id
GET    /miscellaneous/:id/file
PUT    /miscellaneous/:id
DELETE /miscellaneous/:id

POST   /deadlines
GET    /deadlines
PATCH  /deadlines/:id
PATCH  /deadlines/:id/complete
DELETE /deadlines/:id

POST   /reminders
GET    /reminders/due
DELETE /reminders/:id

GET    /subjects
GET    /subjects/:id/resource-types
GET    /subjects/:id/resources
GET    /miscellaneous/categories
GET    /miscellaneous

GET    /storage/status
```
