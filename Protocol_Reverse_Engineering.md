# JK BMS v5.10.1 — Bluetooth Low Energy Protocol Reverse Engineering

**App:** com.jktech.bms (JK BMS) v5.10.1 (build 230)
**APK:** jk-bms-5-10-1.apk
**Architecture:** Qt 6 + C++ native (arm64-v8a), Java BLE layer
**Analysis date:** 2026-02-07

---

## 1. Architecture Overview

The app has a layered architecture:

```
┌─────────────────────────────────────────────┐
│  QML UI (RealTimePage, SettingsPage, etc.)   │
├─────────────────────────────────────────────┤
│  Transfer class (C++ / libjkbms.so)         │
│  ├── sendCommand() / sendCommandImm()       │
│  ├── Table/Frame protocol engine            │
│  └── CRC-8 ROHC checksum                    │
├─────────────────────────────────────────────┤
│  libprotocore.so — Protocol frame engine    │
│  ├── J::Table (frame parsing/building)      │
│  ├── J::Frame (multi-table containers)      │
│  ├── J::Check (CRC-8/16/32, XOR, Sum)      │
│  └── J::Item (field definitions)            │
├─────────────────────────────────────────────┤
│  JBleChannel (C++ BLE abstraction)          │
│  ├── startScan() / stopScan()              │
│  ├── writeData(QByteArray)                  │
│  └── setRecvDataCallback()                  │
├─────────────────────────────────────────────┤
│  Java BLE Layer (com.smartsoft.ble)          │
│  ├── BleService — GATT operations           │
│  ├── Bluetooth — scan/connect/notify        │
│  └── NativeClass — JNI bridge to C++        │
├─────────────────────────────────────────────┤
│  Android Bluetooth LE Stack                 │
└─────────────────────────────────────────────┘
```

---

## 2. BLE Service & Characteristic UUIDs

| Role | UUID | Notes |
|------|------|-------|
| **Notify Service** | `0000ffe0-0000-1000-8000-00805f9b34fb` | Primary GATT service |
| **Write Service** | `0000ffe1-0000-1000-8000-00805f9b34fb` | For command writes |
| **Notify Characteristic** | `0000ffe1-0000-1000-8000-00805f9b34fb` | Device → App notifications |
| **Write Characteristic** | `0000ffe2-0000-1000-8000-00805f9b34fb` | App → Device commands |
| **Read Characteristic** | `0000ffe3-0000-1000-8000-00805f9b34fb` | Read operations |
| **CCCD (Descriptor)** | `00002902-0000-1000-8000-00805f9b34fb` | Enable notifications (standard) |

**Connection setup sequence:**
1. Scan for BLE devices (filtered by advertisement data — see §3)
2. Connect via GATT
3. Request MTU = **188 bytes** (0xBC) — default 20 bytes
4. Discover services → find `0000ffe0` service
5. Subscribe to `0000ffe1` characteristic notifications (write `[0x01, 0x00]` to CCCD)
6. Set write type = 1 (WRITE_TYPE_NO_RESPONSE) on `0000ffe2`
7. Signal `readyToWrite` → C++ native code starts protocol communication

---

## 3. Device Identification (Advertisement Filtering)

The app filters BLE advertisements using manufacturer-specific data. Supported device ID prefixes:

| ID | Format | Description |
|----|--------|-------------|
| `4A4B` | Short | "JK" prefix (ASCII 0x4A=J, 0x4B=K) |
| `4A4B0001` | Extended | JK with version |
| `F000` | Short | Variant |
| `F00088A03CA54AE421E0` | Full | Full UUID format |
| `650B` | Short | Variant |
| `C1EA` | Short | Variant |
| `0B2D` | Short | Variant |
| `4458` / `44585A4E424C45` | Short/Full | "DX" / "DXZNEBLE" |

**Advertisement data parsing** (62-byte payload):
- Byte 5: `0xE0` → JDY-type BLE module indicator
- Byte 6: `0xFF` → Manufacturer-specific data flag
- Byte 13: Device VID (vendor ID)
- Byte 14: Device variant code:
  - `0xA5` → Standard
  - `0xB1`, `0xB2` → Alternative variants
  - `0xC4`, `0xC5` → Alternative variants
  - Other → `JDY_Other` type

**BLE module types:** `Unknown`, `JDY` (JinDongYun), `JDY_Other`

---

## 4. Packet Transport Layer

### Transmission (App → Device)

Data is split into MTU-sized chunks and sent sequentially:

```
writeData(byte[] data):
    maxPkgSize = MTU - 3     // Effective payload per BLE packet
    fullPackets = data.length / maxPkgSize
    remainder = data.length % maxPkgSize

    for i in 0..fullPackets:
        gatt.writeCharacteristic(data[i*maxPkgSize : (i+1)*maxPkgSize])
        sleep(20ms)           // Inter-packet delay

    if remainder > 0:
        gatt.writeCharacteristic(data[fullPackets*maxPkgSize : end])
```

- **Default MTU:** 20 bytes → 17-byte payload
- **Requested MTU:** 188 bytes → 185-byte payload
- **Inter-packet delay:** 20ms
- **Write type:** WRITE_TYPE_NO_RESPONSE (type 1)

### Reception (Device → App)

```
onCharacteristicChanged(gatt, characteristic):
    if characteristic.UUID == UUID_NOTIFY_CHARACTERISTIC:
        data = characteristic.getValue()
        NativeClass.dataReceived(cppPointer, deviceAddress, data)
        // → routes to Transfer.notifyCallback()
        // → Table.updateRecv() parses frame
        // → Checksum verified
        // → Signal tableRecvChanged() emitted
```

---

## 5. Protocol Frame Structure

The protocol uses a **table/frame** hierarchical model defined in `.jsonds` files (JSON-based protocol definitions compiled into the native library).

### Frame Components

```
┌──────────┬─────────────────────┬──────────┐
│  Header  │      Payload        │ Checksum │
│ (domain) │  (items/domains)    │ (CRC-8)  │
└──────────┴─────────────────────┴──────────┘
```

- **Header:** Frame identification and metadata (domain-based)
- **Payload:** Hierarchical structure of Items organized by Domains
- **Checksum:** CRC-8 ROHC algorithm (primary), with support for CRC-16/32, XOR-Sum, and arithmetic Sum

### Checksum Algorithm: CRC-8 ROHC

```
Function: __crc8_rohc(const uint8_t *data, uint16_t length)
Algorithm: CRC-8/ROHC
Polynomial: 0x07 (ROHC standard)
Init value: 0xFF
XOR out: 0x00
Reflect in: true
Reflect out: true
```

The protocol engine also supports configurable checksums via `JCheck`:
- CRC-8, CRC-16, CRC-32 (with configurable polynomial, init, xor_out, refin, refout)
- XOR-Sum (byte-level XOR)
- Arithmetic Sum

### Known Frame Types

| Frame Domain | Purpose |
|-------------|---------|
| `frame/03` | Device info (software version, manufacturer device ID) |
| `frame/06` | Query/response frame |

### Protocol Tables

| Table Name | Path | Purpose |
|------------|------|---------|
| `table_recv` | `JKTech/JK_BMS/table_respond` | Incoming data from BMS |
| `table_respond` | `JKTech/JK_BMS/table_respond` | Response/configuration data |

The QML layer accesses protocol data through domain-based lookups:
```javascript
// Domain-based frame lookup pattern
var table = Transfer.tableRecv.tableByDomain('frame/03', ProtoMeta.DomainMark)
```

---

## 6. Data Fields & Protocol Items

### Real-Time Battery Status (from `table_recv`)

| Field Name | Unit | Description |
|-----------|------|-------------|
| Battery Voltage (V) | V | Total pack voltage |
| Battery Current (A) | A | Charge/discharge current |
| SOC Cap. Remain (AH) | Ah | Remaining capacity |
| SOC Full Charge Cap. (AH) | Ah | Full charge capacity |
| Min Cell Voltage (V) | V | Lowest cell voltage |
| Max Cell Voltage (V) | V | Highest cell voltage |
| Min Cell Voltage No | — | Cell # with lowest voltage |
| Max Cell Voltage No | — | Cell # with highest voltage |
| Min Temp (C) | °C | Lowest temperature |
| Max Temp (C) | °C | Highest temperature |
| Temp MOS (C) | °C | MOSFET temperature |
| Heat Current (A) | A | Heating current |
| Battery T1–T5 | °C | Temperature sensors 1–5 |
| Balance Status | — | Active balancing state |
| Charge MOS Status | — | Charge MOSFET on/off |
| Discharge MOS Status | — | Discharge MOSFET on/off |
| Heating Status | — | Heater on/off |
| Charger Status | — | Charger detected |
| Power on times | — | Boot counter |

### Protection & Alarm Status Bits

| Protection | Released State |
|-----------|---------------|
| Charge overcurrent protection | Charge overcurrent protection is released |
| Discharge overcurrent protection | Discharge overcurrent protection is released |
| Charge over temperature protection | Charge over temperature protection is released |
| Discharge over temperature protection | Discharge over-temperature protection is released |
| Charge low temperature protection | Charge low temperature protection is released |
| Discharge under temperature protection | Discharge under temperature protection Release |
| Cell overcharge protection | Cell overcharge protection is released |
| Cell undervoltage protection | Cell undervoltage protection is released |
| Battery overcharge protection | Battery overcharge protection is released |
| Battery undervoltage protection | Battery undervoltage protection is released |
| MOS over temperature protection | MOS over-temperature protection is released |
| Charge short circuit protection | Charge short circuit protection is released |
| Discharge short circuit protection | Discharge short circuit protection released |
| Discharge level 2 short circuit protection | — |
| Discharge OCP II / OCP III | — |
| Abnormal current sensor | Abnormal release of current sensor |
| Abnormal cancellation of coprocessor comm. | — |
| Charge MOS abnormal | — |
| Discharge MOS abnormal | — |

### Control Commands (via `Transfer`)

| Command | Method | Description |
|---------|--------|-------------|
| Start/stop charge | `enableCharge(bool, int)` | Toggle charge MOSFET |
| Start/stop discharge | `enableDischarge(bool, int)` | Toggle discharge MOSFET |
| Emergency | `enableEmerg(bool)` | Emergency shutdown |
| Emergency Ex | `enableEmergEx(bool)` | Extended emergency |
| Switch status | `switchStatus(int index, bool)` | Toggle specific output |
| Set password | `sendPassword(QString)` | Set/verify BLE password |
| Set device name | `setDeviceName(QString)` | Change BMS Bluetooth name |
| Read config | `requestReadConfig(bool isBle, bool isSub)` | Read all configuration |
| Write setting | `setSettingsValue(JItem*, QVariant)` | Write individual parameter |
| Erase all data | `requestEraseAllData()` | Factory reset |
| Time calibration | `sendTimeCalibration(bool set)` | Sync RTC |
| Query detail logs | `requestQueryDetailLogs(int count)` | Retrieve event logs |
| Query system log | `requestQuerySystemLog()` | Retrieve system log |
| Export config | `exportConfig(QString path)` | Save config to file |
| Import config | `importConfig(QString path, pwd)` | Load config from file |
| Charger smart reco | `chargerSmartReco(bool)` | Smart charger reconnect |
| Voltage calibration | implied | Calibrate voltage readings |
| Equalize enable | `equEnable()` | Enable cell equalization |
| Factory date | `sendFactoryDate(QString)` | Set manufacturing date |
| LCD/Buzzer trigger | `sendLcdBuzzerTrigger(ushort)` | Trigger display/buzzer |
| Dry contact triggers | `sendDry1Trigger/sendDry2Trigger(ushort)` | Configurable relay triggers |
| User data | `sendUserData/sendUserData2()` | Custom user data fields |

---

## 7. Multi-Interface Protocol Support

The BMS supports multiple communication interfaces, each with selectable protocol variants:

### UART Protocols (21 options)

| ID | Protocol |
|----|----------|
| 0 | 4G-GPS Remote module Common protocol V4.2 |
| 1 | JK BMS RS485 Modbus V1.0 |
| 2 | NIU U SERIES |
| 3 | China tower shared battery cabinet V1.1 |
| 4 | PACE RS485 Modbus V1.3 |
| 5 | PYLON low voltage Protocol RS485 V3.5 |
| 6 | Growatt BMS RS485 Protocol 1xSxxP ESS Rev2.01 |
| 7 | Voltronic Inverter and BMS 485 comm protocol |
| 8 | China tower shared battery cabinet V2.0 |
| 9 | WOW RS485 Modbus V1.3 |
| 10 | JK BMS LCD Protocol V2.0 |
| 11 | UART1 User customization |
| 12 | UART2 User customization |
| 13 | (9600) JK BMS RS485 Modbus V1.0 |
| 14 | (9600) PYLON low voltage Protocol RS485 V3.5 |
| 15 | JK BMS PBxx SERIES LCD Protocol V1.0 |
| 16 | JK BMS LIN BUS V1.0 |
| 17–20 | RS485 Protocol 17–20 (reserved) |

### CAN Protocols (21 options)

| ID | Protocol |
|----|----------|
| 0 | JK BMS CAN Protocol (250K) V2.0 |
| 1 | Deye Low-voltage hybrid inverter CAN V1.0 |
| 2 | PYLON Low-voltage V1.2 |
| 3 | Growatt BMS CAN-Bus low-voltage Rev 05 |
| 4 | Victron CANbus BMS protocol |
| 5 | MEGAREVO Hybrid BMS CAN V1.0 |
| 6 | JK BMS CAN Protocol (500K) V2.0 |
| 7 | INVT BMS CAN Bus protocol V1.02 |
| 8 | GoodWe LV BMS Protocol (EX/EM/S-BP/BP) |
| 9 | FSS ConnectingBat V1.0 |
| 10 | MUST PV1800F CAN V1.04.04 |
| 11 | LuxpowerTek Battery CAN protocol V01 |
| 12–13 | CAN BUS User customization 1–2 |
| 14–20 | CAN BUS Protocol 014–020 (reserved) |

### Dry Contact / Relay Trigger Events

| ID | Trigger |
|----|---------|
| 0 | OFF |
| 1 | Low SOC |
| 2 | Battery Over Voltage |
| 3 | Battery Under Voltage |
| 4 | Battery Cell Over Voltage |
| 5 | Battery Cell Under Voltage |
| 6 | Charge Over Current |
| 7 | Discharge Over Current |
| 8 | Battery Over Temperature |
| 9 | MOSFET Over Temperature |
| 10 | System Alarm |
| 11 | Battery Low Temperature |
| 12 | Remote Control |
| 13 | Above SOC |
| 14 | MOSFET Abnormal |

---

## 8. Supported BMS Models

| Model | Description |
|-------|-------------|
| JK-B1A24S | 24-series LiFePO4 |
| JK-B1A24S-P | 24-series with parallel |
| JK-B1A24S-PLW | 24-series parallel low-power |
| JK-B1A24S-PSR | 24-series parallel SR variant |
| JK-B1A32S | 32-series |
| JK-BXAXS-XP | Multi-cell configurable variant |
| JK-DZ08-B1A24S | 8-series variant |

---

## 9. Authentication & Security

- **Bluetooth Password:** Protocol requires a `bluetoothPwd` section in the configuration. Without it, the error "Protocol is invalid! (Has no section bluetoothPwd)" is thrown.
- **Device pairing:** Uses a password-based pairing mechanism — the prompt says: `"<device>" want to make a pair, please input password`
- **Settings password:** Separate from device password, protectable: "Modify password of settings"
- **Configuration import/export:** Password-protected

---

## 10. JNI Bridge (Java ↔ C++)

The `NativeClass` provides the JNI interface between the Java BLE layer and the C++ protocol engine:

| Native Method | Signature | Purpose |
|--------------|-----------|---------|
| `dataReceived` | `(J, String, byte[])` | BLE data → C++ protocol parser |
| `deviceAdded` | `(J, String, String, String, int, int)` | New device discovered |
| `deviceConnected` | `(J, String, String)` | GATT connected |
| `deviceDisconnected` | `(J, String, String)` | GATT disconnected |
| `deviceUpdateName` | `(J, String, String)` | Device name changed |
| `deviceUpdateRSSI` | `(J, String, int)` | RSSI updated |
| `readyToWrite` | `(J, String)` | BLE ready for commands |
| `permissionStateChanged` | `(J, boolean, String)` | Permission granted/denied |

The first parameter `J` (long) is always a C++ object pointer (`mCppThis`), enabling the JNI call to route to the correct `Transfer` instance.

---

## 11. Protocol Definition Files

The protocol is defined in external `.jsonds` files compiled into the native library:

| Resource Path | Content |
|--------------|---------|
| `:/resource/protocol/en_US.jsonds` | English protocol spec (table/frame definitions) |
| `:/resource/protocol/global-en_US.jsonds` | Shared/global definitions |
| `:/resource/others/devices.json` | Supported device list |
| `:/resource/others/protocols-{}.json` | Protocol variant definitions |

These define: frame structure, field offsets, data types, endianness, domains, checksums, and default values. The protocol engine (`libprotocore.so`) parses these definitions to dynamically handle different BMS models and firmware versions.

Key data handling functions in libprotocore:
- `setDataToBuffer` / `dataFromBuffer` — Field-level read/write
- `isBigEndian` — Endianness per field
- `followOffset` — Dynamic offset resolution
- `prettyValue` — Display formatting
- `orgDataStringFromData` — Raw data conversion

---

## 12. Data Processing Pipeline

### Receive Path (BMS → App)

```
1. BLE PHY → Android BT stack
2. onCharacteristicChanged(ffe1 notification)
3. NativeClass.dataReceived(ptr, addr, byte[])
4. Transfer.notifyCallback()
5. Table.setBuffer(raw_data)
6. Table.updateRecv(true)
7. Check.processCheckCrc() — verify CRC-8 ROHC
8. Items extracted from domains by offset/length
9. Signal: tableRecvChanged()
10. QML UI updates (bindings to Item.data)
```

### Send Path (App → BMS)

```
1. QML: Transfer.sendCommand(cmd_id, params)
2. Table.updateSend(flags) — build frame
3. Items written to buffer by offset/length
4. Check.createCrcTable() + processCheckCrc() — append CRC
5. JSuperChannel.writeData(QByteArray)
6. BleService.writeData(byte[])
7. Chunk into MTU-sized packets (185 bytes max)
8. writeCharacteristic(ffe2) × N with 20ms delays
```

---

## 13. Summary for Implementation

To communicate with a JK BMS device over BLE:

1. **Scan** for devices advertising manufacturer data starting with `4A4B` (or other known prefixes)
2. **Connect** via GATT and request MTU 188
3. **Discover services** and find service `0000ffe0`
4. **Enable notifications** on characteristic `0000ffe1` (write `[0x01, 0x00]` to descriptor `00002902`)
5. **Write commands** to characteristic `0000ffe2` (chunked to MTU-3 with 20ms delays)
6. **Receive data** via notifications on `0000ffe1`
7. **Verify** all received frames with CRC-8 ROHC checksum
8. **Parse** frame payload according to the table/domain/item structure

The actual byte-level protocol format (field offsets, command IDs, frame headers) is defined in the `.jsonds` files compiled into `libjkbms_arm64-v8a.so`. These are not extractable as plain text but could be captured via live BLE traffic analysis (e.g., using nRF Connect, Wireshark with BLE sniffer, or Android BLE logging).

### Recommended Next Steps

- **Live traffic capture:** Use Android BLE HCI snoop log or nRF Connect to capture actual protocol frames
- **Frida instrumentation:** Hook `NativeClass.dataReceived()` and `BleService.writeData()` to log all traffic at the Java layer
- **Cross-reference with community projects:** Several open-source JK BMS implementations exist that document the wire protocol (e.g., ESPHome jk_bms component, jkbms Python library)
