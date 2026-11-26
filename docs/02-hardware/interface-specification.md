# Hardware Interface Specification

## 1. Shared I2C Bus (`I2C_NUM_0`)
- **Pins:** `GPIO1` (`SDA`), `GPIO7` (`SCL`), 3.3 V logic, external 4.7 kΩ pull-ups.
- **Clock:** `400 kHz` (Fast Mode; supported by OV5640 SCCB, ES8311, and PCF8575).
- **Address Map (Zero Collisions):**

| Device | 7-Bit Address | Pin Strapping | Driver |
|---|---|---|---|
| **ES8311 Audio Codec** | `0x18` | `CE` = GND (`0x19` if `CE` = VDD) | `espressif/esp_codec_dev` |
| **PCF8575 GPIO Expander** | `0x20` | `A2=0, A1=0, A0=0` | `bjq_pcf8575` (`REQ-001`) |
| **OV5640 Camera SCCB** | `0x3C` | Fixed (`0x78` write / `0x79` read 8-bit) | `espressif/esp32-camera` |

- **Bus Sharing Requirement:** `board_hal` creates a single `i2c_master_bus_handle_t` on `I2C_NUM_0` (`driver/i2c_master.h`). All three devices attach to this bus; `esp32-camera` must be configured to reuse the initialized `I2C_NUM_0` port (`sccb_i2c_port = 0`, `pin_sccb_sda = -1`) without installing the legacy I2C driver.

## 2. RFID SPI Bus (`SPI2_HOST`)
- **Device:** NXP MFRC522 (`bjq_mfrc522` v0.3.0).
- **Mode:** SPI Mode 0 (`CPOL=0, CPHA=0`), MSB first.
- **Clock:** `4 MHz` default (`CONFIG_BJQ_MFRC522_DEFAULT_SPI_CLOCK_HZ`), up to `10 MHz` max per MFRC522 datasheet.
- **DMA:** `SPI_DMA_CH_AUTO`.

## 3. Camera DVP Parallel Interface (`LCD_CAM`)
- **Device:** OmniVision OV5640 (`espressif/esp32-camera`).
- **Format:** On-sensor hardware **JPEG** (`PIXFORMAT_JPEG`), `FRAMESIZE_VGA` (640x480) default live stream, `FRAMESIZE_SVGA` (800x600) or `FRAMESIZE_XGA` (1024x768) for event snapshots.
- **Clock:** `XCLK = 20 MHz` (tunable to 16 MHz if EMI affects Wi-Fi/RFID).
- **Frame Buffers:** 2 buffers allocated in 8 MB Octal PSRAM (`CAMERA_FB_IN_PSRAM`, `CAMERA_GRAB_LATEST`).

## 4. Audio I2S Full-Duplex Interface (`I2S_NUM_0`)
- **Device:** ES8311 Mono Codec + NS4150B Class-D Amplifier (`espressif/esp_codec_dev`).
- **Mode:** I2S Standard (Philips), ESP32-S3 as Master, ES8311 as Slave, Full-Duplex (TX + RX).
- **Sample Format:** `16 kHz`, `16-bit signed PCM`, Mono (`MCLK = 256 * 16 kHz = 4.096 MHz`).
- **PA Control:** `NS4150B CTRL` driven via PCF8575 `P00` (`AUDIO_PA_EN_N`) through an inverting transistor/P-MOSFET stage; unmuted only during active intercom or local prompt playback to eliminate idle hiss and power draw.

## 5. Lock Actuator Stage (`GPIO21`)
- **Driver:** Logic-level N-channel MOSFET (e.g., IRLZ44N / AO3400A with gate 100 Ω series + 10 kΩ pull-down to GND) or opto-isolated relay module switching the 12 V fail-secure electric strike.
- **Protection:** Fast Schottky/TVS flyback diode (e.g., 1N5819 / SMBJ15A) directly across solenoid terminals to clamp inductive kickback.
- **Duty Cycle Protection:** Firmware enforces a hard maximum continuous energization cap (default 15 s) to protect intermittent-duty solenoids from overheating.
