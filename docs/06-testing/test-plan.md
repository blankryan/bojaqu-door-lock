# Test Plan

## Test Pyramid
1. **Shared Component Unit Tests (CI in `bjq-esp-component`):**
   - `bjq_pcf8575`: Unity tests for bounds checks, `input_mask` invariant protection, shadow state, and INT attachment.
   - `bjq_mfrc522 v0.3.0`: Unity tests for ISO 14443-3A UID/SAK + ISO-DEP ATS parser, PCB block toggling, chaining, WTX, and Wokwi simulation.
   - `bjq_desfire`: Unity tests against official NXP AN10922 (AES-128 key diversification) and AN12196 (`AuthenticateEV2First` + `CommMode.Full`) test vectors using a mock APDU responder.
2. **Product Unit & Logic Tests (`bojaqu-door-lock/test/`):**
   - Lock FSM transitions (auto-relock, door-ajar, forced-open, 15 s solenoid hard cap, exit-button override of lockout).
   - MQTTS remote-unlock replay protection (expired `exp`, duplicate `request_id`, invalid signature).
   - BLE offline challenge-response nonce expiry & single-use invalidation.
   - OTA safety interlock gating (defer reboot while door open, unlocked, or in active WebRTC call).
3. **Wokwi Simulation (`dev` preset):**
   - Automated smoke test verifying `dev` build boots, probes MFRC522 (`VersionReg=0x92`), reads test card UID, and drives `GPIO21` lock state cleanly.
4. **Hardware-in-the-Loop & End-to-End Integration (Rev A Bench):**
   - See [Hardware Validation](hardware-validation.md) and [Integration Testing](integration-testing.md).
