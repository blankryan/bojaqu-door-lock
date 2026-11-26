# Features — Bojaqu Smart Door-Lock

## v1 Core Features
1. **RFID Lock & Unlock (`bjq_mfrc522` + `bjq_desfire`)**
   - Tap an authorized **MIFARE DESFire EV2/EV3** card (AES-128 mutual auth with per-card diversified keys in `prod`; UID/mock mode permitted only in `dev`) to unlock—or tap while unlocked to immediately re-lock.
   - Local encrypted NVS credential cache ensures 100% offline operation; Android app enrolls, schedules, and revokes cards over MQTTS.
   - Brute-force lockout (exponential cooldown after 5 invalid taps) with snapshot capture and security alert.
2. **Mobile Lock Control & Offline BLE Fallback (Android)**
   - Remote unlock/lock from anywhere via Bojaqu Cloud (`MQTTS` QoS 1 with expiry timestamp + nonce replay protection).
   - Nearby offline unlock over **NimBLE GATT** using per-session challenge-response authentication when Wi-Fi/internet is unavailable.
   - Initial Wi-Fi setup via BLE Provisioning (Security 2 / SRP6a + QR PoP).
3. **Live Camera View & Event Snapshots (`OV5640` + `espressif/esp32-camera`)**
   - On-demand live camera stream to Android via **WebRTC (`espressif/esp_peer`)** using OV5640 hardware JPEG (VGA 640x480 / SVGA 800x600 @ 10–15 fps).
   - Automatic HTTPS JPEG snapshot upload on doorbell ring, unlock, forced-door alarm, or repeated invalid RFID attempts.
4. **Two-Way Voice Intercom & Local Prompts (`ES8311 + NS4150B` + `espressif/esp_codec_dev`)**
   - Real-time full-duplex two-way audio over WebRTC SRTP (Opus / G.711A @ 16 kHz mono).
   - Local chimes and voice prompts played from the `assets` flash partition (doorbell chime, access granted/denied, pairing ready, OTA notice).
5. **Door State & Physical Safety**
   - 12 V fail-secure electric strike/solenoid with configurable auto-relock timer (default 5 s after door closes or unlock timeout).
   - Magnetic reed sensor for door open/closed state, door-left-ajar alerts, and forced-entry alarms.
   - Physical doorbell call button, indoor exit unlock button, and enclosure tamper switch.
6. **Secure Dual-Bank OTA Updates (`esp_https_ota`)**
   - Cloud-notified (MQTTS) + 24-hour periodic HTTPS manifest checks with SHA-256 + Secure Boot v2 verification.
   - Door-closed-and-locked safety interlock before reboot, user-confirmed or auto-update preference in Android, and automatic rollback if post-boot health self-test fails.

## Planned Post-v1 Extensibility
- **Home Assistant Integration:** Native MQTT Discovery adapter exposing lock entity, door sensor, tamper sensor, and snapshot/camera triggers.
- **ESP RainMaker / Matter over Wi-Fi:** Additional control-plane adapter modules hosted inside the 5 MB OTA app headroom without altering core lock domain logic.
