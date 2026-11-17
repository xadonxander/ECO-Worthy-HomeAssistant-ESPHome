# ECO-WORTHY MPPT — ESPHome package + Home Assistant dashboard

A community-maintained Modbus RTU integration for an ECO-WORTHY MPPT solar charge controller, derived from captured traffic and iterative testing on the **60A** model. **Not an official ECO-WORTHY project.** The **40A model has not been tested** and must not be assumed register-compatible.

## What this repository provides

- A reusable **MPPT-only ESPHome package** — controller polling, register decoding, PV/battery/load/temperature measurements, faults, energy history, and readback-verified writes.
- A minimal **ESP32-S3 example** that imports the package over Git.
- A complete five-view Home Assistant dashboard (Readings, Settings, Configuration, History, Diagnostics).
- Installation, compatibility, Modbus register notes and troubleshooting documentation.

**No JK BMS, BLE, battery-bank aggregation, or third-party BMS component is included or required.** Battery readings shown here are the *MPPT controller's* own measurements; a separately installed JK BMS, if any, remains a separate Home Assistant device.

## Files

| File | Use |
| --- | --- |
| [esphome/ecoworthy-mppt-package.yaml](esphome/ecoworthy-mppt-package.yaml) | Reusable MPPT Modbus package; imported into a host device config |
| [examples/esp32-s3.yaml](examples/esp32-s3.yaml) | Device-specific ESP32-S3 configuration importing the package |
| [homeassistant/dashboard.yaml](homeassistant/dashboard.yaml) | Home Assistant Lovelace dashboard |
| [docs/INSTALLATION.md](docs/INSTALLATION.md) | Setup, dependencies, ESPHome packages, HACS and dashboard |
| [docs/MODBUS.md](docs/MODBUS.md) | Register map, captures, read/write behavior, confidence levels |
| [docs/COMPATIBILITY.md](docs/COMPATIBILITY.md) | Tested 60A, untested 40A, future RP2350 + Ethernet |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Serial, load, temperature, battery Save, chart problems |
| [docs/RESEARCH_NOTES.md](docs/RESEARCH_NOTES.md) | Historical reverse-engineering notes (superseded claims may appear) |

## Dependencies

**Required for the ESP32-S3 example:** ECO-WORTHY 60A controller with observed RS485 RTU interface; ESP32-S3 board supported by ESPHome; a **3.3V-compatible RS485 transceiver**; Wi-Fi; ESPHome builder; correct power/wiring. The example uses **GPIO16 RX**, **GPIO17 TX**, **GPIO18 RS485 direction**, **115200 baud 8N1**, slave **1**. These GPIOs are *not* the charger's connector pinout. Disconnect the original **BT-02** before using this active Modbus master.

**Required software:** ESPHome compatible with your ESP32-S3 host; Home Assistant with its built-in [ESPHome integration](https://www.home-assistant.io/integrations/esphome/). The dashboard also needs these **frontend cards**, installed via [HACS](https://www.hacs.xyz/) or manually: [card-mod](https://github.com/thomasloven/lovelace-card-mod), [ApexCharts Card](https://github.com/RomRider/apexcharts-card), and [Tabbed Card](https://github.com/kinghat/tabbed-card). HACS is an optional installation manager, not a runtime requirement. **There is no JK BMS dependency.**

## Quick setup

1. Back up the original firmware/YAML, any controller battery settings and Home Assistant dashboard.
2. Copy [examples/esp32-s3.yaml](examples/esp32-s3.yaml) to your ESPHome config (for example `/config/esphome/eco-worthy-mppt.yaml`). It pulls the MPPT package from GitHub.
3. Adjust device name, GPIOs and Wi-Fi as needed. Put secrets in the **local ESPHome `secrets.yaml`**, never in Git:

   ```yaml
   wifi_ssid: "YOUR_WIFI"
   wifi_password: "YOUR_PASSWORD"
   mppt_api_key: "YOUR_VALID_ESPHOME_ENCRYPTION_KEY"
   mppt_ota_password: "YOUR_OTA_PASSWORD"
   mppt_fallback_password: "YOUR_FALLBACK_AP_PASSWORD"
   ```

   Preserve existing API and OTA credentials when updating a running device.
4. Unplug/isolate the OEM BT-02 so there is **only one RS485 master**.
5. Validate, compile and flash with ESPHome. **The modular package has been parsed and checked for missing ID references, but has not yet been independently compiled or hardware-tested following extraction.** Check read-only readings before testing any writes.
6. Add the device in Home Assistant's native ESPHome integration.
7. Install the three frontend cards, create dashboard URL path `eco-worthy-mppt-60a-charger`, and paste [homeassistant/dashboard.yaml](homeassistant/dashboard.yaml) in the Raw configuration editor. Update entity ID prefixes if yours differ.

See [complete installation instructions](docs/INSTALLATION.md).

## Import as a package

The ESP32-S3 example already uses:

```yaml
packages:
  ecoworthy:
    url: https://github.com/xadonxander/ECO-Worthy-HomeAssistant-ESPHome
    ref: main
    files:
      - esphome/ecoworthy-mppt-package.yaml
    refresh: 1d
```

For production use, pin `ref` to a reviewed release/commit rather than updating automatically from `main`. A package refresh does **not** automatically compile/flash firmware.

The package contains **MPPT entities and Modbus logic only**. Your host YAML owns ESPHome board selection, API, OTA, logger, Wi-Fi or Ethernet, and other unrelated integrations. An RP2350 with Ethernet and an RS485 transceiver is a **possible future host**, but its firmware has **not** been ported or verified. The current example is ESP32-S3 only.

## Confirmed behavior and cautions

- **Load output toggle and load mode dropdown:** both use Modbus function `0x10` (FC16) to write one holding register, `0x1513`. Selectable modes: On (`0x13FC`), Off (`0x13FD`), Exclusive Light Control (`0x13EC`), Shut down load in 1h (`0x13ED`), Shut down load in 2h (`0x13EE`). The dropdown tracks `0x1513`; the toggle tracks actual load state from `0x1514`.
- **Temperature selector** writes one register, `0x1512` (0 Celsius, 1 Fahrenheit), then checks the charger's response. The dashboard avoids double conversion.
- **Battery Configuration Save** writes **10 registers** from `0x1515` through `0x151E`; ensure the values match your battery requirements before saving.
- **History:** 31 daily and 12 monthly slot values, newest plotted **left**.
- Never write to unknown registers, expose write functions to untrusted networks, or connect two unsynchronized RS485 masters.

## Compatibility

| Model/host | Status |
| --- | --- |
| ECO-WORTHY 60A MPPT | Tested in original setup; this standalone package still needs independent post-refactor hardware validation |
| ECO-WORTHY 40A MPPT | **Not tested** |
| ESP32-S3 + Wi-Fi + RS485 | Included host example |
| RP2350 + Ethernet + RS485 | Feasible architecture, **not yet ported/tested** |

The published MPPT package has no Bluetooth or JK BMS code. For a separate JK BMS installation, configure it independently in ESPHome or Home Assistant; this project does not install it.

## License

No license has been selected by the repository owner yet.
