# State Machines

## 1. Lock Controller State Machine
```mermaid
stateDiagram-v2
    [*] --> LOCKED
    LOCKED --> UNLOCKING : Authorized RFID / MQTTS / BLE / Exit Btn
    LOCKED --> FORCED_OPEN_ALARM : Reed opens while LOCKED
    LOCKED --> LOCKOUT : 5 invalid RFID taps in 60s

    UNLOCKING --> UNLOCKED : GPIO21 HIGH + Auto-Relock Timer Start
    UNLOCKED --> DOOR_OPEN : Reed detects door opened
    UNLOCKED --> LOCKED : Auto-Relock Timeout (default 5s) OR RFID tap re-lock OR MQTTS lock

    DOOR_OPEN --> LOCKED : Reed detects door closed (+ settle delay 500ms) -> GPIO21 LOW
    DOOR_OPEN --> DOOR_AJAR_ALARM : Open > ajar threshold (default 30s)
    DOOR_AJAR_ALARM --> LOCKED : Door closed -> GPIO21 LOW

    FORCED_OPEN_ALARM --> LOCKED : Door closed + Admin ack (alarm logged + snapshot uploaded)
    LOCKOUT --> LOCKED : Cooldown expired (30s -> 120s) OR Admin remote unlock
```

### Hard Safety Invariants
- `GPIO21` (`LOCK_ACTUATOR_EN`) is driven LOW immediately when transitioning to `LOCKED`, `FORCED_OPEN_ALARM`, or `LOCKOUT`, and is hardware-clamped by a **15 s max solenoid energization watchdog** even if the door remains ajar.
- Physical **indoor exit button (`P12`)** always overrides `LOCKOUT` for life-safety egress.

## 2. Media / WebRTC Intercom State Machine
```mermaid
stateDiagram-v2
    [*] --> IDLE_CAM_OFF
    IDLE_CAM_OFF --> SNAPSHOT_CAPTURE : Doorbell / Unlock / Alarm / Lockout
    SNAPSHOT_CAPTURE --> IDLE_CAM_OFF : HTTPS JPEG upload complete -> CAM_PWDN HIGH

    IDLE_CAM_OFF --> SIGNALING : Doorbell Call Answered OR Android Live View Request
    SIGNALING --> STREAMING : ICE Connected + DTLS-SRTP Ready
    SIGNALING --> IDLE_CAM_OFF : Timeout (20s) / Rejected / Error

    STREAMING --> TEARDOWN : Hangup / Idle Timeout (120s) / OTA Preempt / Wi-Fi Lost
    TEARDOWN --> IDLE_CAM_OFF : Close esp_peer, mute NS4150B PA, CAM_PWDN HIGH
```

## 3. OTA Update State Machine
```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> CHECKING : MQTTS cmd/ota_available OR 24h Timer
    CHECKING --> IDLE : Up-to-date OR Manifest Invalid
    CHECKING --> AWAITING_CONFIRM : New version & Auto-Update OFF
    CHECKING --> DOWNLOADING : New version & (Auto-Update ON OR Urgent)

    AWAITING_CONFIRM --> DOWNLOADING : Android MQTTS cmd/ota_confirm
    AWAITING_CONFIRM --> IDLE : User Deferred / Dismissed

    DOWNLOADING --> VERIFYING : Inactive slot written (esp_https_ota)
    DOWNLOADING --> IDLE : Network Error (report progress/failure)

    VERIFYING --> WAITING_SAFE_REBOOT : SHA-256 + Secure Boot v2 + SecVer OK
    VERIFYING --> IDLE : Verification Failed (discard slot, alert cloud)

    WAITING_SAFE_REBOOT --> REBOOTING : Door CLOSED & LOCKED & No Call & 10s RFID Quiet
    REBOOTING --> PENDING_VERIFY : Boot new slot
    PENDING_VERIFY --> IDLE : Self-Test PASS -> mark_app_valid_cancel_rollback()
    PENDING_VERIFY --> ROLLBACK : Self-Test FAIL / Crash / WDT -> Bootloader reverts slot
```
