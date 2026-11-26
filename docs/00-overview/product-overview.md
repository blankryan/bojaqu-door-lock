# Product Overview — Bojaqu Smart Door-Lock

## Purpose
`bojaqu-door-lock` is a mains-powered smart door-lock and video-intercom firmware built on **ESP32-S3-WROOM-1-N16R8** (16 MB Flash + 8 MB Octal PSRAM) running **ESP-IDF v6.1.0**. It combines offline-capable cryptographic RFID access control, remote Android lock control, live OV5640 camera viewing, two-way voice intercom (ES8311 + NS4150B), and secure signed Over-The-Air (OTA) firmware updates.

## System Context
```mermaid
flowchart LR
    Visitor["Visitor / Homeowner at Door"] -->|DESFire EV2/EV3 Card| RFID["MFRC522 (SPI2)"]
    Visitor -->|Doorbell / Voice / Face| Media["OV5640 + ES8311/NS4150B"]
    Lock["ESP32-S3-WROOM-1-N16R8\nbojaqu-door-lock"] -->|GPIO21 Boot-Safe| Strike["12V Fail-Secure Strike"]
    Lock -->|I2C0 + INT| Exp["PCF8575 (Buttons, Reed, LEDs, PA/RST)"]
    RFID --> Lock
    Media <--> Lock

    Lock <-->|MQTTS (TLS mTLS)| Cloud["Bojaqu Cloud\n(MQTT Broker + API + FCM + OTA CDN)"]
    Lock <-->|HTTPS Snapshots & OTA| Cloud
    Lock <-->|WebRTC DTLS-SRTP + STUN/TURN| Android["Bojaqu Android App"]
    Cloud <-->|HTTPS REST / WS / FCM| Android
    Lock <-->|NimBLE Provisioning & Offline Unlock| Android
```

## Key Design Pillars
- **Offline-First Physical Reliability:** Cryptographic RFID unlock and indoor exit button work locally with zero internet dependency; fail-secure 12 V strike remains locked across resets and power loss.
- **Real-Time Mobile Experience:** Always-on Wi-Fi enables sub-second remote unlock via MQTTS, push notifications with OV5640 snapshots, and on-demand WebRTC live video + two-way audio intercom.
- **Hardware-Rooted Security (`prod`):** Secure Boot v2, Flash Encryption (Release mode), NVS Encryption, Anti-Rollback, per-device X.509 mTLS, BLE Security 2 (SRP6a), and DESFire EV2/EV3 AES-128 mutual auth.
- **Ecosystem Extensibility:** Decoupled adapter layer designed for future Home Assistant (MQTT Discovery), ESP RainMaker, and Matter over Wi-Fi integration.
