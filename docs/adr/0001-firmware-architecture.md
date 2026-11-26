# ADR-0001: Modular Firmware Architecture and Shared Component Boundary

- Status: Accepted
- Date: 2026-11-26
- Scope: Product
- Owners: Bojaqu Firmware Engineering

## Context
`bojaqu-door-lock` is an ESP-IDF v6.1 smart door-lock firmware targeting `esp32s3` with 16 MB flash and 8 MB Octal PSRAM. It combines physical access control (MFRC522 RFID + 12V fail-secure lock actuator + reed/button inputs), real-time multimedia (OV5640 camera + ES8311/NS4150B audio codec), mobile/cloud communication (MQTTS + NimBLE + WebRTC), and signed HTTPS OTA updates.

Two organizational constraints govern component selection and architecture:
1. Official Espressif ESP-IDF Registry components (`espressif/*`) and built-in ESP-IDF v6.1 components must be preferred whenever available.
2. When no official Espressif component exists, reusable hardware drivers and security primitives must be requested and hosted in the shared Bojaqu component repository (`https://github.com/blankryan/bjq-esp-component.git`) so they can be shared across Bojaqu products (e.g., door-lock, water-level, future smart-home nodes).
3. Although v1 integrates with the Bojaqu Cloud + Android app, the firmware must remain extensible to Home Assistant (MQTT Discovery), ESP RainMaker, or Matter over Wi-Fi without rewriting core door-lock domain logic.

## Decision
Adopt a **four-layer modular firmware architecture** inside `bojaqu-door-lock`, combined with a strict **official-first + shared-driver component policy**:

1. **Application / Domain Layer (`bojaqu-door-lock` local):**
   - `lock_controller`: Deterministic lock state machine (locked, unlocking, unlocked, door-ajar, forced-open, lockout), auto-relock timer, door-closed OTA interlock.
   - `access_manager`: Local offline credential authorization (DESFire EV2/EV3 AES-128 in `prod`, dev-only UID/mock fallback in `dev`), rate-limiting, and audit event emission.
   - `media_controller`: On-demand WebRTC intercom session coordinator, HTTPS event snapshot uploader, and local voice-prompt/chime playback.
   - `ota_manager`: Cloud-notified + 24h periodic HTTPS manifest check, safety-gated OTA download, post-boot self-test, and rollback confirmation.
2. **Communication & Ecosystem Adapter Layer (`bojaqu-door-lock` local):**
   - Internal typed event/command bus decouples domain services from external protocols.
   - v1 adapters: `cloud_mqtt_adapter` (MQTTS via `espressif/mqtt`), `ble_adapter` (NimBLE GATT provisioning + encrypted nearby unlock), `webrtc_adapter` (`espressif/esp_peer`).
   - Future adapters (Home Assistant MQTT Discovery, ESP RainMaker, Matter) plug into the same domain command/event interface.
3. **Board Support / HAL Layer (`bojaqu-door-lock` local):**
   - Owns SPI2, shared I2C0, I2S0, LCD_CAM DVP, native boot-safe `LOCK_ACTUATOR_EN` GPIO, and PCF8575 pin routing.
4. **Reusable Drivers & Primitives (External Dependencies):**
   - **Official Espressif Registry / Built-in ESP-IDF v6.1:**
     - `espressif/esp32-camera` (OV5640 hardware JPEG)
     - `espressif/esp_codec_dev` (ES8311 + NS4150B PA control; replaces deprecated `espressif/es8311`)
     - `espressif/esp_audio_codec` (Opus/G.711/local audio prompts)
     - `espressif/esp_peer` + `espressif/esp_libsrtp` (WebRTC ICE/DTLS/SRTP/SCTP)
     - `espressif/mqtt` (MQTTS), `espressif/network_provisioning` (BLE Wi-Fi provisioning)
     - Built-in IDF: `esp_https_ota`, `app_update`, `esp_http_client`, `bt` (NimBLE), `nvs_flash`, `spiffs`, `mbedtls`
   - **Shared Bojaqu Repository (`bjq-esp-component`):**
     - `drivers/bjq_mfrc522`: Existing `v0.2.0`; request `v0.3.0` adding ISO/IEC 14443-4 ISO-DEP RATS/APDU transceive.
     - `drivers/bjq_pcf8575`: New component request for PCF8575 16-bit I2C GPIO expander with INT support (no official `espressif/*` driver exists).
     - `security/bjq_desfire`: New component request for MIFARE DESFire EV2/EV3 AES-128 mutual auth over an injected APDU transport callback.

## Alternatives considered
- **Monolithic `main/` application calling peripheral drivers directly:** Rejected because protocol changes (e.g., adding Home Assistant or Matter) or hardware revisions would require invasive rewrites across lock safety logic.
- **Using third-party registry components (`abobija/rc522`, `esp-idf-lib/pcf8575`, `joltwallet/littlefs`):** Rejected because organizational policy requires official `espressif/*` components or Bojaqu-owned components in `bjq-esp-component` to guarantee ESP-IDF v6.1 compatibility, security auditing, and cross-product API consistency.
- **Putting door-lock business logic and OTA orchestration in `bjq-esp-component`:** Rejected for v1 to avoid premature abstraction across differing Bojaqu products; only hardware drivers and cryptographic card primitives are placed in `bjq-esp-component`, while cross-product cloud/OTA wrappers can be extracted later once a second product stabilizes identical needs.

## Consequences
- **Benefits:** Clean separation of safety-critical lock logic from cloud/mobile transports; official Espressif maintenance for complex stacks (WebRTC, camera, codec, MQTT); reusable, IDF-v6.1-native drivers across Bojaqu products.
- **Trade-offs / Constraints:** Full v1 RFID cryptographic auth and PCF8575 hardware bring-up depend on delivering three component requests in `bjq-esp-component` (`REQ-001`, `REQ-002`, `REQ-003`).
- **Follow-up work:** Submit `REQ-001` (`bjq_pcf8575`), `REQ-002` (`bjq_mfrc522 v0.3.0`), and `REQ-003` (`bjq_desfire`) to `blankryan/bjq-esp-component` and pin released tags in `main/idf_component.yml`.

## References
- [Firmware Architecture](../03-firmware/architecture.md)
- [Component Integration & Registry Matrix](../03-firmware/component-integration.md)
- [REQ-001: bjq_pcf8575](../03-firmware/component-requests/REQ-001-bjq-pcf8575.md) ([bjq-esp-component#6](https://github.com/blankryan/bjq-esp-component/issues/6))
- [REQ-002: bjq_mfrc522 v0.3.0](../03-firmware/component-requests/REQ-002-bjq-mfrc522-v0.3.0-isodep.md) ([bjq-esp-component#7](https://github.com/blankryan/bjq-esp-component/issues/7))
- [REQ-003: bjq_desfire](../03-firmware/component-requests/REQ-003-bjq-desfire.md) ([bjq-esp-component#8](https://github.com/blankryan/bjq-esp-component/issues/8))
- [ADR-0002](0002-hardware-platform-and-gpio-budget.md), [ADR-0003](0003-mobile-communication-and-control-plane.md)
