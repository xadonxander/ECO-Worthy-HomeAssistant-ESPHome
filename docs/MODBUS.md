# ECO-WORTHY MPPT Modbus RTU findings

These notes document **observed traffic from one ECO-WORTHY 60 A MPPT controller**, supported by the captures in [RESEARCH_NOTES.md](RESEARCH_NOTES.md) and the working ESPHome configuration. This is not an official protocol specification. Some semantics remain tentative. Addresses are **zero-based hex holding-register addresses**, not 4xxxx notation.

## Physical/transport protocol

| Detail | Observation |
|---|---|
| Physical bus | RS485, Modbus RTU |
| Device/slave | `0x01` (1) |
| Serial | 115200 baud; working ESPHome uses 8N1 |
| Read | Function `0x03` (read holding registers) |
| Write | Function `0x10` hex / **16 decimal** (write multiple holding registers) |
| Main read | `0x1500`, **34 registers** through `0x1521` |
| History read | `0x1600`, **43 registers** through `0x162A` |
| Battery save | FC16 starting `0x1515`, **10 registers** |
| Load control | FC16 starting `0x1513`, **1 register** |
| Temperature selection | FC16 starting `0x1512`, **1 register** in working ESPHome implementation |

**FC10 is hexadecimal function 0x10, not a count of ten registers.** Modbus RTU CRC16 uses polynomial 0xA001 and transmits low CRC byte first.

## Main holding register map

| Address | Meaning | Scaling / values | Confidence |
|---|---|---|---|
| `0x1500` | Controller ID | raw ID; observed 1 | Identified |
| `0x1501` | PV voltage | raw ÷ 10 V | Identified |
| `0x1502` | Battery voltage | raw ÷ 10 V | Verified |
| `0x1503` | Charging current | raw ÷ 10 A | Identified |
| `0x1504` | Load voltage | raw ÷ 10 V | Identified |
| `0x1505` | Load current | raw ÷ 10 A | Identified |
| `0x1506` | Charging power | raw W | Identified |
| `0x1507` | Load power | raw W | Identified |
| `0x1508` | External temperature | Numeric value interpreted in **controller-selected unit** | App compared |
| `0x1509` | Controller internal temperature | Numeric value in controller-selected unit; identity probable | Probable |
| `0x150A` | Controller-reported SOC | raw %; distinct from JK BMS | Identified |
| `0x150B` | Charging activity | 0/1; **not charging enable** | Tentative |
| `0x150C` | Fault bitmask | bit 0 overvoltage; bit 7 low voltage noted | Partially mapped |
| `0x150D` | Total generated energy | raw ÷ 10 kWh | App compared |
| `0x150E` | Unknown | no validated decoding | Unknown |
| `0x150F` | Unknown | observed 4; **not confirmed controller type** | Unknown |
| `0x1510` | Unknown | raw | Unknown |
| `0x1511` | Unknown | raw | Unknown |
| `0x1512` | Temperature unit | **0 Celsius; 1 Fahrenheit** | Observed/readback |
| `0x1513` | Load mode | Five verified values: see mode table below | User-verified app modes |
| `0x1514` | Load status | 0 OFF; 1 ON | Verified readback |
| `0x1515` | Battery chemistry | 0 Custom; 4 LiFePO4 confirmed in this setup | Captured/app compared |
| `0x1516` | System voltage | 4 = 48 V; other values unconfirmed here | Verified 48 V |
| `0x1517` | Overdischarge protection | raw ÷ 10 V | Identified |
| `0x1518` | Overcharge protection | raw ÷ 10 V | Identified |
| `0x1519` | Overdischarge recovery | raw ÷ 10 V | Identified |
| `0x151A` | Overcharge recovery | raw ÷ 10 V | Identified |
| `0x151B` | Constant charging voltage | raw ÷ 10 V | Identified |
| `0x151C` | Float charging voltage | raw ÷ 10 V, `0xFFFF` unset | Identified |
| `0x151D` | Equalization voltage | raw ÷ 10 V, `0xFFFF` unset | Identified |
| `0x151E` | Equalization duration | raw minutes, `0xFFFF` unset | Identified |
| `0x151F` | Maximum charging current | raw ÷10 A; 600 probably means 60 A | Probable |
| `0x1520` | Unknown | raw | Unknown |
| `0x1521` | Unknown | raw | Unknown |

The charger reports its own SOC; it is not equivalent to the JK BMS SOC.

## Captured reads

Main read request:
```text
01 03 15 00 00 22 C1 DF
```
The response uses a `0x44` (68-byte) data payload holding 34 words.

History uses starting address `0x1600`, count `0x002B` (43). The first 12 slots `0x1600–0x160B` represent monthly production; the next 31 `0x160C–0x162A` daily production. Values were interpreted as raw ÷10 kWh. In the working dashboard the first slot represents current period, and the chart places newest at the **left**. Exact calendar rollover semantics on other firmware builds are not established.

## Complete load mode map (holding register 0x1513 / decimal 5395)

All five values below were verified against the ECO-WORTHY app in the original installation. Load mode is distinct from **actual load output state**, reported separately in `0x1514`.

| ECO-WORTHY app mode | Hex | Decimal |
| --- | --- | ---: |
| On | `0x13FC` | 5116 |
| Off | `0x13FD` | 5117 |
| Exclusive Light Control | `0x13EC` | 5100 |
| Shut down load in 1h | `0x13ED` | 5101 |
| Shut down load in 2h | `0x13EE` | 5102 |

The ESPHome `MPPT Load Mode` select writes a single register using Modbus FC16 (function `0x10`) and reads back register `0x1513` to synchronize the selected mode. The independent `MPPT Load Output` switch **reads actual state from `0x1514`** and retains its explicit On/Off behavior. In light control and timed modes, the physical output can change autonomously; `0x1514` is authoritative. The mode names/values are verified for this charger; behavior and supported modes on the 40A remain untested.

## Captured load ON write (OEM app)

```text
Request:  01 10 15 13 00 01 02 13 FC ED 43
Response: 01 10 15 13 00 01 F4 00
```

- `01`: slave address
- `10`: function code **0x10** (= FC16)
- `15 13`: start `0x1513`
- `00 01`: one register
- `02`: two payload bytes
- `13 FC`: ON mode
- `ED 43`: CRC low/high

The following read showed `0x1513=0x13FC` and `0x1514=1`. The OFF value seen in reads is `0x13FD`; **the OFF request CRC was computed from the command pattern rather than independently captured**. The ESPHome code generates Modbus CRC dynamically. A separate battery write is not involved in load switching.

## Battery Save: all ten contiguous registers

The OEM app sent FC16/FC10 with starting address `0x1515`, count `0x000A` (decimal 10) and 20 payload bytes. The exact ordered fields are:

| Word | Register | Meaning |
|---:|---|---|
| 1 | `0x1515` | Battery chemistry |
| 2 | `0x1516` | System voltage |
| 3 | `0x1517` | Overdischarge protection |
| 4 | `0x1518` | Overcharge protection |
| 5 | `0x1519` | Overdischarge recovery |
| 6 | `0x151A` | Overcharge recovery |
| 7 | `0x151B` | Constant charging voltage |
| 8 | `0x151C` | Float voltage |
| 9 | `0x151D` | Equalization voltage |
| 10 | `0x151E` | Equalization duration |

Example observed settings: LiFePO4 code 4, 48 V code 4, 44.0 V protection, 58.4 V overcharge protection, 48.0 V low-voltage recovery, 56.0 V high-voltage recovery, 57.2 V constant voltage; `0xFFFF` for unused float/equalization settings. **These are observations, not recommended setpoints for another battery.** Editing them can affect safety-critical charging behavior.

## Temperature selection and double conversion bug

The temperature unit register is `0x1512` with 0=Celsius, 1=Fahrenheit. The working ESPHome configuration waits for an idle Modbus bus, sends a single-register FC16 write, checks the acknowledgement and follows the next register readback. The first attempt failed because a pending poll could cause the selection to be dropped; the latest configuration waits for the bus to become available.

The Home Assistant dashboard **does not** compute `C × 9/5 + 32` on these readings. A previous double conversion incorrectly turned a true ~60 °F into ~140 °F. Instead display the reported numbers and label matching the controller's **readback** selection.

## Fault flags / unknowns

- `0x150C` bit `0x0001`: battery overvoltage; `0x0080`: controller-low-battery condition, based on observed events. Remaining fault bits not mapped.
- **Charging enable/disable** remains **unknown**. `0x1513` is *load* switching, not charging control.
- `0x150E, 0x150F, 0x1510, 0x1511, 0x1520, 0x1521` remain unidentified. **Do not write unknown addresses.**
- Avoid using the OEM BT-02 and ESPHome as simultaneous RS485 masters.
- Verify every write against acknowledgement **and subsequent readback**, not only by assuming a UI value changed.

See the [full historical decoding notes](RESEARCH_NOTES.md) for raw data snapshots and earlier hypotheses; these curated findings supersede conflicting older assumptions.
