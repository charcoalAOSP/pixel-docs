# Lockscreen Bypass & Visual Overlays

## 1. Keyguard Lockscreen Bypass Mechanism

Lockscreen bypass governs whether a successful face recognition event instantly dismisses the keyguard or leaves the user on the lockscreen until an explicit swipe gesture is made.

### 1.1 State Engine (`KeyguardBypassController`)

* Reads preference: `Settings.Secure.face_unlock_dismisses_keyguard`.
* Checks `canBypass()` conditions:
  ```kotlin
  fun canBypass(): Boolean {
      if (getBypassEnabled()) {
          return bouncerShowing ||
                 keyguardTransitionInteractor.getCurrentState() == KeyguardState.ALTERNATE_BOUNCER ||
                 !(statusBarStateController.getState() != 1 ||
                   launchingAffordance ||
                   isPulseExpanding ||
                   qsExpanded)
      }
      return false
  }
  ```

### 1.2 Execution Paths
* **Instant Unlock (`canBypass() == true`)**:
  * Calls `BiometricUnlockController.startWakeAndUnlock()`, immediately transitioning the scene to `KeyguardState.GONE` (Home screen / App).
* **Pending Unlock (`canBypass() == false`)**:
  * Saves authentication result in `PendingUnlock`.
  * The device is marked unlocked internally, notifications can be read, and the next swipe up dismisses the lockscreen without re-running biometrics.

---

## 2. Dynamic Camera Cutout Rim Overlay

Pixel devices render an animated glowing ring around the front camera hole during active face scanning.

### 2.1 Cutout View Binding (`FaceScanningOverlay` & `FaceScanningProviderFactoryImpl`)
* Subclasses `ScreenDecorations.DisplayCutoutView`.
* `FaceScanningProviderFactoryImpl` queries `DisplayCutout.getFillBuiltInDisplayCutout()` and `DisplayInfo.rotation` to determine exact physical camera cutout geometry across portrait, landscape, and folded states.

### 2.2 Matrix Transformation & Rendering
* Computes path bounding box:
  ```java
  Path path = new Path(this.protectionPath);
  Matrix matrix = new Matrix();
  RectF rectF = new RectF();
  path.computeBounds(rectF, true);
  matrix.setScale(rimProgress, rimProgress, rectF.centerX(), rectF.centerY());
  path.transform(matrix);
  canvas.drawPath(path, this.rimPaint);
  ```
* **Color Transitions**:
  * Blends smoothly from `lockscreenAnimationColor` to `onScrimColor` depending on shade expansion, bouncer presence, or dark theme state.
  * Transitions dynamically to green checkmark or red error pulses upon completion.

---

## 3. Low-Light Illumination (`LowLightFaceInteractor`)
* On optical ML devices (Pixel 7/8/9), low-light detection activates screen illumination during keyguard wake:
  * `LowLightFaceInteractor` senses low lux / ambient light conditions.
  * `LowLightFaceAnimationBinder` drives high-brightness screen illumination to illuminate the user's face for camera recognition.
