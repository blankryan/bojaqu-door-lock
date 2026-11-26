# Production Release & OTA Runbook

## 1. Versioning & Security Version Policy
- **Semantic Version (`PROJECT_VER`):** `MAJOR.MINOR.PATCH` (e.g., `1.0.0`), embedded in ESP-IDF app descriptor (`esp_app_desc_t`).
- **Anti-Rollback Security Version (`CONFIG_BOOTLOADER_APP_SEC_VER`):** Increment **ONLY** for security advisories where downgrading to an older release must be permanently blocked (ESP32-S3 has finite eFuse anti-rollback bits).

## 2. Release Pipeline (CI)
1. Tag release (`vX.Y.Z`) after `dev` and `prod` CI builds + Unity/Wokwi + hardware regression pass.
2. CI retrieves Secure Boot v2 RSA-3072 signing key from HSM/KMS (never stored in Git), runs `cmake --preset prod && cmake --build --preset prod`, and produces signed `bojaqu-door-lock-vX.Y.Z.bin`.
3. CI computes SHA-256 of the signed binary, uploads binary to immutable HTTPS CDN path, and generates a signed release manifest (`manifest.json`):
   ```json
   {
     "product": "bojaqu-door-lock",
     "hw_rev": "rev-a",
     "version": "1.0.1",
     "sec_ver": 1,
     "urgent": false,
     "min_from_version": "1.0.0",
     "url": "https://ota.bojaqu.example/door-lock/v1.0.1/bojaqu-door-lock.bin",
     "sha256": "<64-hex-sha256>",
     "size_bytes": 2418944,
     "changelog": "Fix WebRTC ICE timeout and improve DESFire tap latency.",
     "sig": "<ed25519-manifest-signature>"
   }
   ```

## 3. Staged OTA Rollout
1. **Canary (Internal Devices — 24 h):** Publish MQTTS `cmd/ota_available` to canary group; monitor `ota_complete`, `coredump` count, and rollback rate (must be **0%**).
2. **Staged Waves (10% -> 50% -> 100%):** Gradually publish availability; devices with Auto-Update ON install when door is closed & locked, while devices with User-Confirmed mode prompt homeowners in Android.
3. **Emergency Halt:** Revoke manifest / stop MQTTS broadcast immediately if rollback or crash metrics exceed threshold; existing devices automatically stay on or roll back to the previous verified OTA slot.
