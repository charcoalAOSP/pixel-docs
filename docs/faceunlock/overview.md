# Face Unlock Architecture Overview

## Subsystem Division

The Face Unlock subsystem in Google's Android implementation (Pixel devices) is architecturally divided between two main packages:

1. **`SettingsGoogle` (`com.google.android.settings.biometrics.face`)**:
   - Handles the interactive enrollment wizard, foldable posture checks, education Lottie animations, camera preview rendering (Camera2 vs HAL surface), gaze tracking dialogues, and preference controllers.
2. **`SystemUIGoogle` (`com.android.systemui.deviceentry` / `com.google.android.systemui`)**:
   - Handles real-time keyguard authentication, sensor-driven triggers (lift-to-wake, bouncer, UDFPS touch), gating condition evaluation, dual operation modes (Authenticate vs Detect), camera cutout pulsing rim animations, and lockscreen bypass transitions.

---

## High-Level Architecture Diagram

```text
                      +-----------------------------------------------------+
                      |                   SettingsGoogle                    |
                      |  * Face Enrollment Wizard (Multi-angle/Single capture)|
                      |  * Foldable Posture Guidance (FaceEnrollFoldPage)   |
                      |  * Attention / Gaze Detection Check                 |
                      |  * Face Settings & Lock Screen Bypass Toggles       |
                      +--------------------------+--------------------------+
                                                 | Hardware Token & Config
                                                 v
                      +-----------------------------------------------------+
                      |                   SystemUIGoogle                    |
                      |  * Reactive Face Auth Repository & Interactor Flow   |
                      |  * Gating Checks Matrix (Display, Posture, Policy)  |
                      |  * Dual Modes: Authenticate vs Detect (Presence)     |
                      |  * Lift-To-Wake Face Trigger (Pickup Sensor)        |
                      |  * Keyguard Lockscreen Bypass Controller            |
                      |  * Camera Cutout Animated Rim (FaceScanningOverlay)  |
                      |  * Low-Light Screen Flash / Illumination            |
                      +-----------------------------------------------------+
                                                 |
                                                 v
                                       +-------------------+
                                       | FaceManager / HAL |
                                       +-------------------+
```

---

## Core Component Map

| Component | Responsibility | Key Classes |
| :--- | :--- | :--- |
| **Settings Provider** | Injects Google enrollment activities & capabilities | `FaceFeatureProviderGoogleImpl`, `FaceEnrollActivityClassProviderGoogle` |
| **Enrollment Wizard** | Multi-step user onboarding & preview | `FaceEnrollIntroductionGoogle`, `FaceEnrollFoldPage`, `FaceEnrollEducationGoogle`, `FaceEnrollEnrolling`, `FaceEnrollPreviewFragment`, `FaceEnrollConfirmation` |
| **Enrollment HAL Bridge** | Manages hardware session tokens and callbacks | `FaceEnrollSidecar`, `FaceUpdater` -> `FaceManager.enroll()` |
| **Auth Repository** | Single reactive source of truth for auth status | `DeviceEntryFaceAuthRepositoryImpl`, `BiometricSettingsRepositoryImpl` |
| **Auth Interactor** | Coordinates SystemUI triggers & bouncer states | `SystemUIDeviceEntryFaceAuthInteractor`, `DeviceEntryFaceAuthStatusInteractor` |
| **Sensor Trigger** | Listens to pickup/lift sensor | `LiftToRunFaceAuthBinder` (`Sensor.TYPE_PICK_UP_GESTURE = 25`) |
| **Keyguard Bypass** | Controls instant unlock vs swipe gesture | `KeyguardBypassController` (`face_unlock_dismisses_keyguard`) |
| **Cutout Animation** | Renders pulsing ring around camera hole | `FaceScanningOverlay`, `FaceScanningProviderFactoryImpl` |
| **Low-Light Assist** | Detects darkness and triggers screen flash | `LowLightFaceInteractor`, `LowLightFaceAnimationBinder` |
