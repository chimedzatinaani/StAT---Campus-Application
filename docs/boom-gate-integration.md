# Boom Gate Integration

## Overview

This document describes the intended integration path between the StAT app and a physical boom gate controller. The current implementation handles the data side of gate access (NFC card scanning, ENTRY/EXIT event recording, and vehicle registration) and is designed to be extended with hardware gate control.

The app currently does **not** send direct signals to open or close a gate. All events are recorded in the app and synced to Supabase. The integration points described here represent the designed extension path.

## Current State

The security guard flow as implemented:

1. Driver presents NFC card at the window
2. Guard holds device near card → card identifier read via NFC
3. App looks up student + vehicle (local SQLite first, then Supabase)
4. Guard taps ENTRY or EXIT button
5. Event recorded locally → synced to Supabase `access_events`
6. Gate is operated manually (physical button/lever)

The data infrastructure is complete. What remains is wiring the software event to a physical gate signal.

## NFC Card Format

All gate access NFC cards carry an NDEF Text Record (UTF-8, language code `en`) with the payload:

```
SAT:CARD:<identifier>
```

Where `<identifier>` is the alphanumeric string stored in `nfc_cards.identifier` in the database. The `SAT:` prefix distinguishes gate cards from attendance session tokens (`ATT:`) written to temporary NFC tags.

Cards are written using `NfcService.writeNfcCard()` at registration time via `StudentRegistrationScreen`.

## Integration Options

### Option A - App triggers gate via local network

The phone connects to a gate controller on the campus LAN (TCP socket or HTTP). After a successful ENTRY event, the app sends a signal to open the boom gate.

**Implementation point:** `EventRecorder.recordEventWithValidation()` in `lib/features/access/event_recorder.dart`. After the event is successfully recorded locally, add a call to a `GateControllerService` that sends the open signal.

```dart
// Pseudocode
await _recorder.recordEventWithValidation(student, 'ENTRY');
if (eventType == 'ENTRY') {
  await gateController.openGate();
}
```

The `GateControllerService` would be a Riverpod `Provider` configured with the gate IP and port.

### Option B - Gate controller reads Supabase in real time

The gate controller is a separate device (Raspberry Pi, Arduino + ESP32, etc.) that subscribes to the Supabase `access_events` table via the Supabase Realtime WebSocket. When a new ENTRY event arrives, it activates the gate relay.

This is the lowest-friction approach - no changes to the Flutter app. The gate controller logic is entirely separate from the mobile app.

**Supabase side:** Enable Realtime for `access_events`:
```sql
ALTER PUBLICATION supabase_realtime ADD TABLE access_events;
```

**Controller side:** Subscribe to inserts on `access_events` filtered by `event_type = 'ENTRY'` and the relevant gate location identifier.

### Option C - Gate has its own NFC reader

A fixed NFC reader at the gate (connected to a controller) reads the card independently. The controller queries Supabase directly (`nfc_cards` table) to verify the card is active, then opens the gate and logs the event to `access_events`.

In this model, the guard's phone is used only for manual lookup, registration, and oversight — not for primary gate control.

## `access_events` Table (Supabase)

This is the sync destination for all gate events. Recommended schema (extend as needed for hardware integration):

```sql
CREATE TABLE IF NOT EXISTS access_events (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  nfc_card_id  TEXT NOT NULL,
  event_type   TEXT NOT NULL CHECK (event_type IN ('ENTRY', 'EXIT')),
  timestamp    TIMESTAMPTZ NOT NULL,
  is_override  BOOLEAN NOT NULL DEFAULT false,
  recorded_by  UUID REFERENCES profiles(id),
  gate_id      TEXT,          -- optional: identifies which gate
  synced       BOOLEAN NOT NULL DEFAULT true,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_access_events_card  ON access_events(nfc_card_id);
CREATE INDEX idx_access_events_time  ON access_events(timestamp DESC);
CREATE INDEX idx_access_events_type  ON access_events(event_type);
```

## Offline Behaviour

The app is designed to remain functional when the campus network is unavailable:

- Events are recorded to local `pending_events` immediately
- `SyncService.syncPendingEvents()` uploads them to `access_events` when connectivity returns
- The local student/vehicle/NFC card registry allows card lookup without network access

For hardware integration, Option B (controller subscribes to Supabase) would operate in near-real-time when online. During an outage, if a gate controller is used, it may need its own local fallback (cached list of approved card identifiers).

## NFC Hardware Notes

The app uses `flutter_nfc_reader` for both reading and writing NFC tags. The plugin activates Android's NFC Reader Mode (`NfcAdapter.enableReaderMode`) which:
- Suppresses the Android system NFC popup entirely
- Allows the app to receive tag data silently in the background
- Is active whenever the security scanner screen is open

For writing (student registration, attendance tag creation), the plugin's Reader Mode is briefly paused via `FlutterNfcReader.disableReaderMode()`, the write is performed through a dedicated `MethodChannel` to `MainActivity`, then Reader Mode is restored. This means the guard can write a card immediately after registration without leaving the screen.

Cards used for gate access must be NDEF-formatted NFC tags (NTAG213/215/216 or equivalent ISO 14443-3A tags) that support plain NDEF Text Records.
