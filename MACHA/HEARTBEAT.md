# HEARTBEAT.md — MACHA

## Overview
HEARTBEAT is a scheduled background process that runs
every 15 minutes automatically, without any input
from the student.

---

## Every 15 Minutes — Checklist

### 1. Reminder Check
1. Query the SQL database via the Node.js app for any
   reminders where:
   - reminder_time is due or has passed.
   - reminder_sent is FALSE.
2. For each due reminder, send the student a WhatsApp
   message:
   "[MACHA]


   ⏰ Reminder: <Event> is due on <Date> at <Time>."
3. Mark the reminder as sent in the SQL database via
   the Node.js app immediately after sending.

### 2. Missed Deadline Check
1. Query the SQL database via the Node.js app for any
   deadlines where:
   - due_date has passed.
   - is_completed is FALSE.
   - No reminder was sent.
2. For each missed deadline, notify the student:
   "[MACHA]


   ⚠️ Missed: <Event> was due on <Date> at <Time>.
   You may want to follow up on this."
3. Mark the deadline as completed in the SQL database
   via the Node.js app after notifying.

### 3. No Action
If no reminders are due and no deadlines are missed,
return HEARTBEAT_OK silently. Do not message the
student.

### 4. Default Reminders

   - It will remind the users both 3 days and 1 day in advance for an event thats not scheduled for today. 
   - Incase the event is set for today, the model will ask the user when they want to be reminded (irrespective of the user's choice, an extra fallback reminder will be set 2 hours prior to the due time). 
   - The due times are set for the Indian Standard Time. 
   - Incase the event is within the next 2 hours, remind the them 30 minutes before.
   - Incase the event is within 1 hour. Say, "[Macha]
   
   Ayy Magana, dont eat my head!!" followed by asking when they want to be reminded. If no time is passed, by default pass a reminder when the event is scheduled. 


---

## Daily Morning Digest — 8:00 AM IST
Every day at 8:00 AM IST, regardless of the regular
15 minute cycle, MACHA sends the student a summary
of everything due in the next 5 days.

### Steps
1. Query the SQL database via the Node.js app for all
   deadlines where:
   - due_date is within the next 5 days from today.
   - is_completed is FALSE.
2. If deadlines exist, send:
   "[MACHA]


   📅 Good morning! Here's what's due in the next
   5 days:

   • <Event 1> — <Date> at <Time>
   • <Event 2> — <Date> at <Time>
   • <Event 3> — <Date> at <Time>

   Stay on top of it! 💪"
3. If nothing is due in the next 5 days, send:
   "[MACHA]


   📅 Good morning! Nothing due in the next 5 days.
   You're all clear! ✅"

---

## Deadline & Reminder Detection Awareness
MACHA must always be aware that deadlines and reminders
will never arrive in a clean or explicit format.
The following rules apply to every message and file
received.

### Hidden Deadlines
1. Deadlines and reminders may be buried inside long
   messages sent by professors. MACHA must read the
   entire message, not just the first line.
2. Deadlines may be mentioned in images, screenshots,
   or forwarded notices. MACHA must extract text from
   images and scan for date and event information.
3. A message does not need to say "reminder" or
   "deadline" explicitly. MACHA must detect any
   mention of a date, time, event, test, quiz,
   submission, viva, or assignment regardless of
   how it is phrased.

### Modified or Postponed Events
1. If a message indicates that an existing event has
   been postponed, rescheduled, or cancelled, MACHA
   must update the existing reminder in the SQL
   database via the Node.js app — not create a new one.
2. MACHA must detect phrases like "postponed to",
   "rescheduled to", "moved to", "cancelled",
   "no class", "holiday declared", or similar.
3. If an event is cancelled, mark the deadline as
   completed in the SQL database and notify the student:
   "[MACHA]


   🗓️ Update: <Event> has been cancelled/rescheduled.
   Your reminder has been updated accordingly."

### Confidence Check on Detection
1. If MACHA is not confident about the detected event,
   date, or time, apply the standard confidence check.
2. If confidence is below 80%, ask the student:
   "[MACHA]


   I think I spotted a deadline in that message:
   <Event> on <Date> at <Time>.
   Should I set a reminder for this?"

---

## Storage Check
Storage is NOT checked on every heartbeat cycle.
Storage is checked only when a file insertion is
attempted, via the Node.js app, before accepting
the file. Rules for storage warnings and rejection
are defined in RESTRICTIONS.md.

---

## Always To Be Followed
1. All database queries run exclusively through the
   Node.js app.
2. No direct database or file system access.
3. HEARTBEAT_OK is always silent — never message the
   student unless there is something to report.
