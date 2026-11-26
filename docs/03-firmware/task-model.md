# FreeRTOS Task Model & Core Affinity (ESP32-S3 Dual-Core)

ESP32-S3 provides two 240 MHz Xtensa LX7 cores (**Core 0 / PRO_CPU** and **Core 1 / APP_CPU**). Pinning networking stacks to Core 0 and deterministic access/media tasks to Core 1 prevents WebRTC video/audio bursts from starving RFID unlock or Wi-Fi/BLE interrupts.

## Task Table

| Task Name | Core | Priority | Stack (Bytes) | Role & Responsibilities |
|---|---|---|---|---|
| `wifi` / `btController` / `nimble_host` | Core 0 | IDF default (18–23) | IDF default | ESP-IDF Wi-Fi, LwIP, and NimBLE stacks |
| `mqtt_client` | Core 0 | 6 | 6144 | Persistent MQTTS TLS session, QoS 1 receive/publish, LWT |
| `webrtc_peer` (`esp_peer`) | Core 0 | 8 | 8192 (PSRAM/Int) | ICE/DTLS/SRTP/SCTP transport loop (created on demand during calls!) |
| `http_upload` / `ota_task` | Core 0 | 4 | 8192 | Async HTTPS snapshot POST and background `esp_https_ota` download |
| `lock_core_task` | Core 1 | 12 | 4096 | **Highest app priority**: Lock FSM, `GPIO21` strike timing, PCF8575 `INT` debounce |
| `rfid_task` | Core 1 | 10 | 5120 | MFRC522 100 ms polling, ISO-DEP RATS/APDU, DESFire AES-128 mutual auth |
| `audio_io_task` | Core 1 | 11 | 6144 | Full-duplex `I2S0` read/write, Opus/G.711 codec, local `assets` playback (active during calls/prompts) |
| `cam_capture_task` | Core 1 | 7 | 4096 | Pulls OV5640 hardware JPEG frame buffers from PSRAM and feeds WebRTC / snapshot uploader (active on demand) |
| `sys_health_task` | Core 1 | 3 | 3072 | Task Watchdog supervision, heap/PSRAM telemetry, OTA post-boot self-test |

## Concurrency & Synchronization Rules
1. **Shared `I2C0` Bus Protection:** `bjq_pcf8575`, `esp_codec_dev`, and `esp32-camera` share `I2C_NUM_0`. All device init/control operations serialize cleanly through ESP-IDF's `i2c_master` driver mutex; ISR on `GPIO38` (`IOEXP_INT`) ONLY posts a notification to `lock_core_task`—never performs I2C inside ISR.
2. **On-Demand Media Tasks:** `cam_capture_task` and `webrtc_peer` are spawned only during active intercom/snapshot sessions and torn down afterward, reclaiming internal SRAM and reducing thermal load during idle periods.
3. **Watchdog Policy:** `lock_core_task`, `rfid_task`, and `sys_health_task` subscribe to `esp_task_wdt` (`5 s` timeout; `CONFIG_ESP_TASK_WDT_PANIC=y` in `prod` triggers clean panic-reboot with `GPIO21` hardware pull-down keeping the door locked).
