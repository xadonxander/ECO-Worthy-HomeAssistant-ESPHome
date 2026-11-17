# Hardware compatibility and porting

This repository documents a working **ECO-WORTHY MPPT 60 A controller** integration. The Modbus protocol is not inherently limited to ESP32: any suitable host with a correctly configured RS485 Modbus RTU interface could implement the same requests.

## Status matrix

| Controller | Status | Explanation |
| --- | --- | --- |
| ECO-WORTHY **60A MPPT** | Tested in the original installation | The original combined installation and dashboard were developed around captured traffic from this model; the extracted modular package still needs post-refactor test |
| ECO-WORTHY **40A MPPT** | **Not tested** | A plausible candidate, but identical Modbus registers, baud rate, scaling and write behavior have **not** been established |
| Other ECO-WORTHY controllers | Not tested | Verify protocol independently before issuing writes |

| Host/interface | Status | Notes |
| --- | --- | --- |
| ESP32-S3 + RS485 transceiver + Wi-Fi | Existing implementation | This is the exact host assumed by `esphome/ecoworthy-mppt.yaml` |
| **RP2350 + SPI Ethernet + RS485 transceiver** | **Technically feasible; port not implemented/tested** | ESPHome supports RP2350 and SPI Ethernet; firmware still needs a separate board configuration, compatible UART handling, and non-ESP32 Modbus logic changes |
| Raspberry Pi computer + serial/USB RS485 | Possible alternate implementation, not provided | A Linux host needs different software (e.g. Home Assistant Modbus integration, a Python service, or a Modbus gateway), not ESPHome microcontroller firmware |

### An RP2350 can replace the ESP32 as the MPPT interface host

An RP2350/Pico 2-class board is powerful enough to handle the modest Modbus traffic. Use:

- An RP2350 board with accessible UART/SPI pins
- A **3.3 V-compatible RS485 transceiver** with driver enable/receive enable control, physically connected to the MPPT's RS485 port
- An **SPI Ethernet module** such as W5500, with its required SPI/control pins and wired LAN connection (the module provides network access, *not* RS485)
- An ESPHome firmware configuration specifically written and validated for the RP2350 board
- Home Assistant on the LAN using its ESPHome integration

The **MPPT-only ESPHome package is now host-independent at the YAML organization level**, but its UART direction/timing and lambda code still require compile and hardware validation for RP2350. The [ESP32-S3 example](../examples/esp32-s3.yaml) specifies `esp32:` and Wi-Fi and is **not suitable for RP2350 unchanged**. There is no JK BMS/BLE dependency in the published package.

For the MPPT, the original installation uses Modbus RTU **115200 baud, 8N1, slave ID 1**, with ESP32 GPIO16 RX, GPIO17 TX, and GPIO18 direction. **Those pins are not the charger connector pinout and should not be copied to an RP2350.** Check the actual RS485 wiring and transceiver.

## How to validate the 40A safely

1. Identify the exact 40A model, firmware and communication hardware.
2. Use the factory app/dongle or read-only passive capture to establish electrical levels, baud rate and Modbus slave ID.
3. With only one active bus master, try a **read-only** FC03 request for the known 60A blocks and compare returned values with real measured voltage, current and temperature.
4. Check lengths, scaling, register identities, fault behavior, and energy-history rollover; avoid inferring undocumented registers from a single response.
5. **Do not send battery-setting, load or temperature-setting writes** until their addresses, values and expected response are independently established for the 40A.

## Implementation strategy

Maintain the MPPT core as [one standalone package](../esphome/ecoworthy-mppt-package.yaml). A future RP2350-specific *host* YAML can import it after adapting Ethernet, UART, and RS485 control, with compile/hardware validation. The existing ESP32-S3 host example is separate. Avoid labeling the RP2350 configuration as supported until it has compiled and completed end-to-end hardware testing.

Useful references:
- [ESPHome RP2040/RP2350 platform](https://esphome.io/components/rp2040/)
- [ESPHome Ethernet component](https://esphome.io/components/ethernet/)
- [ESPHome packages](https://esphome.io/components/packages/)
- [Our observed register map](MODBUS.md)
