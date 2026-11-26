# Flashing Guide

## 1. Development Boards (`dev` Preset)
Connect via native **USB-Serial/JTAG** (`GPIO19`/`GPIO20`) or **UART0** (`GPIO43`/`GPIO44`):
```bash
# Build & flash dev preset
cmake --preset dev
cmake --build --preset dev
idf.py -B build-outputs/dev -p <PORT> flash monitor
```
To flash the static voice prompts/chimes SPIFFS image (`assets` at `0xA50000`), use `esptool.py write_flash` or `spiffs_create_partition_image(assets ... FLASH_IN_PROJECT)`.

## 2. Wokwi Simulation (`dev` Preset)
[wokwi.toml](../../wokwi.toml) automatically points to `build-outputs/dev/flasher_args.json` and `build-outputs/dev/bojaqu-door-lock.elf`. After `cmake --build --preset dev`, start Wokwi in VS Code or via `wokwi-cli`.

## 3. Production Units (`prod` Preset)
> [!CAUTION]
> Flashing `prod` enables **Secure Boot v2**, **Flash Encryption (Release mode)**, **Anti-Rollback**, and **disables ROM Download Mode**. Subsequent firmware updates can ONLY be delivered via signed **HTTPS OTA**.

Factory first-boot flashing procedure (using pre-generated encrypted `fctry` NVS + `nvs_keys` + signed bootloader/app):
1. Flash signed bootloader (`0x0000`), partition table (`0x10000`), `otadata`, `nvs_keys`, `fctry`, `ota_0` signed app, and `assets` via factory jig.
2. Boot once on test jig and wait for first-boot Flash Encryption + Secure Boot v2 eFuse burn to finish (do **not** remove 12 V power during initial encryption pass!).
3. Verify boot logs confirm `Secure Boot v2 enabled`, `Flash Encryption Release mode`, `NVS encrypted`, and hardware self-test PASS.
