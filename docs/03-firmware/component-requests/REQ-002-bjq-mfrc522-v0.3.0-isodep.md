# Component Enhancement Request: `bjq_mfrc522` `v0.3.0` (ISO/IEC 14443-4 ISO-DEP + Reset Hook)

- **Target Repository:** `https://github.com/blankryan/bjq-esp-component.git`
- **Tracking Issue:** [blankryan/bjq-esp-component#7](https://github.com/blankryan/bjq-esp-component/issues/7) (Open)
- **Target Component Path:** `drivers/bjq_mfrc522`
- **Current Version:** `bjq_mfrc522/v0.2.0`
- **Proposed Release Tag:** `bjq_mfrc522/v0.3.0`
- **Requesting Product:** `bojaqu-door-lock`
- **Date:** 2026-11-26
- **Related ADRs:** [ADR-0001](../../adr/0001-firmware-architecture.md), [ADR-0002](../../adr/0002-hardware-platform-and-gpio-budget.md), [ADR-0005](../../adr/0005-rfid-cryptographic-authentication.md)

## 1. Justification
`bojaqu-door-lock` requires cryptographic RFID authentication (**MIFARE DESFire EV2/EV3** using AES-128 mutual authentication over ISO/IEC 14443-4 ISO-DEP) in `prod` builds to eliminate UID cloning and MIFARE Classic Crypto1 vulnerabilities (see [ADR-0005](../../adr/0005-rfid-cryptographic-authentication.md)).

Currently, `bjq_mfrc522 v0.2.0` provides ISO/IEC 14443-3A polling, multi-cascade anticollision/select (`UID` + `SAK`), `HLTA`, and register I/O, but lacks ISO/IEC 14443-4 (ISO-DEP) framing and APDU transceive. Additionally, `bojaqu-door-lock` routes `RFID_RST_N` through the `PCF8575` I2C expander (`P03`), so `bjq_mfrc522` needs an optional hardware-reset callback hook alongside `rst_io_num`.

## 2. Backward Compatibility & Layer Boundaries
- **100% backward compatible** with all `v0.2.0` APIs, structs, and Kconfig options (`existing examples/basic` and Wokwi tests continue passing unmodified).
- **Strict layer boundary:** `bjq_mfrc522` handles ONLY **ISO/IEC 14443-4 RF framing** (`RATS`, `ATS`, optional `PPS`, `I-Block` / `R-Block` / `S-Block`, block number toggling, frame chaining, WTX extension, `DESELECT`, hardware CRC_A). High-level DESFire EV2/EV3 commands and AES-128 cryptography belong in `REQ-003` (`bjq_desfire`), NOT in `bjq_mfrc522`.

## 3. Functional Requirements
1. **ISO/IEC 14443-4 Activation (`RATS` / `ATS`):**
   - Check `SAK & 0x20` (ISO 14443-4 compliance bit) or let caller explicitly invoke `bjq_mfrc522_isodep_rats()`.
   - Transmit `RATS` (`0xE0`, `FSDI | CID`) with hardware CRC_A and parse the returned `ATS` (`TL`, `T0`, `TA1`, `TB1`, `TC1`, historical bytes).
   - Derive card max frame size (`FSC`) from `T0` FSCI and frame waiting time (`FWT`) from `TB1` FWI, clamping frame chunks to the MFRC522 64-byte FIFO limit (effective max payload per RF frame <= 60 bytes including PCB/CRC_A, or using MFRC522 FIFO watermark/chaining).
2. **ISO-DEP Half-Duplex Block Protocol (`I-Block` / `R-Block` / `S-Block`):**
   - Maintain per-session ISO-DEP state (`block_num` toggling `0`/`1`, `fsc`, `fwt_ms`, `cid`, `nad`).
   - Support **TX chaining** (splitting outgoing APDUs larger than `min(FSC, 60)` bytes into chained `I-Block`s with chaining bit set and verifying `R(ACK)` between chunks).
   - Support **RX chaining** (reassembling multi-block card responses by sending `R(ACK)` until final unchained `I-Block`).
   - Support **S(WTX)** (Waiting Time Extension requests from DESFire during AES/NVM operations: reply with matching `S(WTX)` and extend deadline by `WTXM * FWT`).
   - Support **R(NAK)** retry on CRC/parity error (configurable max retries, default 3).
3. **ISO-DEP Deactivation (`S(DESELECT)`):**
   - Cleanly deselect ISO-DEP card via `S(DESELECT)` (`0xC2`) so the card returns to HALT state cleanly.
4. **Optional External Hardware Reset Callback (for PCF8575 / IO Expander):**
   - Add an optional `reset_cb` + `reset_user_ctx` (or `bjq_mfrc522_set_reset_callback()`) invoked inside `bjq_mfrc522_reset()` when `rst_io_num == GPIO_NUM_NC`, allowing products with an I2C GPIO expander (`bjq_pcf8575`) to pulse `RFID_RST_N` cleanly on init/recovery.

## 4. Proposed Public C API Additions (`include/bjq_mfrc522.h`)
```c
#define BJQ_MFRC522_ISODEP_ATS_MAX_LEN 32U

typedef struct {
    uint8_t raw[BJQ_MFRC522_ISODEP_ATS_MAX_LEN];
    size_t raw_len;
    uint16_t fsc;       /**< Max card frame size in bytes (from FSCI). */
    uint32_t fwt_ms;    /**< Frame waiting time in ms (from FWI). */
    uint8_t block_num;  /**< Current ISO-DEP block number (0 or 1). */
    uint8_t cid;        /**< Card Identifier (0..14). */
    bool active;        /**< True after successful RATS until DESELECT/reset. */
} bjq_mfrc522_isodep_t;

typedef esp_err_t (*bjq_mfrc522_hw_reset_cb_t)(bool assert_reset, void *user_ctx);

esp_err_t bjq_mfrc522_set_hw_reset_cb(bjq_mfrc522_handle_t handle,
                                      bjq_mfrc522_hw_reset_cb_t cb,
                                      void *user_ctx);

esp_err_t bjq_mfrc522_isodep_rats(bjq_mfrc522_handle_t handle,
                                  uint8_t fsdi,
                                  uint8_t cid,
                                  bjq_mfrc522_isodep_t *out_session);

esp_err_t bjq_mfrc522_isodep_transceive(bjq_mfrc522_handle_t handle,
                                        bjq_mfrc522_isodep_t *session,
                                        const uint8_t *tx_apdu,
                                        size_t tx_len,
                                        uint8_t *rx_apdu,
                                        size_t rx_buf_size,
                                        size_t *out_rx_len);

esp_err_t bjq_mfrc522_isodep_deselect(bjq_mfrc522_handle_t handle,
                                      bjq_mfrc522_isodep_t *session);
```

## 5. Proposed Kconfig Additions
- `CONFIG_BJQ_MFRC522_ISODEP_MAX_RETRIES` (int, default `3`, range `1`..`10`)
- `CONFIG_BJQ_MFRC522_ISODEP_MAX_WTX_COUNT` (int, default `10`, range `1`..`50`)

## 6. Acceptance Criteria & Verification
- [ ] All existing `v0.2.0` Unity tests and Wokwi `examples/basic` continue passing on `esp32s3`.
- [ ] New Unity tests validate `ATS` parser (FSCI/FWI decoding), I-Block PCB toggling, chaining boundary checks, WTX handling, and NULL/size validation.
- [ ] Bench test against physical **MIFARE DESFire EV2 / EV3** card verifies: `REQA` -> `UID/SAK (0x20)` -> `RATS/ATS` -> `GetVersion` (`0x60` + `0xAF` frames) -> `DESELECT`.
- [ ] `CHANGELOG.md` and `docs/architecture.md` updated; tagged as `bjq_mfrc522/v0.3.0`.
