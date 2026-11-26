# Component Request: `bjq_pcf8575` (New Driver Component)

- **Target Repository:** `https://github.com/blankryan/bjq-esp-component.git`
- **Tracking Issue:** [blankryan/bjq-esp-component#6](https://github.com/blankryan/bjq-esp-component/issues/6) (Open)
- **Proposed Path:** `drivers/bjq_pcf8575`
- **Proposed Release Tag:** `bjq_pcf8575/v0.1.0`
- **Requesting Product:** `bojaqu-door-lock`
- **Date:** 2026-11-26
- **Related ADRs:** [ADR-0001](../../adr/0001-firmware-architecture.md), [ADR-0002](../../adr/0002-hardware-platform-and-gpio-budget.md)

## 1. Justification (Why Not ESP Component Registry?)
Checked the official ESP Component Registry on 2026-11-26:
- No official `espressif/*` component exists for the TI/NXP **PCF8575** 16-bit I2C/SMBus I/O expander.
- Only third-party `esp-idf-lib/pcf8575` exists, which does not match Bojaqu's official-first + `bjq-esp-component` ownership and ESP-IDF v6.1 modular driver (`esp_driver_i2c` / `driver/i2c_master.h`) standards.
- `bojaqu-door-lock` uses PCF8575 on the shared `I2C0` bus (alongside OV5640 SCCB and ES8311) to offload 16 slow control/input signals while preserving USB-JTAG, UART0, Octal PSRAM (`GPIO33`–`37`), and a boot-safe native lock GPIO (`GPIO21`).

## 2. Scope & Compatibility
| Item | Requirement |
|---|---|
| Layer | `drivers` |
| ESP-IDF | `>=6.1.0,<7.0.0` |
| Target SoCs | `esp32`, `esp32s3`, `esp32c3`, `esp32c6` |
| License | `Apache-2.0` (`SPDX-FileCopyrightText: 2026 Bojaqu`) |
| Public IDF Deps | `esp_driver_i2c`, `esp_driver_gpio` |
| Private IDF Deps | `log`, `freertos`, `esp_timer` |

### Non-Goals
- Does **not** initialize or delete the shared `i2c_master_bus_handle_t` (caller owns the I2C bus, matching `bjq_mfrc522` bus ownership).
- Does **not** implement product-specific button state machines or lock actuator logic.

## 3. Functional & Hardware Requirements
1. **ESP-IDF v6.1 New I2C Master API:** Use `driver/i2c_master.h` (`i2c_master_bus_add_device`, `i2c_master_transmit`, `i2c_master_receive`); do **not** use legacy `driver/i2c.h`.
2. **I2C Address Support:** Support full 7-bit PCF8575 address range `0x20`–`0x27` (A2:A1:A0), Standard (100 kHz) and Fast Mode (400 kHz).
3. **16-Bit Little-Endian Port Mapping:**
   - Bit `0..7` -> Port 0 (`P00`–`P07`, first byte on wire).
   - Bit `8..15` -> Port 1 (`P10`–`P17`, second byte on wire).
4. **Quasi-Bidirectional Shadow Register Protection (Critical):**
   - PCF8575 has no hardware direction register; writing `0` to an input pin latches it LOW and prevents external input reading.
   - The driver **must** maintain an internal `input_mask` (bits configured as inputs are ALWAYS forced to `1` in every 2-byte I2C write!) and an `output_latch` shadow register protected by an internal FreeRTOS mutex.
   - `bjq_pcf8575_init()` must apply `config->initial_output_mask | config->input_mask` atomically so power-on fail-safe states (e.g., `CAM_PWDN=1`, `AUDIO_PA_EN_N=1`, LEDs OFF) are preserved cleanly.
5. **Open-Drain Interrupt (`INT`) Support:**
   - Optional `int_io_num` (`GPIO_NUM_NC` if polling-only).
   - Configures falling-edge GPIO ISR that signals a caller-provided callback (ISR-safe notification or optional internal worker task / user queue) without performing I2C transactions inside ISR context.
   - Provide `bjq_pcf8575_read_port()` which also clears the PCF8575 hardware `INT` line per datasheet.
6. **Thread Safety & Bus Recovery:**
   - All public read/write/modify APIs must be thread-safe via internal mutex with configurable timeout (`CONFIG_BJQ_PCF8575_DEFAULT_TIMEOUT_MS`).
   - Support atomic masked updates (`bjq_pcf8575_write_masked()`) to prevent read-modify-write races across concurrent tasks (e.g., audio task toggling `AUDIO_PA_EN_N` while LED task updates status LEDs).

## 4. Proposed Public C API (`include/bjq_pcf8575.h`)
```c
typedef struct bjq_pcf8575_ctx *bjq_pcf8575_handle_t;
typedef void (*bjq_pcf8575_int_isr_cb_t)(bjq_pcf8575_handle_t handle, void *user_ctx);

typedef struct {
    i2c_master_bus_handle_t i2c_bus;   /**< Caller-owned initialized I2C master bus handle. */
    uint8_t dev_addr;                  /**< 7-bit I2C address (0x20..0x27). */
    uint32_t scl_speed_hz;             /**< I2C clock (0 -> CONFIG_BJQ_PCF8575_DEFAULT_I2C_FREQ_HZ). */
    uint32_t timeout_ms;               /**< Transaction timeout (0 -> CONFIG_BJQ_PCF8575_DEFAULT_TIMEOUT_MS). */
    uint16_t input_mask;               /**< Bits set to 1 are inputs (always written HIGH on output updates). */
    uint16_t initial_output_mask;      /**< Initial state for output bits (applied atomically on init). */
    gpio_num_t int_io_num;             /**< Optional active-low INT GPIO, or GPIO_NUM_NC if unused. */
    bjq_pcf8575_int_isr_cb_t int_cb;   /**< Optional ISR-context callback when INT falls. */
    void *int_user_ctx;                /**< User pointer passed to int_cb. */
} bjq_pcf8575_config_t;

esp_err_t bjq_pcf8575_init(const bjq_pcf8575_config_t *config, bjq_pcf8575_handle_t *out_handle);
esp_err_t bjq_pcf8575_deinit(bjq_pcf8575_handle_t handle);
esp_err_t bjq_pcf8575_probe(bjq_pcf8575_handle_t handle);
esp_err_t bjq_pcf8575_read_port(bjq_pcf8575_handle_t handle, uint16_t *out_value);
esp_err_t bjq_pcf8575_read_pin(bjq_pcf8575_handle_t handle, uint8_t pin, bool *out_level);
esp_err_t bjq_pcf8575_write_port(bjq_pcf8575_handle_t handle, uint16_t value);
esp_err_t bjq_pcf8575_write_masked(bjq_pcf8575_handle_t handle, uint16_t mask, uint16_t value);
esp_err_t bjq_pcf8575_write_pin(bjq_pcf8575_handle_t handle, uint8_t pin, bool level);
esp_err_t bjq_pcf8575_get_shadow(bjq_pcf8575_handle_t handle, uint16_t *out_latch, uint16_t *out_input_mask);
```

## 5. Proposed Kconfig Options (`Kconfig`)
- `CONFIG_BJQ_PCF8575_DEFAULT_I2C_FREQ_HZ` (int, default `400000`, range `10000`..`400000`)
- `CONFIG_BJQ_PCF8575_DEFAULT_TIMEOUT_MS` (int, default `100`, range `1`..`5000`)

## 6. Acceptance Criteria & Deliverables in `bjq-esp-component`
- [ ] Follows `bjq_mfrc522` layout: `include/`, `src/`, `Kconfig`, `CMakeLists.txt`, `idf_component.yml`, `README.md`, `CHANGELOG.md`, `docs/` (`architecture.md`, `configuration.md`, `concurrency-and-resources.md`, `troubleshooting.md`, `bench-testing.md`).
- [ ] `examples/basic/` builds in CI on `esp32`, `esp32s3`, `esp32c3`, `esp32c6`.
- [ ] `test_apps/` Unity suite verifies argument bounds (`dev_addr`, `pin > 15`, NULL checks), `input_mask` invariant protection during `write_port` / `write_masked`, and ISR attachment/detachment.
- [ ] Bench test verifies PCF8575 @ `0x20`, 400 kHz shared I2C coexistence, output toggling without disturbing input bits, and `INT` edge notification on input change.
- [ ] Tagged as `bjq_pcf8575/v0.1.0` for `main/idf_component.yml` consumption.
