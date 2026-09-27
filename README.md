# StAT — Student & Staff Access Terminal

StAT is a unified digital ecosystem developed to streamline campus security, academic operations and internal commerce into a single mobile platform. It replaces fragmented manual processes such as physical sign-in books, paper attendance registers and cash transactions. 

It is a Flutter mobile application that handles physical entrance access via NFC, student attendance via QR code and NFC, a campus digital wallet, and a campus shop/POS. All data is backed by Supabase (PostgreSQL + Auth + Realtime) with an offline-first SQLite cache on device.

## Executive Summary

StAT currently has 3 main technical modules which are:
1. Security Terminal
- Streamlines perimeter entry points as security personnel verify arriving vehicles using quick NFC ID scans or via manual license plate number lookups reducing peak-hour traffic bottlenecks at university entrances.
- See [docs/security.md](docs/security.md)

2. Campus Wallet & Point of Sale
- Centralizes physical cash handling exclusively to authorized campus cashiers eliminating handling across decentralized vendors which minimizes financial leakage and simplifies accounting reconciliation.
- Students and staff maintain secure account balance for internal transactions, supported by authorizes campus cashiers for top-ups.
- Replaces slow physical counter queues with online digital shop inventory browsing and purchasing. Students buy items in-app and present a digital voucher at pickup, reducing wait times and improving inventory tracking for campus vendors.
See [docs/campus-wallet.md](docs/campus-wallet.md).

3. Academic Attendance Monitoring
- Digitizes lecture check-ins as lecturers can generate dynamic session code for classes allowing students to check in or mark their attendance via their mobile phones. Lecturer can view live, automated attendance rosters and digital registers are stored and can later be downloaded.
- See [docs/attendance.md](docs/attendance.md).

## Tech Stack

| Layer | Technology |
|---|---|
| UI / Framework | Flutter (Dart), Material Design 3 |
| State management | Riverpod (AsyncNotifierProvider) |
| Navigation | go_router with role-based redirect |
| Remote database | Supabase (PostgreSQL 15, Auth, Realtime) |
| Local database | SQLite via sqflite (v14 schema) |
| NFC reading | flutter_nfc_reader (NDEF, Reader Mode) |
| NFC writing | Custom MethodChannel → MainActivity |
| QR scanning | mobile_scanner |
| QR generation | qr_flutter |
| Connectivity | connectivity_plus |

## Roles

Every user authenticates via Supabase Auth and has a `profiles.role` column that determines their starting route and feature access.

| Role | Key capabilities |
|---|---|
| `admin` | Full access to every feature, user management |
| `security` | NFC gate scanning, plate lookup, student registration, activity log |
| `cashier` | Wallet top-up, view all transactions, purchase redemption |
| `lecturer` | Create attendance sessions, view live roster, view session history/reports |
| `student` | Attendance check-in (QR/NFC), personal wallet, campus shop |
| `staff` | Personal wallet, campus shop |
| `merchant_staff` | Campus shop item management, purchase redemption, refunds (with passcode) |

## Database

The app maintains two databases in parallel:

- **Supabase** : source of truth for all data. All mutations go through RPCs with RLS enforcement.
- **SQLite (on-device)** : read cache and offline event queue; synced on connectivity.

See [docs/architecture.md](docs/architecture.md) for the full data model.

## Demo Setup and Access

1. **Download the APK** from the [`release/`](release/README.md) directory in this repository.
2. **Request login credentials** by filling out the [Access Request Form](https://tinaanichimedza.co.zw/stat/access_request/).
3. **Install the APK** on an Android device.
4. **Log in** using the credentials sent to your email and start exploring the app.
