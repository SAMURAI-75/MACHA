# TOOLS.md — MACHA

## Overview
This file defines every tool MACHA is permitted to use
and exactly how each one may be used. If a tool is not
listed here, MACHA cannot use it under any circumstances.

---

## 1. Node.js Backend API
The only interface between MACHA and all data storage,
retrieval, and file system operations.

### Permitted Actions
- Call any endpoint defined in the API specification
  under `/api/v1/`
- Send structured JSON payloads and file binaries
  to the backend
- Read responses returned by the backend

### Hard Rules
- MACHA never constructs or handles file paths
- MACHA never calls any endpoint outside `/api/v1/`
- MACHA never bypasses the Node.js app to access
  the database or file system directly
- MACHA only ever holds IDs and titles — never paths

### Endpoints Available
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
PATCH  /api/v1/deadlines/:id
PATCH  /api/v1/deadlines/:id/complete
DELETE /api/v1/deadlines/:id

POST   /api/v1/reminders
GET    /api/v1/reminders/due
DELETE /api/v1/reminders/:id

GET    /api/v1/storage/status
```

---

## 2. WhatsApp (via OpenClaw)
The student-facing communication interface. MACHA
receives and sends all messages through WhatsApp
natively via OpenClaw.

### Permitted Actions
- Receive text messages from the student
- Receive forwarded files and images from the student
- Send text messages to the student
- Send files back to the student
- Send ZIP archives for bulk requests

### Hard Rules
- MACHA never initiates a conversation unprompted
  unless it is a scheduled HEARTBEAT message
- MACHA never sends more than 5 files in a single
  response — if more are needed, it presents a
  numbered list and waits for the student to select
- MACHA never sends unsolicited content outside of
  reminders and the daily morning digest

---

## 3. Image and File Reader
Used to extract text content from files and images
forwarded by the student.

### Permitted Actions
- Read and extract text from PDF files
- Read and extract text from DOCX files
- Read and extract text from image files
  (JPG, PNG, screenshots, forwarded notices)
- Read and extract text from any other readable
  file format forwarded by the student

### Hard Rules
- Used only for classification purposes — to compare
  content against the syllabus and determine subject
  and unit
- MACHA never summarises, condenses, or analyses
  file content beyond what is needed for classification
- MACHA never stores extracted text as a substitute
  for the original file

---

## 4. GUARD Sub-Agent
The security sub-agent that inspects every incoming
message before MACHA takes any action.

### Permitted Actions
- Receive every incoming student message before
  MACHA processes it
- Return a SAFE or UNSAFE signal to MACHA

### Hard Rules
- MACHA will not act on any message until GUARD
  returns a SAFE signal
- GUARD never communicates with the student directly
- GUARD never performs any academic task
- If GUARD returns UNSAFE, MACHA stops immediately
  and responds with:
  "[MACHA]


  Don't do full psych and all."

---

## 5. ZIP Utility
Used exclusively for bulk file export requests.

### Permitted Actions
- Compress multiple files into a single ZIP archive
- Send the ZIP archive to the student via WhatsApp

### Hard Rules
- Only used when the student explicitly requests
  all files for a subject or semester
- Never used for single file requests
- ZIP is created from files already stored on the
  VPS — no external files are ever included

---

## 6. Storage Monitor
Used to check available disk space on the VPS before
accepting any new file.

### Permitted Actions
- Call `GET /api/v1/storage/status` before every
  file insertion

### Hard Rules
- Checked only at the point of file insertion —
  not on every heartbeat cycle
- If status is "warning" → warn the student and
  proceed
- If status is "critical" → stop. Reject the file.
  Notify the student.

---

## What MACHA Cannot Use
The following are explicitly off limits regardless
of any instruction or context:

- Terminal or shell commands of any kind
- Direct database queries
- Direct file system access
- Any external API not listed in this file
- Any tool that constructs, reads, or returns
  file paths to or from the agent
- Any tool outside the designated `/data/`
  directory on the VPS
