# Build Instructions (`CMakePresets.json`)

This project manages `dev` and `prod` configurations through [CMakePresets.json](../../CMakePresets.json). Both presets layer an environment overlay on top of the shared [sdkconfig.defaults](../../sdkconfig.defaults).

## 1. Development Build (`dev` Preset)
Safe for bench ESP32-S3 devkits and Wokwi simulation (Secure Boot & Flash Encryption are OFF; DEBUG logging is ON).

```bash
# Configure & build via CMake presets
cmake --preset dev
cmake --build --preset dev

# Or using idf.py directly into the dev build directory
idf.py -B build-outputs/dev -D SDKCONFIG=build-outputs/dev/sdkconfig \
  -D "SDKCONFIG_DEFAULTS=sdkconfig.defaults;configs/dev/sdkconfig" build
```

Flash & monitor on a bench board:
```bash
idf.py -B build-outputs/dev -p <PORT> flash monitor
```

## 2. Production Build (`prod` Preset)
> [!CAUTION]
> **`prod` enables Secure Boot v2, Flash Encryption (Release mode), Anti-Rollback, and disables ROM Download Mode.** Flashing a `prod` image onto a physical ESP32-S3 permanently burns hardware eFuses and locks the chip! Never flash `prod` onto a reusable development board.

### Signing Key Requirement (Out-of-Band)
Do **not** commit Secure Boot signing keys to Git. Provide the RSA-3072 signing key path locally or from CI secrets before building `prod`:

```bash
# Example (CI / Release workstation only)
export BJQ_SIGNING_KEY="/secure/path/bojaqu_door_lock_sbv2.pem"

cmake --preset prod -DCONFIG_SECURE_BOOT_SIGNING_KEY="$BJQ_SIGNING_KEY"
cmake --build --preset prod
```
Or configure `CONFIG_SECURE_BOOT_SIGNING_KEY` via an untracked local override / CI overlay when building signed release images (see [Production Release](../../docs/07-deployment/production-release.md)).

## 3. Clean Rebuild (When Changing `sdkconfig` Overlays)
ESP-IDF caches resolved options in `build-outputs/<preset>/sdkconfig`. After editing `sdkconfig.defaults` or `configs/<preset>/sdkconfig`, delete the generated `sdkconfig` before re-configuring:
```bash
rm -f build-outputs/dev/sdkconfig
cmake --preset dev
cmake --build --preset dev
```
