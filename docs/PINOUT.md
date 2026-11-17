# ECO-WORTHY MPPT 8-pin communication connector pinout

This pinout was provided by the owner of a working **ECO-WORTHY 60A MPPT** installation. It describes the **numbered contacts on the charger's communication connector**, **not** the ESP32 GPIO header. The ECO-WORTHY 40A model and other hardware revisions have not been verified against this pinout.

| Connector pin(s) | Signal |
| --- | --- |
| **1 and 3** | **RS485 A** |
| **2 and 4** | **RS485 B** |
| **5 and 6** | **GND** |
| **7 and 8** | **+5 V** |

Pins sharing a signal are duplicates. For a single RS485 connection, use one documented A contact, one B contact and the appropriate ground/reference. The +5 V contacts are separate from A/B.

> **Important orientation warning:** Contact numbering is only useful if you know whether you are looking into the charger socket or at the plug, and where pin 1 is located. A front/back view or connector mechanical orientation **has not been supplied/verified**, so this document intentionally does **not** invent a left-to-right pin drawing. Confirm pin 1, contact numbering and actual voltages with the connector identification and a multimeter before applying power or connecting hardware.

## Connecting an ESP32 or RP2350

The charger uses **RS485 differential signals**. Do **not** connect A/B directly to the microcontroller's UART RX/TX pins. Use a suitable **3.3 V-logic-compatible RS485 transceiver**:

```text
ECO-WORTHY connector              RS485 transceiver              Microcontroller
1 or 3: A ----------------------> A
2 or 4: B ----------------------> B
5 or 6: GND --------------------> GND --------------------------> GND/reference
                                  RO ---------------------------> UART RX
                                  DI <--------------------------- UART TX
                                  DE and /RE <------------------ Direction GPIO
7 or 8: +5V ---- optional power source, only after electrical verification
```

The example **ESP32-S3** firmware uses **GPIO16 RX**, **GPIO17 TX** and **GPIO18 direction**. These are only its local MCU-to-transceiver pin assignments; they **do not** correspond to charger connector pins 16, 17 or 18. For another MCU (for example RP2350), select board-appropriate UART and direction pins.

**Power caution:** Connector pins 7/8 are reported as +5 V. Do **not** connect that directly to 3.3 V GPIO pins or the 3.3 V rail of an ESP32/RP2350. Use a regulator or board-rated 5 V input only if the connector's available current, grounding, and electrical behavior are established. It is safe to leave the +5 V pins unused and power the microcontroller/transceiver independently. Avoid backfeeding either side.

**Bus ownership:** The supplied ESPHome firmware actively sends Modbus RTU requests. Disconnect/isolate the original BT-02 dongle before running it as the RS485 master.

**Serial settings captured on the tested 60A:** 115200 baud, 8N1, Modbus slave address 1. See [MODBUS.md](MODBUS.md) for register meanings and write precautions.
