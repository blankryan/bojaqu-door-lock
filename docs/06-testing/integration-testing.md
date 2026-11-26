# Integration Testing Matrix (Firmware + Cloud + Android)

| Test ID | Scenario | Expected Result |
|---|---|---|
| **IT-001** | BLE Provisioning (Security 2 SRP6a + QR PoP) | Android binds lock to Wi-Fi & Bojaqu account; wrong PoP rejected; lock connects MQTTS with `fctry` mTLS cert. |
| **IT-002** | DESFire EV2/EV3 Enrollment & Offline Unlock | Card enrolled via Android -> Wi-Fi router turned OFF -> tap card unlocks in <= 500 ms; audit log flushes once Wi-Fi returns. |
| **IT-003** | Anti-Cloning & Lockout | Cloned UID-only magic card rejected in `prod`; 5 consecutive bad taps trigger 30 s lockout, snapshot upload, and FCM alert; indoor exit button still unlocks. |
| **IT-004** | Remote Unlock & Replay Rejection | Android over 4G/5G unlocks lock in <= 1.5 s; replaying captured MQTT payload or expired `exp` is rejected with `ack.status="rejected"`. |
| **IT-005** | Offline BLE Nearby Unlock | With Wi-Fi OFF, Android authenticates via NimBLE challenge-response and unlocks; replaying prior nonce fails. |
| **IT-006** | Doorbell -> Snapshot -> WebRTC Intercom | Pressing doorbell plays local chime, uploads JPEG thumbnail, rings Android via FCM, establishes WebRTC VGA MJPEG + two-way audio across cellular NAT (TURN), and unlocks mid-call. |
| **IT-007** | Safe OTA Update & Rollback | Publish valid signed v+1 -> device downloads to inactive 5 MB slot, waits until door is closed & locked, reboots, passes 60 s self-test, marks valid. Repeat with broken self-test image -> bootloader automatically rolls back to previous slot. |
