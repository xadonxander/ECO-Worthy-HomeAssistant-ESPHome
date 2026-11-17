# ECO-WORTHY MPPT 60A — ESPHome + Home Assistant

Community-developed ESPHome firmware and a Home Assistant dashboard for an ECO-WORTHY MPPT 60 A solar charge controller. Based on captured Modbus RTU traffic and iterative testing on one installation; **not an official ECO-WORTHY integration**. Do not assume different models or firmware use the same registers.

## What is included

- Live PV, charging, battery, load, fault, temperature and energy readings
- Load output ON/OFF control with controller readback
- Controller Celsius/Fahrenheit selector with controller-side Modbus write and readback (no dashboard double conversion)
- Battery configuration screen with staged edits, explicit Save and 10-register FC16 write
- Monthly (31 daily records) and yearly (12 monthly records) production graphs; newest entries at the left
- Diagnostics and raw registers
- Two JK BMS BLE instances in the supplied ESPHome firmware (this is **required by the included configuration** unless you remove the JK BMS sections)

**IMPORTANT:** The published ESPHome file is a shareable configuration based on a working installation, not a plug-in binary. You must provide your own BMS Bluetooth MACs and secrets before installing. Device-specific MACs and LAN IPs were deliberately removed.

## Repository files

| Path | Purpose |
| --- | --- |
| [esphome/ecoworthy-mppt.yaml](esphome/ecoworthy-mppt.yaml) | Complete ESP32-S3 ESPHome controller/JK BMS configuration |
| [homeassistant/dashboard.yaml](homeassistant/dashboard.yaml) | Complete five-view Lovelace dashboard |
| [docs/INSTALLATION.md](docs/INSTALLATION.md) | Detailed initial installation, Git updates and validation |
| [docs/MODBUS.md](docs/MODBUS.md) | Confirmed RTU frames, register map and remaining unknowns |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Common control, BLE, UI and UART problems |
| [docs/RESEARCH_NOTES.md](docs/RESEARCH_NOTES.md) | Historical investigation transcript; superseded conclusions may appear |

## Hardware compatibility

| Controller / host | Status | Notes |
| --- | --- | --- |
| ECO-WORTHY MPPT **60A** | **Tested on one installation** | Captured protocol and working ESP32-S3 firmware/dashboard |
| ECO-WORTHY MPPT **40A** | **Untested candidate** | May share the same Modbus protocol; do not assume register addresses, values or writes match until verified |
| ESP32-S3 + RS485 transceiver | **Current supplied firmware** | Wi-Fi + integrated JK BMS Bluetooth monitoring |
| **RP2350 + Ethernet + RS485** | **Feasible, not yet ported/tested** | ESPHome supports RP2350 and SPI Ethernet; must replace ESP32-specific BLE/JK blocks and adapt UART/pins/config |
| Raspberry Pi running Linux + USB/serial RS485 | **Possible alternate architecture, not supported by this YAML** | Requires a different Modbus/HA integration or custom service; ESPHome firmware cannot simply run on a Linux Pi |

**An ESP32 is not inherently required for the MPPT's Modbus protocol.** An RP2350, or another capable Modbus RTU client, can communicate over RS485. However, the exact included YAML **does require ESP32-S3** because it uses ESP32-specific configuration and JK BMS BLE. Do not flash the ESP32 YAML to an RP2350 unchanged. An Ethernet module by itself is not an RS485 interface; an RS485 transceiver is also required. Model compatibility is distinct from host compatibility.

See [hardware compatibility and future RP2350 port notes](docs/COMPATIBILITY.md).

## Dependencies

### Hardware (required for the supplied ESP32-S3 firmware)
- **ECO-WORTHY MPPT 60 A** controller with the observed RS485 Modbus RTU interface (40A model not tested)
- **ESP32-S3** board supported by ESPHome (example target: esp32-s3-devkitc-1)
- **3.3 V RS485 transceiver** wired to the ESP32 UART and driver-enable output
- Compatible power supply, RS485 wiring and Wi-Fi
- **Exclusive control of the RS485 master bus**. Unplug/isolate the original BT-02 dongle before enabling this ESPHome master; two unsynchronized masters can collide
- For the unmodified YAML: **two compatible JK BMS units reachable by Bluetooth**, with correct MAC addresses and protocol setting. Remove/modify JK BMS blocks if using a different battery setup

Example ESP32 pin assignment in the supplied file: **RX GPIO16**, **TX GPIO17**, **RS485 direction GPIO18**; **115200 baud, 8N1, Modbus slave ID 1**. These are GPIO-to-transceiver assignments, **not** a verified controller connector pinout. Verify A/B, ground/reference, transceiver voltage, DE/RE and your particular hardware before energizing.

### Software
- **ESPHome** with ESP32-S3/Arduino framework support and Git access during build
- ESPHome external component **[syssi/esphome-jk-bms](https://github.com/syssi/esphome-jk-bms)**, automatically downloaded by the supplied YAML; use the BLE **jk_bms_ble** component
- **Home Assistant** with the built-in **[ESPHome integration](https://www.home-assistant.io/integrations/esphome/)**
- Home Assistant frontend custom cards (install via [HACS](https://www.hacs.xyz/) or manually):
  1. **[card-mod](https://github.com/thomasloven/lovelace-card-mod)** (provides card_mod styling and custom:mod-card)
  2. **[ApexCharts Card](https://github.com/RomRider/apexcharts-card)** (custom:apexcharts-card)
  3. **[Tabbed Card](https://github.com/kinghat/tabbed-card)** (custom:tabbed-card)
- HACS is the recommended **installation manager**, not a runtime firmware dependency. There is **no custom Home Assistant backend integration** required beyond ESPHome

No fixed ESPHome / HA version has been certified across installations. Use a version supported by the referenced JK BMS component and test after upgrades. The dashboard uses modern HA YAML actions, heading/badge cards, and template rendering.

## Quick start

1. **Back up** existing ESPHome firmware/config, Home Assistant dashboard, battery settings and any API/OTA credentials.
2. Copy **esphome/ecoworthy-mppt.yaml** to your ESPHome configuration directory, usually **/config/esphome/ecoworthy-mppt.yaml**. Replace the two example BLE MAC addresses under substitutions. Change RS485 pins and JK protocol as appropriate. The example does **not** contain the original user's MACs or IP.
3. Add these keys to your existing local **secrets.yaml** (use real values; **never commit this file**):

   ~~~yaml
   wifi_ssid2: "YOUR_WIFI_SSID"
   wifi_password2: "YOUR_WIFI_PASSWORD"
   JKBMS_API_KEY: "YOUR_EXISTING_ESPHOME_API_ENCRYPTION_KEY"
   JKBMS_OTA_PASSWORD: "YOUR_OTA_PASSWORD"
   fallbackAP_Password: "YOUR_FALLBACK_AP_PASSWORD"
   ~~~

   An API encryption key must be a valid ESPHome-generated key. For an existing device, preserve its current API/OTA credentials to avoid access problems.
4. Remove the original **BT-02** from the charger RS485 bus before active ESPHome polling or writes. Check wiring and the controller protocol first.
5. **Validate, compile, flash and review ESPHome logs.** Check read-only measurements before using Load, temperature unit or Battery Save controls. This configuration was not independently compiled in this repository's publishing workflow.
6. Add the ESPHome device through Home Assistant's **ESPHome integration** and verify your generated entity IDs in **Developer Tools → States**.
7. Install **card-mod**, **ApexCharts Card**, and **Tabbed Card** in HACS (Frontend). Refresh the browser if cards are not registered.
8. Create a Home Assistant dashboard with URL path **eco-worthy-mppt-60a-charger**, open **Edit dashboard → Raw configuration editor**, then paste **homeassistant/dashboard.yaml**. The dashboard is written for entity IDs such as **sensor.garage_modbus_bridge_mppt_pv_voltage** and **select.garage_modbus_bridge_mppt_draft_temperature_unit**; if your device/entity names differ, search-replace the prefix throughout the YAML.
9. Test the **Readings**, **Settings**, **Configuration**, **History** and **Diagnostics** views. The Battery Config gear/back links assume **/eco-worthy-mppt-60a-charger/**.

For a file-backed YAML dashboard, see [docs/INSTALLATION.md](docs/INSTALLATION.md).

## Optional Git-based ESPHome updates

ESPHome supports [remote Git packages](https://esphome.io/components/packages/). For example, a local YAML can import the whole configuration:

~~~yaml
packages:
  ecoworthy:
    url: https://github.com/xadonxander/ECO-Worthy-HomeAssistant-ESPHome
    ref: main
    files:
      - esphome/ecoworthy-mppt.yaml
    refresh: 1d

substitutions:
  bms1_mac: "YOUR_FIRST_BMS_MAC"
  bms2_mac: "YOUR_SECOND_BMS_MAC"
~~~

Local secrets still belong in the ESPHome configuration directory. Review the full package and avoid duplicate top-level configuration entries if adding overrides. **Git refresh does not automatically compile or flash new firmware.** For a solar controller that accepts writes, prefer pinned tags/commits over following main unattended.

Home Assistant does **not** automatically import a dashboard from a GitHub URL; copy or synchronize the file into your HA config and reload it after updating.

## Safety and limitations

- **Battery Save** writes **10 consecutive holding registers** from **0x1515 to 0x151E**. Wrong chemistry, voltage or charging limits may damage equipment. Confirm values against the battery/BMS and controller documentation.
- **Load ON/OFF** writes only **one register**, **0x1513**, with **0x13FC** (ON) or **0x13FD** (OFF); state comes from **0x1514**. Modbus **function 0x10 (FC16)** is *not* “10 registers.”
- **Temperature units** use **0x1512** (0 = Celsius, 1 = Fahrenheit). The dashboard does not independently calculate 9/5 + 32.
- Unknown registers must remain read-only. Charging enable/disable is **not** the same as load output switching and is not mapped here.
- Production history is read from 12 monthly and 31 daily slots; newest is intentionally plotted on the **left** using the observed reverse-order display convention. Confirm rollover behavior on other controller versions.
- Do not expose ESPHome or Modbus controls to untrusted networks. Verify every write with ACK and fresh register readback.

## Documentation

Read [installation and dependencies](docs/INSTALLATION.md), [Modbus findings](docs/MODBUS.md) and [troubleshooting](docs/TROUBLESHOOTING.md) before editing write-related registers. See the archived [research notes](docs/RESEARCH_NOTES.md) for captured observations and provisional interpretations.

## License

No open-source license has been selected yet. Add a license only after the repository owner chooses one.
