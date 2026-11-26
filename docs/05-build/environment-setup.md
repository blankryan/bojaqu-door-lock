# Environment Setup

## Prerequisites
- **ESP-IDF:** `v6.1.0` (`>=6.1.0,<7.0.0`) with `esp32s3` toolchain installed (or use the included [.devcontainer](../../.devcontainer/devcontainer.json)).
- **CMake:** `>= 3.22` (supports `CMakePresets.json` schema version 3) + Ninja.
- **Git & GitHub Access:** Required by IDF Component Manager to pull `https://github.com/blankryan/bjq-esp-component.git`.
- **Optional Simulator:** Wokwi CLI / VS Code Wokwi extension using [wokwi.toml](../../wokwi.toml) and [diagram.json](../../diagram.json) for `dev` preset MFRC522 testing.

## Quick Verification
```bash
# Inside ESP-IDF v6.1.0 environment
idf.py --version
cmake --version
cmake --list-presets
```
Expected presets:
- `dev` — Development configuration (`build-outputs/dev`)
- `prod` — Production configuration (`build-outputs/prod`)
