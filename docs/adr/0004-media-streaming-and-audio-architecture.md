# ADR-0004: WebRTC Media Streaming, OV5640 Hardware JPEG, and ES8311 Audio Architecture

- Status: Accepted
- Date: 2026-11-26
- Scope: Product
- Owners: Bojaqu Firmware & Mobile Engineering

## Context
The smart door-lock includes an **OV5640** DVP camera sensor and an **ES8311 + NS4150B** mono audio codec (microphone + speaker amplifier). Homeowners expect to view the door camera from Android across arbitrary NAT/cellular networks, hold two-way voice conversations with visitors, review event snapshots, and hear clear local audio prompts/chimes at the door.

Key ESP32-S3 hardware constraints govern media design:
1. The ESP32-S3 has **no hardware H.264 encoder**; software H.264 (`espressif/esp_h264`) saturates CPU at low resolution/frame rate (~320x240–480x320 @ 5–8 fps) and competes with full-duplex audio and Wi-Fi/TLS.
2. The **OV5640** includes an **onboard hardware JPEG compressor** capable of outputting VGA (640x480) or SVGA (800x600) JPEG frames directly into 8 MB Octal PSRAM via LCD_CAM DMA with low CPU load.
3. The legacy `espressif/es8311` component is deprecated and incompatible with ESP-IDF v6.x (`<6.0`), whereas `espressif/esp_codec_dev` natively supports both ES8311 and NS4150B PA control on ESP-IDF v6.1.

## Decision
1. **On-Demand WebRTC Intercom (`espressif/esp_peer`):**
   - Establish WebRTC sessions only on demand (visitor presses doorbell button or homeowner taps "Live View" in Android), automatically tearing down after an idle timeout (default 60–120 s) to minimize heat, bandwidth, and Wi-Fi contention.
   - Perform SDP offer/answer and ICE candidate signaling over the existing **MQTTS** channel (`cmd/webrtc_signal` / `events/webrtc_signal`), using short-lived STUN/TURN credentials issued by the backend for NAT/cellular traversal.
2. **Camera Driver & Video Transport (`espressif/esp32-camera` + OV5640 Hardware JPEG):**
   - Use official **`espressif/esp32-camera`** (`^2.2.0`) configured for OV5640 DVP JPEG output with 2 frame buffers in Octal PSRAM (`CAMERA_FB_IN_PSRAM`, `CAMERA_GRAB_LATEST`).
   - Stream hardware-compressed JPEG frames over **WebRTC (DataChannel framed MJPEG / RTP MJPEG)** at **VGA 640x480 (default) or SVGA 800x600 @ 10–15 fps** (adaptive JPEG quality 12–20 based on link congestion).
   - Keep camera capture behind a thin internal `camera_hal` interface so a future board revision (e.g., ESP32-P4 with HW H.264) or `espressif/esp_video` migration does not impact callers.
3. **Audio Driver & Two-Way Intercom (`espressif/esp_codec_dev` + `espressif/esp_audio_codec`):**
   - Use official **`espressif/esp_codec_dev`** (`^1.6.2`) for ES8311 I2C control + I2S0 full-duplex PCM (16 kHz, 16-bit, mono) and NS4150B PA mute/unmute via PCF8575 `AUDIO_PA_EN_N`.
   - Stream real-time two-way audio over a **standard WebRTC SRTP audio track** using **Opus** (16 kHz mono, 16–24 kbps) or **G.711A (PCMA)** fallback via `espressif/esp_audio_codec`, with acoustic echo control / half-duplex ducking tuning for the mono enclosure.
   - Play local voice prompts and chimes (card accepted/denied, door unlocked, doorbell ring, pairing mode) from the `assets` SPIFFS partition when no WebRTC call is active.
4. **Event-Driven Still Snapshots (HTTPS):**
   - On doorbell press, unlock event, forced-door alarm, or repeated RFID failures, capture a single OV5640 JPEG frame (SVGA/XGA) and upload it asynchronously via **HTTPS POST (mTLS / presigned URL)** so the Android push notification displays an immediate visitor thumbnail even before WebRTC connects.

## Alternatives considered
- **Software H.264 over WebRTC video track (`espressif/esp_video` + `espressif/esp_h264`):** Rejected as v1 primary path because software H.264 on ESP32-S3 severely limits resolution/FPS, raises chip temperature inside a sealed door-lock housing, and risks audio underruns during simultaneous two-way voice intercom.
- **Cloud Media Relay (WSS/RTSP server transcoding MJPEG + PCM):** Rejected as primary live transport due to continuous server egress cost and ~0.5–1 s extra latency, though WebRTC TURN relay provides an equivalent fallback only when direct P2P ICE fails.
- **Local HTTP MJPEG only (no WebRTC):** Rejected because it fails across NAT/4G/5G networks and does not provide synchronized low-latency encrypted two-way audio.
- **Deprecated `espressif/es8311` driver:** Rejected because it is deprecated and constrains ESP-IDF to `<6.0`.

## Consequences
- **Benefits:** Crisp VGA/SVGA video at 10–15 fps with low ESP32-S3 CPU load; end-to-end DTLS-SRTP media encryption; sub-300 ms voice intercom latency; instant snapshot notifications; official Espressif component support on ESP-IDF v6.1.
- **Trade-offs / Constraints:** The Android app decodes JPEG frames received on the WebRTC video/DataChannel stream (standard Android `BitmapFactory` / `ImageDecoder` on a background thread) rather than relying solely on WebRTC's built-in H.264 `SurfaceViewRenderer`; future Home Assistant native WebRTC card integration may require either a lightweight MJPEG-to-RTSP/WebRTC bridge or optional SW H.264 fallback mode. Full-duplex speaker+mic inside a compact enclosure requires careful acoustic isolation and AEC/ducking calibration during hardware bring-up.
- **Follow-up work:** Benchmark VGA vs SVGA frame rate, PSRAM bandwidth, and audio jitter concurrently on ESP32-S3-WROOM-1-N16R8 rev A hardware; verify `espressif/esp_peer` + `espressif/esp32-camera` + `espressif/esp_codec_dev` dependency resolution on ESP-IDF v6.1.0.

## References
- [Protocol & Communication Workflow](../04-communication/protocol-workflow.md)
- [Memory Management](../03-firmware/memory-management.md)
- [Task Model](../03-firmware/task-model.md)
- [ADR-0001](0001-firmware-architecture.md), [ADR-0002](0002-hardware-platform-and-gpio-budget.md)
