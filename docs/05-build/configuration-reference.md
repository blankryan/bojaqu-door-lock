# Configuration Reference (`dev` vs `prod`)

Configuration merges in order: `sdkconfig.defaults` -> `configs/<preset>/sdkconfig` -> `build-outputs/<preset>/sdkconfig`.

## Preset Comparison Matrix

| Setting Area | Common (`sdkconfig.defaults`) | `dev` (`configs/dev/sdkconfig`) | `prod` (`configs/prod/sdkconfig`) |
|---|---|---|---|
| **Target & Flash** | `esp32s3`, 16 MB QIO @ 80 MHz | Inherited | Inherited |
| **PSRAM** | 8 MB Octal (`SPIRAM_MODE_OCT`, 80 MHz) | Inherited | Inherited |
| **CPU & FreeRTOS** | 240 MHz, 1000 Hz tick | Inherited | Inherited |
| **Partition Table** | Custom `partitions.csv` at `0x10000` | Inherited | Inherited |
| **OTA Rollback** | `BOOTLOADER_APP_ROLLBACK_ENABLE=y` | Anti-Rollback OFF | `BOOTLOADER_APP_ANTI_ROLLBACK=y` |
| **Secure Boot v2** | — | OFF (unsigned dev builds) | **ON** (RSA-3072 signed, key out-of-band) |
| **Flash Encryption** | — | OFF (reversible flashing) | **ON — Release Mode** |
| **NVS Encryption** | — | OFF | **ON** (`nvs_keys` encrypted partition) |
| **ROM Download Mode** | — | Enabled (USB/UART flashing) | **Disabled** (`SECURE_DISABLE_ROM_DL_MODE=y`) |
| **Compiler Opt** | — | `OPTIMIZATION_DEBUG=y` | `OPTIMIZATION_SIZE=y` |
| **Log Level** | — | `DEBUG` | `WARN` (zero sensitive payload logging) |
| **Task Watchdog** | 5 s enabled on both CPUs | Panic OFF (print trace) | **Panic ON** (auto-reboot to fail-safe lock) |
| **RFID Auth Mode** *(App Kconfig)* | DESFire EV2/EV3 AES-128 | `BJQ_RFID_ALLOW_UID_FALLBACK=y` (Wokwi/bench) | UID fallback hard-disabled at compile time |
| **Cloud / OTA Endpoints** *(App Kconfig)* | MQTTS + HTTPS manifest | Staging broker & staging OTA CDN | Production broker & production signed CDN |
