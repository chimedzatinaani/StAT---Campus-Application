# StAT v1.0.0

**Student & Staff Access Terminal — Initial Release**

This is the first public release of StAT, a unified campus management platform built with Flutter. It covers three production-ready modules: campus gate security, academic attendance, and a digital campus wallet with integrated point of sale.

---

## What's in this release

### Campus Gate Security
- NFC card scanning for vehicle entry/exit at campus perimeter gates
- Licence plate lookup with partial match search
- On-the-spot student registration with NFC card writing
- Offline-first event queue — gate events are stored locally and synced to Supabase when connectivity is restored
- Conflict detection for duplicate entry or exit events with guard override support
- Activity log with name/plate/date filtering and CSV export
- Sync status screen with manual trigger for reference data refresh

### Academic Attendance
- Lecturer creates a timed class session which generates a live QR code and optional NFC tag payload
- Students check in by scanning the QR with their camera or tapping the NFC tag
- Live roster powered by Supabase Realtime — new check-ins appear within seconds on the lecturer's screen
- Session history with per-session drill-down into full attendee rosters
- Attendance reports screen for reviewing past sessions

### Campus Wallet & Point of Sale
- Every user has a digital wallet; balances and transactions are managed exclusively through Supabase RPCs
- Cashier top-up flow: search by name, student number, or staff number and deposit cash-backed funds
- Campus shop: students browse items, purchase with wallet balance, and receive a QR voucher for counter pickup
- Merchant redemption via QR scan or 6-character short code
- Refund flow that reverses the transaction and restores wallet balance and item stock
- Offline display cache — last known balance shown with an offline indicator when connectivity is absent

### Platform & Infrastructure
- Role-based access: `admin`, `security`, `cashier`, `lecturer`, `student`, `staff`, `merchant_staff`
- go_router navigation with role-based redirect on every auth/profile change
- Supabase (PostgreSQL 15 + Auth + Realtime) as source of truth; all mutations via `SECURITY DEFINER` RPCs with RLS
- SQLite (sqflite, schema v14) as offline read cache and event queue
- NFC read and write via `flutter_nfc_reader` and a custom `MethodChannel` on `MainActivity`
- QR scanning via `mobile_scanner`; QR generation via `qr_flutter`

---

## Installation

1. Download `StAT-v1.0.0.apk` from the Assets section below.
2. Request login credentials by filling out the [Access Request Form](https://tinaanichimedza.co.zw/stat/access_request/).
3. Install the APK on an Android device (enable "Install from unknown sources" if prompted).
4. Log in with the credentials sent to your email.

---

## Assets

| File | Description |
|---|---|
| [`StAT-v1.0.0.apk`](StAT-v1.0.0.apk) | Android application package — install directly on device |
