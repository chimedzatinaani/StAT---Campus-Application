# Architecture

## Overview

StAT follows a feature-first folder structure with a clear layered architecture inside each feature: `domain → data → presentation`. State is managed by Riverpod providers. The app is offline-capable for read and queuing operations; all balance mutations and attendance check-ins require an online connection.

```
Supabase (PostgreSQL + Auth + Realtime)
        ↑ RPCs / REST / Realtime
Flutter App
  ├── Riverpod Providers
  ├── Feature Repositories  ←→  SQLite (sqflite)
  └── Presentation screens
```

## Navigation & Role Gating

Navigation uses go_router with a `_RouterNotifier` that re-evaluates redirects on every auth/profile change.

Every route in the app is public to authenticated users **unless** it falls into a gated group:

| Group | Allowed roles |
|---|---|
| `/scanner`, `/manual-lookup`, `/register-student`, `/activity-log`, `/local-data`, `/sync-status` | security, admin |
| `/cashier/top-up` | cashier, admin |
| `/shop/redeem` | merchant_staff, security, cashier, admin |
| `/shop/refund` | admin, cashier, merchant_staff |
| `/attendance/record`, `/attendance/schedule`, `/attendance/reports`, `/attendance/create-session`, `/attendance/live-dashboard` | lecturer, admin |
| `/attendance/check-in` | student, admin |

Unauthenticated users are always redirected to `/login`.

## State Management

All state uses Riverpod. The key provider types:

- `AsyncNotifierProvider` — for stateful flows with loading/error/data (scanning, attendance session, student check-in, wallet).
- `FutureProvider` — for one-shot async initialisation (repositories, database instance).
- `Provider` — for synchronous services (NfcService, SyncService).

## Data Layer

### SQLite (sqflite)

The local SQLite database (`campus_security.db`) is at schema version 14. Tables:

| Table | Purpose |
|---|---|
| `students` | Local registry of registered students (synced from Supabase) |
| `vehicles` | Vehicle records linked to students |
| `nfc_cards` | NFC card identifiers linked to students |
| `pending_events` | ENTRY/EXIT gate events queued for sync |
| `app_metadata` | Key/value store for app state |
| `attendance_records` | Legacy manual attendance records |
| `class_schedules` | Cached class schedule data |
| `attendance_summary` | Aggregated attendance summaries |
| `wallet_accounts` | Cached wallet balances per profile |
| `wallet_transactions` | Cached transaction history |
| `shop_items` | Cached campus shop catalogue |
| `purchases` | Cached purchase history |
| `attendance_sessions` | Session records (lecturer creates, synced from Supabase) |
| `session_check_ins` | Per-session student check-ins (synced from Supabase Realtime) |

Migrations are handled in `database_helper.dart`'s `_onUpgrade` method. The database is a singleton accessed via the `databaseProvider` FutureProvider.

### Supabase

Supabase is the source of truth. All writes go through PostgreSQL functions (RPCs) with `SECURITY DEFINER` and `search_path = public`. RLS is enabled on all tables.

#### Core tables (managed outside `schema.sql`)

- `profiles` — one row per auth user; contains `role`, `name`, `email`, `student_id`, `staff_number`
- `students` — student registry with `student_number`, `name`, `level`, `program`
- `vehicles` — vehicle records
- `nfc_cards` — NFC card identifiers
- `access_events` — synced gate ENTRY/EXIT events

#### Tables defined in `schema.sql`

- `shop_items` — campus shop catalogue
- `purchases` — purchase transactions
- `purchase_references` — QR code payload and OTP short code per purchase
- `attendance_sessions` — lecturer-created class sessions with time-limited token
- `session_check_ins` — one row per student per session

#### RPCs

| Function | Called by | Purpose |
|---|---|---|
| `wallet_deposit` | Cashier | Credit a profile's wallet balance |
| `wallet_purchase` | Student (shop) | Debit balance for a purchase |
| `wallet_refund` | Cashier / merchant | Credit balance for a refund |
| `create_shop_purchase` | Student | Atomically debit wallet + create purchase + generate QR/short code |
| `redeem_purchase` | Merchant/cashier | Mark a purchase as collected (single-use) |
| `refund_purchase` | Admin/cashier/merchant | Reverse a purchase and credit wallet |
| `create_attendance_session` | Lecturer | Create a session, generate token, return full row |
| `end_attendance_session` | Lecturer | Deactivate session |
| `student_check_in` | Student | Validate token, insert check-in row, return session + student info |
| `create_single_student` | Security | Register one student + vehicle + NFC card |
| `bulk_import_students` | Admin | Import students from CSV |

## Offline / Sync Strategy

**Gate access events** are written to `pending_events` locally first. `SyncService.syncPendingEvents()` uploads them to `access_events` in Supabase when connectivity is available.

**Reference data** (students, vehicles, NFC cards) is pulled wholesale from Supabase via `SyncService.syncReferenceData()`. This runs atomically inside a SQLite transaction — it deletes all local rows and re-inserts the fresh data.

**Wallet and shop** — all mutations require online connectivity (enforced via `ConnectivityResult.none` check in `WalletRepository`). The balance and transaction list are cached in SQLite after each remote operation for display.

**Attendance** — `createSession` and `studentCheckIn` both require online access (RPC calls). The session and check-in are cached locally after the RPC succeeds. The live roster uses Supabase Realtime (`.stream()`) so the lecturer's roster updates in real time as students check in from their own devices.

## NFC Architecture

Two distinct NFC flows share the same underlying hardware session:

### Gate reading (security scanner)
`ScanningController` → `NfcService.startSession()` → `FlutterNfcReader.onTagDiscovered()` EventChannel → payload parsed for `SAT:CARD:<id>` or `CARD:<id>` format → student looked up in SQLite (then Supabase fallback).

### Attendance tag writing (lecturer)
`NfcService.writeNfcCard()` → `FlutterNfcReader.disableReaderMode()` → `MethodChannel('com.example.stat_security/nfc_write')` → MainActivity registers a one-shot ReaderCallback → tag delivered silently → NDEF Text Record written with `ATT:<token>` payload → `FlutterNfcReader.enableReaderMode()` restored.

### Attendance reading (student)
`NfcService.startSession()` (same as gate reading) → payload validated for `ATT:` prefix → `student_check_in` RPC called.

The NFC tag payload format differs by use case:
- Gate cards: `SAT:CARD:<alphanumeric>` (written at registration)
- Attendance tags: `ATT:<32-char-hex-token>` (written fresh per session)
