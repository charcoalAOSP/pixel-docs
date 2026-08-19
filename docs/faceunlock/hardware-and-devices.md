# Device Variations & Hardware Topologies

The Pixel Face Unlock implementation is a single polymorphic codebase configured dynamically via Runtime Resource Overlays (RROs) and sensor properties:

---

## 1. Matrix by Pixel Generation

| Feature | Pixel 4 / 4 XL | Pixel 7 / 7 Pro / 7a | Pixel 8 / 9 Series | Pixel Fold / 9 Pro Fold |
| :--- | :--- | :--- | :--- | :--- |
| **Sensor Type** | IR Camera + Dot Projector + Flood Illuminator | Single RGB Front Camera (DPAF) | Single RGB Front Camera + Tensor G3/G4 ML | Dual RGB Cameras (Inner & Outer) |
| **Biometric Tier** | **Class 3 (Strong)** | **Class 1 / 2 (Weak / Convenience)** | **Class 3 (Strong)** | **Class 3 (Strong)** |
| **Use for Banking / Wallet** | Yes | No (Keyguard only) | Yes | Yes |
| **Preview Management** | HAL Surface (`should_manage_preview = false`) | App Camera2 (`should_manage_preview = true`) | App Camera2 (`should_manage_preview = true`) | App Camera2 (`should_manage_preview = true`) |
| **Foldable Posture Check** | No | No | No | Yes (`config_face_auth_supported_posture`) |
| **Low-Light Assist** | Hardware IR Emitters | Low-Light Screen Flash | Low-Light Screen Flash | Low-Light Screen Flash |
| **Wake Trigger** | Soli Radar + Lift | Pickup Sensor / Tap | Pickup Sensor / Tap | Pickup Sensor / Hinge Sensor |

---

## 2. Dynamic Hardware Property Evaluation

### 2.1 Biometric Strength
* Evaluated at runtime via `FacePropertyRepositoryImpl.sensorInfo.strength` (`SensorStrength.STRONG` vs non-strong).
* Controls:
  * Enablement of `FaceSettingsAppsPreferenceController` (`face_unlock_apps_enabled`).
  * Displaying Class 3 disclaimer in `FaceEnrollIntroductionGoogle`.
  * Allowing `FACE_AUTH_TRIGGERED_OCCLUDING_APP_REQUESTED` for Google Wallet / Bitwarden.

### 2.2 Foldable Postures
* Configured by `config_face_enroll_supported_posture` and `config_face_auth_supported_posture`:
  * Handled by `FaceEnrollFoldPage` in Settings.
  * Handled by `BiometricSettingsRepositoryImpl.isFaceAuthSupportedInCurrentPosture` in SystemUI.

### 2.3 RRO Resource Configuration Map
* `config_face_settings_should_manage_preview`: `true` on Camera2 devices, `false` on HAL-managed preview devices.
* `config_face_intro_show_less_secure`: `true` for Class 1/2 devices.
* `config_face_settings_attention_supported`: `true` for devices supporting eye-gaze tracking.
* `config_face_auth_wake_up_triggers`: Array of wake reasons permitted to trigger face scanning.
* `config_face_acquire_device_entry_ignorelist`: Vendor HAL acquire codes filtered out per device.
