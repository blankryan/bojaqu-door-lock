# ADR-0006: OTA Update Strategy, Safety Interlocks, and 16 MB Flash Partitioning

- Status: Accepted
- Date: 2026-11-26
- Scope: Platform
- Owners: Bojaqu Firmware & Platform Engineering

## Context
The smart door-lock must support reliable Over-The-Air (OTA) firmware updates for maintenance, security patches, bug fixes, and new features (including future Home Assistant / RainMaker / Matter integrations).

The repository currently sets `CONFIG_PARTITION_TABLE_TWO_OTA=y` in `sdkconfig.defaults`, which uses ESP-IDF's built-in 1 MB OTA slots. Once Wi-Fi, NimBLE, WebRTC (`espressif/esp_peer` + `esp_libsrtp`), `esp32-camera`, `esp_codec_dev`, `esp_audio_codec`, MQTTS, and mbedTLS are linked, the firmware binary will exceed 1 MB—causing build failures or preventing future feature growth—because partition tables cannot be safely resized via OTA on deployed devices.

## Decision
1. **Custom 16 MB Flash Partition Table (`partitions.csv`) with Shared `0x10000` Offset:**
   - Set `CONFIG_PARTITION_TABLE_OFFSET=0x10000` in `sdkconfig.defaults` (shared across `dev` and `prod`) so the 64 KB bootloader region fits Secure Boot v2 + Flash Encryption + Anti-Rollback without offset divergence between presets.
   - Allocate **two 5 MB OTA app slots** (`ota_0` at `0x040000`, `ota_1` at `0x540000`) to leave ample headroom for WebRTC and future Matter/RainMaker stacks.
   - Include dedicated partitions for `otadata` (8 KB), `phy_init` (4 KB), `nvs_keys` (4 KB, `encrypted`), `nvs` (128 KB), `fctry` (44 KB factory NVS for per-device mTLS certs & PoP), `coredump` (64 KB), `assets` (2 MB read-only SPIFFS for local voice prompts/chimes), and `storage` (3,776 KB read/write SPIFFS for offline audit logs and snapshot ring buffer).
2. **Hybrid Cloud-Notified + Periodic Pull HTTPS OTA:**
   - **Discovery:** Device checks a signed JSON release manifest over HTTPS (mTLS / pinned CA) every **24 hours** (with jitter) AND immediately upon receiving an MQTTS `cmd/ota_available` notification.
   - **Policy:** Homeowners choose in the Android app between **User-Confirmed OTA** (app prompts to install) and **Auto-Update** (installs automatically during quiet hours / idle state), with a server flag for urgent security releases.
   - **Transport & Verification:** Download firmware into the inactive OTA slot using ESP-IDF `esp_https_ota` over TLS 1.2/1.3, verifying manifest semantic version + monotonic security version, SHA-256 digest, and **Secure Boot v2 RSA-3072/ECDSA signature** (`prod`) before switching boot partition.
3. **Door-Lock Safety Interlocks:**
   - OTA download may occur in the background while idle (pausing WebRTC/camera streaming), **never blocking local RFID unlock**.
   - Reboot into the newly flashed slot is **strictly gated** until: (a) the door reed sensor confirms the door is **closed**, (b) the lock actuator is **locked**, (c) no WebRTC call or active unlock cycle is in progress, and (d) no card tap occurred within the last 10 seconds.
4. **Automatic Rollback & Post-Boot Self-Test:**
   - Enable `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y` in both presets and `CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK=y` in `prod`.
   - After booting a new image (`ESP_OTA_IMG_PENDING_VERIFY`), firmware runs a **post-boot health self-test** (verifies NVS mount, PCF8575 I2C probe, MFRC522 `VersionReg` probe, Wi-Fi association, and MQTTS broker reconnect within 60 s) before calling `esp_ota_mark_app_valid_cancel_rollback()`. Any crash, watchdog reset, or self-test failure triggers automatic rollback to the previous known-good slot.

## Alternatives considered
- **Default `CONFIG_PARTITION_TABLE_TWO_OTA=y` (1 MB slots):** Rejected because the linked firmware exceeds 1 MB and would block OTA and future ecosystem features.
- **2x 3 MB or 2x 4 MB OTA slots:** Viable, but 16 MB flash comfortably allows **2x 5 MB** while still providing >5.6 MB for `assets` + `storage`, eliminating future flash-ceiling risk if Matter over Wi-Fi is added.
- **BLE / Local Android-Relayed OTA:** Rejected as primary transport due to slow BLE throughput on multi-megabyte images (~5–15 min) and poor reliability; HTTPS pull directly from CDN/backend is fast and resumable.
- **Immediate reboot upon download completion:** Rejected because rebooting while the door is unlocked or mid-entry creates a physical security and UX hazard.

## Consequences
- **Benefits:** Guaranteed flash headroom for v1 and future Matter/RainMaker features; identical flash map in `dev` and `prod`; zero-brick automatic rollback; safe updates that never leave the door stuck unlocked or interrupt entry.
- **Trade-offs / Constraints:** Changing `partitions.csv` later requires full UART/USB re-flashing (cannot be changed via OTA), so the layout must be frozen prior to pilot production; enabling `CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK` in `prod` permanently burns eFuse security version bits when security version increments, so security version bumps must be reserved for security-critical releases only.
- **Follow-up work:** Implement the post-boot self-test checklist in `ota_manager`; configure release CI to sign binaries, generate SHA-256 manifests, and bump `CONFIG_BOOTLOADER_APP_SEC_VER` only on security advisories.

## References
- [Partition Layout](../05-build/partition-layout.md)
- [Production Release & OTA Runbook](../07-deployment/production-release.md)
- [partitions.csv](../../partitions.csv)
- [ADR-0007](0007-security-hardening-and-build-presets.md)
