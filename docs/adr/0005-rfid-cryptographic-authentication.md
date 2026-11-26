# ADR-0005: Cryptographic RFID Authentication (MIFARE DESFire EV2/EV3) and Credential Storage

- Status: Accepted
- Date: 2026-11-26
- Scope: Product
- Owners: Bojaqu Security & Firmware Engineering

## Context
Users unlock and lock the door using 13.56 MHz RFID cards read by an NXP MFRC522 transceiver (`bjq_mfrc522`). Plain ISO/IEC 14443-3A UID whitelisting is trivially defeated with inexpensive magic-UID clone cards, and MIFARE Classic Crypto1 keys can be recovered in seconds with commodity hardware.

Meanwhile, `bjq_mfrc522 v0.2.0` currently provides ISO/IEC 14443-3A UID/SAK reading and register I/O, and the repository uses Wokwi (`wokwi.toml` / `diagram.json`) during development where simulated MFRC522 cards only expose UID/MIFARE Classic behavior.

## Decision
1. **Production Card Standard — MIFARE DESFire EV2/EV3 (AES-128 Mutual Auth):**
   - In `prod`, RFID access requires **ISO/IEC 14443-4 (ISO-DEP)** selection (`RATS`/`ATS`) followed by **MIFARE DESFire EV2/EV3 `AuthenticateEV2First` (AES-128 mutual challenge-response)** using **per-card diversified AES-128 keys** (NIST SP 800-108 CMAC KDF derived from a device/site master key stored in encrypted NVS + card UID + AID) and reading a CMAC-protected credential file (card ID, key version, expiry, access flags).
   - Cards that fail ISO-DEP selection or AES-128 mutual authentication are unconditionally rejected in `prod`.
2. **Component Split in `bjq-esp-component`:**
   - **Request `REQ-002` (`bjq_mfrc522 v0.3.0`):** Extend `drivers/bjq_mfrc522` with generic ISO/IEC 14443-4 ISO-DEP framing (`RATS`, `ATS` parsing, optional `PPS`, I-Block/R-Block/S-Block WTX handling, and APDU transceive) plus an optional reset callback hook so MFRC522 hardware reset can be driven through `bjq_pcf8575`.
   - **Request `REQ-003` (`bjq_desfire v0.1.0`):** Create a hardware-agnostic DESFire EV2/EV3 AES-128 mutual-auth & file-access component in `bjq-esp-component` built on ESP-IDF `mbedtls`/PSA crypto and an injected APDU transceive callback.
3. **Offline-First Local Credential Store & Revocation:**
   - Authorized card records (card ID, salted UID hash, key version, schedule/validity window, enabled/revoked state) and the device master diversification key are stored locally in **encrypted NVS** (`nvs` + `nvs_keys`) so RFID lock/unlock works reliably with zero internet latency or during Wi-Fi outages.
   - Card enrollment, metadata updates, and revocations sync bidirectionally via MQTTS (`cmd/rfid_sync`), with Android-assisted enrollment mode (lock enters timed 30-second enrollment window upon authenticated command).
4. **Brute-Force Protection & Audit:**
   - Track consecutive invalid card taps; after 5 failed attempts within 60 s, engage exponential cooldown (30 s -> 120 s), emit a high-priority MQTTS security alert, and trigger an OV5640 snapshot upload.
5. **Development Preset Fallback (`dev` only):**
   - Gate a compile-time Kconfig option (`CONFIG_BJQ_RFID_ALLOW_UID_FALLBACK`) so **`dev` preset** supports UID-only / Wokwi bench simulation while **`prod` preset** hard-disables UID fallback at compile time (`# CONFIG_BJQ_RFID_ALLOW_UID_FALLBACK is not set`).

## Alternatives considered
- **UID-only whitelist in v1:** Rejected because a physical door lock is a high-trust security boundary where UID cloning is an unacceptable production vulnerability.
- **MIFARE Classic 1K/4K with Crypto1 (`MFAuthent`):** Rejected for `prod` because Crypto1 has well-known practical key-recovery attacks (`nested` / `darkside` / `hardnested`).
- **NTAG21x with 32-bit `PWD_AUTH`:** Rejected because the static 32-bit password is sent in cleartext over RF and lacks mutual challenge-response authentication.
- **Online cloud-only card validation:** Rejected because a home door lock must never lock out residents when Wi-Fi or cloud connectivity is down.

## Consequences
- **Benefits:** Strong anti-cloning security using standard AES-128 mutual authentication; 100% offline unlock reliability; clean separation between low-level MFRC522 RF framing (`bjq_mfrc522`) and high-level DESFire cryptography (`bjq_desfire`); Wokwi and bench development remain unblocked via `dev` preset.
- **Trade-offs / Constraints:** Higher card unit cost (DESFire EV2/EV3 vs Classic); card provisioning requires an enrollment step (either lock-side enrollment with master key or backend/Android HCE provisioning flow); v1 production RFID depends on delivering `REQ-002` and `REQ-003` in `bjq-esp-component`.
- **Follow-up work:** Deliver `REQ-002` (`bjq_mfrc522 v0.3.0`) and `REQ-003` (`bjq_desfire`); define card enrollment UX flow with the Android team; add compile-time `#error` guard in `prod` if `CONFIG_BJQ_RFID_ALLOW_UID_FALLBACK` is enabled.

## References
- [REQ-002: bjq_mfrc522 v0.3.0 ISO-DEP](../03-firmware/component-requests/REQ-002-bjq-mfrc522-v0.3.0-isodep.md) ([bjq-esp-component#7](https://github.com/blankryan/bjq-esp-component/issues/7))
- [REQ-003: bjq_desfire](../03-firmware/component-requests/REQ-003-bjq-desfire.md) ([bjq-esp-component#8](https://github.com/blankryan/bjq-esp-component/issues/8))
- [State Machines](../03-firmware/state-machines.md)
- [ADR-0001](0001-firmware-architecture.md), [ADR-0007](0007-security-hardening-and-build-presets.md)
