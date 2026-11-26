# Board Revisions

| Revision | Status | MCU Module | Key Hardware Changes |
|---|---|---|---|
| **Rev 0 (Bench / Wokwi)** | Active (current `main.c`) | ESP32-S3-DevKitC-1 (Wokwi `board-esp32-s3-devkitc-1`) | MFRC522 wired on `SPI2_HOST` (`GPIO2/4/5/6`); validates `bjq_mfrc522 v0.2.0` and `dev` preset build. |
| **Rev A (Prototype PCB)** | Planned (Target v1) | **ESP32-S3-WROOM-1-N16R8** (16 MB Flash + 8 MB Octal PSRAM) | Full integration: MFRC522 (`GPIO2/4/5/6`), OV5640 DVP, ES8311+NS4150B I2S0, PCF8575 (`0x20`, `INT=GPIO38`), boot-safe 12 V strike driver (`GPIO21` + 10 kΩ pull-down), reed/doorbell/exit/tamper inputs, 12 V -> 5 V -> 3.3 V power tree. |
| **Rev B (Pilot / Production)** | Future | ESP32-S3-WROOM-1-N16R8 | Production enclosure acoustics calibration, ESD/surge hardening on outdoor fascia, optional ESP32-S3 Digital Signature (DS) peripheral provisioning for mTLS key hardware protection. |
