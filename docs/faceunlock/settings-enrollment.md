# SettingsGoogle: Enrollment & Management Pipeline

## 1. Feature Provider & Entry Injection

Google customizes the AOSP Face Settings using feature injection:

* **`FaceFeatureProviderGoogleImpl`** (`com.google.android.settings.biometrics.face.FaceFeatureProviderGoogleImpl`):
  * Overrides `getEnrollActivityClassProvider()` to return `FaceEnrollActivityClassProviderGoogle.INSTANCE`.
  * Overrides `getFaceSettingsFeatureProvider()` to return `FaceSettingsFeatureProviderGoogle.INSTANCE`.
  * Checks attention requirement support via `config_face_settings_attention_supported`.
  * Reads `settings_max_face_enrollable` for maximum face enrollment limits.

---

## 2. The Enrollment Wizard Flow

```text
[Settings / SetupWizard]
          │
          ▼
[FaceEnrollIntroductionGoogle] ──(Checks Class 3 / Parental Consent / Private Profile)
          │
          ▼
[FaceEnrollFoldPage] ──────────(Verifies device posture on foldables)
          │
          ▼
[FaceEnrollEducationGoogle] ───(Interactive Lottie animation & gaze toggle)
          │
          ▼
[FaceEnrollEnrolling] ─────────(Camera2 preview + FaceEnrollSidecar + Debounced Help)
          │
          ▼
[FaceEnrollConfirmation] ──────(Sets lock screen bypass preference)
```

### 2.1 Introduction (`FaceEnrollIntroductionGoogle`)
* Evaluates `isFaceStrong()` to present Class 3 (Strong Biometric) security text (`security_settings_face_enroll_introduction_message_class3_2`) or standard biometric disclosures.
* Checks parental consent requirement via `ParentalControlsUtils.parentConsentRequired()`.
* Handles Private Space profile enrollments (`private_space_face_enroll_introduction_message`).

### 2.2 Foldable Posture Gate (`FaceEnrollFoldPage`)
* Uses `ScreenSizeFoldProvider` to listen for fold/hinge posture transitions.
* If the posture is not among `config_face_enroll_supported_posture`, enrollment is blocked until the device is positioned properly (e.g. unfolded or outer screen folded).

### 2.3 Education (`FaceEnrollEducationGoogle` / `FaceEnrollEducation`)
* Plays interactive Lottie animations demonstrating head rotation (`config_face_education_use_lottie`).
* Exposes `FaceEnrollAccessibilityToggle` (`toggle_gaze` / `switch_diversity`) for accessibility options (e.g., skip 3D head movement, disable gaze tracking).

### 2.4 Active Enrolling (`FaceEnrollEnrolling` & `FaceEnrollPreviewFragment`)
* **Camera Preview Modes**:
  * **App-Managed Preview (`config_face_settings_should_manage_preview == true`)**: `FaceEnrollPreviewFragment` opens the front RGB camera using Camera2 API (`CameraManager.openCamera`), binds the stream to a `SquareTextureView`, and instructs the HAL to enroll with `previewSurface = null`.
  * **HAL-Managed Preview (`false`)**: The biometric HAL directly drives the preview stream.
* **Animation Drawables**:
  * `FaceEnrollAnimationMultiAngleDrawable`: Standard 3D multi-angle capture with rotating segments.
  * `FaceEnrollAnimationSingleCaptureDrawable`: Accessibility single-frame capture mode.
* **Help Message Debouncing**:
  * An inner `HelpController` implements a 10-bucket buffer (`Debouncer`) to prevent visual flickering of HAL error strings ("center head", "too close", "turned too far").
* **Gaze Timeout Dialog**:
  * If gaze acquisition fails repeatedly (`mGazeFailCount >= 10`), triggers `FaceGazeDialog` to allow the user to continue with disabled eye-tracking.

### 2.5 Hardware Interface (`FaceEnrollSidecar` & `FaceUpdater`)
* `FaceEnrollSidecar` retains enrollment state across activity recreation.
* Converts UI flags into disabled feature arrays:
  * Feature `1`: Attention / Eye-tracking required.
  * Feature `2`: Diversity / Multi-angle head turn required.
* Passes the challenge, hardware token, and cancellation signal to `FaceUpdater.enroll()` -> `FaceManager.enroll()`.
* Upon completion (`remaining == 0`), triggers `FaceSafetySource.onBiometricsChanged()` to notify Android Safety Center.

---

## 3. Preference Dashboard (`FaceSettings`)

Inside `FaceSettings`, settings are partitioned among preference controllers:
1. `FaceSettingsKeyguardUnlockPreferenceController`: Controls `face_unlock_keyguard_enabled`.
2. `FaceSettingsAppsPreferenceController`: Controls `face_unlock_apps_enabled` (active on Class 3 devices).
3. `FaceSettingsAttentionPreferenceController`: Controls `face_unlock_attention_required`.
4. `FaceSettingsConfirmPreferenceController`: Controls `face_unlock_always_require_confirmation`.
5. `FaceSettingsLockscreenBypassPreferenceController`: Controls `face_unlock_dismisses_keyguard`.
6. `FaceSettingsRemoveButtonPreferenceController`: Invokes `FaceUpdater.remove()` to delete stored face templates.
