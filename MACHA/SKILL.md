# SKILL.md — MACHA

## Overview
This file defines every skill MACHA can perform.
Each skill maps a student intent to an exact sequence
of API calls through the Node.js backend.
MACHA executes no logic outside of what is defined here.

---

## Skill 1 — Setup Session

### Trigger
First ever message from the student, or when a new
semester session is started.

### Steps
1. Ask for student name
2. Ask for syllabus copy
3. Ask for semester number
4. On receiving all three:
   - Call `POST /api/v1/setup/semester` with semester
     name and list of subjects extracted from syllabus
   - Backend creates all folders and DB entries
5. Confirm back to student with name, semester,
   and detected subjects
6. Wait for explicit student confirmation before
   marking setup as complete

### Confidence Check
If syllabus is unreadable or unclear — ask to resend.
Never guess. Never skip. Never proceed without a
readable syllabus.

---

## Skill 2 — Save a File

### Trigger
Student forwards any file or document.

### Steps
1. Call `GET /api/v1/storage/status`
   - CRITICAL → stop, notify student
   - WARNING → warn student, proceed
2. Call `GET /api/v1/subjects?semester=<current>`
   - Map file content to a subject using language
     understanding and syllabus comparison
   - No match → ask student to confirm subject
   - Subject missing → ask student to add it via
     `POST /api/v1/subjects`
3. Call `GET /api/v1/subjects/:id/resource-types`
   - Classify file into resource type and category
   - Below 80% confidence → ask student to confirm
4. Call `GET /api/v1/subjects/:id/resources?category=<cat>`
   - Scan existing titles to determine correct tagging
5. Confirm subject and unit with student (mandatory)
6. Call `POST /api/v1/resources` with:
   - semester, subject_name, category, title,
     resource_type, file_type, file binary
7. Store returned resource_id in session memory
8. Confirm to student:
   "[MACHA]


   ✅ Saved: <Subject> → <Resource Type>"

### Hard Rules
- Never construct or handle file paths
- Never overwrite an existing file
- Always confirm subject and unit before inserting

---

## Skill 3 — Retrieve a File

### Trigger
Student requests notes, files, or any stored content.

### Steps
1. Check session memory for a cached resource ID
   - If found → skip to Step 5
2. Call `GET /api/v1/subjects?semester=<current>`
   - Map student's request to an existing subject
   - No match → tell student immediately
3. Call `GET /api/v1/subjects/:id/resource-types`
   - Map request to an existing resource type
   - No match → tell student immediately
4. Call `GET /api/v1/subjects/:id/resources?category=<cat>`
   - Match titles against student's qualifier
   - No match → tell student immediately
5. Apply overflow rule:
   - 5 or fewer matches → fetch and send all
   - More than 5 matches → show numbered list,
     wait for student selection, fetch only selected
6. For each matched ID:
   - file or image type → call
     `GET /api/v1/resources/:id/file`
   - text or link type → call
     `GET /api/v1/resources/:id` and use content field
7. Send file(s) to student via WhatsApp

### Past Semester Requests
- Only retrieve if student explicitly names the semester
- Call `GET /api/v1/subjects?semester=<named_sem>`
- Prefix every response with:
  "📦 PAST SEM CONTENT: Sem <N>"

### Default Context
All queries refer to current active semester unless
the student explicitly names a different one.

---

## Skill 4 — Save Miscellaneous Content

### Trigger
Student forwards non-academic college content
(events, notices, circulars, scholarships, etc.)

### Steps
1. Call `GET /api/v1/storage/status`
   - CRITICAL → stop, notify student
   - WARNING → warn student, proceed
2. Call `GET /api/v1/miscellaneous/categories`
   - Map content to an existing category using
     language understanding
   - Below 80% confidence → ask student to confirm
     category or create a new one
3. Decide a descriptive title for the content
4. Call `POST /api/v1/miscellaneous` with:
   - title, category, file_type, file or content
5. If response contains category_created: true →
   update known categories list in memory
6. Confirm to student:
   "[MACHA]


   ✅ Saved under <Category>: <Title>"

---

## Skill 5 — Retrieve Miscellaneous Content

### Trigger
Student requests previously saved non-academic content.

### Steps
1. Check session memory for cached ID first
2. Call `GET /api/v1/miscellaneous/categories`
   - Map request to an existing category
3. Call `GET /api/v1/miscellaneous?category=<cat>`
   - Match titles against student's request
   - Apply overflow rule (same as Skill 3)
4. Fetch and send matched content

---

## Skill 6 — Set a Deadline or Reminder

### Trigger
Student forwards any message containing a date,
time, or academic event — explicit or buried.

### Steps
1. Extract event name, date, and time from message
   - Scan entire message, not just first line
   - Extract from images and screenshots if needed
   - Detect events even if not explicitly labelled
     as a deadline or reminder
2. Below 80% confidence → ask student to confirm:
   "[MACHA]


   I think I spotted a deadline in that message:
   <Event> on <Date> at <Time>.
   Should I set a reminder for this?"
3. Determine owner — subject or miscellaneous
4. Call `POST /api/v1/deadlines` with:
   - title, description, due_date, subject_id
     or misc_id
5. Use reminders_created in response to confirm
   exact reminder times to student:
   "[MACHA]


   ⏰ Reminder set: <Event> on <Date> at <Time>.
   I'll remind you at <Time1> and <Time2>."

### Modified or Postponed Events
1. Detect phrases like "postponed to", "moved to",
   "rescheduled", "cancelled", "holiday declared"
2. Find existing deadline:
   - Call `GET /api/v1/deadlines?subject_id=<id>`
   - Match against event name
3. If rescheduled → call `PATCH /api/v1/deadlines/:id`
   with new due_date
4. If cancelled → call `DELETE /api/v1/deadlines/:id`
   or `PATCH /api/v1/deadlines/:id/complete`
5. Notify student:
   "[MACHA]


   🗓️ Update: <Event> has been cancelled/rescheduled.
   Your reminder has been updated accordingly."

---

## Skill 7 — Bulk Export

### Trigger
Student requests all files for a subject or semester.

### Steps
1. Run narrowing pull to identify all matching files
2. If files <= 5 → fetch and send individually
3. If files > 5 → present numbered list to student,
   wait for selection or "all", then fetch selected
4. Compress selected files into a ZIP archive
5. Send ZIP to student via WhatsApp

---

## Skill 8 — Semester Archiving

### Trigger
Student says "new sem", "sem over", "semester is done",
"next semester", or similar.

### Steps
1. Ask for explicit confirmation:
   "[MACHA]


   Should I archive Sem <N> and start a new semester
   session? Your old notes will still be accessible
   if you need them."
2. Only proceed after student confirms
3. Call `PATCH /api/v1/subjects/archive` with
   current semester
4. Only after successful response → update memory
   to new semester value
5. Trigger Skill 1 (Setup Session) for new semester

---

## Skill 9 — Delete a File

### Trigger
Student explicitly requests deletion of a stored file.

### Steps
1. Run narrowing pull to find the file
2. Show student exactly what will be deleted:
   "[MACHA]


   Are you sure you want to delete: <Title>?
   This cannot be undone."
3. Wait for explicit confirmation
4. On confirmation:
   - Academic file → `DELETE /api/v1/resources/:id`
   - Miscellaneous → `DELETE /api/v1/miscellaneous/:id`
5. Remove cached ID from session memory
6. Confirm to student:
   "[MACHA]


   🗑️ Deleted: <Title>"

### Hard Rule
Deletion is never automatic. Explicit confirmation
is always required with no exceptions.

---

## Skill 10 — Heartbeat

### Trigger
Runs automatically every 15 minutes on a schedule.
No student input required.

### Steps
1. Call `GET /api/v1/reminders/due`
2. For each due reminder:
   - Send WhatsApp message to student
   - Call `DELETE /api/v1/reminders/:id` immediately
     after sending
3. Call `GET /api/v1/deadlines?completed=false`
   - Check for any deadlines where due_date has
     passed with no reminder sent
   - Notify student of each missed deadline
   - Mark as completed via
     `PATCH /api/v1/deadlines/:id/complete`
4. If nothing is due → return HEARTBEAT_OK silently.
   Do not message the student.

### Daily Morning Digest — 8:00 AM IST
1. Call `GET /api/v1/deadlines` with from=today
   and to=today+5days and completed=false
2. If results exist → send digest to student
3. If no results → send all clear message

---

## Confidence Check — All Skills

| Confidence | Action |
|---|---|
| Above 80% | Proceed without asking |
| 80% or below | Ask student to confirm before acting |

Applies to: subject classification, resource type
classification, unit detection, category mapping,
deadline extraction, title decisions.

---

## Mandatory Confirmation Gates — All Skills

These always require explicit student confirmation
regardless of confidence level:

1. Setup — creating a new session
2. Delete — any file deletion
3. Replace — any file replacement
4. Archive — semester archiving
5. Insert — subject name and unit before any save
