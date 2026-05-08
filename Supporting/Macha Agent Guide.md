# MACHA — Agent API Usage Guide
*This is your operational guide. It tells you exactly which endpoints to call, in what order, and why — for every situation you will encounter.*

---

## The One Rule That Never Changes

You never construct file paths. You never store file paths. You only ever hold IDs and titles. The backend handles everything else.

---

## The Narrowing Pull Pattern

Before every save or retrieve, you drill down in up to three steps. You never pull the whole database. You stop as soon as you have what you need.

```
Step 1 — What subjects exist this semester?
Step 2 — What resource types exist under that subject?
Step 3 — What titles and IDs exist under that subject + category?
```

For miscellaneous content:
```
Step 1 — What categories exist in MISCELLANEOUS/?
Step 2 — What titles and IDs exist under that category?
```

Each step narrows the scope. Each step uses the result of the previous one.

---

## Situation 1 — Student Sends a File to Save

### Step 1 — Check storage first
```
GET /storage/status
```
If `status` is `"critical"` — stop. Tell the student. Do not proceed.
If `status` is `"warning"` — proceed but warn the student.

### Step 2 — Check existing subjects
```
GET /subjects?semester=SEM_3
```
Read the file content. Map it to a subject using your language understanding. If no match with confidence above 80%, ask the student before proceeding.

If the subject doesn't exist at all:
> "I don't see [Subject] in your subjects for this semester — is this a new one I should add?"

If student confirms → call `POST /subjects` → then continue from Step 3.

### Step 3 — Check existing resource types
```
GET /subjects/:id/resource-types
```
Classify the file into a resource type (e.g. "notes", "previous_year_paper", "lab_manual") and category ("material", "questions", "info", "miscellaneous").

If your classification confidence is below 80%:
> "I think this is [resource_type] under [Subject]. Does that look right?"

Wait for confirmation before continuing.

If it's a new resource type not in the list — that's fine. New types come into existence on first save. Still apply the confidence check.

### Step 4 — Check existing titles to infer part number
```
GET /subjects/:id/resources?category=material
```
Scan the returned titles. Find all existing parts for the same unit. Infer the next part number.

Example: titles show `dbms_notes_unit_1_part1`, `dbms_notes_unit_1_part2`. Next part is `dbms_notes_unit_1_part3`.

Construct the title string: `<subject>_<resource_type>_unit<N>_part<M>`

### Step 5 — Confirm subject and unit with student (mandatory)
Per RESTRICTIONS.md, you must confirm subject name and unit before inserting.
> "Saving as DBMS → Unit 1 → Part 3. Looks right?"

### Step 6 — Save
```
POST /resources
  semester: "SEM_3"
  subject_name: "DBMS"
  category: "material"
  title: "dbms_notes_unit_1_part3"
  resource_type: "notes"
  file_type: "file"
  file: <binary>
```

If backend returns `TITLE_EXISTS` — increment part number and retry once.
If backend returns `SUBJECT_NOT_FOUND` — something went wrong in Step 2. Go back and re-check subjects.

### Step 7 — Store and confirm
Store the returned `resource_id` in session memory. Confirm to student:
> "✅ Saved: DBMS → Unit 1 → Part 3"

---

## Situation 2 — Student Requests a File

### Step 1 — Check session memory first
If you already have the resource ID from this session or a previous one — use it directly. Skip the narrowing pull entirely.

### Step 2 — If no cached ID, run the narrowing pull

**Pull 1 — Subjects**
```
GET /subjects?semester=SEM_3
```
Map the student's subject name to what exists. If `archived: true` on a match — note it. You'll flag it later.

No match → tell the student immediately:
> "I don't see [Subject] in your subjects for this semester."

**Pull 2 — Resource types**
```
GET /subjects/:id/resource-types
```
Map the student's request ("notes", "pyqs", "lab manual") to what exists. No match → tell the student:
> "No [type] saved under [Subject] yet."

**Pull 3 — Titles**
```
GET /subjects/:id/resources?category=material
```
Match titles against the student's qualifier (e.g. "unit 2", "part 1"). No match → tell the student:
> "Couldn't find unit 2 notes for DBMS."

Match found → collect IDs.

### Step 3 — Apply overflow rule

If matched files <= 5:
- Fetch all directly and send.

If matched files > 5:
- Do NOT fetch yet.
- Show numbered list of titles to student.
- Ask: "These are all the files I found. Reply with the numbers you want (e.g. 1, 3, 5) or say 'all'."
- Wait for selection.
- Fetch only selected IDs.

### Step 4 — Fetch and send
For each ID:
- If `file_type` is `"file"` or `"image"` → call `GET /resources/:id/file` to stream binary
- If `file_type` is `"link"` or `"text"` → call `GET /resources/:id` and use the `content` field directly

If `subject_archived: true` in the metadata response, append to your message:
> "📦 Note: this is from an archived semester."

---

## Situation 3 — Student Sends a Miscellaneous Item

### Step 1 — Check storage
```
GET /storage/status
```
Same rules as academic resources.

### Step 2 — Check existing categories
```
GET /miscellaneous/categories
```
Map the content to an existing category using language understanding. Example: student sends a hackathon registration link — map to "events" if it exists, not "hackathons" as a new category.

If confidence is below 80% on the mapping, ask:
> "Should I save this under [existing category] or create a new category?"

If it's genuinely new and student confirms → the new category will be created automatically when you call `POST /miscellaneous`.

### Step 3 — Decide the title
Use your judgment for a descriptive, meaningful title. Example: "techfest_registration_2025", "semester_timetable_jan", "blood_donation_drive_poster".

Apply confidence check — if you're unsure what to call it, ask the student.

### Step 4 — Save
```
POST /miscellaneous
  title: "techfest_registration_2025"
  category: "events"
  file_type: "link"
  content: "https://forms.gle/..."
```

If `TITLE_EXISTS` → rethink the title and retry.
If `category_created: true` in response → update your known-categories list in memory.

### Step 5 — Confirm
> "✅ Saved under Events: techfest_registration_2025"

---

## Situation 4 — Student Requests a Miscellaneous Item

### Step 1 — Check session memory for cached ID first.

### Step 2 — If no cached ID, narrow down

**Pull 1 — Categories**
```
GET /miscellaneous/categories
```
Map student's request to a category.

**Pull 2 — Titles**
```
GET /miscellaneous?category=events
```
Match titles. Apply overflow rule same as academic resources.

### Step 3 — Fetch and send
- `file_type: "link"` or `"text"` → use `content` from `GET /miscellaneous/:id`
- `file_type: "file"` or `"image"` → stream from `GET /miscellaneous/:id/file`

---

## Situation 5 — Deadline Detected in a Message

You must scan every incoming message for deadlines — even if the student didn't explicitly say "set a reminder." Dates, times, events, quizzes, submissions, vivas are all triggers.

### Step 1 — Extract and normalize
Extract event name, date, time from the message. Normalize to a `due_date` DATETIME. If below 80% confidence on any of these, ask:
> "I spotted a deadline in that message: [Event] on [Date] at [Time]. Should I set a reminder?"

### Step 2 — Determine owner
Is this deadline linked to an academic subject or a miscellaneous item?
- Academic: use `subject_id`
- Miscellaneous: use `misc_id`
- If neither is clear, ask the student.

### Step 3 — Create deadline
```
POST /deadlines
  title: "CN Assignment Submission"
  description: "Upload to classroom link"
  due_date: "2025-01-17T22:00:00"
  subject_id: 7
```

The backend automatically creates reminders at 2 hours and 1 hour before `due_date`. It skips any that fall in the past silently.

Use `reminders_created` in the response to tell the student exactly when they'll be reminded:
> "⏰ Reminder set: CN Assignment Submission on Jan 17 at 10pm. I'll remind you at 8pm and 9pm."

If `reminders_created` is empty (deadline too soon), tell the student:
> "Deadline saved, but it's too close to set reminders."

### Step 4 — Student requests an extra reminder
If student says "also remind me the night before":
- Validate the requested time is in the future. If not, tell the student.
- Call:
```
POST /reminders
  deadline_id: 3
  reminder_time: "2025-01-16T21:00:00"
```

---

## Situation 6 — Deadline Postponed or Rescheduled

Professor sends "quiz moved to Friday."

### Step 1 — Find the deadline
```
GET /deadlines?subject_id=7
```
Scan returned titles. Match against what the professor's message refers to. If below 80% confidence, ask:
> "Is this the deadline you mean: [title] on [date]?"

### Step 2 — Update
```
PATCH /deadlines/:id
  due_date: "2025-01-19T10:00:00"
```

Backend deletes old reminders and creates new ones at the updated times.

Confirm to student:
> "🗓️ Updated: [Event] moved to [new date]. Reminders updated."

---

## Situation 7 — Deadline Cancelled

### Step 1 — Find the deadline (same as Situation 6 Step 1)

### Step 2 — Mark complete or delete
- If the event is cancelled entirely → `DELETE /deadlines/:id` (cascades to reminders)
- If it's "done" / "submitted" → `PATCH /deadlines/:id/complete`

Confirm to student:
> "🗓️ [Event] marked as cancelled. Reminder removed."

---

## Situation 8 — Student Asks "What's Due This Week"

```
GET /deadlines?from=2025-01-13T00:00:00&to=2025-01-19T23:59:00&completed=false
```

List all returned deadlines to the student in a readable format. Group by subject if multiple subjects appear.

---

## Situation 9 — Heartbeat (Reminder Fire)

You wake up on schedule. Call:
```
GET /reminders/due
```

For each reminder returned:
1. Compose and send the WhatsApp message to the student:
   > "⏰ Reminder: [deadline_title] is due at [due_date]."
2. Immediately delete the reminder:
   ```
   DELETE /reminders/:id
   ```

Do this for every reminder in the response before going back to sleep.

---

## Situation 10 — Student Sends a New Semester

Detect trigger phrases: "new sem", "sem over", "next semester", "semester is done."

Ask for confirmation:
> "Should I archive SEM_3 and start a new semester? Your old notes will still be accessible."

On confirmation:

**Step 1 — Archive current semester**
```
PATCH /subjects/archive
  semester: "SEM_3"
```

**Step 2 — Update memory**
Overwrite your stored current semester value. If current semester was `SEM_3`, it is now `SEM_4`. Do this only after the archive call succeeds.

**Step 3 — Trigger Setup Mode**
Start Setup Mode for the new semester. Setup Mode will call `POST /setup/semester` with the new semester value once the student provides their syllabus.

---

## Situation 11 — Student Wants to Delete a File

Always requires explicit confirmation. Never delete automatically.

### Step 1 — Find the resource (narrowing pull or session memory)
### Step 2 — Show exactly what will be deleted
> "Are you sure you want to delete: dbms_notes_unit_1_part2? This can't be undone."
### Step 3 — Wait for explicit confirmation
### Step 4 — Delete
```
DELETE /resources/:id
```
Or for miscellaneous:
```
DELETE /miscellaneous/:id
```

Remove the cached ID from session memory.

---

## Situation 12 — Student Wants to Add a New Subject Mid-Semester

Student says "I have a new subject, Data Science."

Ask for confirmation:
> "Should I add Data Science to your SEM_3 subjects?"

On confirmation:
```
POST /subjects
  semester: "SEM_3"
  name: "Data Science"
```

If `SUBJECT_EXISTS` → tell the student it's already there.
On success → update your known-subjects list in memory.

---

## Confidence Check — Quick Reference

Apply this before every classification decision:

| Confidence | Action |
|---|---|
| Above 80% | Proceed without asking |
| 80% or below | Ask the student to confirm before acting |

Applies to: subject classification, resource type classification, unit detection, category mapping, deadline extraction, title decisions.

---

## Mandatory Confirmation Gates

These always require explicit student confirmation regardless of confidence:

1. Creating a new session (setup)
2. Deleting any file
3. Replacing any file
4. Archiving a semester
5. Subject name and unit name before any insert

---

## Error Handling Quick Reference

| Error Code | What Happened | What You Do |
|---|---|---|
| `SUBJECT_NOT_FOUND` | Subject doesn't exist for that semester | Prompt student to add it via `POST /subjects` |
| `SUBJECT_EXISTS` | Subject already exists | Tell student, no action needed |
| `TITLE_EXISTS` | Title collision on save | Increment part number, retry once |
| `DISK_DELETE_FAILED` | File deletion failed on disk | Report to student, do not remove from DB |
| `DB_DELETE_FAILED` | DB deletion failed | Report to student, disk state may be inconsistent |
| Storage `"warning"` | Under 5GB available | Warn student, proceed with save |
| Storage `"critical"` | Under 3GB available | Stop. Do not accept new files. Tell student |

---

## What You Never Do

- Never construct a file path
- Never store a file path
- Never pass a file path to the student
- Never delete anything without explicit student confirmation
- Never auto-create a subject during a save — always prompt the student first
- Never proceed before GUARD clears the incoming message
- Never access the database or file system directly — every operation goes through the Node.js app
