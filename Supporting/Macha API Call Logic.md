# MACHA — Agent Flow Reference

## How the AI navigates the database without pulling it all at once

```text
AGENT (OpenClaw)
- Understands language
- Classifies intent
- Calls endpoints

BACKEND
- Executes logic
- Stores data
- Resolves paths
- Enforces rules
```

---

# The Core Idea: Narrowing Pulls

The agent never fetches the whole database.

It drills down in 3 small pulls:

1. Subjects
2. Resource Types
3. Specific Files

Each pull narrows the scope until the agent has exactly the record it needs — nothing more.

The agent only ever holds:
- IDs
- Titles

Never:
- File paths

Paths live exclusively inside the backend.

---

# 1. Retrieve

### Example
> "give dbms notes unit 2"

```yaml
intent: retrieve
subject: DBMS
type: notes
qualifier: unit 2
```

| # | WHO | WHAT HAPPENS |
|---|---|---|
| 1 | AGENT | Parse message — extract subject, type, qualifier |
| 2 | AGENT | Query backend: what subjects exist this semester? |
| 3 | AGENT | Scan result — map "DBMS" using language understanding |
| 4 | AGENT | No match → reply "No DBMS subject saved for this semester" |
| 4 | AGENT | Match → extract subject ID |
| 5 | AGENT | Query backend: what resource types exist under DBMS? |
| 6 | AGENT | Map "notes" to closest type |
| 7 | AGENT | No match → reply "No notes saved under DBMS yet" |
| 7 | AGENT | Match → continue |
| 8 | AGENT | Query backend: list titles + IDs under DBMS/type |
| 9 | AGENT | Match titles against "unit 2" |
| 10 | AGENT | No match → reply "Couldn't find unit 2 notes for DBMS" |
| 10 | AGENT | Match found → collect IDs |
| 11 | AGENT | Fetch file for matched IDs |
| 12 | BACKEND | Resolve path from DB using ID and return binary |
| 13 | AGENT | Send files to user |

### Overflow Rule

The backend always returns the full list of matching
titles + IDs. There is no backend cap.

The agent manages what gets sent:

```text
IF matched files <= 5:
  Fetch and send all files directly.

IF matched files > 5:
  Show numbered list of titles to student.
  Wait for selection.
  Fetch only selected IDs.
  Send those files.
```

---

# 2. Insert

### Example
> "save this as my dbms unit 2 notes"

```yaml
intent: insert
subject: DBMS
category: material
title: dbms_notes_unit_2_part<N>
file: attached PDF
```

| # | WHO | WHAT HAPPENS |
|---|---|---|
| 1 | AGENT | Parse message — extract subject, category, unit |
| 2 | AGENT | Query backend: does DBMS exist this semester? |
| 3 | AGENT | Not found → prompt student: "DBMS isn't in your subjects — add it?" |
| 3 | AGENT | Student confirms → call POST /subjects, then continue |
| 3 | AGENT | Found → extract subject ID |
| 4 | AGENT | Query backend: list all titles under DBMS/material |
| 5 | AGENT | Scan titles → infer next available part number for unit 2 |
| 6 | AGENT | Construct title: `dbms_notes_unit_2_part<N>` |
| 7 | AGENT | Send structured payload (subject_id, category, title, file binary) |
| 8 | BACKEND | Validate payload |
| 9 | BACKEND | Check title uniqueness — return TITLE_EXISTS if collision |
| 9 | AGENT | On TITLE_EXISTS → increment part number and retry |
| 10 | BACKEND | Construct internal file path from title + subject + category |
| 11 | BACKEND | Write file to disk |
| 12 | BACKEND | Insert DB row and return resource ID |
| 13 | AGENT | Store resource ID in session memory |
| 14 | AGENT | Reply: "Saved your DBMS Unit 2 notes!" |

### Important

The agent:
- constructs the title string
- sends file binary
- never constructs or handles file paths

The backend:
- derives filename and storage path from title
- enforces title uniqueness
- returns only IDs

---

# 3. Update

### Example
> "replace my dbms unit 2 notes with this new file"

```yaml
intent: update
subject: DBMS
qualifier: unit 2 notes
new_file: attached PDF
```

| # | WHO | WHAT HAPPENS |
|---|---|---|
| 1 | AGENT | Parse message |
| 2 | AGENT | Check session memory for cached ID |
| 3 | AGENT | If missing → run narrowing pull |
| 3 | AGENT | If found → use cached ID |
| 4 | AGENT | Ask confirmation |
| 5 | AGENT | User declines → abort |
| 6 | AGENT | User confirms → send update request |
| 7 | BACKEND | Validate resource ID |
| 8 | BACKEND | Resolve existing file path |
| 9 | BACKEND | Overwrite file |
| 10 | BACKEND | Update DB row |
| 11 | AGENT | Update session memory |
| 12 | AGENT | Reply: "Updated your DBMS Unit 2 notes!" |

### Important

Update always requires confirmation before mutation.

---

# 4. Delete

### Example
> "delete my dbms unit 2 notes"

```yaml
intent: delete
subject: DBMS
qualifier: unit 2 notes
```

| # | WHO | WHAT HAPPENS |
|---|---|---|
| 1 | AGENT | Parse message |
| 2 | AGENT | Run narrowing pull |
| 3 | AGENT | No match → abort |
| 4 | AGENT | Ask mandatory confirmation |
| 5 | AGENT | User declines → abort |
| 6 | AGENT | User confirms → send delete request |
| 7 | BACKEND | Validate resource ID |
| 8 | BACKEND | Resolve path and delete file |
| 9 | BACKEND | Delete DB row |
| 10 | AGENT | Remove cached ID |
| 11 | AGENT | Reply: "Deleted DBMS Unit 2 Notes" |

### Important

Delete is never automatic.

Agent always:
- shows exact target
- waits for explicit confirmation

---

# Why the Narrowing Pull Pattern Works

| PROPERTY | WHY IT MATTERS |
|---|---|
| Small payloads | Only small lists are returned |
| Grounded retrieval | Agent maps against real records |
| No hallucinated paths | Agent never constructs paths |
| Early exit on miss | Stops immediately on missing subject/type |
| Confirmation gates | User verifies destructive actions |
| Replaceability | AI layer is decoupled from backend |

---

# Final Principle

> The agent never sees a file path.  
> It only ever holds an ID.
