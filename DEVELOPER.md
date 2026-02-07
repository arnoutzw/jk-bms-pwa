# JK BMS Monitor PWA — Developer Documentation

> This document is written for AI assistants (Claude, etc.) that may need to modify this codebase in future sessions. It contains everything needed to understand the architecture, protocol, data flow, and conventions without re-reading the entire source.

---

## 1. Project Overview

A self-contained Progressive Web App that connects to up to **3 JK BMS devices simultaneously** over Bluetooth Low Energy (Web Bluetooth API) and displays real-time battery data. The entire app lives in a single HTML file with inline CSS and JS — no build step, no dependencies, no framework.

### Files

| File | Purpose |
|---|---|
| `pwa/index.html` | **The entire app** — HTML structure, CSS styles, and all JavaScript (~1250 lines) |
| `pwa/manifest.json` | PWA manifest (standalone display, dark theme, inline SVG icon) |
| `pwa/sw.js` | Service worker — network-first with cache fallback, caches `index.html` and `manifest.json` |

### Reference Material (not part of the app)

| Path | What it is |
|---|---|
| `esphome-jk-bms-main/components/jk_bms_ble/jk_bms_ble.cpp` | **Authoritative protocol reference** — the esphome open-source implementation that all parsing logic is based on. Key functions: `decode_jk02_cell_info_()` (line ~361), `decode_jk04_cell_info_()` (line ~676), `decode_device_info_()` (line ~1239), `write_register()` (line ~1514), `assemble()` (line ~301) |
| `esphome-jk-bms-main/components/jk_bms_ble/jk_bms_ble.h` | Header with `ProtocolVersion` enum and sensor definitions |
| `JK_BMS_BLE_Protocol_ReverseEngineering.md` | Earlier reverse-engineering report from APK analysis (partially superseded by esphome findings) |

---

## 2. BLE Protocol

### Service & Characteristics

| UUID | Role |
|---|---|
| `0xFFE0` | Primary GATT service |
| `0xFFE1` | **Notify** characteristic — BMS sends data here in ~20-byte BLE chunks |
| `0xFFE2` | **Write** characteristic — app sends commands here (some BMS models use FFE1 for both) |

### Frame Structure (BMS → App)

All frames from the BMS are **exactly 300 bytes**:

```
Byte 0-3:   Header — 55 AA EB 90 (always)
Byte 4:     Frame type (0x01, 0x02, or 0x03)
Byte 5:     Frame counter (increments)
Byte 6-298: Payload (varies by frame type)
Byte 299:   CRC — simple sum of bytes 0-298, masked to 0xFF
```

Frames arrive fragmented across multiple BLE notifications (~20 bytes each). The app accumulates bytes in `rxBuffer`, scans for the `55 AA EB 90` header, and extracts complete 300-byte frames.

### Frame Types (same for ALL protocol versions)

| Type | Hex | Content | Triggered by |
|---|---|---|---|
| Settings | `0x01` | BMS configuration parameters | Sent automatically after connect |
| Cell Data | `0x02` | Real-time cell voltages, current, temps, SOC, errors | Command `0x96` |
| Device Info | `0x03` | Model, firmware, serial, hardware version | Command `0x97` |

**Important:** Frame types are identical across all protocol versions. An earlier version of this code incorrectly swapped 0x01 and 0x02 — this was wrong.

### Command Structure (App → BMS)

Commands are **20 bytes**:

```
Byte 0-3:   Header — AA 55 90 EB (note: reversed from response header)
Byte 4:     Command byte
Byte 5-18:  Zero padding
Byte 19:    CRC — sum of bytes 0-18, masked to 0xFF
```

| Command | Hex | Purpose |
|---|---|---|
| Cell Info / Activate | `0x96` | Request cell data frame; also serves as keepalive |
| Device Info | `0x97` | Request device info frame |

### Protocol Versions

Three protocol variants exist, differing in how cell data (frame type 0x02) is encoded:

| Protocol | Cell voltage format | Cell count | Resistance format | Used by |
|---|---|---|---|---|
| `JK02_24S` | uint16 LE × 0.001V | 24 | uint16 LE × 0.001Ω | Older JK-B models, HW < 3 |
| `JK02_32S` | uint16 LE × 0.001V | 32 | uint16 LE × 0.001Ω | JK_PB models, JK-B with HW ≥ 11 |
| `JK04` | IEEE 754 float32 LE (V) | 24 | IEEE 754 float32 LE (Ω) | JK-B with HW 3-10 |

**Auto-detection** happens from the device info frame (0x03):
- Model starts with `JK_` or contains `PB` → `JK02_32S`
- Model starts with `JK-B` + HW version ≥ 11 → `JK02_32S`
- Model starts with `JK-B` + HW version 3-10 → `JK04`
- Otherwise → `JK02_24S`

The tested/primary device is a **JK-PB2A16S20P** which uses **JK02_32S**.

---

## 3. JK02 Cell Data Frame Layout (0x02)

This is the most complex parser. The `JK02_32S` variant has a 16-byte offset (`ofs`) compared to `JK02_24S` because it has 32 cell slots instead of 24. After the resistance block, the offset doubles (`ofs2 = ofs * 2`).

### Cell Voltages (offset 6)

`numCells × uint16 LE` — each value × 0.001 = volts. Cells with value 0 or > 5.0V are skipped (not connected).

- **24S:** 24 × 2 bytes = 48 bytes (offset 6 to 53)
- **32S:** 32 × 2 bytes = 64 bytes (offset 6 to 69)

### Cell Resistances (offset 64 + ofs)

`numCells × uint16 LE` — each value × 0.001 = ohms.

- **24S (ofs=0):** starts at byte 64
- **32S (ofs=16):** starts at byte 80

### After resistances: ofs2 = ofs × 2

All subsequent fields use `ofs2` (0 for 24S, 32 for 32S):

| Field | Offset | Type | Scale | Notes |
|---|---|---|---|---|
| MOS temp | 112 + ofs2 | int16 LE | × 0.1 °C | 32S only at this offset |
| Pack voltage | 118 + ofs2 | uint32 LE | × 0.001 V | |
| Pack current | 126 + ofs2 | int32 LE | × 0.001 A | Positive = charging |
| Temp sensor 1 | 130 + ofs2 | int16 LE | × 0.1 °C | |
| Temp sensor 2 | 132 + ofs2 | int16 LE | × 0.1 °C | |
| Errors (32S) | 134 + ofs2 | uint16 BE | bitmask | **Big-endian!** 32S only |
| MOS temp (24S) | 134 + ofs2 | int16 LE | × 0.1 °C | 24S only |
| Errors (24S) | 136 + ofs2 | uint16 BE | bitmask | 24S only |
| Balance current | 138 + ofs2 | int16 LE | × 0.001 A | |
| Balance action | 140 + ofs2 | uint8 | 0/1/2 | 0=off, 1=chg, 2=dsg |
| SOC | 141 + ofs2 | uint8 | % | |
| Remaining cap | 142 + ofs2 | uint32 LE | × 0.001 Ah | |
| Full capacity | 146 + ofs2 | uint32 LE | × 0.001 Ah | |
| Cycle count | 150 + ofs2 | uint32 LE | integer | |
| Cycle capacity | 154 + ofs2 | uint32 LE | × 0.001 Ah | |
| SOH | 158 + ofs2 | uint8 | % | |
| Total runtime | 162 + ofs2 | uint32 LE | seconds | |
| Charge MOSFET | 166 + ofs2 | uint8 | 0/1 | |
| Discharge MOSFET | 167 + ofs2 | uint8 | 0/1 | |
| Heating | 183 + ofs2 | uint8 | 0/1 | |
| Temp sensor 5 | 222 + ofs2 | int16 LE | × 0.1 °C | 32S only |
| Temp sensor 4 | 224 + ofs2 | int16 LE | × 0.1 °C | 32S only |
| Temp sensor 3 | 226 + ofs2 | int16 LE | × 0.1 °C | 32S only |

### Error Bitmask (16 bits)

| Bit | Error |
|---|---|
| 0 | Charge overtemp |
| 1 | Charge undertemp |
| 2 | Coprocessor comm error |
| 3 | Cell undervoltage |
| 4 | Pack undervoltage |
| 5 | Discharge overcurrent |
| 6 | Discharge short circuit |
| 7 | Discharge overtemp |
| 8 | Wire resistance |
| 9 | MOSFET overtemp |
| 10 | Cell count mismatch |
| 11 | Current sensor anomaly |
| 12 | Cell overvoltage |
| 13 | Pack overvoltage |
| 14 | Charge overcurrent |
| 15 | Charge short circuit |

---

## 4. JK04 Cell Data Frame Layout (0x02)

Simpler layout using IEEE 754 floats:

| Field | Offset | Type | Count |
|---|---|---|---|
| Cell voltages | 6 | float32 LE | 24 (96 bytes) |
| Cell resistances | 102 | float32 LE | 24 (96 bytes) |
| Balancing flag | 220 | uint8 | |
| Balance current | 222 | float32 LE | |
| Uptime | 286 | uint32 LE | seconds |

Note: JK04 does not report pack voltage directly — it is computed as the sum of cell voltages.

---

## 5. Device Info Frame Layout (0x03)

Same offsets for all protocol versions:

| Field | Offset | Length | Type |
|---|---|---|---|
| Model / Vendor ID | 6 | 16 | ASCII string |
| Hardware version | 22 | 8 | ASCII string |
| Firmware version | 30 | 8 | ASCII string |
| Uptime | 38 | 4 | uint32 LE (seconds) |
| Power-on count | 42 | 4 | uint32 LE |
| Device name | 46 | 16 | ASCII string |
| Passcode | 62 | 16 | ASCII string |
| Manufacturing date | 78 | 8 | ASCII string |
| Serial number | 86 | 11 | ASCII string |

---

## 6. App Architecture

### Multi-Device Slot System

The app supports up to 3 simultaneous BLE connections. State is managed via a `slots[3]` array where each entry is either `null` (empty) or a connection state object.

```
slots = [null, null, null]
         ↓
    createSlotState() → {
      device, server, service,          // Web Bluetooth objects
      charNotify, charWrite,            // GATT characteristics
      rxBuffer: [],                     // Frame assembly buffer
      frameCount, lastFrameTime,        // Diagnostics
      reconnecting: false,              // Auto-reconnect guard
      heartbeatTimer: null,             // setInterval ID
      heartbeatEnabled: false,          // Default: off
      protocolVersion: PROTO.JK02_32S,  // Auto-detected later
      bmsData: { ... }                  // All parsed BMS data
    }
```

### Element ID Convention

All DOM elements for a device slot use the pattern `id="<name>-<slot>"`. The helper function `el(name, slot)` returns the element:

```javascript
el('soc-bar', 0)  →  document.getElementById('soc-bar-0')
el('pack-voltage', 2)  →  document.getElementById('pack-voltage-2')
```

### Dynamic Panel Generation

Device panels are created/destroyed dynamically:
- `createDevicePanel(s)` — inserts HTML before the "Add" card via `insertAdjacentHTML`
- `removeDevicePanel(s)` — removes the panel DOM element
- `updateGlobalStatus()` — updates header counter and shows/hides the Add card

### Color Coding

Each slot has a distinct accent color for its border and active tab highlight:
- Slot 0: green (`--green`)
- Slot 1: blue (`--blue`)
- Slot 2: orange (`--orange`)

Applied via CSS class `.slot-0`, `.slot-1`, `.slot-2` on the `.device-panel` element.

---

## 7. Connection Flow

```
addDevice()
  ├── findFreeSlot() → s
  ├── createSlotState() → slots[s]
  ├── createDevicePanel(s)
  ├── navigator.bluetooth.requestDevice({ filters: [{ services: [0xFFE0] }] })
  ├── device.addEventListener('gattserverdisconnected', () => onDisconnected(s))
  └── connectSlot(s)
        ├── device.gatt.connect()
        ├── getPrimaryService(0xFFE0)
        ├── getCharacteristic(0xFFE1) → charNotify
        ├── getCharacteristic(0xFFE2) → charWrite (fallback to FFE1)
        ├── charNotify.startNotifications()
        ├── charNotify.addEventListener('characteristicvaluechanged', (e) => onNotification(s, e))
        ├── sleep(500) → sendDeviceInfoCmd(s)   // 0x97 — triggers protocol auto-detect
        ├── sleep(300) → sendActivate(s)         // 0x96 — starts cell data stream
        └── startHeartbeat(s)                    // Only if toggle is checked (default: off)
```

### Disconnection & Reconnect

When `gattserverdisconnected` fires:
1. Status dot turns gray
2. Heartbeat timer cleared
3. Auto-reconnect attempted after 3 seconds (unless `reconnecting` flag set)
4. Manual disconnect (`disconnectSlot`) sets `reconnecting = true` to prevent auto-reconnect, then removes the panel and nulls the slot

### Heartbeat

The BMS disconnects after ~15-25 seconds without commands. The heartbeat sends `0x96` (cell info request) every 5 seconds. It is **off by default** and controlled per-device via a toggle in the Raw tab. The toggle checkbox ID is `hb-toggle-<slot>`.

---

## 8. Data Flow

```
BLE notification (20 bytes)
  → onNotification(s, event)
    → append bytes to slots[s].rxBuffer
    → extractFrames(s)
      → scan for header 55 AA EB 90
      → extract 300-byte frame
      → verify CRC (sum of first 299 bytes)
      → processFrame(s, frame)
        → switch on frame[4]:
          0x01 → log only (settings — not parsed)
          0x02 → parseCellData(s, frame)
                  → dispatches to parseJK02CellInfo() or parseJK04CellInfo()
                  → writes to slots[s].bmsData
                  → calls updateSlotUI(s)
          0x03 → parseDeviceInfo(s, frame)
                  → writes model/fw/serial to bmsData
                  → auto-detects protocol version
                  → updates device name in panel header
```

---

## 9. UI Structure Per Device Panel

Each device panel contains:

```
.device-panel.slot-{s}
  ├── .panel-header
  │     ├── .slot-badge ("BMS 1/2/3")
  │     ├── .status-dot (connecting/connected/disconnected)
  │     ├── .device-name
  │     └── disconnect button (✕)
  └── .panel-body
        ├── .frame-counter (frame #, type, protocol)
        ├── SOC card (bar + remain/full Ah)
        ├── .stats-grid (voltage, current, power, cycles)
        ├── .tabs (Cells | Temps | Status | Raw)
        ├── #tab-cells-{s} — cell voltage bars with ▲▼ markers
        ├── #tab-temps-{s} — temperature grid (MOS, T1-T5)
        ├── #tab-status-{s} — device info rows + MOS on/off + alarms
        └── #tab-raw-{s} — activate/devinfo buttons, heartbeat toggle, hex dump, event log
```

Tab switching is per-device via `showTab(s, name, btn)`.

---

## 10. CSS Design System

Dark theme using CSS custom properties:

| Variable | Value | Usage |
|---|---|---|
| `--bg` | `#0f172a` | Page background |
| `--surface` | `#1e293b` | Card/panel background |
| `--surface2` | `#334155` | Nested card / borders |
| `--text` | `#f1f5f9` | Primary text |
| `--text2` | `#94a3b8` | Secondary text |
| `--text3` | `#64748b` | Tertiary / labels |
| `--green` | `#22c55e` | OK / charging / slot 0 |
| `--blue` | `#3b82f6` | Voltage / slot 1 |
| `--orange` | `#f97316` | High deviation / slot 2 |
| `--red` | `#ef4444` | Error / discharging / off |
| `--yellow` | `#eab308` | Warning / power |

The layout uses CSS Grid (`repeat(auto-fill, minmax(340px, 1fr))`) so panels stack on mobile and sit side-by-side on wider screens. Container max-width is 1200px.

---

## 11. Key Functions Reference

### Slot Management
| Function | Purpose |
|---|---|
| `createSlotState()` | Returns a new connection state object with fresh bmsData |
| `createBmsData()` | Returns a new empty BMS data object |
| `findFreeSlot()` | Returns first null index in `slots[]`, or -1 |
| `connectedCount()` | Returns number of non-null slots |
| `el(id, s)` | Shorthand for `document.getElementById(id + '-' + s)` |

### Connection
| Function | Purpose |
|---|---|
| `addDevice()` | Full scan+connect flow — finds free slot, creates panel, scans BLE, connects |
| `connectSlot(s)` | GATT connect, find service/chars, enable notifications, request info+data |
| `disconnectSlot(s)` | Manual disconnect — clears heartbeat, disconnects GATT, removes panel, nulls slot |
| `onDisconnected(s)` | Auto-reconnect handler (3s delay) |
| `setSlotStatus(s, status)` | Updates status dot CSS class ('connecting'/'connected'/'disconnected') |

### Commands
| Function | Purpose |
|---|---|
| `buildCommand(cmd)` | Builds a 20-byte command frame with CRC |
| `sendCmd(s, data)` | Writes to charWrite (with fallback to writeValue) |
| `sendActivate(s)` | Sends 0x96 (cell info request) |
| `sendDeviceInfoCmd(s)` | Sends 0x97 (device info request) |
| `startHeartbeat(s)` | Starts 5s interval sending 0x96 (if toggle checked) |
| `toggleHeartbeat(s, enabled)` | Start/stop heartbeat from UI toggle |

### Frame Processing
| Function | Purpose |
|---|---|
| `onNotification(s, event)` | Appends BLE notification bytes to rxBuffer, calls extractFrames |
| `extractFrames(s)` | Scans rxBuffer for 300-byte frames with valid CRC |
| `processFrame(s, frame)` | Routes frame by type (0x01/0x02/0x03) |

### Parsers
| Function | Purpose |
|---|---|
| `parseDeviceInfo(s, frame)` | Parses 0x03 frame — model, FW, serial; **auto-detects protocol version** |
| `parseCellData(s, frame)` | Dispatches to JK02 or JK04 parser based on protocol |
| `parseJK02CellInfo(s, frame)` | Parses JK02_24S/32S — uint16 voltages, offset-aware field extraction |
| `parseJK04CellInfo(s, frame)` | Parses JK04 — IEEE float32 voltages/resistances |

### Data Helpers
| Function | Purpose |
|---|---|
| `jkGet16/jkGet32/jkGetS16/jkGetS32` | Little-endian integer reads from DataView |
| `jkGetFloat` | IEEE 754 float32 LE read |
| `readString(frame, start, maxLen)` | Extract null-terminated ASCII string |
| `errorBitsToString(bitmask)` | Convert error bitmask to human-readable string |

### UI Updates
| Function | Purpose |
|---|---|
| `updateSlotUI(s)` | Main UI refresh — updates all stats, SOC, cells, temps, MOS, alarms |
| `updateCellDisplay(s)` | Renders cell voltage bars with deviation coloring and ▲▼ markers |
| `updateTempDisplay(s)` | Renders temperature grid with hot/warm/normal coloring |
| `updateMos(s, key, isOn)` | Updates a MOS status indicator (chg/dsg/bal) |
| `updateHexDump(s, frame)` | Renders color-coded hex dump (header=blue, data=white, CRC=yellow) |
| `showTab(s, name, btn)` | Switches tabs within a device panel |
| `updateGlobalStatus()` | Updates header device count and add-card visibility |
| `createDevicePanel(s)` | Generates and inserts dashboard HTML for a slot |
| `removeDevicePanel(s)` | Removes dashboard HTML for a slot |

---

## 12. Known Gotchas & Past Mistakes

1. **Frame types are the same for ALL protocols.** 0x01 = settings, 0x02 = cell data, 0x03 = device info. An earlier version incorrectly swapped 0x01 and 0x02 for "JK04" — this was wrong and caused garbage parsing.

2. **JK-PB models use JK02_32S, not JK04.** The `PB` in the model name does not mean JK04. Protocol detection is based on model string prefix and hardware version, not the product line name.

3. **The error bitmask at offset 134+ofs2 is big-endian** (unlike everything else in the frame which is little-endian). It is read as `(frame[offset] << 8) | frame[offset+1]`.

4. **The offset doubling matters.** For JK02_32S: `ofs = 16` (extra cells), then `ofs2 = ofs * 2 = 32` for fields after the resistance block. This is because both the cell block AND resistance block are each 16 bytes longer, so the cumulative offset is doubled.

5. **Temperature sensors 3-5 are at offsets 226, 224, 222** (in that order, NOT sequential). This matches the esphome source.

6. **The BMS disconnects without periodic commands.** Without heartbeat (0x96 every ~5s), the BMS drops the connection after 15-25 seconds. Heartbeat defaults to OFF — the user must enable it per device.

7. **Web Bluetooth requires HTTPS or localhost.** The PWA must be served over HTTPS (or from localhost) for `navigator.bluetooth` to be available.

8. **Some BMS models don't have 0xFFE2.** The code falls back to using 0xFFE1 for writes if 0xFFE2 is not found.

9. **Frame assembly needs header scanning.** BLE notifications can split across frame boundaries. The `extractFrames()` function scans for the `55 AA EB 90` header pattern and only processes complete 300-byte frames after CRC validation.

---

## 13. Extending the App

### Adding a new data field from cell data frame

1. Find the offset in the esphome source (`decode_jk02_cell_info_()` or `decode_jk04_cell_info_()`)
2. Add the field to `createBmsData()` with a default value
3. Read it in `parseJK02CellInfo()` and/or `parseJK04CellInfo()` using the appropriate helper (`jkGet16`, `jkGetS16`, etc.), remembering to add `ofs2` for JK02
4. Display it in `updateSlotUI()` and add the HTML element in `createDevicePanel()` with ID pattern `newfield-${s}`

### Adding a new command

1. Add the command byte to the `BLE` constants object
2. Create an `async function sendNewCommand(s)` that calls `sendCmd(s, buildCommand(BLE.CMD_NEW))`
3. Add a button in the Raw tab template in `createDevicePanel()`: `onclick="sendNewCommand(${s})"`

### Adding a 4th device slot

1. Change `MAX_SLOTS` from 3 to 4
2. Extend `slots` array: `const slots = [null, null, null, null]`
3. Add a 4th color to `SLOT_COLORS` (e.g., `'purple'`)
4. Add CSS for `.slot-3` border color and `.slot-3 .tab.active` highlight
5. Add `--purple` CSS variable

### Parsing settings frames (0x01)

Settings frames (type 0x01) are received but currently only logged. To parse them, add a `parseSettings(s, frame)` function and call it from the `processFrame()` switch statement. The esphome function `decode_jk02_settings_()` (line ~1148 in the .cpp) has the full offset table.

---

## 14. Service Worker

The service worker (`sw.js`) uses a **network-first** strategy: it tries to fetch from the network, and falls back to cache on failure. On install, it pre-caches `index.html` and `manifest.json`. The cache is named `jk-bms-v1` — bump this version string when deploying updates to force cache refresh.
