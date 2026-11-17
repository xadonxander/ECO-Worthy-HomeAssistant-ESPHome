# Installation — ECO-WORTHY MPPT-only package

This release intentionally contains **no JK BMS support, BLE clients, or battery-bank calculations**. The controller's own battery voltage, charging and SOC measurements remain available.

## Required dependencies

| Item | Requirement |
| --- | --- |
| Charger | ECO-WORTHY MPPT **60A** with tested RS485 Modbus RTU behavior; 40A untested |
| Controller host | **ESP32-S3** for the supplied example; other ESPHome-compatible hosts may work with a separate port |
| RS485 hardware | 3.3V logic-compatible UART-to-RS485 transceiver, correctly wired to the charger |
| Network | Wi-Fi with ESP32-S3 example; an Ethernet-capable host is a possible alternative |
| Firmware environment | ESPHome (ESP32-S3 Arduino framework in the example); Git accessible at configuration/build time |
| HA integration | Built-in Home Assistant ESPHome integration |
| Dashboard frontend | **card-mod**, **ApexCharts Card**, **Tabbed Card**; install via HACS or manually |

**Not required:** `syssi/esphome-jk-bms`, any JK BMS device or MAC address, ESP32 BLE tracker/client, external BMS library, HACS backend integration.

## Wiring and bus ownership

The example uses ESP32 GPIO16 RX, GPIO17 TX, GPIO18 RS485 driver-enable direction, with RTU 115200 baud, 8N1 and device ID 1. Verify the exact charger socket pinout, A/B, reference ground and transceiver supply separately. These GPIO assignments are **not** a verified charger socket wiring diagram.

**Unplug/isolate the OEM BT-02** before flashing the new active-master firmware or allowing polling/writes. Two simultaneous masters may collide.

## Install ESPHome

1. Back up your currently working ESPHome YAML, secrets, credentials and current controller battery settings.
2. Copy [`examples/esp32-s3.yaml`](../examples/esp32-s3.yaml) to your ESPHome directory as `eco-worthy-mppt.yaml` (commonly `/config/esphome/`).
3. The example imports [`esphome/ecoworthy-mppt-package.yaml`](../esphome/ecoworthy-mppt-package.yaml) with `packages:`. The package has **only MPPT components**. The host file defines the ESP32 board, Wi-Fi, API, OTA, logger and fallback AP.
4. Update GPIO substitutions and the hostname as appropriate. Put these exact keys into the local **ESPHome `secrets.yaml`**:

   ```yaml
   wifi_ssid: "MY_WIFI"
   wifi_password: "MY_PASSWORD"
   mppt_api_key: "VALID_ESPHOME_API_ENCRYPTION_KEY"
   mppt_ota_password: "MY_OTA_PASSWORD"
   mppt_fallback_password: "MY_STRONG_FALLBACK_PASSWORD"
   ```

   Generate a proper ESPHome API encryption key with ESPHome's UI/CLI, not by copying the placeholder. Preserve the previous device API/OTA credentials if reusing a provisioned device. Do **not** commit `secrets.yaml`.
5. Disconnect the BT-02, verify RS485 hardware and choose the correct serial port. Validate your complete ESPHome configuration and compile **before flashing**. This refactor has received YAML structural and ID-reference checks, but **has not been compiled/hardware-tested in the released modular form**.
6. Flash the ESP32-S3 and inspect UART logs for valid FC03 reads/CRC before trying any writes. In Home Assistant, add the device via **Settings → Devices & Services → ESPHome**.
7. Verify sensor names and state changes. The HA dashboard was authored against `garage_modbus_bridge_` entities; your HA may generate a different prefix depending on the host's name/registry. Adjust dashboard entity IDs if necessary.

## Install Home Assistant dashboard

1. Install [card-mod](https://github.com/thomasloven/lovelace-card-mod), [ApexCharts Card](https://github.com/RomRider/apexcharts-card) and [Tabbed Card](https://github.com/kinghat/tabbed-card) through HACS Frontend or manually. Register frontend resources and hard-refresh HA.
2. Create a new dashboard with URL path `eco-worthy-mppt-60a-charger`.
3. Paste [`homeassistant/dashboard.yaml`](../homeassistant/dashboard.yaml) using **Edit dashboard → Raw configuration editor**.
4. Check all five views: Readings, Settings, Configuration, History and Diagnostics. The gear/Cancel links assume `/eco-worthy-mppt-60a-charger/`. Change these paths if you choose another slug.
5. Verify load and temperature controls respond to the controller readback before changing battery limits. The history chart intentionally plots newest values at the left.

### Optional file-backed dashboard

For `/config/dashboards/ecoworthy-mppt.yaml`, merge into HA's **existing** `configuration.yaml`:

```yaml
lovelace:
  dashboards:
    eco-worthy-mppt-60a-charger:
      mode: yaml
      title: ECO-WORTHY MPPT
      icon: mdi:solar-power
      show_in_sidebar: true
      filename: dashboards/ecoworthy-mppt.yaml
```

Home Assistant does **not** automatically sync a GitHub dashboard. Copy/sync the file yourself and reload the dashboard when updated.

## Remote package and modularity

ESPHome `packages` can combine unrelated integrations from different files. Use the MPPT package from this repo and, if desired, a **separate** JK BMS package/configuration maintained independently. Do not add duplicate device boot settings (`esphome:`, `esp32:`, `api:`, `wifi:`) inside the MPPT package.

```yaml
packages:
  ecoworthy:
    url: https://github.com/xadonxander/ECO-Worthy-HomeAssistant-ESPHome
    ref: main
    files:
      - esphome/ecoworthy-mppt-package.yaml
    refresh: 1d
```

Pin `ref` to a tested tag/commit when deploying to a live charger. Pulling an updated package does **not** flash a new build: compile and install explicitly. The MPPT package is not a standalone firmware; you must provide a supported host/device configuration.

## Protocol notes

The package reads live registers `0x1500–0x1521` and history `0x1600–0x162A`, writes `0x1513` for load ON/OFF and `0x1512` for °C/°F, and writes `0x1515–0x151E` **only on explicit Battery Save**. See [MODBUS.md](MODBUS.md) before changing any write logic. A write success should be checked by RTU ACK **and fresh readback**.

## Rollback / update safety

Keep a known-good copy of the original combined ESPHome firmware. Do **not** automatically flash a newly refactored package over the existing device until compilation and test reads succeed. Failure to load package/configuration must not trigger attempts to write settings. Test first on the bench where possible.
