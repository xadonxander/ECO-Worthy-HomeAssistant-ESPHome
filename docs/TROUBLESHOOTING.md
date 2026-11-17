# Troubleshooting

Read [INSTALLATION.md](INSTALLATION.md) before changing wiring or firmware, and [MODBUS.md](MODBUS.md) for address definitions.

## ESPHome boots but there are no MPPT readings

- Confirm the original BT-02 module is disconnected; avoid two Modbus masters.
- Confirm ESP32 UART pin mapping **RX GPIO16 / TX GPIO17 / DE GPIO18** matches your build. These are firmware defaults, not an official charger connector pinout.
- Check transceiver power/logic levels, RS485 A/B polarity, ground reference and DE/RE timing.
- The observed controller speaks **115200 baud, 8N1, slave ID 1**. Check logs for valid read response framing and CRC.
- If using a changed controller or firmware revision, addresses may differ.

## Temperature selector immediately reverts

The selector must **actually write** `0x1512`; dashboard-only changes do not modify controller state. In the working firmware the `mppt_set_temperature_unit` script waits up to five seconds for Modbus polling/writes to finish, sends **FC16 to a single register**, and uses the controller's readback for the displayed selection.

- Filter ESPHome logs for `mppt_temp`.
- Look for `FC10 requested temperature unit`, acknowledgement and readback verification.
- A missing ACK suggests bus collision, direction/timing issue, failed CRC, incompatible firmware or unsupported write.
- If write ACK succeeded but register `0x1512` still returns the prior value, treat the controller as the source of truth and investigate its supported unit-setting command.

## Fahrenheit approximately 140 °F when ambient is approximately 60 °F

An early dashboard applied `C*9/5+32` to a value already in Fahrenheit. **Do not convert a second time.** In this project, the controller handles the temperature unit; the dashboard displays the controller's numerical reading and a unit label derived from readback. If readings still disagree, inspect the raw values of `0x1508`, `0x1509`, `0x1512`.

## Load switch does not work

- Verify Modbus **function `0x10`** (FC16) with **count 1** writing `0x1513` to `0x13FC` (ON) or `0x13FD` (OFF).
- Read `0x1514` for actual load state. Do not equate `0x1513` mode with proof of load power.
- Check logs for Modbus ACK and whether a polling/battery configuration operation is holding the serial bus.
- The `0x1513` command is **load output**, **not the charging ON/OFF switch**.

## Battery configuration Save does not persist

- The Configuration view stages data in ESPHome's draft entities; **Save** sends the block of **10 registers `0x1515–0x151E`**.
- Verify the actual device values and chemistry before retrying. A failed write must not be represented as a saved value.
- Do not use guessed battery code enums or inferred safety limits.
- Check `mppt_fc10` messages; examine request, ACK and next readback.
- On communication failure, leave the battery settings unchanged until a verified correction is possible.

## HA dashboard says custom card does not exist

Install and register all three frontend resources:

1. **card-mod** — `custom:mod-card` and `card_mod`.
2. **ApexCharts Card** — `custom:apexcharts-card`.
3. **Tabbed Card** — `custom:tabbed-card`.

Install through HACS Frontend or install/register manually, then hard-reload the browser.

## Dashboard entities show Unavailable

- Add the ESPHome device to the native HA ESPHome integration.
- The example dashboard expects entity IDs prefixed `garage_modbus_bridge_`; HA may use a different generated prefix. Confirm via **Developer Tools → States** and replace entity IDs in the dashboard consistently.
- Confirm matching API encryption credentials, network connectivity, OTA/reboot status and ESPHome log errors.

## Modular configuration and separate BMS

This repository publishes **MPPT only**. There are no JK BMS BLE clients or associated dependencies. If your previous combined firmware monitored JK BMS batteries, configure them separately in another ESPHome device or an independently maintained package. The ECO-WORTHY dashboard intentionally uses the **charger-reported** measurements, not JK BMS state.


## Security and upgrades

- Do not commit `secrets.yaml`, API keys, private MAC addresses, internal IPs or service credentials.
- Do not expose RS485/Modbus control externally. Prefer reviewed, pinned Git revisions over automatic use of untested firmware commits.
- Preserve the last working firmware and dashboard YAML before updating. Building/reflashing ESPHome and replacing the dashboard are **separate deployment steps**.
