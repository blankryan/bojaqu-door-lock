# Functional Requirements

| ID | Category | Requirement | Verification |
|---|---|---|---|
| **FR-001** | RFID Access | The lock shall authenticate MIFARE DESFire EV2/EV3 cards using AES-128 mutual auth (`AuthenticateEV2First`) and per-card diversified keys in `prod` builds. | `bjq_desfire` vector tests + hardware bench |
| **FR-002** | RFID Access | Authorized card tap shall unlock the door when locked (auto-relocking after configurable timeout once door closes) or immediately re-lock if tapped while unlocked. | Integration test |
| **FR-003** | RFID Access | RFID verification and unlock shall function completely offline using the local encrypted NVS credential store. | Disconnect Wi-Fi + card tap test |
| **FR-004** | RFID Management | The Android app shall enroll, list, schedule, and revoke RFID cards via cloud MQTTS synchronization (`cmd/rfid_sync`). | End-to-end cloud/app test |
| **FR-005** | Mobile Control | Authorized Android users shall remotely unlock and lock the door via MQTTS with replay protection (`request_id`, UTC expiry, server signature). | MQTTS protocol + replay test |
| **FR-006** | Offline BLE | Authorized Android users shall unlock the door nearby over NimBLE GATT using challenge-response authentication when Wi-Fi/cloud is unavailable. | BLE GATT integration test |
| **FR-007** | Provisioning | Unprovisioned devices shall support BLE Wi-Fi provisioning using Security 2 (SRP6a) and a per-device QR Proof of Possession. | Provisioning test |
| **FR-008** | Live Video | Upon doorbell press or Android "Live View" request, the lock shall stream OV5640 hardware JPEG video (VGA/SVGA @ 10–15 fps) over WebRTC (`espressif/esp_peer`). | WebRTC LAN + 4G NAT test |
| **FR-009** | Event Snapshots | The lock shall capture and upload an OV5640 JPEG snapshot over HTTPS on doorbell ring, unlock, forced-door alarm, and brute-force RFID lockout. | HTTPS upload + FCM payload test |
| **FR-010** | Two-Way Audio | During an active WebRTC call, the lock shall stream full-duplex mono voice audio (ES8311 mic + NS4150B speaker) over WebRTC SRTP (Opus / G.711A). | Intercom latency & audio quality test |
| **FR-011** | Local Audio | The lock shall play local chimes/voice prompts (doorbell, unlock, deny, pairing, OTA) from the `assets` partition via ES8311 + NS4150B. | Hardware bench test |
| **FR-012** | Door & Sensors | The lock shall monitor the magnetic reed door sensor, doorbell button, indoor exit button, and enclosure tamper switch via PCF8575 interrupt, reporting state changes and alarms. | Hardware I/O & alarm test |
| **FR-013** | OTA Updates | The lock shall discover available firmware via MQTTS notifications and 24-hour HTTPS manifest checks, verify SHA-256 + Secure Boot v2 signature, enforce door-closed-and-locked safety before reboot, and automatically roll back on post-boot self-test failure. | Dual-slot OTA + fault-injection rollback test |
| **FR-014** | Build Presets | The project shall build cleanly for `dev` and `prod` presets via `CMakePresets.json`, with `dev` permitting Wokwi/UID fallback and `prod` enforcing hardware security. | CI preset build matrix |
