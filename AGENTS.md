# AGENTS.md — `bojaqu-door-lock`

Instructions for AI coding agents working in the **Bojaqu Smart Door-Lock** ESP-IDF firmware repository.

## 1. Project Snapshot

- **Product:** Mains-powered (12 V DC) smart door-lock and video-intercom firmware (`bojaqu-door-lock`).
- **Target Hardware:** `ESP32-S3-WROOM-1-N16R8` (`esp32s3`, dual-core 240 MHz, **16 MB Quad Flash**, **8 MB Octal PSRAM**), NXP MFRC522 (`SPI2`), OmniVision OV5640 (`LCD_CAM` + `I2C0`), ES8311 + NS4150B (`I2S0` + `I2C0`), PCF8575 (`I2C0` + `INT`), 12 V fail-secure electric strike (`GPIO21`).
- **Framework & Build:** **ESP-IDF `v6.1.0`** (`>=6.1.0,<7.0.0`), CMake `>=3.22` + Ninja, `CMakePresets.json` (schema v3), `MINIMAL_BUILD ON`.
- **Current Stage:** Architecture, hardware pinout, ADRs (`docs/adr/0001`–`0007`), 16 MB partition layout (`partitions.csv`), `dev`/`prod` presets, and Wokwi simulation are established. `main/main.c` currently runs the `bjq_mfrc522` (`v0.2.0`) SPI2 RFID polling baseline while shared component requests (`REQ-001`–`REQ-003`) and local `components/` modules are queued for implementation.

## 2. Environment Setup

### Prerequisites
- **ESP-IDF `v6.1.0`** (`>=6.1.0,<7.0.0`) with the `esp32s3` toolchain sourced in the shell (or use `.devcontainer/devcontainer.json` based on `espressif/idf`).
- **CMake `>=3.22`** and **Ninja** (required for `CMakePresets.json` v3).
- **Git + GitHub access** so IDF Component Manager can pull `https://github.com/blankryan/bjq-esp-component.git`.
- **Optional:** Wokwi CLI or VS Code Wokwi extension (`wokwi.toml`, `diagram.json`) for `dev` preset simulation.

### Quick Verification
```bash
idf.py --version
cmake --version
cmake --list-presets   # Expected: dev, prod
```

### IDE / Language Server Notes
- `.clangd` removes GCC-only flags (`Remove: [-f*, -m*]`). After configuring `dev`, `compile_commands.json` is generated in `build-outputs/dev/`.
- `.vscode/` and `.devcontainer/` are gitignored; do not commit machine-specific paths.

## 3. Build, Flash, and Simulation (`CMakePresets.json`)

Configuration merges in strict order:
`sdkconfig.defaults` -> `configs/<preset>/sdkconfig` -> `build-outputs/<preset>/sdkconfig`

### Development Build (`dev` Preset — Default for Local Work)
Safe for bench ESP32-S3 devkits and Wokwi simulation (DIO flash, `SPIRAM_IGNORE_NOTFOUND=y`, DEBUG logs, Secure Boot/Flash/NVS Encryption OFF, ROM Download Mode ON):
```bash
cmake --preset dev
cmake --build --preset dev
idf.py -B build-outputs/dev -p <PORT> flash monitor
```
Equivalent `idf.py` build:
```bash
idf.py -B build-outputs/dev -D SDKCONFIG=build-outputs/dev/sdkconfig \
  -D "SDKCONFIG_DEFAULTS=sdkconfig.defaults;configs/dev/sdkconfig" build
```

### Wokwi Simulation (`dev` Preset)
`wokwi.toml` points directly to `build-outputs/dev/flasher_args.json` and `build-outputs/dev/bojaqu-door-lock.elf`. `diagram.json` simulates `board-esp32-s3-devkitc-1` (16 MB Flash, 8 MB Octal PSRAM) wired to `board-mfrc522` on `SPI2_HOST` (`GPIO2/4/5/6`). Build `dev` first, then start Wokwi via `wokwi-cli` or VS Code.

### Production Build (`prod` Preset)
> **CRITICAL:** `prod` enables **Secure Boot v2 (RSA-3072)**, **Flash Encryption (Release mode)**, **NVS Encryption**, **Anti-Rollback**, and **disables ROM Download Mode**. Flashing `prod` onto physical hardware permanently burns eFuses and locks the chip. **Never** flash `prod` onto a reusable development board.

Provide the signing key out-of-band (never commit `*.pem`, `*.key`, or `secure_boot_signing_key*`):
```bash
cmake --preset prod -DCONFIG_SECURE_BOOT_SIGNING_KEY="/secure/path/bojaqu_door_lock_sbv2.pem"
cmake --build --preset prod
```

### Clean Rebuild After Editing `sdkconfig` Overlays
ESP-IDF caches resolved options in `build-outputs/<preset>/sdkconfig`. After editing `sdkconfig.defaults` or `configs/<preset>/sdkconfig`, delete the cached `sdkconfig` before reconfiguring:
```bash
rm -f build-outputs/dev/sdkconfig
cmake --preset dev
cmake --build --preset dev
```

## 4. Code Formatting & C Style Rules (ESP-IDF 4-Space K&R — `.clang-format`)

All C and header files (`*.c`, `*.h`) must strictly follow **ESP-IDF 4-space K&R C style**, enforced by **`.clang-format`** at the repository root (requires `clang-format >= 16`, also picked up automatically by `clangd` / VS Code):

```bash
# Format a file in place
clang-format -i main/main.c

# Verify formatting without modifying files (CI check)
clang-format --dry-run --Werror main/main.c
```

### Formatting Specification (`.clang-format`)
- **Indentation:** **4 spaces** per level; **never** use tabs (`IndentWidth: 4`, `UseTab: Never`).
- **Column Limit:** **120 characters** maximum (`ColumnLimit: 120`).
- **Brace Style (ESP-IDF K&R / Linux):**
  - **Function definitions:** Opening `{` on a **separate new line**.
  - **Control flow & types** (`if`, `else`, `for`, `while`, `switch`, `struct`, `enum`): Opening `{` on the **same line**, preceded by a single space (`if (cond) {`, `} else if (other) {`).
  - **Always use braces `{ ... }`** for single-statement `if`/`else`/`for`/`while` bodies.
  - **Vertical spacing:** One blank line between functions; no blank line immediately inside a function's opening `{` or closing `}`.
- **Horizontal Spacing:**
  - Single space after control-flow keywords (`if (`, `for (`, `while (`, `switch (`), none between function name and `(`.
  - Single space around binary operators (`a + b`, `err != ESP_OK`); no space for unary operators, `.`, or `->`.
  - No trailing whitespace; avoid brittle manual column alignment unless guarded with `// clang-format off` / `// clang-format on`.
- **Pointers:** Right-aligned to the identifier (`PointerAlignment: Right`), e.g., `const uint8_t *data`, `esp_err_t *out_err`.
- **Includes:** Preserve intentional grouping (`SortIncludes: Never`):
  1. Standard C headers (`<inttypes.h>`, `<stdbool.h>`, `<stddef.h>`, `<stdint.h>`, `<string.h>`)
  2. `"sdkconfig.h"` (when needed)
  3. FreeRTOS headers (`"freertos/FreeRTOS.h"`, `"freertos/task.h"`, ...)
  4. ESP-IDF driver & system headers (`"driver/..."`, `"esp_err.h"`, `"esp_log.h"`, ...)
  5. Bojaqu shared / local component headers (`"bjq_mfrc522.h"`, local module headers)
- **Line Endings & EOF:** Unix **`LF`** line endings (`LineEnding: LF`) and a single trailing newline at EOF (`InsertNewlineAtEOF: true`).
- **Diff Hygiene:** When editing an existing file, format only touched lines/functions unless rewriting the file from scratch.

```c
/* Example matching ESP-IDF 4-space K&R style */
static esp_err_t lock_apply_state(bool unlock)
{
    if (unlock) {
        return gpio_set_level(GPIO_NUM_21, 1);
    } else {
        return gpio_set_level(GPIO_NUM_21, 0);
    }
}
```

### C Coding & API Conventions
- **SPDX License Header:** Start every `.c` and `.h` file with:
  ```c
  /*
   * SPDX-FileCopyrightText: 2026 Bojaqu
   *
   * SPDX-License-Identifier: Apache-2.0
   */
  ```
- **Headers:** Use `#pragma once` and `#ifdef __cplusplus` / `extern "C"` guards. Document public types, constants, and APIs with Doxygen (`/** @brief ... @param[in/out] ... @return ... */`).
- **Naming & Visibility:**
  - Namespace public symbols per module (`bjq_<module>_...`, types ending in `_t`, opaque handles ending in `_handle_t`).
  - Mark file-local functions and state `static`; prefix static variables with `s_` (e.g., `static bool s_locked;`).
  - Define `static const char *TAG = "<module>";` for `ESP_LOG*`.
- **Error Handling:** Return `esp_err_t` from fallible functions; validate pointer/range arguments (`ESP_ERR_INVALID_ARG`) and propagate errors cleanly. Reserve `ESP_ERROR_CHECK()` for unrecoverable startup invariants in `app_main`.
- **ESP-IDF v6.1 Drivers Only:** Use modern modular drivers (`driver/i2c_master.h`, `driver/spi_master.h`, `driver/i2s_std.h`, `driver/gpio.h`). Never use legacy `driver/i2c.h` or deprecated `espressif/es8311`.
- **Secret Hygiene:** Zeroize cryptographic keys, nonces, and credential buffers via `mbedtls_platform_zeroize()`. Never log secrets, keys, nonces, or raw credential material at any log level.

## 5. Repository Structure & Architecture

### Current Layout
```text
bojaqu-door-lock/
├── CMakeLists.txt             # Top-level ESP-IDF project (MINIMAL_BUILD ON)
├── CMakePresets.json          # dev and prod configure/build presets
├── partitions.csv             # Custom 16 MB dual-OTA (2x 5 MB) partition table
├── sdkconfig.defaults         # Shared ESP32-S3, 16 MB flash, 8 MB Octal PSRAM base
├── configs/
│   ├── dev/sdkconfig          # Development overlay (bench & Wokwi)
│   └── prod/sdkconfig         # Production hardened overlay (SBv2, Flash/NVS Enc)
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml      # Managed dependencies (currently bjq_mfrc522/v0.2.0)
│   └── main.c                 # Entry point (currently MFRC522 SPI2 baseline)
├── docs/                      # Product specs, hardware, firmware, ADRs, component requests
├── test/                      # Product unit & logic tests (planned)
├── tools/                     # Helper scripts (planned)
├── diagram.json & wokwi.toml  # Wokwi ESP32-S3 + MFRC522 simulation
├── .clang-format              # ESP-IDF 4-space K&R C formatting rules
├── .clangd                    # clangd compile-flag filter
└── AGENTS.md                  # AI coding agent instructions
```

### Layered Architecture (ADR-0001)
Planned local modules under `components/` keep safety-critical lock logic decoupled from transports and drivers:
- **Adapters (`bjq_comms_mgr`):** MQTTS (`espressif/mqtt`), NimBLE GATT + Wi-Fi provisioning (`espressif/network_provisioning`), WebRTC (`espressif/esp_peer`), extensible to Home Assistant / RainMaker / Matter.
- **Typed Internal Bus:** `esp_event` + FreeRTOS queues (`lock_cmd_t` / `lock_event_t`).
- **Domain Services:** `bjq_lock_core` (Lock FSM, auto-relock, safety interlocks), `bjq_access_mgr` (DESFire policy, NVS store, lockout), `bjq_media_svc` (OV5640 snapshots/MJPEG, ES8311 audio prompts/intercom), `bjq_ota_svc` (manifest check, `esp_https_ota`, safety gate, rollback self-test).
- **Board HAL (`bjq_lock_hal`):** Owns `SPI2_HOST`, shared `I2C_NUM_0`, `I2S_NUM_0`, `LCD_CAM`, `GPIO21` strike, and PCF8575 routing.

### Dependency Sourcing Policy (Strict Priority — ADR-0001)
1. **Built-in ESP-IDF `v6.1.0` components** (`bt`/NimBLE, `esp_https_ota`, `app_update`, `nvs_flash`, `spiffs`, `mbedtls`, `esp_driver_*`).
2. **Official Espressif Registry (`espressif/*`)**: `esp32-camera`, `esp_codec_dev`, `esp_audio_codec`, `esp_peer`, `mqtt`, `network_provisioning`.
3. **Bojaqu Shared Components (`blankryan/bjq-esp-component`)**: `bjq_mfrc522`, `bjq_pcf8575` (`REQ-001`), `bjq_desfire` (`REQ-003`).
- **Do not** introduce third-party community components (`abobija/rc522`, `esp-idf-lib/pcf8575`, `joltwallet/littlefs`, etc.) or deprecated `espressif/es8311`.

## 6. Hardware, Runtime & Security Guardrails

### Pinout & Electrical Constraints (`docs/02-hardware/pin-mapping.md`)
- **Reserved — DO NOT USE:** `GPIO26`–`32` (internal flash), `GPIO33`–`37` (**8 MB Octal PSRAM**), `GPIO19`–`20` (USB-JTAG), `GPIO43`–`44` (UART0), `GPIO0`, `3`, `45`, `46` (strapping).
- **Lock Actuator (`GPIO21`):** Dedicated boot-safe native GPIO with 10 kΩ external pull-down (`HIGH` = unlock). **Never** route lock actuation through PCF8575. Enforce a 15 s max solenoid energization watchdog; indoor exit button (`PCF8575 P12`) always overrides `LOCKOUT`.
- **Shared Bus Ownership:** `board_hal` initializes `I2C_NUM_0` (`GPIO1` SDA, `GPIO7` SCL, 400 kHz: ES8311 `0x18`, PCF8575 `0x20`, OV5640 `0x3C`) and `SPI2_HOST` (`GPIO2/4/5/6`) **once**. Child drivers attach as devices and never delete the bus. Configure `esp32-camera` with `sccb_i2c_port = 0`, `pin_sccb_sda = -1` so it shares `I2C_NUM_0` without installing legacy I2C.
- **PCF8575 Invariant:** Port 1 (`P10`–`P17`, `input_mask = 0xFF00`) must **always** be written `1` on every output update; Port 0 (`P00`–`P07`) defaults HIGH (`0x00FF`) keeping `AUDIO_PA_EN_N` muted, `CAM_PWDN` powered down, and LEDs OFF. The `GPIO38` (`IOEXP_INT`) ISR must only notify a task—never execute I2C in ISR context.

### Dual-Core & Memory Discipline (`docs/03-firmware/task-model.md`, `memory-management.md`)
- **Core 0:** Wi-Fi/LwIP/NimBLE, `mqtt_client`, on-demand `webrtc_peer`, `http_upload`/`ota_task`.
- **Core 1:** `lock_core_task` (prio 12, highest app priority), `audio_io_task` (11), `rfid_task` (10), `cam_capture_task` (7), `sys_health_task` (3).
- **RAM Placement:** Keep SPI2/I2S0 DMA buffers and lock/RFID control state in internal SRAM (`MALLOC_CAP_DMA`), preserving `>= 40 KB` minimum free internal SRAM. Offload mbedTLS (`CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y`), OV5640 frames (`CAMERA_FB_IN_PSRAM`), and WebRTC buffers to 8 MB Octal PSRAM. Pause active WebRTC sessions during OTA flash writes.

### Partitions, OTA & Security (`partitions.csv`, `ADR-0005`–`0007`)
- **Partitions (`0x10000` shared offset):** Dual 5 MB OTA slots (`ota_0`, `ota_1`), `otadata`, `phy_init`, `nvs_keys` (`encrypted`), `nvs`, `fctry`, `coredump`, `assets`, `storage`. **Never** add the `encrypted` flag to `nvs` or `fctry` in `partitions.csv` (NVS encryption uses software AES-XTS keyed from `nvs_keys`).
- **RFID Policy:** Production requires MIFARE DESFire EV2/EV3 AES-128 mutual auth (`bjq_desfire`). `CONFIG_BJQ_RFID_ALLOW_UID_FALLBACK` is strictly `dev`-only and must be hard-disabled in `prod`.
- **Remote Unlock Validation:** Verify UTC `exp` (via SNTP), `request_id` deduplication, and cryptographic signature before energizing `GPIO21`.
- **OTA Interlocks:** Never block local RFID during download. Gate reboot until door is **closed**, **locked**, no active call, and 10 s RFID quiet. Pass post-boot self-test within 60 s before `esp_ota_mark_app_valid_cancel_rollback()`. Increment `CONFIG_BOOTLOADER_APP_SEC_VER` **only** for security advisories (finite eFuse bits).

## 7. Testing, Git Hygiene & Documentation

- **Test Pyramid (`docs/06-testing/test-plan.md`):**
  1. Shared driver/crypto Unity tests in `bjq-esp-component` (including NXP AN10922/AN12196 vectors for `bjq_desfire`).
  2. Product unit tests in `test/` (Lock FSM, 15 s solenoid cap, replay rejection, BLE nonce expiry, OTA safety gate).
  3. Wokwi simulation smoke test on `dev` preset (`VersionReg=0x92`, card UID read).
  4. Rev A hardware validation (`docs/06-testing/hardware-validation.md`) and E2E matrix (`docs/06-testing/integration-testing.md`).
- **Never Commit:** `build-outputs/`, `managed_components/`, root `/sdkconfig`, `compile_commands.json`, `*.pem`, `*.key`, or `secure_boot_signing_key*`.
- **Commits & Docs:** Use Conventional Commits (`feat(...):`, `fix(...):`, `docs:`). Record major design changes as ADRs in `docs/adr/` and keep `docs/` synchronized with code and config updates.

### Key Documentation References
- Build & Config: `docs/05-build/environment-setup.md`, `build-instructions.md`, `configuration-reference.md`, `partition-layout.md`
- Architecture & Tasks: `docs/03-firmware/architecture.md`, `component-integration.md`, `task-model.md`, `memory-management.md`, `state-machines.md`
- Hardware & Comms: `docs/02-hardware/pin-mapping.md`, `interface-specification.md`, `docs/04-communication/protocol-workflow.md`
- Decisions: `docs/adr/0001-firmware-architecture.md` through `0007-security-hardening-and-build-presets.md`
