# Possible Future Integrations

This document describes two hardware extensions that can be layered on top of the existing StAT gate security system without altering its core data architecture. Both integrate at well-defined points in the current codebase and feed into the same `pending_events → access_events` pipeline that is already in production.

---

## Feature 1 — Automated Boom Gate Control

### Overview

The current guard workflow records an ENTRY or EXIT event and then manually operates the physical gate. This feature closes that gap by sending an open signal to a boom gate controller immediately after a successful event record — making the gate response automatic and guard-independent.

The app already has everything it needs on the data side. What this adds is a network call to gate hardware at the right moment in the existing flow.

### Where the Trigger Lives

The entire gate event flow converges in a single method:

**`lib/features/access/event_recorder.dart` → `EventRecorder._doRecord()`**

This method is called regardless of how the event was initiated — NFC card tap, plate number lookup, or manual lookup screen. It handles both online (direct Supabase insert) and offline (local SQLite queue) paths. The boom gate signal should fire immediately after a successful record, before the snackbar is shown:

```dart
// Inside EventRecorder._doRecord(), after the Supabase insert / repo.recordEvent() calls:

if (eventType == 'ENTRY') {
  // Non-blocking — gate failure should not prevent event recording
  unawaited(ref.read(gateControllerServiceProvider).openGate());
}
```

`GateControllerService` would be a Riverpod `Provider` that holds the gate controller's IP address and port, read from `dart_defines.json` (which already exists in the project at `.dart_defines.json`).

### Integration Options

#### Option A — App sends HTTP signal over campus LAN

After `_doRecord()` succeeds, the Flutter app sends an HTTP POST to the gate controller on the local network. The controller (a Raspberry Pi, Arduino + ESP8266, or commercial relay board) listens for this request and activates the relay.

```
Guard scans NFC card
       ↓
EventRecorder._doRecord() records event (SQLite + Supabase)
       ↓
GateControllerService.openGate()
       ↓  HTTP POST  (campus LAN)
Gate Controller Board
       ↓
Relay activates → barrier lifts
```

**Implementation sketch:**

```dart
// lib/core/gate/gate_controller_service.dart

class GateControllerService {
  final String _baseUrl; // e.g. 'http://192.168.1.50:8080'

  GateControllerService(this._baseUrl);

  Future<void> openGate() async {
    try {
      await http.post(Uri.parse('$_baseUrl/open'),
          headers: {'X-Gate-Token': '<shared_secret>'});
    } catch (e) {
      // Log but do not rethrow — a gate service failure
      // must never block the access event from being recorded.
      debugPrint('Gate signal failed: $e');
    }
  }
}
```

The gate endpoint should use a shared secret token (not tied to Supabase Auth) since it operates on the local network independently of the internet.

**When the device is offline:** The event still records to `pending_events` via `SyncService.syncPendingEvents()` as it does today. The gate signal, however, requires LAN connectivity to the controller — if the guard's phone cannot reach the controller IP, the gate must be operated manually. This is acceptable for the offline case since the campus LAN is typically more stable than internet connectivity.

#### Option B — Gate controller subscribes to Supabase Realtime

The gate controller is a standalone device (Raspberry Pi or similar) that subscribes to the `access_events` table via the Supabase Realtime WebSocket. When a new ENTRY row appears it activates the relay. No changes to the Flutter app are needed for this option.

```sql
-- Enable Realtime for access_events (run once in Supabase SQL Editor)
ALTER PUBLICATION supabase_realtime ADD TABLE access_events;
```

```python
# Gate controller (Raspberry Pi) — pseudocode
supabase.table('access_events') \
  .on('INSERT', lambda payload: open_gate() if payload['new']['event_type'] == 'ENTRY' else None) \
  .subscribe()
```

This option has a slightly higher latency (Realtime round-trip vs. direct LAN call) but requires zero changes to the mobile app and works from any network-connected controller.

#### Option C — Fixed NFC reader at the gate post

A dedicated NFC reader is mounted at the gate post, connected to a controller that queries Supabase `nfc_cards` directly to verify the card is active, then writes to `access_events` and activates the relay. In this model the guard's phone is used only for registration, manual overrides, and activity log review — not for primary gate control. The `is_active` column on `nfc_cards` acts as the gate authorisation flag.

### Recommended `access_events` Schema Extension

The existing sync destination already supports most of what is needed. A `gate_id` column is the only addition required to support multi-gate campuses:

```sql
ALTER TABLE access_events ADD COLUMN IF NOT EXISTS gate_id TEXT;
-- 'MAIN_GATE', 'RESIDENCE_GATE', etc.
-- Populated by GateControllerService or the controller hardware.
```

### Conflict and Override Behaviour

The existing `EventRecorder._showOverrideDialog()` and `is_override` flag are unaffected. For Option A, the gate open signal is only sent on non-override ENTRY events — override events still record but require the guard to manually operate the barrier, preserving the existing safety behaviour.

---

## Feature 2 — OCR Licence Plate Recognition

### Overview

A camera-based automatic licence plate recognition (ALPR) system that captures a vehicle's plate as it approaches the gate and feeds the recognised plate into the same vehicle lookup and event recording pipeline already used by the manual plate lookup tab in `NfcScannerScreen`.

A fully working proof-of-concept for this feature already exists in the `ocr_poc/` directory. It uses a Python backend (`FastAPI + OpenCV + Pytesseract`) paired with a browser-based frontend that streams from the device camera, sends frames to the backend for OCR, and displays real-time results with student and vehicle details pulled from Supabase.

### Two Deployment Modes

#### Mode 1 — In-App Camera Scan (Flutter, guard-initiated)

The guard opens the scanner screen and taps a camera button. `mobile_scanner` (already in `pubspec.yaml`) captures a frame, which is sent to the OCR backend. The recognised plate is passed directly to `VehicleRepository.searchVehicles()` — the same method used by the existing manual plate tab — and from there into `EventRecorder.recordEventWithValidation()`.

```
Guard taps "Scan Plate" button (new tab in NfcScannerScreen)
       ↓
mobile_scanner captures frame
       ↓  HTTP POST (image bytes)
OCR Backend (FastAPI + OpenCV + Pytesseract)
       ↓  { plate: "ABC123GP", confidence: 0.91 }
VehicleRepository.searchVehicles("ABC123GP")
       ↓
_showDetails(result)   ← same sheet used by the manual plate tab
       ↓
Guard confirms → EventRecorder.recordEventWithValidation()
       ↓
pending_events (SQLite) → access_events (Supabase)
```

This mode keeps a guard in the loop for confirmation before the event is committed — appropriate for a staffed gate.

**Integration point in `NfcScannerScreen`:** Add a third tab (`Tab(icon: Icon(Icons.camera_alt), text: 'Camera')`) or add a camera icon button to the existing plate tab. The logic mirrors `_lookupPlate()` but sources the query string from the OCR result instead of the text field.

#### Mode 2 — Automated Fixed Camera (Edge PC, guard-free)

A fixed camera is mounted at the gate post and connected to a laptop or Raspberry Pi running the Python engine from `ocr_poc/backend/`. The engine continuously ingests video frames, extracts the plate string, queries Supabase, determines ENTRY or EXIT based on the vehicle's last event (same logic as `AccessRepository.getExpectedNextEvent()`), and writes directly to `access_events`. The Flutter activity log updates in real time via Supabase Realtime.

```
Fixed camera (USB or IP) → Python engine (ocr_poc/backend)
       ↓  frame ingestion (OpenCV)
       ↓  OCR (Pytesseract / EasyOCR)
       ↓  plate string extracted + validated (regex + 30-sec cooldown)
       ↓  Supabase query: vehicles JOIN students JOIN nfc_cards
       ↓  AccessRepository logic: getExpectedNextEvent()
       ↓  INSERT into access_events (recorded_by = 'SYSTEM:CAMERA_01')
       ↓
Flutter activity log updates live (Supabase Realtime stream)
       ↓  (if boom gate wired) relay activates
```

The `recorded_by` field uses a system identifier string (`SYSTEM:CAMERA_01`) rather than a user UUID. The existing schema accepts `NULL` or any text value here, so no migration is needed.

### OCR Backend Stack

The POC backend at `ocr_poc/backend/` uses:

| Component | Purpose |
|---|---|
| `FastAPI` | HTTP endpoint that receives image frames |
| `OpenCV` | Frame pre-processing (grayscale, threshold, contour detection) |
| `Pytesseract` | Text extraction from isolated plate region |
| `supabase-py` | Direct database lookup and event insertion |
| `Pillow` / `numpy` | Image manipulation |

The backend exposes a single endpoint:

```
POST /scan
Content-Type: multipart/form-data
Body: { image: <file>, event_type: "ENTRY" | "EXIT" }

Response: {
  plate: "ABC123GP",
  confidence: 0.91,
  student_name: "Jane Doe",
  vehicle: "Toyota Corolla",
  authorized: true,
  event_id: "<uuid>"
}
```

The Flutter app calls this endpoint for Mode 1. The Python engine calls its own internal equivalent for Mode 2.

### How It Fits the Existing Data Architecture

OCR does not introduce any new tables or RPCs. The recognised plate is treated identically to a manually typed plate — it goes through `VehicleRepository.searchVehicles()`, resolves to a student record and `nfc_card_id`, and then passes through the existing event recording path. The activity log, sync status screen, and Supabase `access_events` table see no difference between an NFC-originated event, a manual plate event, and a camera-recognised event except for the optional source tag in `recorded_by`.

### Confidence Threshold and Fallback

When the OCR confidence score is below a defined threshold (e.g. `< 0.75`), the result is not committed automatically. Instead:

- **Mode 1 (in-app):** The guard sees the low-confidence plate string pre-filled in the plate text field and can correct it before searching.
- **Mode 2 (automated):** The engine logs an `UNREADABLE` audit snapshot locally and does not write to `access_events`. An alert is shown on the POC dashboard so a guard can intervene.

### Combining Both Features

When OCR (Mode 2) and boom gate (Option A or B) are both active, the full automated flow becomes:

```
Vehicle approaches gate
       ↓
Fixed camera → Python engine extracts plate
       ↓
Supabase query confirms vehicle is registered
       ↓
access_events INSERT (event_type = 'ENTRY', recorded_by = 'SYSTEM:CAMERA_01')
       ↓  (Option A) HTTP signal to gate controller
       ↓  (Option B) Realtime subscription on gate controller fires
Barrier lifts automatically
       ↓
Flutter activity log shows new event within seconds
```

In this configuration the guard's phone transitions from a primary input device to a monitoring and intervention tool — used when a plate is unreadable, a vehicle is unregistered, or an override is needed.
