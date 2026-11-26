# Non-Functional Requirements

| ID | Category | Target / Constraint |
|---|---|---|
| **NFR-001** | RFID Unlock Latency | <= 500 ms from DESFire card tap to lock strike actuation (offline or online). |
| **NFR-002** | Remote Unlock Latency | <= 1.5 s end-to-end (Android tap -> Cloud -> MQTTS -> strike actuation) on healthy Wi-Fi. |
| **NFR-003** | WebRTC Setup & Media Latency | <= 3.0 s from doorbell press / Live View tap to first video frame + audio; <= 300 ms two-way voice glass-to-glass latency. |
| **NFR-004** | Video Frame Rate | >= 10 fps steady at VGA (640x480) hardware JPEG during concurrent two-way audio intercom. |
| **NFR-005** | Security (`prod`) | Secure Boot v2, Flash Encryption (Release mode), NVS Encryption, Anti-Rollback, ROM DL disabled, X.509 mTLS, BLE Security 2, DESFire AES-128 mutual auth; zero plaintext secrets in logs or flash. |
| **NFR-006** | Availability & Watchdog | Task Watchdog (`5 s`, panic-to-reboot in `prod`); automatic Wi-Fi/MQTTS reconnection with exponential backoff; offline RFID & exit button unaffected by network outages. |
| **NFR-007** | Flash & Memory Budget | App binary <= 3.5 MB (inside 5 MB OTA slot, leaving >= 1.5 MB growth headroom); minimum free internal DMA-capable SRAM >= 40 KB during peak WebRTC + MQTTS + RFID activity. |
| **NFR-008** | Fail-Safe Hardware State | Lock actuator `GPIO21` stays physically LOW (locked) across power-on, brownout, reset, panic, and I2C bus hang; audio PA muted and camera powered down at boot. |
| **NFR-009** | Maintainability | All significant architecture decisions recorded as ADRs in `docs/adr/`; reusable hardware/security drivers hosted in `bjq-esp-component` with Unity test coverage. |
