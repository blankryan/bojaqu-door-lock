# ADR-0003: Mobile Communication Topology and Control Plane (MQTTS + NimBLE)

- Status: Accepted
- Date: 2026-11-26
- Scope: Platform
- Owners: Bojaqu Firmware & Mobile Engineering

## Context
Homeowners use an Android application to provision the door lock, remotely unlock the door from anywhere, receive real-time visitor/door/RFID alerts, manage authorized RFID cards, initiate live camera/audio intercom sessions, and control OTA updates. The lock must also support initial Wi-Fi setup and optional nearby unlock when internet/Wi-Fi is unavailable, while remaining architecturally ready for future integration with **Home Assistant**, **ESP RainMaker**, or **Matter over Wi-Fi**.

## Decision
Adopt a **Hybrid Cloud + BLE topology** with a **decoupled transport adapter layer**:

1. **Cloud Control & Telemetry Plane (`MQTTS` over TLS 1.2/1.3):**
   - Device maintains a persistent MQTT 3.1.1 / 5.0 connection (`espressif/mqtt`) to the Bojaqu MQTT broker over port `8883` using per-device X.509 mTLS certificates (`prod`) or staging TLS credentials (`dev`).
   - Structured topic hierarchy under `bojaqu/v1/locks/{device_id}/...` for commands (`cmd/unlock`, `cmd/lock`, `cmd/rfid_sync`, `cmd/webrtc_signal`, `cmd/ota`), command acknowledgments (`ack`), retained state (`state`), real-time events (`events`), and Last Will & Testament (`status` online/offline).
   - All security-critical commands (especially `cmd/unlock`) include a monotonic/UUID `request_id`, UTC expiry timestamp (`exp`, verified via SNTP), and short-lived server authorization signature/nonce to prevent replay attacks; commands are processed with MQTT QoS 1 and idempotent deduplication.
   - The Android app communicates with the Bojaqu Cloud Backend via HTTPS REST / WebSocket + Firebase Cloud Messaging (FCM) push notifications for doorbell rings, forced-door alarms, and OTA availability.
2. **Local BLE Plane (`NimBLE`):**
   - **Wi-Fi Provisioning:** Uses `espressif/network_provisioning` over BLE with **Security 2 (SRP6a)** and a per-device QR-code Proof of Possession (PoP).
   - **Offline Nearby Unlock:** Exposes a custom Bojaqu BLE GATT Lock Service using **challenge-response AES-GCM/HMAC authentication** (device issues a fresh 16-byte random nonce per session; Android signs with a device-scoped key provisioned during pairing) so the homeowner can unlock locally even if home Wi-Fi or internet is down.
3. **Ecosystem Extensibility Boundary:**
   - All adapters translate external payloads into internal domain commands (`lock_cmd_t`) and subscribe to domain events (`lock_event_t`).
   - Choosing MQTTS directly enables a future **Home Assistant MQTT Discovery** adapter with minimal overhead, and isolates domain logic so **ESP RainMaker** or **Matter over Wi-Fi** adapters can be added in later releases.

## Alternatives considered
- **Local Wi-Fi LAN only (mDNS + local HTTP/WebSocket):** Rejected because homeowners cannot unlock the door, answer visitors, or receive alerts when away from home.
- **Secure WebSocket (WSS) instead of MQTTS:** Rejected because MQTT natively provides broker pub/sub fan-out, QoS 1 delivery, retained state, LWT online/offline detection, and a direct bridge to Home Assistant MQTT Discovery and ESP RainMaker (AWS IoT MQTT).
- **HTTPS REST polling / Server-Sent Events (SSE):** Rejected due to higher latency for remote unlock and doorbell signaling, plus extra TLS handshake overhead.
- **ESP RainMaker or Matter exclusively from day one:** Rejected because v1 requires custom Bojaqu Android app features (DESFire card enrollment, WebRTC intercom, custom OTA gating), whereas an adapter boundary lets us add RainMaker/Home Assistant/Matter cleanly afterward.

## Consequences
- **Benefits:** Instant remote access from anywhere on 4G/5G/Wi-Fi; offline resilience via BLE challenge-response unlock and local RFID; low-latency push events; seamless path to Home Assistant and RainMaker.
- **Trade-offs / Constraints:** Requires operating an MQTT broker + backend API + FCM push gateway; NimBLE + Wi-Fi coexistence on ESP32-S3 must share the 2.4 GHz radio cleanly (BLE advertising duty-cycled after provisioning; BLE pauses or reduces scan/adv intervals during active WebRTC streams); SNTP time sync is required for command expiry validation.
- **Follow-up work:** Finalize JSON schema contracts in [Protocol & Communication Workflow](../04-communication/protocol-workflow.md) with the Android and backend teams; define BLE GATT UUIDs and challenge-response crypto vectors.

## References
- [Protocol & Communication Workflow](../04-communication/protocol-workflow.md)
- [ADR-0001](0001-firmware-architecture.md), [ADR-0004](0004-media-streaming-and-audio-architecture.md), [ADR-0007](0007-security-hardening-and-build-presets.md)
