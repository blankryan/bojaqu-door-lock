# Power Tree — Mains/DC Architecture (Rev A)

## Power Distribution Diagram
```mermaid
flowchart LR
    VIN["12V DC Adapter\n(12V >= 2.0A, TVS + Fuse)"] --> SOL["12V Actuator Rail\n(Electric Strike / Solenoid\n+ Flyback Diode)"]
    VIN --> BUCK5["5V Buck Regulator\n(>= 2.0A, e.g., MP1584 / TPS54302)"]
    BUCK5 --> AMP["NS4150B Speaker Amp\n(5V, up to 0.8A peak)"]
    BUCK5 --> BUCK33["3.3V Logic Buck / LDO\n(>= 1.5A Low-Noise)"]
    BUCK33 --> ESP["ESP32-S3-WROOM-1-N16R8\n(3.3V, ~500mA TX peak)"]
    BUCK33 --> RFID["MFRC522 RFID\n(3.3V, ~30mA RF peak)"]
    BUCK33 --> PCF["PCF8575 + Pull-ups\n(3.3V, ~15mA + LEDs)"]
    BUCK33 --> CODEC["ES8311 Analog/Digital\n(3.3V RC-filtered)"]
    BUCK33 --> CAM_LDO["OV5640 Local LDOs\n(2.8V AVDD + 1.5V DVDD\n+ 2.8V/3.3V DOVDD)"]
```

## Current Budget Estimate
| Rail / Load | Typical | Peak | Notes |
|---|---|---|---|
| **12 V Solenoid / Electric Strike** | 0 mA (locked) | 400–800 mA | Energized only during unlock window (5–15 s max) |
| **5 V -> NS4150B (4 Ω / 8 Ω speaker)** | 1 mA (muted) | 600–800 mA | Active only during intercom or voice prompts |
| **3.3 V -> ESP32-S3 (Wi-Fi + BLE + PSRAM)** | 120–180 mA | 500 mA | Wi-Fi TX bursts; add 220 µF + 10 µF + 0.1 µF near VDD |
| **3.3 V -> OV5640 Camera** | < 1 mA (PWDN) | 140 mA | Powered down via `CAM_PWDN` when idle |
| **3.3 V -> MFRC522 (RF active)** | 20–30 mA | 40 mA | Continuous 13.56 MHz polling |
| **3.3 V -> ES8311 + PCF8575 + LEDs** | 15 mA | 40 mA | Keep separate analog ground return for ES8311 mic |
| **Total 12 V Input Budget** | ~2.5 W idle | ~15 W peak (unlock + call + Wi-Fi TX) | Minimum **12 V / 2.0 A (24 W)** UL/CE-listed DC supply |

## Design Precautions
- **Brownout Prevention:** Isolate the 12 V solenoid ground return to a star ground near the input barrel jack/terminal block and place >= 470 µF bulk capacitance on 12 V and >= 220 µF on 3.3 V so solenoid inrush never trips ESP32-S3 brownout detection.
- **Audio Noise Isolation:** Power NS4150B from the 5 V rail (not 3.3 V) and feed ES8311 AVDD through a ferrite bead + 10 µF/0.1 µF filter to keep Wi-Fi TX buzz out of the microphone.
