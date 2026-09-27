# Campus Gate Security

## Overview

Security guards use the app as a handheld terminal to manage vehicle access through the campus gate. When a vehicle approaches, the guard taps the driver's NFC card to identify them, then records an ENTRY or EXIT event. If the card cannot be scanned, the guard can look up the vehicle by licence plate number. All events are stored locally first and synced to Supabase when online.

## Roles

| Role | Access |
|---|---|
| security | NFC scanning, plate lookup, student registration, activity log, sync status |
| admin | Same as security, plus all other admin features |

## NFC Gate Scanner (`/scanner`)

The main security screen has two tabs:

### Tab 1 - NFC Card

The app begins listening for NFC tags as soon as the screen loads (`ScanningController.startScanning()`). When a guard holds a student's NFC card near the device:

1. `NfcService` receives the tag via `FlutterNfcReader.onTagDiscovered()` EventChannel
2. The card identifier is extracted from the NDEF Text Record (expected format: `SAT:CARD:<alphanumeric>` or `CARD:<alphanumeric>`)
3. `ScanningController` looks up the identifier in local SQLite first
4. If not found locally and the device is online, it queries Supabase `nfc_cards` with a join to `students` and `vehicles`
5. The student name, vehicle make/model, and plate number are displayed

The guard then taps **Entry** or **Exit** to record the event. The app checks the last recorded event for that card to determine the expected next state:
- If the last event was ENTRY, EXIT is highlighted as the expected action
- If a guard records the same event type twice (e.g. two ENTRYs), a conflict warning appears with an option to override

Events are written to the local `pending_events` table with `synced = 0`. The sync service uploads them to Supabase `access_events` when connectivity is available.

If a student is not found (unknown card), an error state is shown with a "Try Again" button.

### Tab 2 - Plate Number

The guard types all or part of a licence plate number. The app searches the local `vehicles` table (joined to `students`). If only one result is found, a details sheet opens immediately. If multiple matches exist, a list is shown for the guard to select from.

The details sheet shows the student name, vehicle details, and the expected next event. The guard can record ENTRY or EXIT the same way as the NFC tab.

NFC scanning is **paused** while the guard is on the plate tab (to save battery) and resumes when switching back.

## Event State Logic

The app prevents double-ENTRY or double-EXIT mistakes by tracking the expected next event:

```
No history  →  Either ENTRY or EXIT is valid (first-time card)
Last = ENTRY  →  EXIT is expected; ENTRY shows a conflict warning
Last = EXIT   →  ENTRY is expected; EXIT shows a conflict warning
```

When a conflict is detected, the guard sees a dialog explaining the inconsistency. They can confirm to record the event as an **override** (`is_override = true` in `pending_events`).

## Student Registration (`/register-student`)

Security guards can register new students on the spot. The registration form collects:

- Full name (required)
- Student number (optional)
- Phone number (optional)
- Level / year (optional)
- Vehicle details: make, model, colour, plate number (all optional)

Submitting the form calls the `create_single_student` Supabase RPC, which creates the student record, vehicle record (if provided), and a new NFC card entry in one atomic operation.

After registration, the guard is prompted to write the student's new card identifier to a physical NFC card. This uses `NfcService.writeNfcCard()` which writes an NDEF Text Record with payload `SAT:CARD:<identifier>`.

## Activity Log (`/activity-log`)

A filterable log of all ENTRY/EXIT events stored locally. The guard can filter by:
- Student name (partial match)
- Plate number (partial match)
- Date range (start and end date)

Results are fetched from the local `pending_events` table joined to `nfc_cards`, `students`, and `vehicles`. The log can be exported as a CSV file via the share sheet (`share_plus`).

## Sync Status (`/sync-status`)

Shows how many events are pending upload and allows the guard to manually trigger a sync. The sync process:

1. Checks connectivity - aborts if offline
2. Uploads all `pending_events` rows with `synced = 0` to Supabase `access_events`
3. Marks each successfully uploaded event as `synced = 1`

A separate "Sync Reference Data" button triggers a full refresh of the student/vehicle/NFC card registry from Supabase (used when new students have been registered on another device).

## Manual Lookup (`/manual-lookup`)

A search screen where a guard or admin can look up a student by name, student number, or NFC identifier. Results show full student details including linked vehicle and NFC card status.

## Local Data Viewer (`/local-data`)

A debug/admin screen that displays the raw contents of local SQLite tables (students, vehicles, NFC cards). Used to verify sync state.

## SQLite Tables

```sql
students (id, student_number, name, phone_number, level, program)
vehicles (id, student_id, make, model, color, plate_number)
nfc_cards (id, identifier, student_id, is_active)
pending_events (
  id, nfc_card_id, event_type,  -- 'ENTRY' | 'EXIT'
  timestamp, synced, is_override, recorded_by
)
```

## Supabase Tables

- `students` - master student registry
- `vehicles` - vehicle records
- `nfc_cards` - NFC card identifiers with `is_active` flag
- `access_events` - synced gate events (destination for `pending_events` sync)
