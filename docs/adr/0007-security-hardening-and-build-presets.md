# ADR-0007: Security Hardening Baseline and CMakePresets (`dev` vs `prod`) Strategy

- Status: Accepted
- Date: 2026-11-26
- Scope: Platform
- Owners: Bojaqu Security & Firmware Engineering

## Context
A physical smart door-lock is exposed to both network attackers and physical tampering (open enclosure, SPI flash dump, UART/JTAG probing, firmware downgrade). At the same time, engineers need fast iterative builds, verbose logging, Wokwi simulation, and non-destructive USB/JTAG debugging during development.

The project uses `CMakePresets.json` with two configure presets (`dev` and `prod`) that layer `configs/dev/sdkconfig` or `configs/prod/sdkconfig` on top of `sdkconfig.defaults`. Previously, `CMakePresets.json` contained a copy-paste path bug in `prod` (`./build-outputs/water_level/sdkconfig`), and both overlay files were empty.

## Decision
Establish a strict **two-preset build configuration (`dev` vs `prod`)** via `CMakePresets.json` with full hardware-rooted security enforced in `prod`:

1. **Common Baseline (`sdkconfig.defaults`):**
   - Target `esp32s3`, 16 MB Flash (`QIO`/80 MHz), 8 MB Octal PSRAM (`SPIRAM_MODE_OCT`, 80 MHz), 240 MHz CPU, `CONFIG_PARTITION_TABLE_OFFSET=0x10000`, custom `partitions.csv`, `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`, NimBLE enabled, mbedTLS CA bundle enabled, FreeRTOS `1000 Hz` tick rate, Task Watchdog enabled.
2. **Development Preset (`dev` -> `configs/dev/sdkconfig`, output `build-outputs/dev`):**
   - Reversible bench/Wokwi flashing: **Secure Boot OFF**, **Flash Encryption OFF**, **NVS Encryption OFF**, **UART/USB ROM Download Mode ENABLED**.
   - Diagnostics: `CONFIG_LOG_DEFAULT_LEVEL_DEBUG=y`, `CONFIG_COMPILER_OPTIMIZATION_DEBUG=y`, assertions enabled, coredump to flash/UART.
   - Application dev hooks (Kconfig): staging MQTT/HTTPS endpoints, `CONFIG_BJQ_RFID_ALLOW_UID_FALLBACK=y` for Wokwi/bench UID testing.
3. **Production Preset (`prod` -> `configs/prod/sdkconfig`, output `build-outputs/prod`):**
   - **Hardware Root of Trust:**
     - **Secure Boot v2** (`CONFIG_SECURE_BOOT=y`, `CONFIG_SECURE_BOOT_V2_ENABLED=y`, RSA-3072 / ECDSA signing in CI/HSM; signing keys NEVER committed to Git).
     - **Flash Encryption — Release Mode** (`CONFIG_SECURE_FLASH_ENC_ENABLED=y`, `CONFIG_SECURE_FLASH_ENCRYPTION_MODE_RELEASE=y`).
     - **NVS Encryption** (`CONFIG_NVS_ENCRYPTION=y` backed by `nvs_keys` encrypted partition) protecting Wi-Fi creds, DESFire master diversification key, and local access store.
     - **Anti-Rollback** (`CONFIG_BOOTLOADER_APP_ANTI_ROLLBACK=y`) blocking firmware downgrades to vulnerable versions.
     - **Physical Interface Lockdown:** Disable ROM download mode / JTAG (`CONFIG_SECURE_DISABLE_ROM_DL_MODE=y`) so flash/RAM cannot be dumped from the outdoor unit.
   - **Network & Provisioning Security:**
     - **BLE Provisioning Security 2 (SRP6a)** with unique per-device QR Proof of Possession stored in `fctry` NVS.
     - **Per-device X.509 mTLS** certificate & private key stored in factory-provisioned encrypted `fctry` NVS (migrating to ESP32-S3 Digital Signature peripheral where feasible in manufacturing Rev B).
     - **Hardened Runtime:** `CONFIG_LOG_DEFAULT_LEVEL_WARN=y`, `CONFIG_COMPILER_OPTIMIZATION_SIZE=y`, Task Watchdog panic-to-reboot (`CONFIG_ESP_TASK_WDT_PANIC=y`), coredump to flash, and compile-time enforcement disabling UID-only RFID fallback.
4. **Fix `CMakePresets.json`:**
   - Correct `prod` `SDKCONFIG` cache variable from `./build-outputs/water_level/sdkconfig` to `./build-outputs/prod/sdkconfig`, and add matching `buildPresets` (`dev`, `prod`).

## Alternatives considered
- **Single build configuration with runtime debug flags:** Rejected because runtime flags cannot disable irreversible eFuse features (Secure Boot v2, Flash Encryption Release mode) or eliminate debug symbol/log surface area in production.
- **Deferring Secure Boot v2 & Flash Encryption until after v1:** Rejected because physical door-lock hardware without flash encryption allows an attacker with brief physical access to read Wi-Fi credentials, mTLS private keys, and DESFire master keys directly from the 16 MB SPI flash chip.
- **Flash Encryption in Development Mode for `prod`:** Rejected because Development Mode leaves UART re-flashing open and is explicitly unsupported for fielded production units by Espressif.

## Consequences
- **Benefits:** Production devices resist physical flash extraction, unsigned firmware injection, version rollback, and credential theft; developers retain fast, non-destructive `dev` builds and Wokwi simulation from the same repository.
- **Trade-offs / Constraints:** **`prod` builds burn irreversible eFuses on physical ESP32-S3 modules**—engineers must NEVER flash a `prod` image onto a reusable development board unless intending to permanently lock it; production builds require an external signing key path (`SECURE_BOOT_SIGNING_KEY` / CI KMS) outside Git; factory provisioning must flash unique `fctry` NVS binary per device.
- **Follow-up work:** Set up CI signing key handling (never in repo); document factory provisioning flow in [Provisioning](../07-deployment/provisioning.md) and [Production Release](../07-deployment/production-release.md); add Kconfig options for `BJQ_RFID_ALLOW_UID_FALLBACK` and staging/prod endpoints when scaffolding application components.

## References
- [CMakePresets.json](../../CMakePresets.json)
- [Build Instructions](../05-build/build-instructions.md)
- [Configuration Reference](../05-build/configuration-reference.md)
- [Provisioning](../07-deployment/provisioning.md)
- [ADR-0005](0005-rfid-cryptographic-authentication.md), [ADR-0006](0006-ota-update-and-flash-partitioning.md)
