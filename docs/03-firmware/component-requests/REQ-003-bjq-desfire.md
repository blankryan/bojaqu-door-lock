# Component Request: `bjq_desfire` (New Security/Protocol Component)

- **Target Repository:** `https://github.com/blankryan/bjq-esp-component.git`
- **Tracking Issue:** [blankryan/bjq-esp-component#8](https://github.com/blankryan/bjq-esp-component/issues/8) (Open)
- **Proposed Path:** `security/bjq_desfire` (or `protocols/bjq_desfire` per repo layer taxonomy)
- **Proposed Release Tag:** `bjq_desfire/v0.1.0`
- **Requesting Product:** `bojaqu-door-lock`
- **Date:** 2026-11-26
- **Related ADRs:** [ADR-0001](../../adr/0001-firmware-architecture.md), [ADR-0005](../../adr/0005-rfid-cryptographic-authentication.md)

## 1. Justification
`bojaqu-door-lock` uses **MIFARE DESFire EV2/EV3** cards with **AES-128 mutual authentication (`AuthenticateEV2First`)** and per-card diversified keys in `prod` builds. No official `espressif/*` component implements MIFARE DESFire EV2/EV3 secure messaging.

Keeping DESFire cryptography in a dedicated, transceiver-agnostic component in `bjq-esp-component` allows reuse across any Bojaqu access-control product (whether backed by `bjq_mfrc522`, PN532, or PN5180) and isolates security-critical AES/CMAC/session-key derivation for independent unit testing against NXP AN12196 / AN10922 test vectors.

## 2. Scope & Compatibility
| Item | Requirement |
|---|---|
| Layer | `security` (or `protocols`) |
| ESP-IDF | `>=6.1.0,<7.0.0` |
| Target SoCs | `esp32`, `esp32s3`, `esp32c3`, `esp32c6` |
| License | `Apache-2.0` (`SPDX-FileCopyrightText: 2026 Bojaqu`) |
| Public IDF Deps | `esp_common` |
| Private IDF Deps | `mbedtls`, `log`, `esp_hw_support` (`esp_fill_random`) |
| Intra-Repo Deps | **None** (uses injected ISO-DEP APDU transceive callback; no hard coupling to `bjq_mfrc522`) |

### Non-Goals
- Does **not** touch SPI, GPIO, or MFRC522 registers.
- Does **not** own NVS storage, user lists, or lock actuation decisions.
- Does **not** implement legacy DES/3DES/Crypto1 authentication (AES-128 EV2/EV3 only).

## 3. Functional & Security Requirements
1. **Injected APDU Transport:**
   - Caller passes an ISO-DEP APDU transceive function pointer (`bjq_desfire_apdu_transceive_fn_t`) wrapping `bjq_mfrc522_isodep_transceive()` (or a mock APDU responder in host/Unity tests).
2. **Per-Card Key Diversification (NXP AN10922 — AES-128 CMAC):**
   - Implement `bjq_desfire_diversify_aes128_key(master_key, div_input, div_input_len, out_card_key)` using AES-128-CMAC (`0x01 || UID || AID || SystemIdentifier`) so a compromised card never compromises other cards or the master key.
3. **Core DESFire EV2/EV3 Commands (Native Wrapped ISO 7816-4 Framing):**
   - `GetVersion` (`0x60` / `0xAF`) to verify genuine DESFire EV2/EV3 silicon metadata.
   - `SelectApplication` (`0x5A`, 3-byte AID).
   - `AuthenticateEV2First` (`0x71`, AES-128):
     - Decrypts `RndB` using AES-128-CBC (zero IV), generates cryptographically secure `RndA` via hardware TRNG (`esp_fill_random`), sends `E(K, RndA || RndB')`, verifies `TI || RndA' || PDcap2 || PCDcap2` using constant-time comparison (`mbedtls_ct_memcmp` / constant-time loop).
     - Derives `SesAuthENCKey` and `SesAuthMACKey` per NXP AN12196 (AES-128-CMAC SV1/SV2) and initializes `CmdCtr = 0`.
   - `ReadData` (`0xAD` / `0xBD`) in **CommMode.MAC** and **CommMode.Full** (AES-128-CBC encrypted + 8-byte truncated AES-128-CMAC verified, incrementing `CmdCtr`).
   - Provisioning helpers (used during Android-assisted card enrollment): `CreateApplication`, `CreateStdDataFile`, `ChangeKey` (AES-128), `WriteData` (`CommMode.Full`).
4. **Secret Hygiene:**
   - Zeroize all temporary plaintext buffers, nonces (`RndA`/`RndB`), and session keys (`SesAuthENCKey`, `SesAuthMACKey`) on deinit, auth failure, or session end using `mbedtls_platform_zeroize()`.
   - Never log raw keys, nonces, or decrypted credential bytes even at `ESP_LOG_DEBUG`.

## 4. Proposed Public C API (`include/bjq_desfire.h`)
```c
#define BJQ_DESFIRE_AES128_KEY_LEN 16U
#define BJQ_DESFIRE_AID_LEN        3U

typedef esp_err_t (*bjq_desfire_apdu_transceive_fn_t)(
    void *transport_ctx,
    const uint8_t *tx_apdu,
    size_t tx_len,
    uint8_t *rx_apdu,
    size_t rx_buf_size,
    size_t *out_rx_len);

typedef enum {
    BJQ_DESFIRE_COMM_PLAIN = 0,
    BJQ_DESFIRE_COMM_MAC   = 1,
    BJQ_DESFIRE_COMM_FULL  = 3,
} bjq_desfire_comm_mode_t;

typedef struct bjq_desfire_ctx *bjq_desfire_handle_t;

typedef struct {
    bjq_desfire_apdu_transceive_fn_t transceive;
    void *transport_ctx;
} bjq_desfire_config_t;

esp_err_t bjq_desfire_init(const bjq_desfire_config_t *config, bjq_desfire_handle_t *out_handle);
esp_err_t bjq_desfire_deinit(bjq_desfire_handle_t handle);

esp_err_t bjq_desfire_diversify_aes128_key(const uint8_t master_key[BJQ_DESFIRE_AES128_KEY_LEN],
                                           const uint8_t *div_input,
                                           size_t div_input_len,
                                           uint8_t out_key[BJQ_DESFIRE_AES128_KEY_LEN]);

esp_err_t bjq_desfire_select_application(bjq_desfire_handle_t handle,
                                         const uint8_t aid[BJQ_DESFIRE_AID_LEN]);

esp_err_t bjq_desfire_auth_ev2_first(bjq_desfire_handle_t handle,
                                     uint8_t key_no,
                                     const uint8_t aes_key[BJQ_DESFIRE_AES128_KEY_LEN]);

esp_err_t bjq_desfire_read_data(bjq_desfire_handle_t handle,
                                uint8_t file_id,
                                uint32_t offset,
                                size_t length,
                                bjq_desfire_comm_mode_t comm_mode,
                                uint8_t *out_data,
                                size_t out_buf_size);

esp_err_t bjq_desfire_write_data(bjq_desfire_handle_t handle,
                                 uint8_t file_id,
                                 uint32_t offset,
                                 const uint8_t *data,
                                 size_t length,
                                 bjq_desfire_comm_mode_t comm_mode);

esp_err_t bjq_desfire_clear_session(bjq_desfire_handle_t handle);
```

## 5. Acceptance Criteria & Verification
- [ ] Unity test suite verifies **official NXP AN10922 AES-128 key diversification test vectors** and **NXP AN12196 `AuthenticateEV2First` + `CommMode.Full` session key / CMAC / IV test vectors** using a mock APDU responder (no physical hardware needed in CI!).
- [ ] Negative security tests verify tampered CMAC rejection, mismatched `RndA'` rejection, counter desync rejection, and session zeroization on error.
- [ ] End-to-end bench verification on `esp32s3` paired with `bjq_mfrc522 v0.3.0` and a physical MIFARE DESFire EV2/EV3 card.
- [ ] Tagged as `bjq_desfire/v0.1.0` for `main/idf_component.yml` consumption.
