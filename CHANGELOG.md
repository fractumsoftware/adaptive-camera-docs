# Changelog - Adaptive Camera Suite (UE 5.8)

All notable changes to the **Adaptive Camera** plugin line by **Fractum Software** are documented in this file.

---

## [1.0.0] - Initial Fab Release (Unreal Engine 5.8)

### Target Platforms & Engine
- Built and verified for **Unreal Engine 5.8** on **Windows (`Win64`)** and **Linux** (`Development` and `Shipping`, `-Rocket -StrictIncludes`).
- Standalone isolated packaging for all four plugins with zero cross-plugin dependencies.
- 33 built-in plugin C++ automation tests covering spring convergence, accessibility overrides, perspective scoping, RDG shader parameter snapshots, and coexistence.

### Adaptive Camera: Shooter (`1.0.0`)
- Native First-Person & Third-Person `UCameraModifier` with runtime perspective and shoulder switching, `ECC_Camera` sweeps, and First-Person-only grenade throw kinematics (`TriggerGrenadeThrow`, `StartGrenadeWindup`, `ReleaseGrenadeThrow`).
- Spring-damper weapon recoil, ADS FOV transitions, dual hip/ADS raycast autofocus DOF with recursive pawn/weapon exclusion, and tactical Q/E lean.
- Post-tonemap ViewRect-safe RDG HLSL shader with decoupled dark suppression tunnel vs. directional crimson damage feedback, ADS barrel distortion, flashbang recovery (plus Dark Flashbang accessibility mode), and A/B Split Compare.
- Dockable Editor Studio panel, `UAdaptiveCameraShooterAnimNotify`, 3 DataAsset presets (`MasterIntensity = 0.65`), and playable `/AdaptiveCamera_Shooter/Maps/L_Showcase`.

### Adaptive Camera: Horror (`1.0.0`)
- First-Person Only survival-horror camera modifier with tension-coupled physiological breathing, cardiac pulse oscillator, grounded bodycam stride, turn inertia, low-sanity FOV compression/Dutch tilt, and directional 3D jump scare springs.
- Low-light oscillatory focus hunting DOF with reciprocal `1 / f-stop` scaling.
- Post-tonemap ViewRect-safe RDG HLSL shader with optical fisheye distortion, dual-band VHS tracking drift, temporal tape grain, scanlines, chroma bleed, monochrome IR Night Vision mode, and A/B Split Compare.
- Dockable Editor Studio panel with 30 Hz live preview, `UAdaptiveCameraHorrorAnimNotify`, 3 DataAsset presets (`MasterIntensity = 0.65`), and playable `/AdaptiveCamera_Horror/Maps/L_Showcase`.

### Adaptive Camera: Action - Adventure (`1.0.0`)
- Third-Person `UCameraModifier` cooperating with `USpringArmComponent`, orbital yaw inertia, translation lag, `ECC_Camera` obstruction sweeps, and diagonal low-angle boss lock-on framing with dynamic Dutch tilt and runtime duel-side flipping.
- Execution and dialogue Rack Focus DOF with exploration autofocus fallback.
- Continuous low-health camera sway/crimson edge pulse and healing relief camera lift/luminous edge-to-center recovery wave.
- Post-tonemap ViewRect-safe RDG HLSL shader with socket/world-projected 12-tap radial blur, chromatic shift, S-curve hit-stop grading, Perfect Parry flash, and A/B Split Compare.
- 6-tab dockable Editor Studio panel, `UAdaptiveCameraActionAnimNotify`, 3 DataAsset presets (`MasterIntensity = 0.65`), and playable `/AdaptiveCamera_Action/Maps/L_Showcase`.

### Adaptive Camera: Strategy - City Building (`1.0.0`)
- Top-Down / Isometric / Tactical Aerial camera modifier with damped panning, momentum tilt, orbital yaw damping, zoom-to-pitch curve control, non-invasive Overlay Mode (`bDriveAltitudeAndPitch = false`), owning pawn XY sync, and altitude-coupled Tilt-Shift DOF.
- Single-pass post-tonemap ViewRect-safe RDG HLSL shader with true 3D World-Space `SceneDepth` + Large World Coordinates (LWC) tactical sonar wave and world grid (with automatic 2D fallback), directional and custom-colored perimeter map alerts, distance- and altitude-attenuated artillery shocks, and miniature edge defocus.
- Low-altitude battlefield combat proximity rumble with zero-code `UAdaptiveCameraStrategyCombatZoneComponent` and `OnCombatProximityIntensityChanged` audio delegate.
- 5-tab dockable Editor Studio panel, `UAdaptiveCameraStrategyAnimNotify`, 3 DataAsset presets (`MasterIntensity = 0.65`), and playable `/AdaptiveCamera_Strategy/Maps/L_Showcase`.
