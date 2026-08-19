# SystemUIGoogle: Authentication Engine & Gating Matrix

## 1. Architecture: Kotlin Coroutines & Flow Pipeline

SystemUIGoogle implements a reactive biometric pipeline centered on Kotlin Flows, StateFlows, Repositories, and Interactors.

```text
[System Events: Wake, Lift, Bouncer, UDFPS, Shade]
                        │
                        ▼
    [SystemUIDeviceEntryFaceAuthInteractor]
                        │
                        ▼
      [DeviceEntryFaceAuthRepositoryImpl]
                        │
           ┌────────────┴────────────┐
           │ (Evaluate Gating Flow)  │
           ▼                         ▼
   canRunFaceAuth == true     canRunDetection == true
           │                         │
           ▼                         ▼
    [authenticate()]           [detectFace()]
           │                         │
           └────────────┬────────────┘
                        │
                        ▼
      [FaceManager / Biometric HAL]
```

---

## 2. The Gating Conditions Matrix

Before invoking hardware authentication or detection, `DeviceEntryFaceAuthRepositoryImpl.gatingConditionsForAuthAndDetect()` combines multiple reactive StateFlows:

### Common Gates (Required for both Auth & Detect)
1. **`displayIsNotOffWhileFullyTransitionedToAwake`**: Display must be powered on and not in transition to off.
2. **`isFaceAuthEnrolledAndEnabled`**: Face biometric is enrolled and enabled for the current active user.
3. **`keyguardNotGoneAndNotTransitioningToGone`**: Keyguard is visible or actively displaying.
4. **`deviceNotTransitioningToAsleepState`**: Power state is not falling asleep/dozing.
5. **`secureCameraNotActiveOrAnyBouncerIsShowing`**: Secure camera preview is not occluding unless bouncer is active.
6. **`isFaceAuthSupportedInCurrentPosture`**: Foldable device is in a valid supported posture.
7. **`userHasNotLockedDownDevice`**: Device is not in lockdown mode.
8. **`doesNotRequirePrimaryAuthOnBouncerForSecureLockDevice`**: No forced PIN/pattern/password lock.
9. **`isKeyguardShowing`**: Keyguard window state is visible.
10. **`userSwitchingNotInProgress`**: User switching is idle.

### Authentication-Specific Gates (`canRunFaceAuth`)
* **`isNotInLockOutState`**: No active permanent or timed biometric lockout.
* **`keyguardIsNotDismissible`**: Keyguard is not already unlocked/dismissible.
* **`isFaceAuthCurrentlyAllowed`**: Biometric auth allowed by `StrongAuthTracker` and policy.
* **`faceNotAuthenticated`**: Current user is not yet authenticated.

### Detection-Specific Gates (`canRunFaceDetect`)
* **`isBypassEnabled`**: Lockscreen bypass enabled.
* **`faceAuthIsNotCurrentlyAllowedOrCurrentUserIsTrusted`**: Auth not permitted, but presence detection is safe.
* **`udfpsAuthIsNotPossibleAnymore`**: UDFPS is not currently handling authentication.

---

## 3. Trigger Sources & `FaceAuthUiEvent`

| Event Name | Source / Interactor Trigger |
| :--- | :--- |
| `FACE_AUTH_UPDATED_KEYGUARD_VISIBILITY_CHANGED` | Waking from AOD / Off / Dozing to Lockscreen |
| `FACE_AUTH_TRIGGERED_PICK_UP_GESTURE_TRIGGERED` | Device lifted via `LiftToRunFaceAuthBinder` (`Sensor.TYPE_PICK_UP_GESTURE = 25`) |
| `FACE_AUTH_UPDATED_PRIMARY_BOUNCER_SHOWN` | Primary PIN/Pattern/Password bouncer showing |
| `FACE_AUTH_TRIGGERED_ALTERNATE_BIOMETRIC_BOUNCER_SHOWN` | Alternate biometric bouncer (e.g. UDFPS bouncer) active |
| `FACE_AUTH_TRIGGERED_UDFPS_POINTER_DOWN` | Finger touching the under-display fingerprint sensor |
| `FACE_AUTH_TRIGGERED_NOTIFICATION_PANEL_CLICKED` | User tapped on a lockscreen notification |
| `FACE_AUTH_TRIGGERED_QS_EXPANDED` | Quick Settings shade pulled down |
| `FACE_AUTH_TRIGGERED_OCCLUDING_APP_REQUESTED` | Wallet or occluding activity request (Class 3 sensors) |

---

## 4. Operation Modes: Authenticate vs Detect

* **`authenticate()`**:
  * Invokes `FaceManager.authenticate()`.
  * Returns cryptographic hardware tokens to authenticate secure transactions and unlock the device.
  * Consumes lockout failure attempts upon mismatched faces.
* **`detectFace()`**:
  * Invokes `FaceManager.detectFace()`.
  * Validates face presence without generating biometric unlock tokens or advancing lockout failure counters.
  * Used when the device is already unlocked by a TrustAgent or during non-strong authentication scenarios.

---

## 5. Error Recovery & Watchdog Mechanisms
* **Hardware Retry Loop**: On transient HAL errors (`ERROR_HW_UNAVAILABLE` / `ERROR_UNABLE_TO_PROCESS`), `DeviceEntryFaceAuthRepositoryImpl` retries up to 5 times using coroutine launch tracing (`handleFaceHardwareError`).
* **Cancellation Watchdog**: When `cancel()` is dispatched to the biometric HAL, a 3-second coroutine delay watchdog job runs. If the HAL fails to acknowledge cancellation within 3000ms, SystemUI forcibly cleans up the pending state and logs a watchdog event.
