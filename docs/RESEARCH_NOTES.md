> Historical source notes from captured traffic. Some earlier interpretations were provisional; use [MODBUS.md](MODBUS.md) for curated current findings.

# ECO-WORTHY MPPT — Modbus RTU decoding notes

**Compiled:** 2026-11-15  
**Source:** previously captured ESPHome RS485 scanner logs and ESPHome charger sensor configuration.  
**Purpose:** preserve the observed protocol, raw frames, register values, and mappings for native ESPHome integration.

## Current consolidated register map (updated 2026-11-15)

This section is the **current reference**. The original capture transcripts and earlier interpretations below are retained as historical evidence; where an older caution conflicts with this section, use this section. Latest main-block snapshot: 19:19:34 (November 15). Register addresses are strictly ascending. **Verified** means matched to app/user-confirmed behavior; **identified** means supported by traffic; **probable** and **unknown** are not confirmed.

| Register | Description | Raw | Decoded | Status |
|---|---|---:|---|---|
| `0x1500` | Controller ID | 1 | ID 1 | Identified |
| `0x1501` | PV voltage | 0 | 0.0 V | Identified |
| `0x1502` | Battery voltage | 519 | 51.9 V | Verified |
| `0x1503` | Charging current | 0 | 0.0 A | Identified |
| `0x1504` | Load voltage | 0 | 0.0 V | Identified |
| `0x1505` | Load current | 0 | 0.0 A | Identified |
| `0x1506` | Charging power | 0 | 0 W | Identified |
| `0x1507` | Load power | 0 | 0 W | Identified |
| `0x1508` | External temperature | 77 | 77 °F | App verified |
| `0x1509` | Internal controller temperature | 59 | 59 °F | Probable |
| `0x150A` | Controller-reported SOC | 100 | 100% | Identified; differs from JK BMS SOC |
| `0x150B` | Charging active status | 0 | Inactive | Tentative; **not established as charging enable switch** |
| `0x150C` | Fault bitmask | 0 | No active faults | Partially mapped |
| `0x150D` | Total generated energy | 146 | 14.6 kWh | App verified; ÷10 |
| `0x150E` | Unknown | 0 | Possible energy counter high word | Unverified hypothesis |
| `0x150F` | Unknown | 4 | Unknown | Unidentified; **do not label controller type** |
| `0x1510` | Unknown | 0 | Unknown | Unidentified |
| `0x1511` | Unknown | 0 | Unknown | Unidentified |
| `0x1512` | Temperature unit selection | 1 | Fahrenheit (1=°F, 0=°C) | Verified via observed app write |
| `0x1513` | **Load** operating mode | 5117 | `0x13FD` = OFF | Identified; **not charging enable** |
| `0x1514` | Load output state | 0 | OFF | Identified |
| `0x1515` | Battery chemistry | 4 | LiFePO₄ | User verified; 0=Custom |
| `0x1516` | Battery system voltage | 4 | 48 V | Verified |
| `0x1517` | Overdischarge protection | 440 | 44.0 V | Identified |
| `0x1518` | Overcharge protection | 584 | 58.4 V | Identified |
| `0x1519` | Overdischarge recovery | 480 | 48.0 V | Identified |
| `0x151A` | Overcharge recovery | 560 | 56.0 V | Identified |
| `0x151B` | Constant charging voltage | 572 | 57.2 V | Identified |
| `0x151C` | Float voltage | 65535 | `0xFFFF` (unset/not applicable) | Identified |
| `0x151D` | Equalization voltage | 65535 | `0xFFFF` (unset/not applicable) | Identified |
| `0x151E` | Equalization duration | 65535 | `0xFFFF` (unset/not applicable) | Identified |
| `0x151F` | Maximum charging current | 600 | **60.0 A** (÷10) | Probable; app does not expose editing |
| `0x1520` | Unknown | 1 | Unknown | Unidentified |
| `0x1521` | Unknown | 2 | Unknown | Unidentified |

### Fault bits in `0x150C`

| Bit | Mask | Interpretation | Evidence |
|---|---|---|---|
| 0 | `0x0001` | Battery overvoltage | Previously identified when 12 V mode was selected on ~52 V battery |
| 7 | `0x0080` | Controller battery voltage too low | Observed as reported controller voltage collapsed, cleared on recovery |
| All others | — | Unknown | Not mapped |

Multiple bits may be set simultaneously. Low-voltage event showed controller-reported voltage as low as 24.6 V while JK BMS stayed near 51.7 V; investigate connection/sensing rather than assuming batteries were actually that low.

### Historical generation `0x1600–0x162A`

43 words: **12 monthly (`0x1600–0x160B`) and 31 daily (`0x160C–0x162A`)** generation records, with raw ÷10 = kWh. Earlier investigation identified newest/current record first in each group; exact rollover/date alignment still needs confirmation. During production, only current day and current month should increase; older records should remain unchanged until rollover. Full 43-register example and per-register values appear below.

### App fields not yet tied to a register

- **Charging ON/OFF switch:** separate from load ON/OFF; register and write payload not yet established. `0x150B` reports charging activity, not necessarily user enable state.
- **Controller type:** app displays **MPPT Controller**, read-only. Could be hardcoded or derived from device identity; **no register confirmed**. In particular, `0x150F=4` is not evidence of controller type.

**Remaining unidentified main-block registers:** `0x150E`, `0x150F`, `0x1510`, `0x1511`, `0x1520`, `0x1521` (six). Keep raw values visible for passive comparison; avoid writes to unknown registers.

---

## Verified from captured traffic

| Setting | Observation |
|---|---|
| Physical interface | RS485, Modbus RTU |
| Baud rate | **115200** (scanner repeatedly reported valid Modbus frames at this speed) |
| Slave address | **1** (`0x01`) |
| Read function | **03** — holding registers |
| Write function observed | **16** (`0x10`) — write multiple holding registers |
| Read block A | Start `0x1500`, count `0x0022` = **34 registers** |
| Read block B | Start `0x1600`, count `0x002B` = **43 registers** |
| Write block observed | Start `0x1515`, count `0x000A` = **10 registers** |
| Data encoding | 16-bit register words, high byte first within each word |
| Captured scanner status | `Locked - Modbus detected`; valid frame count >4,400 in sample |

The scanner log verifies the baud rate and the captured RTU traffic. **Parity/stop bits were configured as 8N1 in the ESPHome bridge**, but the capture alone does not independently establish serial framing. The capture was passive; it does not prove the replacement ESPHome master can poll successfully until tested.

## Register map used in the ESPHome charger configuration

Addresses are **zero-based Modbus holding-register addresses** (not 4xxxx display notation). The mappings and scaling below were explicitly present in the earlier `modbus_bridge_single_file.yaml` configuration. Some have strong consistency with the observed raw data; the exact semantic labels are not independently proven for every register.

| Address (hex) | Decimal | Name in configuration | Type | Scaling | Example raw from capture | Decoded example |
|---|---:|---|---|---|---:|---:|
| `0x1501` | 5377 | MPPT PV Voltage | unsigned 16-bit | × 0.1 V | `0x034A` = 842 | **84.2 V** |
| `0x1502` | 5378 | MPPT Battery Voltage | unsigned 16-bit | × 0.1 V | `0x020F` = 527 | **52.7 V** |
| `0x1503` | 5379 | MPPT Charging Current | unsigned 16-bit | × 0.1 A | `0x0026` = 38 | **3.8 A** |
| `0x1506` | 5382 | MPPT Charging Power | unsigned 16-bit | × 1 W | `0x00C8` = 200 | **200 W** |
| `0x1507` | 5383 | MPPT Load Power | unsigned 16-bit | × 1 W | `0x0005` = 5 | **5 W** |
| `0x150A` | 5386 | MPPT Reported SOC | unsigned 16-bit | × 1 % | `0x0043` = 67 | **67%** |

**Interpretation caution:** `0x1506` = 200 W is consistent with 52.7 V × 3.8 A ≈ 200 W. `0x1507` and `0x150A` were assigned labels in the saved configuration, but their meaning should not be treated as independently validated solely by this snapshot. At 13:51:20, `0x150A` was `0x0064` (100); at 13:51:37, it was `0x0043` (67). Do not mix values from those two snapshots. Historical note: the energy scale was not yet confirmed at the time of this earlier snapshot. Subsequent app comparison established `0x150D` raw ÷10 = kWh; see the current consolidated map above.

## Full captured block A: `0x1500`–`0x1521`

Captured at approximately 13:51:37:

**Request (8 bytes):**

```text
01 03 15 00 00 22 C1 DF
```

- `01`: slave 1
- `03`: read holding registers
- `15 00`: start `0x1500`
- `00 22`: read 34 registers
- `C1 DF`: CRC bytes, low byte first

**Response (73 bytes):**

```text
01 03 44
00 01 03 4A 02 0F 00 26 02 0F 00 01 00 C8 00 05
00 4D 00 66 00 43 00 01 00 00 00 8E 00 00 00 04
00 00 00 00 00 01 13 FC 00 01 00 00 00 04 01 C0
02 58 01 E8 02 48 02 48 02 28 02 50 00 78 02 58
00 01 00 02
9E C0
```

`44` = 68 payload bytes = 34 registers. `9E C0` are response CRC bytes.

| Register | Raw hex | Unsigned decimal |
|---|---|---:|
| `0x1500` | `0x0001` | 1 |
| `0x1501` | `0x034A` | 842 |
| `0x1502` | `0x020F` | 527 |
| `0x1503` | `0x0026` | 38 |
| `0x1504` | `0x020F` | 527 |
| `0x1505` | `0x0001` | 1 |
| `0x1506` | `0x00C8` | 200 |
| `0x1507` | `0x0005` | 5 |
| `0x1508` | `0x004D` | 77 |
| `0x1509` | `0x0066` | 102 |
| `0x150A` | `0x0043` | 67 |
| `0x150B` | `0x0001` | 1 |
| `0x150C` | `0x0000` | 0 |
| `0x150D` | `0x008E` | 142 |
| `0x150E` | `0x0000` | 0 |
| `0x150F` | `0x0004` | 4 |
| `0x1510` | `0x0000` | 0 |
| `0x1511` | `0x0000` | 0 |
| `0x1512` | `0x0001` | 1 |
| `0x1513` | `0x13FC` | 5116 |
| `0x1514` | `0x0001` | 1 |
| `0x1515` | `0x0000` | 0 |
| `0x1516` | `0x0004` | 4 |
| `0x1517` | `0x01C0` | 448 |
| `0x1518` | `0x0258` | 600 |
| `0x1519` | `0x01E8` | 488 |
| `0x151A` | `0x0248` | 584 |
| `0x151B` | `0x0248` | 584 |
| `0x151C` | `0x0228` | 552 |
| `0x151D` | `0x0250` | 592 |
| `0x151E` | `0x0078` | 120 |
| `0x151F` | `0x0258` | 600 |
| `0x1520` | `0x0001` | 1 |
| `0x1521` | `0x0002` | 2 |

## Captured block B: `0x1600`–`0x162A` (all 43 registers)

The original passive capture contains **all 43 registers**. An earlier version of this document accidentally omitted the zero word at `0x160B`, shifting the subsequent address labels. The table below is rebuilt from the 86 captured payload bytes (not hand-transcribed).

**Request:** `01 03 16 00 00 2B 01 9D`

**Response:**

```text
01 03 56
00 5A 00 EE 01 46 03 C1 08 24 01 18 00 00 00 00
00 00 00 00 00 00 00 00 00 1E 00 12 00 04 00 01
00 02 00 01 00 01 00 04 00 05 00 05 00 05 00 06
00 05 00 04 00 00 00 00 00 05 00 05 00 05 00 09
00 06 00 05 00 04 00 05 00 04 00 05 00 07 00 07
00 08 00 07 00 07
CB 51
```

`0x56` = 86 payload bytes = 43 words; `CB 51` = CRC bytes.

### Recovered historical-generation mapping (from November 14 analysis)

The app reads **12 monthly generation totals followed by 31 daily generation totals**, all stored as unsigned 16-bit integers with **raw ÷ 10 = kWh**. The earlier reverse-engineering conversation identified this layout; it was mistakenly omitted from previous versions of this document.

- `0x1600–0x160B`: **12 monthly values**, current month first, then previous months (relative offsets 0–11).
- `0x160C–0x162A`: **31 daily values**, most recent day first, then previous days (relative offsets 0–30).
- **Date rollover and exact day/calendar alignment remain to be verified.** Do not assign fixed calendar dates without the capture timestamp and rollover test.
- Values below belong to the 13:51 capture. An earlier 14:xx snapshot showed `0x1600=9.2 kWh` and `0x160C=3.1 kWh`, versus `9.0` and `3.0 kWh` here; these are different samples, not different scales.

| Register | Raw hex | Raw decimal | Decoded kWh | App field / relative position |
|---|---|---:|---:|---|
| `0x1600` | `0x005A` | 90 | 9.0 | Monthly generation — current month |
| `0x1601` | `0x00EE` | 238 | 23.8 | Monthly generation — 1 month ago |
| `0x1602` | `0x0146` | 326 | 32.6 | Monthly generation — 2 months ago |
| `0x1603` | `0x03C1` | 961 | 96.1 | Monthly generation — 3 months ago |
| `0x1604` | `0x0824` | 2084 | 208.4 | Monthly generation — 4 months ago |
| `0x1605` | `0x0118` | 280 | 28.0 | Monthly generation — 5 months ago |
| `0x1606` | `0x0000` | 0 | 0.0 | Monthly generation — 6 months ago |
| `0x1607` | `0x0000` | 0 | 0.0 | Monthly generation — 7 months ago |
| `0x1608` | `0x0000` | 0 | 0.0 | Monthly generation — 8 months ago |
| `0x1609` | `0x0000` | 0 | 0.0 | Monthly generation — 9 months ago |
| `0x160A` | `0x0000` | 0 | 0.0 | Monthly generation — 10 months ago |
| `0x160B` | `0x0000` | 0 | 0.0 | Monthly generation — 11 months ago |
| `0x160C` | `0x001E` | 30 | 3.0 | Daily generation — most recent day |
| `0x160D` | `0x0012` | 18 | 1.8 | Daily generation — 1 day before most recent |
| `0x160E` | `0x0004` | 4 | 0.4 | Daily generation — 2 days before most recent |
| `0x160F` | `0x0001` | 1 | 0.1 | Daily generation — 3 days before most recent |
| `0x1610` | `0x0002` | 2 | 0.2 | Daily generation — 4 days before most recent |
| `0x1611` | `0x0001` | 1 | 0.1 | Daily generation — 5 days before most recent |
| `0x1612` | `0x0001` | 1 | 0.1 | Daily generation — 6 days before most recent |
| `0x1613` | `0x0004` | 4 | 0.4 | Daily generation — 7 days before most recent |
| `0x1614` | `0x0005` | 5 | 0.5 | Daily generation — 8 days before most recent |
| `0x1615` | `0x0005` | 5 | 0.5 | Daily generation — 9 days before most recent |
| `0x1616` | `0x0005` | 5 | 0.5 | Daily generation — 10 days before most recent |
| `0x1617` | `0x0006` | 6 | 0.6 | Daily generation — 11 days before most recent |
| `0x1618` | `0x0005` | 5 | 0.5 | Daily generation — 12 days before most recent |
| `0x1619` | `0x0004` | 4 | 0.4 | Daily generation — 13 days before most recent |
| `0x161A` | `0x0000` | 0 | 0.0 | Daily generation — 14 days before most recent |
| `0x161B` | `0x0000` | 0 | 0.0 | Daily generation — 15 days before most recent |
| `0x161C` | `0x0005` | 5 | 0.5 | Daily generation — 16 days before most recent |
| `0x161D` | `0x0005` | 5 | 0.5 | Daily generation — 17 days before most recent |
| `0x161E` | `0x0005` | 5 | 0.5 | Daily generation — 18 days before most recent |
| `0x161F` | `0x0009` | 9 | 0.9 | Daily generation — 19 days before most recent |
| `0x1620` | `0x0006` | 6 | 0.6 | Daily generation — 20 days before most recent |
| `0x1621` | `0x0005` | 5 | 0.5 | Daily generation — 21 days before most recent |
| `0x1622` | `0x0004` | 4 | 0.4 | Daily generation — 22 days before most recent |
| `0x1623` | `0x0005` | 5 | 0.5 | Daily generation — 23 days before most recent |
| `0x1624` | `0x0004` | 4 | 0.4 | Daily generation — 24 days before most recent |
| `0x1625` | `0x0005` | 5 | 0.5 | Daily generation — 25 days before most recent |
| `0x1626` | `0x0007` | 7 | 0.7 | Daily generation — 26 days before most recent |
| `0x1627` | `0x0007` | 7 | 0.7 | Daily generation — 27 days before most recent |
| `0x1628` | `0x0008` | 8 | 0.8 | Daily generation — 28 days before most recent |
| `0x1629` | `0x0007` | 7 | 0.7 | Daily generation — 29 days before most recent |
| `0x162A` | `0x0007` | 7 | 0.7 | Daily generation — 30 days before most recent |

## Captured battery-type change / custom configuration write

At **13:51:23**, the BT-02/app sent Modbus function `0x10` (write multiple registers), starting at `0x1515`. The user confirmed that this operation corresponded to selecting **Custom** battery mode, that the ten fields below occur in the same order as the app, and that `0` means Custom, `4` means 48 V, and `120` means 120 minutes.

**Request:**

```text
01 10 15 15 00 0A 14 00 00 00 04 01 C0 02 58 01 E8 02 48 02 48 02 28 02 50 00 78 06 A2
```

**Acknowledgment:** `01 10 15 15 00 0A 55 C6`

This is **10 registers / 20 data bytes**, not a 10-byte write.

| Register | App field (user-confirmed order) | Raw hex | Decoded value |
|---|---|---|---|
| `0x1515` | Battery Type | `0x0000` | **Custom (code 0)** |
| `0x1516` | Battery Voltage | `0x0004` | **48 V system (code 4)** |
| `0x1517` | Overdischarge protection voltage | `0x01C0` | **44.8 V** |
| `0x1518` | Overcharge protection voltage | `0x0258` | **60.0 V** |
| `0x1519` | Overdischarge recovery voltage | `0x01E8` | **48.8 V** |
| `0x151A` | Overcharge recovery voltage | `0x0248` | **58.4 V** |
| `0x151B` | Constant charging voltage | `0x0248` | **58.4 V** |
| `0x151C` | Float charging voltage | `0x0228` | **55.2 V** |
| `0x151D` | Equalized charging voltage | `0x0250` | **59.2 V** |
| `0x151E` | Equalized charging time | `0x0078` | **120 minutes** |

**Status:** All ten field names, their order, and the captured values are confirmed by the user. Voltage settings use 0.1 V per count. The user confirmed the time unit is minutes. Codes for battery types other than Custom and system voltages other than 48 V were not captured.

**Safety:** Preserve this as an observed write, not a command to replay automatically. Reading configuration is safe; writing charging limits requires explicit control and validation.


## ESPHome bridge hardware and integration

| Component | Existing configuration |
|---|---|
| Controller | ESP32-S3 N16R8 (`modbus_bridge`) |
| Transceiver | MAX3485 RS485 |
| RX / TX / DE-RE | GPIO16 / GPIO17 / GPIO18 |
| UART | 115200, 8N1 |
| Modbus slave | 1 |
| Original external master | BT-02 RS485 Bluetooth adapter |
| Home Assistant | Native ESPHome API already connected, two JK BMS devices working over BLE |

The BT-02 and ESPHome must not both act as masters on the same two-wire RS485 bus. When changing to active ESPHome polling, disconnect the BT-02. Preserve the JK BMS BLE configuration.

## Native ESPHome implementation reference

The recovered sensor entries use `platform: modbus_controller`, `register_type: holding`, and `value_type: U_WORD`. The charger uses `modbus_controller` address `0x01` and the UART pins above. Add the six entries to the **existing** `sensor:` list, not a second top-level key. No separate Home Assistant Modbus integration is needed once the native ESPHome sensors are installed.

## What remains unknown

- Semantic labels for the remaining **unmapped** monitoring/diagnostic registers in `0x1500`. **All 43 `0x1600–0x162A` registers are mapped** to 12 monthly and 31 daily generation-history values (raw ÷ 10 kWh).
- The high-word/overflow interpretation of `0x150E` remains unknown; `0x150D` raw ÷10 = kWh was subsequently verified against the app.
- Safety bounds and enumeration values for **other** battery types/nominal voltages; the ten captured Custom/48 V settings and their order are decoded.
- Whether the new native ESPHome active-master configuration has successfully polled the charger; the recovered log documents passive traffic capture.

This file separates **captured values**, **existing ESPHome sensor labels**, and **user-confirmed battery configuration fields**. The complete second block is preserved without omissions.

## Recovered earlier conversation findings (2026-11-15)

**Important:** The preceding document was an incomplete consolidation, not the full state of the reverse engineering. The following register interpretations were recovered from prior conversation context and supersede earlier claims that these fields are unidentified. Keep observed values separate from inferred scaling.

| Register | Recovered field | Decoding / observed behavior |
|---|---|---|
| `0x1501` | PV voltage | U16 × 0.1 V |
| `0x1502` | Battery voltage | U16 × 0.1 V |
| `0x1503` | Charging current | U16 × 0.1 A |
| `0x1504` | Load voltage | U16 × 0.1 V |
| `0x1505` | Load current | U16 × 0.1 A |
| `0x1506` | Charging power | U16, watts |
| `0x1507` | Load power | U16, watts |
| `0x150A` | Reported SOC | U16, percent; earlier capture showed 100%; other snapshot showed 67%, so do not assume fixed 100% |
| `0x150B` | Charging active | 1 observed when charging |
| `0x150C` | Fault indication | 0 normal; 1 observed during overvoltage |
| `0x150D` | Total energy counter | Earlier reading 0x008D = 141 was provisionally interpreted; subsequent app comparison confirmed ÷10 kWh. Later 0x008E = 142 raw. |
| `0x1513` | Load mode | Codes below |
| `0x1514` | Load output status | 0 OFF; 1 ON observed in separate snapshots |
| `0x1515–0x151E` | Custom battery configuration | All ten app fields and scaling documented above |

### Previously decoded load-mode codes (`0x1513`)

| Raw value | App selection |
|---|---|
| `0x13FC` | ON |
| `0x13FD` | OFF |
| `0x13EC` | Light control |
| `0x13ED` | 1 hour |
| `0x13EE` | 2 hours |

### Additional live capture recovered from earlier analysis (14:28:47–14:29:01)

| Register | First value | Last value | Interpretation |
|---|---|---|---|
| `0x1501` | 85.6 V | 86.6 V | PV voltage |
| `0x1502` | 52.1 V | 52.1 V | Battery voltage |
| `0x1503` | 2.1 A | 2.7 A | Charging current |
| `0x1506` | 109 W | 140 W | Charging power |
| `0x150A` | 100% | 100% | Reported SOC |
| `0x150B` | 1 | 1 | Charging active |
| `0x150C` | 0 | 0 | No active fault |
| `0x150D` | 14.3 (provisional interpretation) | 14.3 | Energy counter; units unverified |
| `0x1513` | `0x13FD` | `0x13FD` | Load mode OFF |
| `0x1514` | 0 | 0 | Load output OFF |

**Recovered November 14 mapping:** All 43 registers at `0x1600–0x162A` are generation-history entries: 12 months (`0x1600–0x160B`) and 31 days (`0x160C–0x162A`), newest first, each scaled by 0.1 kWh. The complete per-register mapping is in the block B table above. Calendar rollover alignment still needs validation.