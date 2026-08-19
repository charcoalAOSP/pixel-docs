# Pixel Documentation (`pixel-docs`)

Technical documentation and reverse-engineering notes on Google Pixel proprietary components, customizations, and framework subsystems across Android / AOSP.

---

## Table of Contents

- [Biometrics: Face Unlock Subsystem](docs/faceunlock/overview.md)
  - [Architecture Overview](docs/faceunlock/overview.md)
  - [SettingsGoogle: Enrollment & Management Flow](docs/faceunlock/settings-enrollment.md)
  - [SystemUIGoogle: Authentication Engine & Gating Matrix](docs/faceunlock/systemui-auth.md)
  - [Device Variations & Hardware Topologies](docs/faceunlock/hardware-and-devices.md)
  - [Lockscreen Bypass & UI Visual Overlays](docs/faceunlock/bypass-and-ui.md)

---

## Repository Structure

```text
pixel-docs/
├── README.md
└── docs/
    └── faceunlock/
        ├── overview.md              # System-wide overview & sequence diagram
        ├── settings-enrollment.md   # SettingsGoogle enrollment wizard & sidecar
        ├── systemui-auth.md         # SystemUIGoogle reactive flow repository & interactor
        ├── hardware-and-devices.md  # Pixel 4, 7, 8, 9, and Fold hardware adaptations
        └── bypass-and-ui.md         # Keyguard bypass and camera cutout rim overlay
```

---

## Contributing

Maintained by **charcoalAOSP**. When adding documentation:
- Include decompiled class names and package locations.
- Reference relevant Android CDD / Biometric security specifications where applicable.
- Document Runtime Resource Overlay (RRO) configs and System property overrides.
