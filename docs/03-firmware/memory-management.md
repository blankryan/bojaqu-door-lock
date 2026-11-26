# Memory Management (Internal SRAM + 8 MB Octal PSRAM)

ESP32-S3-WROOM-1-N16R8 provides **~512 KB internal SRAM** (shared by IRAM, DRAM, DMA descriptors, Wi-Fi/BLE, and TLS) plus **8 MB Octal PSRAM** (`SPIRAM_MODE_OCT`, 80 MHz). Correct placement of buffers between internal SRAM and PSRAM is mandatory to avoid DMA faults and out-of-memory crashes during concurrent MQTTS + NimBLE + WebRTC + Camera + Audio operation.

## Placement Policy

| Allocation | Memory Region | Approx. Budget | Rationale |
|---|---|---|---|
| Wi-Fi + LwIP + NimBLE runtime | Internal SRAM (RX/TX dynamic in PSRAM where allowed) | ~130–160 KB | Radio ISR & low-latency DMA descriptors |
| Lock FSM, RFID, SPI2 DMA, I2S0 DMA buffers | Internal DMA-capable SRAM (`MALLOC_CAP_DMA`) | ~40 KB | Deterministic real-time access and DMA requirement |
| mbedTLS sessions (MQTTS + HTTPS + WebRTC DTLS) | **External PSRAM** (`CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y`) | ~120–180 KB | Offloads 3+ simultaneous TLS/DTLS contexts (~40–60 KB each) out of precious internal SRAM |
| OV5640 Camera Frame Buffers (2x JPEG VGA/SVGA) | **External PSRAM** (`CAMERA_FB_IN_PSRAM`) | ~200–400 KB | Hardware JPEG frames ~25–80 KB each; 2 buffers support continuous `GRAB_LATEST` |
| WebRTC (`esp_peer`) jitter/retransmit/SCTP buffers | **External PSRAM** (`MALLOC_CAP_SPIRAM`) | ~250–400 KB | Only allocated during active intercom calls |
| Audio PCM / Opus / Local Prompt Decode Buffers | **External PSRAM** (DMA ring in internal SRAM) | ~64–128 KB | Prevents internal heap fragmentation |
| Offline Audit Log / Snapshot Upload Staging | **External PSRAM** | ~128 KB | Staging JPEG buffer before HTTPS POST |

## Key `sdkconfig` Memory Settings
- `CONFIG_SPIRAM=y`, `CONFIG_SPIRAM_MODE_OCT=y`, `CONFIG_SPIRAM_SPEED_80M=y`
- `CONFIG_SPIRAM_USE_MALLOC=y` with `CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=4096` (small control structs stay in fast internal SRAM; allocations >= 4 KB automatically land in 8 MB PSRAM)
- `CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y`
- `CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y`
- `CONFIG_SPIRAM_FETCH_INSTRUCTIONS=y` and `CONFIG_SPIRAM_RODATA=y` (optional if flash cache XIP contention appears during camera DMA + SPI flash writes)

## Runtime Guardrails
- `sys_health_task` monitors `heap_caps_get_minimum_free_size(MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT)` and publishes telemetry over MQTTS; alert threshold triggers if free internal SRAM drops below **40 KB**.
- During **OTA flash writes**, active WebRTC sessions are gracefully paused to eliminate simultaneous Octal PSRAM + SPI flash erase cache stalls.
