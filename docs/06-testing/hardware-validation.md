# Hardware Validation Checklist (Rev A Board)

## Power & Boot Safety
- [ ] Verify 12 V -> 5 V -> 3.3 V rails under idle and peak (solenoid + Wi-Fi TX + speaker) load; zero ESP32-S3 brownout resets during 50 consecutive solenoid actuations.
- [ ] Scope `GPIO21` (`LOCK_ACTUATOR_EN`) during power-on, EN reset, and watchdog panic: must never glitch HIGH above MOSFET threshold.
- [ ] Verify NS4150B speaker produces zero loud pop on boot (`PCF8575 P00 AUDIO_PA_EN_N` remains HIGH/muted until firmware unmutes).
- [ ] Verify `GPIO33`–`GPIO37` remain completely unconnected and 8 MB Octal PSRAM initializes at 80 MHz (`SPIRAM` test passes).

## Peripheral Interfaces
- [ ] **Shared `I2C0` (400 kHz):** Clean ACK scan for ES8311 (`0x18`), PCF8575 (`0x20`), and OV5640 (`0x3C`) simultaneously without bus lockup.
- [ ] **PCF8575:** Confirm toggling Port 0 outputs (`P00`–`P07`) never clears Port 1 input pull-ups (`P10`–`P17`); verify `GPIO38` `INT` fires cleanly on reed, doorbell, exit, and tamper events.
- [ ] **MFRC522 (`SPI2`):** Verify `VersionReg=0x91`/`0x92`, `RFID_RST_N` via `PCF8575 P03`, and >= 25 mm read distance on physical **MIFARE DESFire EV2/EV3** cards through enclosure bezel.
- [ ] **OV5640 (`LCD_CAM`):** Capture sustained VGA (640x480) hardware JPEG at >= 12 fps and SVGA/XGA snapshots with zero DMA sync errors while Wi-Fi TX is active.
- [ ] **ES8311 + NS4150B (`I2S0`):** Record mic and play local prompt at 16 kHz mono; verify acoustic echo margin and no Wi-Fi buzz on mic audio.
- [ ] **Thermal:** Run continuous 10-minute WebRTC video+audio stress test inside enclosure; verify ESP32-S3 and OV5640 temperatures remain within operating limits.
