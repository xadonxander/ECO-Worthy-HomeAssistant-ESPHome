# Installation and configuration

This guide accompanies [the README](../README.md), [the ESPHome configuration](../esphome/ecoworthy-mppt.yaml), and the [Home Assistant dashboard](../homeassistant/dashboard.yaml). It describes **the specific active-master ESP32-S3 installation**; other controllers and firmware may differ.

## Dependencies checklist

| Dependency | Why required |
|---|---|
| ECO-WORTHY MPPT 60 A with observed RS485 port | Modbus RTU device, slave ID 1 in captured traffic |
| ESP32-S3 supported by ESPHome | Runs the provided active-master firmware |
| Suitable 3.3 V RS485 transceiver, wiring and power | Connects controller RS485 bus to ESP32 UART; controller connector wiring must be independently verified |
| Wi-Fi and ESPHome builder | Firmware configuration, build and OTA |
| Home Assistant + native ESPHome integration | Publishes controller and JK BMS entities |
| `syssi/esphome-jk-bms` external component | Required by supplied two-JK-BMS BLE configuration; ESPHome fetches it from Git |
| Two JK BMS BLE devices with their MACs | **Needed only if retaining the included JK sections** |
| HACS or manual frontend resource installation | Install card-mod, ApexCharts Card, Tabbed Card |
| card-mod | Settings layout, style overrides, `custom:mod-card` |
| ApexCharts Card | `custom:apexcharts-card` historical energy plots |
| Tabbed Card | `custom:tabbed-card` Month/Year selector |

Check that your ESPHome version can build the installed JK BMS external component. This configuration has not been independently compiled in a clean GitHub CI build.

## Wiring and bus ownership

Configured ESP32-S3 GPIOs: RX=GPIO16, TX=GPIO17, DE/RE direction=GPIO18. Serial protocol observed: **115200 baud, 8N1**, Modbus RTU **slave ID 1**.

The GPIO numbers are **ESP32-side assignments**, not a pinout for the charger socket. Verify transceiver power (3.3 V logic), polarity, ground, DE/RE wiring, cable termination, and actual charger connector pinout. Incorrect connections can damage equipment.

**Disconnect/isolate the original ECO-WORTHY BT-02 Modbus master** before allowing the ESPHome firmware to poll or write. The bus is not safe for two uncoordinated masters.

## ESPHome installation (complete configuration)

1. Make a backup of the existing device YAML, secrets and current controller battery settings.
2. Copy `esphome/ecoworthy-mppt.yaml` into your ESPHome configuration directory (`/config/esphome/` on many HAOS installs).
3. Edit `substitutions:` for your actual BLE MAC addresses. The provided `AA:BB:CC:DD:EE:01/02` are examples, **not real BMS addresses**. Adjust GPIOs, protocol version, device name or BLE sections as needed. If no JK BMS devices are present, remove JK BMS-related components, sensors, globals and other references; this is a project-specific configuration, not a JK-free turnkey profile.
4. Add these keys to the **ESPHome** `secrets.yaml` and keep that file private:

   ```yaml
   wifi_ssid2: "YOUR_SSID"
   wifi_password2: "YOUR_PASSWORD"
   JKBMS_API_KEY: "YOUR_BASE64_ESPHOME_API_ENCRYPTION_KEY"
   JKBMS_OTA_PASSWORD: "YOUR_EXISTING_OR_NEW_OTA_PASSWORD"
   fallbackAP_Password: "YOUR_STRONG_FALLBACK_PASSWORD"
   ```

   Generate the API key with ESPHome's supported key generator. Existing installations should preserve the original API/OTA secrets if updating the same device.
5. Set the correct Wi-Fi network. If hostname discovery fails, add your own `wifi.use_address` (local IP/hostname) under `wifi:`; the originating installation IP was removed.
6. Validate in ESPHome, compile, then flash via USB or your already-authorized OTA path. Review the logs for RS485 reads and Modbus CRC/acknowledgement failures before touching any control.
7. Add/discover the device with **Settings → Devices & Services → ESPHome** and verify entities under **Developer Tools → States**. In this installation, entity IDs were prefixed `garage_modbus_bridge_`. HA may assign different IDs: update the dashboard references to match.

## Home Assistant dashboard installation

1. In HACS, install these *Frontend* resources: [card-mod](https://github.com/thomasloven/lovelace-card-mod), [ApexCharts Card](https://github.com/RomRider/apexcharts-card), [Tabbed Card](https://github.com/kinghat/tabbed-card). Manual installs are also possible if you register resource URLs correctly. Refresh the browser after installing resources.
2. Create a new dashboard with URL path **`eco-worthy-mppt-60a-charger`** and open its **Raw configuration editor**.
3. Paste all of `homeassistant/dashboard.yaml`, save, then revisit the dashboard.
4. Check the five views: **Readings**, **Settings**, **Configuration** (subview), **History**, **Diagnostics**.
5. The battery-config gear navigation and cancel link use **`/eco-worthy-mppt-60a-charger/`**. If your dashboard slug differs, replace these paths in the YAML.
6. If HA entities are not found, inspect a few live ESPHome entities and replace `garage_modbus_bridge_` with the actual prefix consistently.
7. The Month/Year charts derive displayed dates from rolling 31-day/12-month slots and deliberately place **newest on the left**. This display mapping was validated by the original user; test controller rollover separately.

### Optional file-backed dashboard

Copy `homeassistant/dashboard.yaml` to `/config/dashboards/ecoworthy-mppt.yaml` and merge into **existing** `configuration.yaml`:

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

Ensure the dashboard key is a supported unique slug and that its filename is relative to HA's config directory. A file-backed dashboard is not automatically refreshed from GitHub; use a separate trusted checkout/sync process and reload the dashboard when needed.

### Optional ESPHome Git package

ESPHome remote packages can load the YAML during ESPHome configuration processing:

```yaml
packages:
  eco_worthy:
    url: https://github.com/xadonxander/ECO-Worthy-HomeAssistant-ESPHome
    ref: main
    files:
      - esphome/ecoworthy-mppt.yaml
    refresh: 1d
substitutions:
  bms1_mac: "YOUR_REAL_BMS1_MAC"
  bms2_mac: "YOUR_REAL_BMS2_MAC"
```

For predictable and secure updates, pin `ref` to a reviewed release tag/commit rather than `main` and verify local overrides. ESPHome package refreshing **does not flash** a device. You must compile and install new firmware deliberately.

## Control operation

- **Load switch:** Writes a single value to holding register `0x1513` with Modbus function `0x10`; verifies status from `0x1514`.
- **Temperature selector:** Requests `0x1512=0` for Celsius or `=1` for Fahrenheit, waits for the UART bus to be free, verifies acknowledgement and reads back the controller setting. The dashboard displays the returned temperature values and label **without a second conversion**.
- **Battery configuration:** The Configuration subview edits draft values locally. **Save** writes ten contiguous holding registers `0x1515–0x151E` in one FC16 transaction. Do not press Save without double-checking battery chemistry/voltage/charge limits.
- **History:** Read-only daily and monthly production registers.

## Updates and rollback

- Commit and review changes on GitHub, back up current working files, validate/compile in ESPHome, then explicitly flash a known-good build.
- Download/update the HA dashboard separately. Confirm the entity IDs and custom frontend cards.
- Keep a prior known-good YAML/firmware backup for immediate rollback; do not automate unreviewed firmware flashes on a live solar charger.
