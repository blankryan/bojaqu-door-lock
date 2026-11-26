# Pin Mapping — ESP32-S3-WROOM-1-N16R8 + PCF8575 (Rev A)

> [!IMPORTANT]
> **ESP32-S3-WROOM-1-N16R8 Reserved Pins (DO NOT USE):**
> - `GPIO26`–`GPIO32`: Internal SPI Flash (not broken out)
> - `GPIO33`–`GPIO37`: Internal 8 MB Octal PSRAM (**MUST remain unconnected**)
> - `GPIO19`, `GPIO20`: Reserved for native USB-Serial/JTAG debugging
> - `GPIO43`, `GPIO44`: Reserved for UART0 console/flashing
> - `GPIO0`, `GPIO3`, `GPIO45`, `GPIO46`: Boot/JTAG/VDD_SPI strapping pins (kept free)

## 1. Native ESP32-S3 GPIO Allocation (25 of 25 Clean GPIOs)

| GPIO | Signal Name | Peripheral / Bus | Direction | Notes / Electrical |
|---|---|---|---|---|
| **GPIO1** | `I2C0_SDA` | Shared `I2C_NUM_0` | In/Out (OD) | External 4.7 kΩ pull-up to 3.3 V (OV5640 `0x3C`, ES8311 `0x18`, PCF8575 `0x20`) |
| **GPIO2** | `RFID_MISO` | `SPI2_HOST` (MFRC522) | Input | Matches existing [main.c](../../main/main.c) & [diagram.json](../../diagram.json) |
| **GPIO4** | `RFID_MOSI` | `SPI2_HOST` (MFRC522) | Output | Matches existing [main.c](../../main/main.c) & [diagram.json](../../diagram.json) |
| **GPIO5** | `RFID_SCLK` | `SPI2_HOST` (MFRC522) | Output | Matches existing [main.c](../../main/main.c) & [diagram.json](../../diagram.json) |
| **GPIO6** | `RFID_CS` | `SPI2_HOST` (MFRC522) | Output | Active-low chip select; external 10 kΩ pull-up recommended |
| **GPIO7** | `I2C0_SCL` | Shared `I2C_NUM_0` | In/Out (OD) | External 4.7 kΩ pull-up to 3.3 V (400 kHz Fast Mode) |
| **GPIO8** | `CAM_VSYNC` | `LCD_CAM` (OV5640) | Input | Frame sync |
| **GPIO9** | `CAM_HREF` | `LCD_CAM` (OV5640) | Input | Line sync |
| **GPIO10** | `CAM_D0` (Y2) | `LCD_CAM` (OV5640) | Input | Parallel data bit 0 |
| **GPIO11** | `CAM_D1` (Y3) | `LCD_CAM` (OV5640) | Input | Parallel data bit 1 |
| **GPIO12** | `CAM_D2` (Y4) | `LCD_CAM` (OV5640) | Input | Parallel data bit 2 |
| **GPIO13** | `CAM_PCLK` | `LCD_CAM` (OV5640) | Input | Pixel clock (keep trace short & away from SPI/RF) |
| **GPIO14** | `CAM_D3` (Y5) | `LCD_CAM` (OV5640) | Input | Parallel data bit 3 |
| **GPIO15** | `CAM_XCLK` | `LCD_CAM` (OV5640) | Output | 20 MHz master clock to OV5640 (series 33 Ω damping resistor) |
| **GPIO16** | `CAM_D4` (Y6) | `LCD_CAM` (OV5640) | Input | Parallel data bit 4 |
| **GPIO17** | `CAM_D5` (Y7) | `LCD_CAM` (OV5640) | Input | Parallel data bit 5 |
| **GPIO18** | `CAM_D6` (Y8) | `LCD_CAM` (OV5640) | Input | Parallel data bit 6 |
| **GPIO21** | `LOCK_ACTUATOR_EN` | Lock MOSFET / Opto-Relay | Output | **Boot-safe dedicated GPIO**; external 10 kΩ pull-down to GND (`HIGH` = unlock) |
| **GPIO38** | `IOEXP_INT` | PCF8575 `INT` | Input | Active-low falling-edge ISR; external 10 kΩ pull-up to 3.3 V |
| **GPIO39** | `I2S0_MCLK` | `I2S_NUM_0` (ES8311) | Output | 4.096 MHz (`256 * 16 kHz`); series 33 Ω resistor |
| **GPIO40** | `I2S0_SCLK` | `I2S_NUM_0` (ES8311) | Output | I2S bit clock (`BCLK`) |
| **GPIO41** | `I2S0_LRCK` | `I2S_NUM_0` (ES8311) | Output | I2S word select (`WS` / `LRCK`) |
| **GPIO42** | `I2S0_DOUT` | `I2S_NUM_0` (ES8311) | Output | Audio TX to ES8311 `DSDIN` (DAC -> NS4150B speaker) |
| **GPIO47** | `CAM_D7` (Y9) | `LCD_CAM` (OV5640) | Input | Parallel data bit 7 |
| **GPIO48** | `I2S0_DIN` | `I2S_NUM_0` (ES8311) | Input | Audio RX from ES8311 `ASDOUT` (ADC mic -> ESP32-S3) |

## 2. PCF8575 16-Bit I2C Expander Pin Allocation (Address `0x20`)

PCF8575 defaults to `0xFFFF` (all pins weakly HIGH) on power-on.
- **Output Mask (`initial_output_mask = 0x00FF`)**: Port 0 (`P00`–`P07`) starts HIGH (fail-safe idle/off).
- **Input Mask (`input_mask = 0xFF00`)**: Port 1 (`P10`–`P17`) is ALWAYS written `1` on every I2C write so weak pull-ups remain active for input sensing.

| PCF8575 Pin | Bit | Signal Name | Dir | Active Level | Power-On State (`1`) | Description |
|---|---|---|---|---|---|---|
| **P00** | 0 | `AUDIO_PA_EN_N` | Out | Active-LOW | Muted (OFF) | Drives inverter/P-MOSFET to NS4150B `CTRL` (prevents boot pop!) |
| **P01** | 1 | `CAM_PWDN` | Out | Active-HIGH | Powered Down | OV5640 power-down pin; set LOW only when camera active |
| **P02** | 2 | `CAM_RST_N` | Out | Active-LOW | Released | Pulsed LOW >= 10 ms during OV5640 init/recovery |
| **P03** | 3 | `RFID_RST_N` | Out | Active-LOW | Released | Wired via `bjq_mfrc522_set_hw_reset_cb()` for hard reset |
| **P04** | 4 | `LED_STATUS_R_N` | Out | Active-LOW | OFF | Status LED Red (current-sinking to PCF8575, <= 10 mA) |
| **P05** | 5 | `LED_STATUS_G_N` | Out | Active-LOW | OFF | Status LED Green (<= 10 mA) |
| **P06** | 6 | `LED_STATUS_B_N` | Out | Active-LOW | OFF | Status LED Blue / BLE Provisioning (<= 10 mA) |
| **P07** | 7 | `CAM_FLASH_LED_N` | Out | Active-LOW | OFF | Drives P-MOSFET for low-light camera fill LED |
| **P10** | 8 | `DOOR_REED_SW` | In | Active-LOW | Pull-up | Closed-to-GND when door is closed (100 nF RC debounce) |
| **P11** | 9 | `BTN_DOORBELL` | In | Active-LOW | Pull-up | Outdoor visitor call button to GND (ESD TVS + RC debounce) |
| **P12** | 10 | `BTN_EXIT` | In | Active-LOW | Pull-up | Indoor manual unlock request button to GND |
| **P13** | 11 | `TAMPER_SW` | In | Active-HIGH (NC) | Pull-up | NC switch held to GND when cover closed; opens on tamper |
| **P14** | 12 | `RFID_IRQ_N` | In | Active-LOW | Pull-up | Optional MFRC522 IRQ output |
| **P15**–**P17** | 13–15 | `SPARE_IN_0..2` | In | — | Pull-up | Reserved for future sensors (e.g., bolt sensor, keypad IRQ) |
