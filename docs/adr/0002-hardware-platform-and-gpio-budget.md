# ADR-0002: Hardware Platform, Power Model, and PCF8575 GPIO Budget

- Status: Accepted
- Date: 2026-11-26
- Scope: Hardware
- Owners: Bojaqu Hardware & Firmware Engineering

## Context
The smart door-lock integrates an `ESP32-S3` module with 16 MB flash, OV5640 DVP camera, MFRC522 SPI RFID reader, ES8311 + NS4150B mono audio codec, a physical lock actuator, and door/visitor inputs (magnetic reed sensor, doorbell call button, indoor exit button, enclosure tamper switch).

Two critical hardware constraints shape the platform:
1. **Memory & Power:** OV5640 frame buffering, concurrent TLS/MQTTS + NimBLE + WebRTC SRTP stacks, and full-duplex audio require large RAM buffers and instant responsiveness for remote unlock and live intercom calls.
2. **GPIO Scarcity on ESP32-S3-WROOM-1-N16R8:** Using 8 MB Octal PSRAM reserves `GPIO33`–`GPIO37` internally (in addition to `GPIO26`–`GPIO32` for internal flash). Preserving `UART0` (`GPIO43`/`GPIO44`), native USB-Serial/JTAG (`GPIO19`/`GPIO20`), and strapping pins (`GPIO0`, `GPIO3`, `GPIO45`, `GPIO46`) leaves **25 clean GPIOs**. Direct wiring of OV5640 DVP (12), shared I2C (2), MFRC522 SPI (4), ES8311 I2S + PA_EN (6), and lock/reed/buttons/LEDs (6+) exceeds 30 pins.

## Decision
1. **SoC Module & Power Architecture:** Standardize on **ESP32-S3-WROOM-1-N16R8** (16 MB Quad Flash + 8 MB Octal PSRAM) powered continuously from **12 V DC mains adapter** (with isolated/regulated 5 V and 3.3 V rails, plus separate bulk capacitance and flyback protection for the 12 V actuator rail). Wi-Fi stays always-on (no deep-sleep wake latency).
2. **Lock Actuator & Door Sensing:** Use a **12 V fail-secure electric strike / solenoid** driven via a low-side N-channel MOSFET (or opto-isolated relay) on a **dedicated boot-safe native ESP32-S3 GPIO (`GPIO21`)** with an external 10 kΩ pull-down resistor to GND. Pair with a magnetic reed door-closed sensor, outdoor doorbell button, indoor exit button, and enclosure tamper switch.
3. **I2C GPIO Expansion via PCF8575:** Add a **PCF8575** 16-bit I2C/SMBus I/O expander with interrupt (`0x20`) on the shared `I2C0` bus (`GPIO1` SDA, `GPIO7` SCL, shared with OV5640 SCCB `0x3C` and ES8311 `0x18`), with its open-drain `INT` pin routed to `GPIO38`.
   - Offload slow control/status I/O to PCF8575 Port 0 (`P00`–`P07`: inverted/active-low `AUDIO_PA_EN_N`, `CAM_PWDN`, `CAM_RST_N`, `RFID_RST_N`, RGB status LEDs, fill LED) and inputs to Port 1 (`P10`–`P17`: reed sensor, doorbell button, exit button, tamper switch, optional RFID IRQ).
   - Never route the lock actuator through the PCF8575, because PCF8575 quasi-bidirectional pins boot weakly HIGH (`0xFFFF`) and could fail to release if the I2C bus stalls.
4. **Preserve Existing MFRC522 Dev Wiring:** Retain `SPI2_HOST` pins `GPIO2` (MISO), `GPIO4` (MOSI), `GPIO5` (SCLK), and `GPIO6` (CS) to remain compatible with the existing bench setup and `diagram.json` Wokwi simulation.

## Alternatives considered
- **ESP32-S3-WROOM-1-N16 (no PSRAM):** Rejected because OV5640 VGA/SVGA frame buffers plus WebRTC + Wi-Fi + NimBLE + TLS exhaust internal SRAM (~512 KB).
- **Battery-powered deep-sleep architecture:** Rejected because continuous Wi-Fi availability is required for low-latency remote unlock and on-demand live camera/audio intercom from Android.
- **Direct ESP32-S3 GPIOs only (repurposing USB `GPIO19`/`GPIO20` and strapping pins):** Rejected because it eliminates native USB-Serial/JTAG debugging, risks boot glitches on strapping pins, and leaves zero spare GPIOs for resets, LEDs, or tamper sensing.
- **TCA9554 / PCA9555 I2C expander:** Evaluated (official `espressif/esp_io_expander_tca95xx_16bit` exists), but **PCF8575** was selected by hardware availability/BOM preference; because no official `espressif/*` PCF8575 driver exists, `bjq_pcf8575` is requested in `bjq-esp-component`.
- **12 V fail-safe magnetic lock / motorized deadbolt:** Rejected for v1; fail-secure electric strike stays locked during power loss (with mechanical key/indoor egress override) and requires only a single boot-safe GPIO.

## Consequences
- **Benefits:** All 25 clean ESP32-S3 GPIOs are utilized with zero conflicts; Octal PSRAM (`GPIO33`–`37`), USB-JTAG (`GPIO19`–`20`), and UART0 (`GPIO43`–`44`) remain untouched; lock actuator is fail-secure across reset, brownout, and I2C faults; PCF8575 power-on `0xFFFF` state keeps camera powered down, LEDs off, and audio PA muted (with active-low PA stage).
- **Trade-offs / Constraints:** Requires `bjq_pcf8575` driver in `bjq-esp-component`; shared `I2C0` bus must be mutex-protected across `esp32-camera` SCCB, `esp_codec_dev` ES8311, and `bjq_pcf8575`; PCF8575 quasi-bidirectional outputs cannot source high current directly (active-low sinking or transistor/MOSFET buffering required for `AUDIO_PA_EN_N` and LEDs).
- **Follow-up work:** Build PCB/schematic Rev A honoring the active-low PA stage, 10 kΩ pull-down on `GPIO21`, 4.7 kΩ I2C pull-ups, and solenoid flyback diode; deliver `REQ-001` (`bjq_pcf8575`).

## References
- [Hardware Overview](../02-hardware/hardware-overview.md)
- [Pin Mapping](../02-hardware/pin-mapping.md)
- [Interface Specification](../02-hardware/interface-specification.md)
- [Power Tree](../02-hardware/power-tree.md)
- [REQ-001: bjq_pcf8575](../03-firmware/component-requests/REQ-001-bjq-pcf8575.md) ([bjq-esp-component#6](https://github.com/blankryan/bjq-esp-component/issues/6))
