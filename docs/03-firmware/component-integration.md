# Component Integration & Registry Matrix

## 1. Sourcing Policy
Per [ADR-0001](../adr/0001-firmware-architecture.md), `bojaqu-door-lock` resolves external dependencies in strict priority order:
1. **Built-in ESP-IDF v6.1.0 components** (official core drivers & networking/security stacks).
2. **Official Espressif ESP Component Registry (`espressif/*`)** packages compatible with `ESP-IDF >=6.1.0` and `esp32s3`.
3. **Bojaqu Shared Component Repository (`https://github.com/blankryan/bjq-esp-component.git`)** when no official `espressif/*` component exists—keeping reusable drivers and security primitives integrated across all Bojaqu firmware repos.

Third-party community components (`abobija/rc522`, `esp-idf-lib/pcf8575`, etc.) are intentionally excluded to maintain single-vendor security auditing and ESP-IDF v6.1 API compatibility.

## 2. Complete Dependency Matrix

| Subsystem / Peripheral | Component | Source | Target Version | Status | Notes |
|---|---|---|---|---|---|
| **RFID Transceiver (MFRC522)** | `bjq_mfrc522` | `bjq-esp-component` (`drivers/bjq_mfrc522`) | `v0.2.0` -> `v0.3.0` | Integrated (`v0.2.0`); **Issue [#7](https://github.com/blankryan/bjq-esp-component/issues/7)** | [REQ-002](component-requests/REQ-002-bjq-mfrc522-v0.3.0-isodep.md) adds ISO-DEP RATS/APDU & reset callback |
| **RFID Crypto Auth (DESFire EV2/EV3)** | `bjq_desfire` | `bjq-esp-component` (`security/bjq_desfire`) | `v0.1.0` | **Issue [#8](https://github.com/blankryan/bjq-esp-component/issues/8)** | [REQ-003](component-requests/REQ-003-bjq-desfire.md) AES-128 EV2First + CMAC/enciphered file R/W |
| **16-bit I2C GPIO Expander (PCF8575)** | `bjq_pcf8575` | `bjq-esp-component` (`drivers/bjq_pcf8575`) | `v0.1.0` | **Issue [#6](https://github.com/blankryan/bjq-esp-component/issues/6)** | [REQ-001](component-requests/REQ-001-bjq-pcf8575.md) no official `espressif/*` PCF8575 driver exists |
| **Camera Sensor (OV5640 DVP)** | `espressif/esp32-camera` | Official Registry | `^2.2.0` | Verified (`IDF >=5.1`, S3 supported) | Hardware JPEG output directly into Octal PSRAM |
| **Audio Codec (ES8311 + NS4150B)** | `espressif/esp_codec_dev` | Official Registry | `^1.6.2` | Verified (`IDF >=4.0`, S3 supported) | Natively supports ES8311 + NS4150 PA control; replaces deprecated `espressif/es8311` |
| **Audio Compression (Opus / G.711)** | `espressif/esp_audio_codec` | Official Registry | `^2.0.0` (resolve latest IDF 6.1) | Official Registry | Encodes/decodes WebRTC voice intercom & local audio prompts |
| **WebRTC PeerConnection** | `espressif/esp_peer` | Official Registry | `^1.5.8` | Verified (`espressif/esp_libsrtp` transitive) | ICE/STUN/TURN, DTLS-SRTP audio, DataChannel/RTP MJPEG |
| **MQTT over TLS (MQTTS)** | `espressif/mqtt` | Official Registry / IDF | Latest IDF 6.1 | Official Registry | Control plane, telemetry, WebRTC signaling, OTA trigger |
| **BLE Wi-Fi Provisioning** | `espressif/network_provisioning` | Official Registry | Latest IDF 6.1 | Official Registry | BLE Security 2 (SRP6a) + QR PoP |
| **BLE Stack (GATT Offline Unlock)** | `bt` (NimBLE) | Built-in ESP-IDF 6.1 | `6.1.0` | Built-in | Low-RAM BLE host coexisting with Wi-Fi |
| **OTA & HTTPS Client** | `esp_https_ota`, `app_update`, `esp_http_client` | Built-in ESP-IDF 6.1 | `6.1.0` | Built-in | Dual-OTA download, SHA-256/signature check, rollback |
| **Storage & Security** | `nvs_flash`, `spiffs`, `mbedtls`, `esp_driver_*` | Built-in ESP-IDF 6.1 | `6.1.0` | Built-in | Encrypted NVS, `assets`/`storage` SPIFFS, TLS/mTLS/AES |

## 3. Target `main/idf_component.yml` (Once Requests Are Tagged)
> [!NOTE]
> Currently [main/idf_component.yml](../../main/idf_component.yml) pins `bjq_mfrc522/v0.2.0` so the existing `dev` baseline builds today. Once `REQ-001`, `REQ-002`, and `REQ-003` are tagged in `blankryan/bjq-esp-component` and firmware implementation starts, update `main/idf_component.yml` to:

```yaml
dependencies:
  idf:
    version: ">=6.1.0,<7.0.0"

  # Official Espressif ESP Component Registry
  espressif/esp32-camera: "^2.2.0"
  espressif/esp_codec_dev: "^1.6.2"
  espressif/esp_audio_codec: "*"
  espressif/esp_peer: "^1.5.8"
  espressif/mqtt: "*"
  espressif/network_provisioning: "*"

  # Bojaqu Shared Components (https://github.com/blankryan/bjq-esp-component.git)
  bjq_mfrc522:
    git: "https://github.com/blankryan/bjq-esp-component.git"
    path: "drivers/bjq_mfrc522"
    version: "bjq_mfrc522/v0.3.0"
  bjq_pcf8575:
    git: "https://github.com/blankryan/bjq-esp-component.git"
    path: "drivers/bjq_pcf8575"
    version: "bjq_pcf8575/v0.1.0"
  bjq_desfire:
    git: "https://github.com/blankryan/bjq-esp-component.git"
    path: "security/bjq_desfire"
    version: "bjq_desfire/v0.1.0"
```

## 4. Shared Bus Ownership Rules
- **Shared `I2C0` Bus (`GPIO1` SDA, `GPIO7` SCL):** Initialized ONCE in `board_hal` using ESP-IDF v6.1 `i2c_new_master_bus()`. `bjq_pcf8575` (`0x20`), `esp_codec_dev` ES8311 (`0x18`), and `esp32-camera` OV5640 SCCB (`0x3C`) attach as devices on the shared bus (Note: verify `esp32-camera` SCCB I2C-master sharing flag `CONFIG_SCCB_CLK_FREQ` / `sccb_i2c_port` so it does not double-install the legacy I2C driver on `I2C_NUM_0`!).
- **`SPI2_HOST` (`GPIO2/4/5/6`):** Initialized ONCE in `board_hal`; `bjq_mfrc522` attaches as a device and never frees the bus.
- **`I2S_NUM_0` (`GPIO39/40/41/42/48`):** Initialized in `audio_hal` in full-duplex standard Philips mode (16 kHz, 16-bit, mono, MCLK = 256 × fs = 4.096 MHz) and passed into `esp_codec_dev`.

## 5. Tracked Component Requests Checklist
- [x] File [REQ-001: `bjq_pcf8575`](component-requests/REQ-001-bjq-pcf8575.md) -> [blankryan/bjq-esp-component#6](https://github.com/blankryan/bjq-esp-component/issues/6)
- [x] File [REQ-002: `bjq_mfrc522 v0.3.0` ISO-DEP](component-requests/REQ-002-bjq-mfrc522-v0.3.0-isodep.md) -> [blankryan/bjq-esp-component#7](https://github.com/blankryan/bjq-esp-component/issues/7)
- [x] File [REQ-003: `bjq_desfire`](component-requests/REQ-003-bjq-desfire.md) -> [blankryan/bjq-esp-component#8](https://github.com/blankryan/bjq-esp-component/issues/8)
