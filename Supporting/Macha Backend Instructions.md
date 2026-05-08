# MACHA Backend — Claude Code Build Instructions

You are building the complete Node.js backend for MACHA, a WhatsApp academic assistant chatbot. Read every section of this document before writing a single line of code. Do not skip ahead.

---

## What You Are Building

A REST API that serves as the exclusive data layer between an AI agent (OpenClaw) and the file system + MySQL database. The agent never touches the file system or database directly — every operation goes through this backend.

The backend is responsible for:
- Storing and retrieving academic files organized by semester, subject, and category
- Storing and retrieving miscellaneous non-academic content
- Managing deadlines and reminders
- Reporting disk storage status

---

## Stack — No Deviations

- **Runtime:** Node.js
- **Framework:** Express
- **Database driver:** mysql2 (promise-based, no ORM, no query builders)
- **File uploads:** multer (diskStorage only, never memoryStorage)
- **Disk monitoring:** check-disk-space
- **Config:** dotenv
- **Dev runner:** nodemon

Install exactly these packages, nothing else:
```
npm install express mysql2 multer check-disk-space dotenv
npm install --save-dev nodemon
```

---

## Folder Structure — Create Exactly This

```
/backend
  /src
    /routes
      setup.js
      subjects.js
      resources.js
      miscellaneous.js
      deadlines.js
      reminders.js
      storage.js
    /services
      db.js
      fileSystem.js
    /middleware
      upload.js
      errorHandler.js
    app.js
    server.js
  .env.example
  package.json
```

---

## Environment Variables

### `.env.example` must contain exactly:
```
PORT=3000
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=yourpassword
DB_NAME=macha
DATA_DIR=/absolute/path/to/data
TEMP_DIR=/absolute/path/to/temp
```

---

## package.json scripts

```json
"scripts": {
  "start": "node src/server.js",
  "dev": "nodemon src/server.js"
}
```

---

## API Base Path

Every route is prefixed with `/api/v1`. Example: `POST /api/v1/setup/semester`.

---

## Complete Endpoint Index

Build every single one of these. Do not skip any.

```
POST   /api/v1/setup/semester
POST   /api/v1/subjects
PATCH  /api/v1/subjects/archive
GET    /api/v1/subjects
GET    /api/v1/subjects/:id/resource-types
GET    /api/v1/subjects/:id/resources

POST   /api/v1/resources
GET    /api/v1/resources/:id
GET    /api/v1/resources/:id/file
PUT    /api/v1/resources/:id
DELETE /api/v1/resources/:id

POST   /api/v1/miscellaneous
GET    /api/v1/miscellaneous/categories
GET    /api/v1/miscellaneous
GET    /api/v1/miscellaneous/:id
GET    /api/v1/miscellaneous/:id/file
PUT    /api/v1/miscellaneous/:id
DELETE /api/v1/miscellaneous/:id

POST   /api/v1/deadlines
GET    /api/v1/deadlines
PATCH  /api/v1/deadlines/:id/complete
PATCH  /api/v1/deadlines/:id
DELETE /api/v1/deadlines/:id

POST   /api/v1/reminders
GET    /api/v1/reminders/due
DELETE /api/v1/reminders/:id

GET    /api/v1/storage/status
```

---

## Database Schema

Run this exact SQL to create the database. Do not modify it.

```sql
CREATE TABLE subjects (
  id       INT AUTO_INCREMENT PRIMARY KEY,
  name     VARCHAR(100) NOT NULL,
  semester VARCHAR(20)  NOT NULL,
  archived BOOLEAN      DEFAULT FALSE NOT NULL,
  UNIQUE (name, semester)
);

CREATE TABLE resources (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  subject_id    INT          NOT NULL,
  title         VARCHAR(200) NOT NULL,
  resource_type VARCHAR(50)  NOT NULL,
  category      VARCHAR(50)  NOT NULL,
  file_path     VARCHAR(500) DEFAULT NULL,
  content       TEXT         DEFAULT NULL,
  file_type     VARCHAR(20)  DEFAULT NULL,
  created_at    TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT chk_resource_has_content
    CHECK (file_path IS NOT NULL OR content IS NOT NULL),
  CONSTRAINT chk_resource_file_type
    CHECK (file_type IN ('file', 'image', 'text', 'link')),
  CONSTRAINT uq_resource_title
    UNIQUE (subject_id, category, title),
  FOREIGN KEY (subject_id) REFERENCES subjects(id),
  INDEX idx_resources_subject (subject_id)
);

CREATE TABLE miscellaneous (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  title      VARCHAR(200) NOT NULL,
  category   VARCHAR(50)  NOT NULL,
  content    TEXT         DEFAULT NULL,
  file_path  VARCHAR(500) DEFAULT NULL,
  file_type  VARCHAR(20)  DEFAULT NULL,
  created_at TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT chk_misc_has_content
    CHECK (file_path IS NOT NULL OR content IS NOT NULL),
  CONSTRAINT chk_misc_file_type
    CHECK (file_type IN ('file', 'image', 'text', 'link')),
  INDEX idx_misc_category (category)
);

CREATE TABLE deadlines (
  id           INT AUTO_INCREMENT PRIMARY KEY,
  subject_id   INT          DEFAULT NULL,
  misc_id      INT          DEFAULT NULL,
  title        VARCHAR(200) NOT NULL,
  description  TEXT         DEFAULT NULL,
  due_date     DATETIME     NOT NULL,
  is_completed BOOLEAN      DEFAULT FALSE NOT NULL,
  created_at   TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT chk_deadline_one_owner
    CHECK (
      (subject_id IS NOT NULL AND misc_id IS NULL) OR
      (subject_id IS NULL     AND misc_id IS NOT NULL)
    ),
  FOREIGN KEY (subject_id) REFERENCES subjects(id),
  FOREIGN KEY (misc_id)    REFERENCES miscellaneous(id),
  INDEX idx_deadlines_subject (subject_id),
  INDEX idx_deadlines_misc    (misc_id),
  INDEX idx_deadlines_due     (due_date)
);

CREATE TABLE reminders (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  deadline_id   INT       NOT NULL,
  reminder_time DATETIME  NOT NULL,
  created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (deadline_id) REFERENCES deadlines(id) ON DELETE CASCADE,
  INDEX idx_reminders_time     (reminder_time),
  INDEX idx_reminders_deadline (deadline_id)
);
```

---

## Core Design Rules — Non-Negotiable

### 1. Path Derivation
The backend derives all file paths internally. The agent never receives, constructs, or stores file paths.

Path format for academic resources:
```
<DATA_DIR>/<SEMESTER>/<subject_name>/<category>/<title>
```
Example: `/data/SEM_3/DBMS/material/dbms_notes_unit_1_part1`

Path format for miscellaneous items:
```
<DATA_DIR>/MISCELLANEOUS/<category>/<title>
```
Example: `/data/MISCELLANEOUS/events/techfest_registration_2025`

**No file extensions.** Files are stored without extension. Content-Type is served from the `file_type` DB field.

### 2. Path Safety
Every derived path must be validated before any fs operation. If the resolved absolute path does not start with `path.resolve(DATA_DIR)`, throw an error immediately. This prevents path traversal attacks.

### 3. Atomic Subject Creation — DB Before FS
When creating a subject:
1. Insert into DB first
2. Create folders on disk second
3. If fs mkdir fails after DB insert succeeds → DELETE the DB row → throw error
4. Never leave an orphaned DB row without folders, or folders without a DB row

### 4. Multer Temp Flow — Exact Order
1. Multer writes uploaded file to `TEMP_DIR` with a random hex filename
2. Backend performs DB insert
3. If DB insert fails → delete temp file → return error
4. If DB insert succeeds → move temp file to derived final path
5. If file move fails → DELETE DB row → delete temp file → throw error
6. Never write directly to the final path via Multer

### 5. File Types
- `"file"` — catch-all for all binary formats (pdf, docx, pptx, zip, etc.)
- `"image"` — image files
- `"text"` — plain text content stored in DB `content` column
- `"link"` — URL stored in DB `content` column

For `"file"` and `"image"`: binary is stored on disk, `file_path` is populated, `content` is NULL.
For `"text"` and `"link"`: string is stored in `content` column, `file_path` is NULL, no disk write.

### 6. resource_type is Free-Form
The `resource_type` field is a plain string. No enum, no CHECK constraint, no validation against a fixed list. Store and return it exactly as the agent sends it.

### 7. Failures Are Always Loud
Never swallow errors silently. Every failure returns a response with the exact error code defined in this document. Log every error server-side.

### 8. Separation of Concerns
- Route files: input validation and calling services only
- Service files: all business logic, DB queries, fs operations
- No DB queries in route files
- No fs operations in route files

---

## Service: `src/services/db.js`

Export a single mysql2 connection pool.

```javascript
const pool = mysql.createPool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT, 10),
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
  timezone: '+00:00',
});
```

All queries use `pool.execute()`. Never use `pool.query()`.

---

## Service: `src/services/fileSystem.js`

Export the following functions. Implement every one completely.

```
deriveResourcePath(semester, subjectName, category, title) → string
deriveMiscPath(category, title) → string
assertWithinDataDir(resolvedPath) → throws if path escapes DATA_DIR
createSubjectFolders(semester, subjectName) → creates subject dir + 4 subcategories
ensureMiscCategoryFolder(category) → creates category dir, returns boolean (true if newly created)
moveTempToFinal(tempPath, finalPath) → moves file, creates parent dirs
deleteTempFile(tempPath) → deletes silently, never throws
deleteStoredFile(filePath) → deletes, throws on failure
getFileStream(filePath, fileType) → validates path exists, returns { contentType }
```

The four fixed subcategories created with every subject: `material`, `questions`, `info`, `miscellaneous`.

Content-Type mapping:
- `"file"` → `"application/octet-stream"`
- `"image"` → `"image/jpeg"`

---

## Middleware: `src/middleware/upload.js`

Multer with `diskStorage`. Destination is `process.env.TEMP_DIR`. Filename is `crypto.randomBytes(16).toString('hex')` — no extension.

---

## Middleware: `src/middleware/errorHandler.js`

Four-argument Express error handler `(err, req, res, next)`. Must:
- Log the error server-side
- If `req.file` exists, attempt to delete `req.file.path` (cleanup orphaned temp file)
- Return exactly: `{ "error": { "code": "INTERNAL_ERROR", "message": "..." } }` with status 500
- Never expose stack traces in the response body

---

## `src/server.js`

On startup:
1. Load `.env` via `require('dotenv').config()`
2. Create `DATA_DIR` directory recursively if it doesn't exist
3. Create `TEMP_DIR` directory recursively if it doesn't exist
4. Create `DATA_DIR/MISCELLANEOUS` directory recursively if it doesn't exist
5. Test DB connection — get a connection from the pool and release it
6. If any startup step fails → log error → `process.exit(1)`
7. Start Express server on `process.env.PORT || 3000`

---

## `src/app.js`

Mount all routes under `/api/v1`. Register error handler last.

Route order matters — register specific static routes before dynamic `:id` routes within the same router file.

---

## Error Response Format

All error responses follow this exact shape:
```json
{ "error": { "code": "ERROR_CODE", "message": "Human readable description" } }
```

---

## All Error Codes — Use Exactly These

| Code | HTTP Status | When |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Missing or invalid input fields |
| `SUBJECT_NOT_FOUND` | 404 | Subject lookup by name+semester fails |
| `SUBJECT_EXISTS` | 409 | Subject with same name+semester already exists |
| `ALREADY_ARCHIVED` | 400 | All subjects for semester already archived |
| `TITLE_EXISTS` | 409 | Title collision on save |
| `NOT_FOUND` | 404 | Any record lookup by ID fails |
| `DISK_DELETE_FAILED` | 500 | File could not be deleted from disk |
| `DB_DELETE_FAILED` | 500 | DB row could not be deleted (after disk delete succeeded) |
| `INTERNAL_ERROR` | 500 | Unhandled / unexpected errors |

---

## Endpoint Specifications

### `POST /api/v1/setup/semester`

**Request body (JSON):**
```json
{ "semester": "SEM_3", "subjects": ["DBMS", "OS", "DAA", "CN"] }
```

**Response 200:**
```json
{ "semester": "SEM_3", "created": ["DBMS", "OS", "DAA", "CN"], "skipped": [] }
```

**Behavior:**
- For each subject in the array:
  - Check DB — if `(name, semester)` already exists → add to `skipped`, continue
  - DB insert first, then `createSubjectFolders`
  - If fs fails → roll back DB insert → throw
  - If DB insert throws `ER_DUP_ENTRY` → add to `skipped`, continue
  - If everything succeeds → add to `created`
- Ensure semester-level folder exists (idempotent mkdir)
- Return `created` and `skipped` arrays

---

### `POST /api/v1/subjects`

**Request body (JSON):**
```json
{ "semester": "SEM_3", "name": "Machine Learning" }
```

**Response 201:**
```json
{ "subject_id": 12, "name": "Machine Learning", "semester": "SEM_3" }
```

**Errors:** `SUBJECT_EXISTS` 409

**Behavior:** Same atomic DB-before-fs pattern. Return 409 if duplicate.

---

### `PATCH /api/v1/subjects/archive`

**Request body (JSON):**
```json
{ "semester": "SEM_3" }
```

**Response 200:**
```json
{ "semester": "SEM_3", "archived_count": 4 }
```

**Errors:** 404 if no subjects found for semester. `ALREADY_ARCHIVED` 400 if every subject for that semester already has `archived = TRUE`.

**Behavior:**
- Fetch all subjects for semester
- If none found → 404
- If all have `archived = TRUE` → 400 `ALREADY_ARCHIVED`
- Run `UPDATE subjects SET archived = TRUE WHERE semester = ?`
- Return total count of subjects for that semester as `archived_count`
- Folders on disk are NOT touched

**Important:** This route must be registered as `router.patch('/archive', ...)` before any `router.patch('/:id', ...)` to prevent Express routing `/archive` as a dynamic param.

---

### `GET /api/v1/subjects?semester=SEM_3`

**Response 200:**
```json
{
  "subjects": [
    { "id": 7, "name": "DBMS", "archived": false },
    { "id": 8, "name": "OS", "archived": false }
  ]
}
```

`semester` query param is required. Return `archived` as boolean (not 0/1).

---

### `GET /api/v1/subjects/:id/resource-types`

**Response 200:**
```json
{
  "subject_id": 7,
  "resource_types": [
    { "resource_type": "notes", "category": "material" },
    { "resource_type": "previous_year_paper", "category": "questions" }
  ]
}
```

Query: `SELECT DISTINCT resource_type, category FROM resources WHERE subject_id = ?`

Return `SUBJECT_NOT_FOUND` 404 if subject ID doesn't exist.

---

### `GET /api/v1/subjects/:id/resources?category=material`

**Response 200:**
```json
{
  "subject_id": 7,
  "category": "material",
  "resources": [
    { "id": 42, "title": "dbms_notes_unit_1_part1", "file_type": "file" },
    { "id": 43, "title": "dbms_notes_unit_1_part2", "file_type": "file" }
  ]
}
```

`category` query param is required. Return `SUBJECT_NOT_FOUND` 404 if subject doesn't exist.

---

### `POST /api/v1/resources`

**Request:** `multipart/form-data`

| Field | Type | Required | Notes |
|---|---|---|---|
| semester | string | yes | |
| subject_name | string | yes | |
| category | string | yes | must be: `material`, `questions`, `info`, `miscellaneous` |
| title | string | yes | agent-constructed string |
| resource_type | string | yes | free-form, no validation |
| file_type | string | yes | must be: `file`, `image`, `text`, `link` |
| file | binary | if file_type is `file` or `image` | |
| content | string | if file_type is `text` or `link` | |

**Response 201:**
```json
{ "resource_id": 42, "subject_id": 7 }
```

**Errors:** `SUBJECT_NOT_FOUND` 404, `TITLE_EXISTS` 409

**Behavior:**
1. Validate all required fields
2. Validate `category` is one of the four allowed values
3. Validate `file_type` is one of the four allowed values
4. If `file_type` is `file`/`image` and no file uploaded → 400
5. If `file_type` is `text`/`link` and no `content` → 400 (clean up temp file if present)
6. Look up subject by `(subject_name, semester)` → `SUBJECT_NOT_FOUND` if missing
7. Check title uniqueness → `TITLE_EXISTS` if collision
8. Derive `finalPath` for `file`/`image` types — NULL for `text`/`link`
9. DB insert with `file_path = finalPath` (or NULL) and `content` (or NULL)
10. On DB failure → delete temp file → return error
11. On DB success + temp file present → `moveTempToFinal`
12. On move failure → roll back DB insert → delete temp file → throw

---

### `GET /api/v1/resources/:id`

**Response 200:**
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

Query must JOIN subjects to get `subject_archived`. Return `subject_archived` as boolean.
`content` field: return actual content string only if `file_type` is `text` or `link`. Otherwise return `null` even if DB has a value.

---

### `GET /api/v1/resources/:id/file`

Streams the binary file.

**Behavior:**
- Return 404 if resource not found
- Return 404 if `file_type` is `text` or `link` (no binary exists)
- Set `Content-Type` header from file_type mapping
- Set `Content-Disposition: attachment; filename="<title>"`
- Stream file using `fs.createReadStream`
- Handle stream error event → call `next(err)`

---

### `PUT /api/v1/resources/:id`

**Request:** `multipart/form-data` — at least one field required

| Field | Type |
|---|---|
| title | string (optional) |
| file | binary (optional) |
| content | string (optional) |
| file_type | string (optional) |

**Response 200:**
```json
{ "resource_id": 42, "updated": true }
```

**Errors:** 400 if no fields provided, 404 if not found

**Behavior:**
- Fetch current record (JOIN subjects for semester + subject_name)
- Build UPDATE query from provided fields only
- If new file provided: derive new path using `(semester, subject_name, category, newTitle)`, add `file_path` to update
- Run DB update
- If file provided: `moveTempToFinal` to new path
- If old file_path differs from new → attempt to delete old file (non-fatal if it fails)
- If file move fails → delete temp file → throw

---

### `DELETE /api/v1/resources/:id`

**Response 200:**
```json
{ "resource_id": 42, "deleted": true }
```

**Errors:** 404 if not found, `DISK_DELETE_FAILED` 500, `DB_DELETE_FAILED` 500

**Behavior — order matters:**
1. Fetch record
2. If `file_type` is `file` or `image` and `file_path` exists → delete from disk first
3. If disk delete throws → return `DISK_DELETE_FAILED` 500 immediately (do NOT attempt DB delete)
4. Delete from DB
5. If DB delete throws → return `DB_DELETE_FAILED` 500 (disk is already deleted — agent needs to know about this mismatch)

---

### `POST /api/v1/miscellaneous`

**Request:** `multipart/form-data`

| Field | Type | Required |
|---|---|---|
| title | string | yes |
| category | string | yes |
| file_type | string | yes |
| file | binary | if file_type is `file` or `image` |
| content | string | if file_type is `text` or `link` |

**Response 201:**
```json
{ "misc_id": 15, "category_created": true }
```

`category_created` is `true` if the category folder was newly created, `false` if it already existed.

**Errors:** `TITLE_EXISTS` 409

**Behavior:**
1. Validate fields
2. Check title uniqueness within category
3. Call `ensureMiscCategoryFolder(category)` — capture boolean return value
4. Derive `finalPath` for file/image, NULL for text/link
5. DB insert first
6. On DB failure → delete temp file → return error
7. On DB success → move temp file if applicable
8. On move failure → roll back DB insert → delete temp → throw
9. Return `{ misc_id, category_created }`

---

### `GET /api/v1/miscellaneous/categories`

**Response 200:**
```json
{ "categories": ["events", "scholarships", "timetable_changes"] }
```

Query: `SELECT DISTINCT category FROM miscellaneous ORDER BY category ASC`

**Important:** This route must be registered before `GET /miscellaneous/:id` in the router to prevent Express matching "categories" as an ID param.

---

### `GET /api/v1/miscellaneous?category=events`

**Response 200:**
```json
{
  "category": "events",
  "items": [
    { "id": 15, "title": "techfest_registration_2025", "file_type": "link" }
  ]
}
```

`category` query param is required.

**Important:** This route (`GET /`) must also be registered before `GET /:id`.

---

### `GET /api/v1/miscellaneous/:id`

**Response 200:**
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

Return `content` only if `file_type` is `text` or `link`. Otherwise return `null`.

---

### `GET /api/v1/miscellaneous/:id/file`

Same streaming behavior as `GET /api/v1/resources/:id/file`. Return 404 for `text` or `link` types.

---

### `PUT /api/v1/miscellaneous/:id`

Same pattern as `PUT /api/v1/resources/:id`. At least one field required. 404 if not found.

---

### `DELETE /api/v1/miscellaneous/:id`

Same disk-then-DB pattern as `DELETE /api/v1/resources/:id`. FK `ON DELETE CASCADE` in the schema handles linked deadline rows automatically.

---

### `POST /api/v1/deadlines`

**Request body (JSON):**
```json
{
  "title": "CN Assignment Submission",
  "description": "Upload to classroom link",
  "due_date": "2025-01-17T22:00:00",
  "subject_id": 7
}
```
OR (for miscellaneous-linked deadline):
```json
{
  "title": "Hackathon Registration Closes",
  "due_date": "2025-01-20T23:59:00",
  "misc_id": 15
}
```

**Response 201:**
```json
{
  "deadline_id": 3,
  "reminders_created": ["2025-01-17T20:00:00", "2025-01-17T21:00:00"]
}
```

**Errors:**
- 400 if both `subject_id` and `misc_id` are provided
- 400 if neither `subject_id` nor `misc_id` is provided
- 400 if `due_date` is in the past
- 400 if `due_date` is not a valid datetime
- 404 if `subject_id` or `misc_id` does not exist

**Behavior:**
1. Validate presence of `title` and `due_date`
2. Validate exactly one of `subject_id` / `misc_id`
3. Parse `due_date` → if `NaN` → 400
4. If `due_date <= Date.now()` → 400
5. Verify owner exists in DB → 404 if not
6. Insert deadline row
7. Calculate reminder times: `due_date - 2 hours` and `due_date - 1 hour`
8. For each candidate: if `reminder_time > Date.now()` → insert reminder row
9. Collect and return the reminder times that were actually inserted as ISO strings

---

### `GET /api/v1/deadlines`

**Query params (all optional):**
```
subject_id   int
misc_id      int
from         datetime    defaults to NOW()
to           datetime
completed    boolean     defaults to false
```

**Response 200:**
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

Build the WHERE clause dynamically from provided params. Return `is_completed` as boolean. Order by `due_date ASC`.

---

### `PATCH /api/v1/deadlines/:id/complete`

**Response 200:**
```json
{ "deadline_id": 3, "is_completed": true }
```

Sets `is_completed = TRUE`. 404 if not found.

**Important:** Register `router.patch('/:id/complete', ...)` before `router.patch('/:id', ...)` to prevent route conflict.

---

### `PATCH /api/v1/deadlines/:id`

**Request body (JSON — send only fields that changed):**
```json
{ "due_date": "2025-01-19T22:00:00" }
```

**Response 200:**
```json
{ "deadline_id": 3, "updated": true }
```

**Behavior:**
- Accept `title`, `description`, `due_date` — update only provided fields
- If `due_date` is provided and valid:
  1. `DELETE FROM reminders WHERE deadline_id = ?`
  2. Recreate default reminders at new times (skip any in the past)
- 400 if no fields provided
- 404 if deadline not found

---

### `DELETE /api/v1/deadlines/:id`

**Response 200:**
```json
{ "deadline_id": 3, "deleted": true }
```

`ON DELETE CASCADE` in the schema handles linked reminders automatically. 404 if not found.

---

### `POST /api/v1/reminders`

**Request body (JSON):**
```json
{ "deadline_id": 3, "reminder_time": "2025-01-16T20:00:00" }
```

**Response 201:**
```json
{ "reminder_id": 8 }
```

**Errors:**
- 400 if `reminder_time` is after `due_date` of the linked deadline
- 404 if `deadline_id` not found

**Behavior:** Fetch deadline to get its `due_date`. Validate `reminder_time <= due_date`. Insert.

---

### `GET /api/v1/reminders/due`

**Query param:** `as_of` (datetime, optional, defaults to `NOW()`)

**Response 200:**
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

**Query:**
```sql
SELECT r.id AS reminder_id, r.deadline_id, r.reminder_time,
       d.title AS deadline_title, d.due_date
FROM reminders r
JOIN deadlines d ON d.id = r.deadline_id
WHERE r.reminder_time <= ?
  AND d.is_completed = FALSE
ORDER BY r.reminder_time ASC
```

**Important:** Register `router.get('/due', ...)` before `router.delete('/:id', ...)` to prevent "due" being matched as an ID.

---

### `DELETE /api/v1/reminders/:id`

**Response 200:**
```json
{ "reminder_id": 8, "deleted": true }
```

404 if not found.

---

### `GET /api/v1/storage/status`

**Response 200:**
```json
{ "available_gb": 4.2, "status": "warning" }
```

**Behavior:**
- Call `checkDiskSpace(path.resolve(process.env.DATA_DIR))`
- `check-disk-space` exports `.default` — import as `const checkDiskSpace = require('check-disk-space').default`
- `available_gb` = `diskInfo.free / (1024 * 1024 * 1024)` rounded to 2 decimal places
- Status thresholds:
  - `"critical"` — free < 3 GB
  - `"warning"` — free < 5 GB
  - `"ok"` — free >= 5 GB

---

## Route Ordering Rules — Critical

Express matches routes in registration order. Get these wrong and requests will hit the wrong handler.

### In `subjects.js`:
Register in this order:
1. `POST /` 
2. `PATCH /archive` ← must come before any `/:id` route
3. `GET /`
4. `GET /:id/resource-types`
5. `GET /:id/resources`

### In `miscellaneous.js`:
Register in this order:
1. `GET /categories` ← must come before `GET /:id`
2. `GET /` ← must come before `GET /:id`
3. `POST /`
4. `GET /:id`
5. `GET /:id/file`
6. `PUT /:id`
7. `DELETE /:id`

### In `deadlines.js`:
Register in this order:
1. `POST /`
2. `GET /`
3. `PATCH /:id/complete` ← must come before `PATCH /:id`
4. `PATCH /:id`
5. `DELETE /:id`

### In `reminders.js`:
Register in this order:
1. `GET /due` ← must come before `DELETE /:id`
2. `POST /`
3. `DELETE /:id`

---

## Datetime Handling

All datetimes stored in MySQL as `DATETIME`. When inserting, convert JS Date to MySQL-compatible string: `new Date(ms).toISOString().slice(0, 19).replace('T', ' ')` → produces `"2025-01-17 22:00:00"`.

Set pool timezone to `'+00:00'` to ensure consistent UTC handling.

---

## Boolean Handling from MySQL

MySQL returns BOOLEAN columns as `0`/`1` integers in Node.js. When returning these in responses, convert explicitly:
```javascript
archived: r.archived === 1 || r.archived === true
```

---

## Temp File Cleanup — Every Code Path

Every route that accepts a file upload must clean up the temp file in every failure path. There must be no code path where a temp file is left on disk after an error. Check every early return in your POST and PUT routes.

---

## What the Backend Never Does

- Never constructs a file path that it returns to the agent
- Never includes `file_path` in any API response
- Never auto-creates a subject during a resource save
- Never deletes anything without being explicitly called to do so
- Never swallows an error silently

---

## Verification Checklist Before Finishing

Go through this list and confirm every item is true before calling the implementation complete.

- [ ] All 26 endpoints are implemented
- [ ] `PATCH /subjects/archive` is registered before any `/:id` route in subjects router
- [ ] `GET /miscellaneous/categories` and `GET /miscellaneous/` are registered before `GET /miscellaneous/:id`
- [ ] `PATCH /deadlines/:id/complete` is registered before `PATCH /deadlines/:id`
- [ ] `GET /reminders/due` is registered before `DELETE /reminders/:id`
- [ ] Every error code in the error code table is used in the correct endpoint
- [ ] `file_path` never appears in any response body
- [ ] Temp file is deleted in every failure path of every upload route
- [ ] DB insert happens before fs operation in every subject/resource/miscellaneous creation
- [ ] DB row is rolled back if fs operation fails after insert
- [ ] `resource_type` has no validation against a fixed list anywhere
- [ ] `archived` and `is_completed` are returned as booleans, not integers
- [ ] `content` field returns `null` for `file` and `image` types in metadata responses
- [ ] `check-disk-space` is imported as `.default`
- [ ] `server.js` exits with code 1 if DB connection or directory setup fails on startup
- [ ] Error handler is the last middleware registered in `app.js`
