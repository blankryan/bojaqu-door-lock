# Provisioning (Factory & End-User)

## 1. Factory Provisioning (Manufacturing Line — Before Final Lockdown)
Each production unit receives a unique identity flashed into the `fctry` NVS partition (44 KB at `0x035000`, encrypted via `nvs_keys`):
1. Generate unique `device_id` (UUID/serial) and per-device **X.509 client keypair + CSR** signed by the Bojaqu Device CA (certificate + private key stored in `fctry` NVS; future Rev B may migrate private key to ESP32-S3 Digital Signature peripheral).
2. Generate unique high-entropy **BLE Security 2 SRP6a salt + verifier** (`PoP`) and print the pairing QR code label (`BJQ:<device_id>:<pop>`) placed inside the indoor battery/service cover.
3. Generate and store the encrypted cloud command signature verification public key (Ed25519) in `fctry`.
4. Generate per-unit **DESFire master diversification key** (or bind on cloud claim) stored in encrypted `nvs`.
5. Register `device_id` and X.509 certificate fingerprint in the Bojaqu Cloud device registry.

## 2. End-User Provisioning (Android App via BLE Security 2)
1. User powers on lock; if unprovisioned, status LED blinks blue (`PCF8575 P06`) and NimBLE advertises `espressif/network_provisioning` service (`BJQ-LOCK-XXXX`).
2. User scans QR code in Bojaqu Android app -> establishes BLE Security 2 (SRP6a) encrypted channel.
3. Android sends home Wi-Fi credentials + short-lived user claim token.
4. Lock joins Wi-Fi, connects MQTTS with `fctry` mTLS cert, completes cloud ownership bind, receives its user-scoped **BLE Offline Unlock Key** (saved to encrypted `nvs`), and exits provisioning mode.

## 3. Factory Reset
Holding the indoor physical reset sequence (e.g., indoor exit button + tamper/service button for 10 s, or authenticated owner unbind via Android) erases user `nvs` (Wi-Fi, RFID cards, offline BLE key) while preserving factory `fctry` identity so the lock can be re-provisioned cleanly.
