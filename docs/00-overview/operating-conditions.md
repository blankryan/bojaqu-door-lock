# Operating Conditions

| Parameter | Specification | Notes |
|---|---|---|
| **Supply Voltage** | 12 V DC nominal (10.8 V – 13.8 V), >= 2.0 A | Powers 12 V solenoid/strike + regulated 5 V / 3.3 V logic rails |
| **Operating Temperature** | -10 °C to +55 °C (ambient) | Watch ESP32-S3 + OV5640 thermal rise during sustained WebRTC calls; enforce call idle timeout (60–120 s) |
| **Relative Humidity** | 10% – 85% RH non-condensing | Outdoor fascia (camera lens, RFID antenna, doorbell button) requires IP54+ gasket sealing |
| **Wi-Fi Environment** | 2.4 GHz IEEE 802.11 b/g/n (RSSI >= -75 dBm recommended) | Required for MQTTS, WebRTC video/audio, and HTTPS OTA; BLE + RFID continue working if Wi-Fi drops |
| **BLE Range** | Up to ~5 m through residential door | Used for BLE Provisioning and nearby offline unlock |
| **RFID Read Distance** | 15 – 40 mm (13.56 MHz ISO/IEC 14443-A/4) | Non-metallic antenna bezel required over MFRC522 coil |
| **Power Loss Behavior** | **Fail-Secure** (remains locked) | Mechanical key cylinder / indoor manual egress override required per local building code |
