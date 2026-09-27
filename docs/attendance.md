# Attendance

## Overview

The attendance system is session-based. A lecturer creates a timed class session which generates a one-time QR code and optional NFC tag payload. Students check in by scanning the QR with their camera or tapping the NFC tag. The lecturer sees students appearing on a live roster in real time via Supabase Realtime.

## Roles

| Role | Access |
|---|---|
| Lecturer | Create sessions, view live roster, end sessions, view history and reports |
| Admin | Same as lecturer for all sessions |
| Student | Check in to any active session by scanning QR or tapping NFC |

## Lecturer Flow

### 1. Create a Session (`/attendance/create-session`)

The screen has two tabs: **Session** and **History**.

On the Session tab, the lecturer fills in:
- Course code (e.g. `BUS101`)
- Course name (e.g. `Business Studies`)
- Lecturer name (pre-filled from their profile)
- Duration: 30 min / 1h / 1.5h / 2h / 3h

Tapping **Start Session** calls the `create_attendance_session` Supabase RPC, which:
1. Generates a unique 32-hex-char token server-side
2. Inserts a row into `attendance_sessions` with `is_active = true` and an `end_time` calculated from the duration
3. Returns the full session row including `lecturer_id`, `token`, `start_time`, `end_time`

The session is cached locally in SQLite and displayed immediately.

### 2. QR Code Display

Once a session is active, a QR code is displayed on screen. The QR data is `ATT:<token>`. The lecturer can project their screen or let students scan directly from the phone.

### 3. NFC Tag Write (optional)

The lecturer can optionally write the session token to a physical NFC tag. Tapping **Write to NFC Tag** calls `NfcService.writeNfcCard()` which writes an NDEF Text Record with payload `ATT:<token>`. The tag can then be placed at the room door for tap-to-check-in.

### 4. Live Roster (`/attendance/live-dashboard`)

The live dashboard shows:
- Session header (course code, name, lecturer, countdown timer)
- Number of checked-in students
- Scrollable roster list — each row shows: method icon (QR/NFC), student name, student number, programme, and check-in time

The roster is powered by a Supabase Realtime `.stream()` subscription on `session_check_ins` filtered by `session_id`. Each new check-in from any student device appears within seconds without any manual refresh.

On app restart or navigation away and back, the controller's `build()` method automatically calls `fetchActiveSession()` from Supabase to recover the session, re-seeds the roster from `getCheckIns()`, and re-subscribes to the realtime stream.

### 5. End Session

Tapping **End Session** (after confirmation) calls `end_attendance_session` RPC, which sets `is_active = false` and `end_time = NOW()` on the server. Students can no longer check in after this point.

### 6. Session History

The **History** tab on the session screen shows all past sessions for the current lecturer, fetched from Supabase and cached locally. Each session card shows:
- Course code and name
- Date and time
- Number of attendees
- Live/Ended status badge

Tapping a session opens a detail view showing the full roster (name, student number, programme, method, time), fetched directly from Supabase.

## Student Flow (`/attendance/check-in`)

The student check-in screen runs two input methods simultaneously:

### Camera QR Scan
`mobile_scanner` opens the rear camera. When a barcode starting with `ATT:` is detected, the token is extracted and submitted.

### NFC Tap
`NfcService.startSession()` listens for NFC tags. When a tag with an NDEF payload starting with `ATT:` is read, it is submitted.

### Submission
Both methods call `student_check_in` RPC with the token and method (`'qr'` or `'nfc'`). The RPC:
1. Looks up the session by token (must be `is_active = true` and `NOW()` within `[start_time, end_time]`)
2. Looks up the student's profile (must have role `'student'`)
3. Resolves the `student_id` from `profiles.student_id` (UUID of the `students` row) or falls back to `profiles.email`
4. Fetches the student's `program` from the `students` table
5. Inserts a row into `session_check_ins` with `UNIQUE (session_id, student_id)` — duplicate check-ins raise an exception
6. Returns the check-in details including course info

On success, the student sees a confirmation card showing their name, student ID, course, method, and time.

## Reports (`/attendance/reports`)

The reports screen shows the same session history as the lecturer's History tab, with the same drill-down into per-session rosters. Each session report shows:
- Course code, name, lecturer name, date
- Total attendees
- Full roster: name, student number, programme, method, time

## Database

### Supabase tables

```sql
attendance_sessions (
  id UUID, course_code TEXT, course_name TEXT,
  lecturer_id UUID, lecturer_name TEXT,
  start_time TIMESTAMPTZ, end_time TIMESTAMPTZ,
  token TEXT UNIQUE, is_active BOOLEAN, created_at TIMESTAMPTZ
)

session_check_ins (
  id UUID, session_id UUID, student_id TEXT,
  student_name TEXT, profile_id UUID,
  checked_in_at TIMESTAMPTZ, method TEXT, program TEXT,
  UNIQUE (session_id, student_id)
)
```

Both tables are added to `supabase_realtime` publication so Realtime streams work.

### SQLite cache tables

`attendance_sessions` and `session_check_ins` mirror the Supabase schema with SQLite-compatible column types. They serve as offline display caches; all writes go through Supabase RPCs first.

## Error Handling

| Error | Student sees |
|---|---|
| Token not found / session expired | "This session has ended or the code has expired." |
| Already checked in | "You have already checked in for this session." |
| Missing student ID on profile | "Your profile is missing a student ID. Contact admin." |
| Non-student role | "Only students can check in via this screen." |
