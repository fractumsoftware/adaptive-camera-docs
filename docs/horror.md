# Adaptive Camera: Horror - Survival Horror, Bodycam & VHS Camera FX (UE 5.8)

**Fab Title:** `Adaptive Camera: Horror | Reactive Motion, Focus Hunting & VHS FX (C++)`  
**Author:** Ricardo Pinto / Fractum Software  
**Support:** `fractumsoftware@gmail.com`  
**Target Engine:** Unreal Engine 5.8 (`Win64`, `Linux` - SM5/SM6 Desktop Renderers)  
**Camera Perspective:** **First-Person Only**

Procedural **First-Person Only** camera modifier, low-light focus hunting DOF, directional jump scare shock springs, accessibility comfort scaling, and post-tonemap HLSL bodycam/VHS/IR shader pass for survival horror, found-footage, and bodycam games in **Unreal Engine 5.8**.

---

## 1. Architecture & Modules

**Adaptive Camera: Horror** (`Plugins/AdaptiveCamera_Horror/`) is a standalone C++ plugin with no external plugin or game-module dependencies. It can be installed on its own or alongside the other plugins in the **Adaptive Camera** suite (`Adaptive Camera: Shooter`, `Adaptive Camera: Action - Adventure`, `Adaptive Camera: Strategy - City Building`) in the same project.

- **Public Symbol Prefix:** `UAdaptiveCameraHorror*`, `FAdaptiveCameraHorror*`, and `EAdaptiveCameraHorror*`; primary asset type `AdaptiveCameraHorrorProfile`.
- **Perspective Scope (First-Person Only):** Built specifically for first-person character, bodycam, and found-footage camera rigs (including head-mounted or chest-mounted camera components). Third-person orbit and spring-arm chase cameras are not supported; bodycam stride cadence, turn-inertia harness lag/roll, directional jump scare whip, and forward center-screen raycast focus hunting assume a first-person camera origin.
- **Feature Scope:** Tension-driven breathing, cardiac pulse, grounded bodycam footstep stride weight, turn inertia harness lag/roll, directional jump scare spring recovery, stress vignette and low-sanity FOV/Dutch tilt, raycast autofocus with low-light focus hunting, runtime overrides and accessibility comfort controls, animation notifies, and a post-tonemap HLSL bodycam/VHS/IR shader.
- **`MasterIntensity` Scaling:** The profile asset (`UAdaptiveCameraHorrorProfileDataAsset::MasterIntensity`, default `0.65`) and every category struct (`MasterIntensity`, default `1.0`, range `[0.0 .. 1.0]`) expose independent intensity scalars, where `0.0` disables output and `1.0` applies full configured values without compounding attenuation.
- **Module Breakdown (`AdaptiveCamera_Horror.uplugin`):**

| Module | Type | `LoadingPhase` | `PlatformAllowList` | Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **`AdaptiveCameraHorror`** | `Runtime` | `PostConfigInit` | `Win64`, `Linux` | Mounts `/Plugin/AdaptiveCamera_Horror` (`Shaders/AdaptiveCameraHorrorFX.usf`), registers `r.AdaptiveCameraHorror.Enable` / `r.AdaptiveCameraHorror.Debug` CVars and `FAdaptiveCameraHorrorViewExtension`, and provides the runtime classes (`UAdaptiveCameraHorrorProfileDataAsset`, `UAdaptiveCameraHorrorCameraModifier`, `UAdaptiveCameraHorrorComponent`, `UAdaptiveCameraHorrorAnimNotify`, `AAdaptiveCameraHorrorShowcasePawn`, and `FAdaptiveCameraHorror*` structs). Zero dependencies on `UnrealEd`, `Slate`, `PropertyEditor`, `ToolMenus`, or `CinematicCamera`. |
| **`AdaptiveCameraHorrorEditor`** | `Editor` | `PostEngineInit` | `Win64`, `Linux` | Registers `FAdaptiveCameraHorrorEditorStyle` (`AdaptiveCameraHorrorEditorStyle` using `Resources/Icon128.png`), the dockable **Adaptive Camera: Horror** panel (`AdaptiveCameraHorrorStudioTab`), the **Adaptive Camera: Horror** toolbar button (`AdaptiveCameraHorrorStudioToolbarButton`) in `LevelEditor.LevelEditorToolBar.PlayToolBar` (section `AdaptiveCameraHorrorToolbar`), Live Preview, A/B Split Compare, DataAsset saving, Viewport apply/revert (`Ctrl+Z`), and Live PIE Impulse testing. |

- **Render Thread, Multi-Viewport Isolation & HLSL Pipeline (`/Plugin/AdaptiveCamera_Horror`):**
  - The global pixel shader `FAdaptiveCameraHorrorFXPS` (`Shaders/AdaptiveCameraHorrorFX.usf`) runs at `EPostProcessingPass::Tonemap` via RDG (`AddDrawScreenPass`).
  - **Console Variables:** `r.AdaptiveCameraHorror.Enable` (default `1`) toggles the HLSL pass; `r.AdaptiveCameraHorror.Debug` (default `0`) renders a ViewRect-safe telemetry bar showing intensity, glitch/chroma, IR mode, and split-compare divider position.
  - **Per-Viewport & Split-Screen Isolation:** `FAdaptiveCameraHorrorViewExtension` isolates up to 4 local players across multiple scenes via scene-keyed `RuntimeSlots` (`MaxRuntimeShaderSlots = 16`) from the Editor Viewport preview (`EditorPreview`) and rejects SceneCaptures, ReflectionCaptures, PlanarReflections, and VirtualTextures. `BeginRenderViewFamily` snapshots a POD bundle to the Render Thread, and each `SubscribeToPostProcessingPass` delegate captures its resolved view parameters by value.
  - **ViewRect Handling:** Maps `UVAndScreenPos.xy` through `Input_UVViewportMin` and `Input_UVViewportSizeInverse` and clamps all sample UVs to `[Input_UVViewportBilinearMin, Input_UVViewportBilinearMax]` for Screen Percentage, split-screen, and Movie Render Queue compatibility.
  - **Shader Modes:** Dual-band organic VHS tracking drift, tuned temporal luma/chroma tape noise, radial chromatic aberration + horizontal analog chroma bleed, Night Vision / Bodycam IR phosphor monochrome mode (`NightVisionIntensity`), and A/B Split Compare (`bEnableSplitCompare`, `SplitComparePosition`).

---

## 2. Installation & Integration

> **Perspective Compatibility (First-Person Only):** `Adaptive Camera: Horror` supports **First-Person Only** camera setups (first-person character, bodycam, and found-footage rigs). Third-person spring-arm or orbit cameras are not supported by this plugin.

### Step 1 - Enable the Plugin & Add `UAdaptiveCameraHorrorComponent`
1. Copy `AdaptiveCamera_Horror/` into `<YourProject>/Plugins/` and enable it under **Edit -> Plugins -> Camera -> Adaptive Camera: Horror** or in `<YourProject>.uproject`:
   ```json
   "Plugins": [
     { "Name": "AdaptiveCamera_Horror", "Enabled": true }
   ]
   ```
2. Open your player `ACharacter` or `APawn` Blueprint/C++ class, click **Add Component**, and add **Adaptive Camera: Horror Component** (`UAdaptiveCameraHorrorComponent`).
3. On local player clients (skipping dedicated servers and remote proxies automatically), the component registers a `UAdaptiveCameraHorrorCameraModifier` on the player's `APlayerCameraManager` during `BeginPlay` (or once possessed), passively samples owner movement velocity and grounded state each tick for bodycam stride cadence, and cleans up in `EndPlay`.

### Step 2 - Select, Tune, or Create a `UAdaptiveCameraHorrorProfileDataAsset`
1. **Included Presets (`Content/Presets/`):** Three Data Assets are included (enable **Show Plugin Content** in Content Browser settings):
   - `/AdaptiveCamera_Horror/Presets/DA_Horror_FoundFootageVHS`
   - `/AdaptiveCamera_Horror/Presets/DA_Horror_BodycamParanormal`
   - `/AdaptiveCamera_Horror/Presets/DA_Horror_PsychologicalDread`
   You can also create new assets via **Right-Click -> Miscellaneous -> Data Asset -> AdaptiveCameraHorrorProfileDataAsset** or directly from the Studio panel via **Save as DataAsset...**.
2. **Adaptive Camera: Horror Panel:** Click **Adaptive Camera: Horror** in the Level Editor Play toolbar (or open **Window / Tools -> Adaptive Camera: Horror**, Tab ID `AdaptiveCameraHorrorStudioTab`):
   - Pick a Data Asset from the Asset Registry dropdown, click **Temp Scratch** for in-memory editing, or click **Save as DataAsset...** to export a reusable asset.
   - Load any preset (**Found Footage VHS**, **Bodycam Paranormal**, **Psychological Dread**) and adjust parameters across the 6 tabs (**Breathing & Pulse**, **Stress Vignette**, **Focus Hunting DOF**, **Bodycam / VHS HLSL**, **Accessibility**, **Profiles**). Primary parameters are shown by default; secondary tuning is grouped under **Advanced**.
   - Enable **Live Preview** for instant HLSL feedback in the Level Editor viewport, toggle **A/B Split Compare** with the `0..1` divider slider, use **Apply to Viewport** / **Revert Viewport** (`Ctrl+Z` supported) for `ACineCameraActor` and `APostProcessVolume` previews, or trigger **Live PIE Impulse** buttons (**Test Tension Spike**, **Test Jump Scare**, **Test VHS Glitch**) while running Play-In-Editor.
3. Assign your profile to `UAdaptiveCameraHorrorComponent::DefaultProfile` (and set `DefaultProfileBlendTime`, default `0.25 s`).
4. **Included Showcase Level (`/AdaptiveCamera_Horror/Maps/L_Showcase`):** Open `Content/Maps/L_Showcase` (with **Show Plugin Content** enabled) to inspect a damp sub-level maintenance corridor with emergency crimson lighting, 3 `ACineCameraActor` rigs (`CAM_01_Hero_Showcase`, `CAM_02_VHS_Entity_Threshold`, `CAM_03_Security_CCTV_Angle`), an exposure-locked `APostProcessVolume`, and a pre-wired `ShowcasePlayerPawn` (`AAdaptiveCameraHorrorShowcasePawn`) ready for **Live Preview**, **A/B Split Compare**, **Apply / Revert Viewport**, and **PIE / Live PIE Tester**.
   - **Playable `L_Showcase` Controls (`AAdaptiveCameraHorrorShowcasePawn`):** `[WASD]` Walk, `[Shift]` Run, `[1]` Cycle Fear Level (`0.0 / 0.5 / 1.0`), `[2]` Toggle Low Sanity, `[3]` Trigger VHS Glitch, `[4]` Trigger Jump Scare, `[F]` Toggle Darkness, `[N]` Toggle IR Night Vision, `[Tab]` Cycle Presets, `[0]` Reset, `[H]` Toggle Controls Legend.
   - **First-Person Viewmodel & Low-Light DOF Note:** Low-light presets lower f-stop (`ApertureFStop` / `PanicApertureFStop` down to `2.2`) to accentuate focus hunting. Keep flashlight/arm viewmodel geometry clear of the near focus bound or adjust `ApertureFStop` / `MinFocusDistance` if near-camera viewmodels soften excessively.
   > **Packaging Note (`DirectoriesToNeverCook`):** `Content/Showcase/` and `Content/Maps/` are optional demonstration assets provided for evaluation and viewport preview. Add `/AdaptiveCamera_Horror/Maps` and `/AdaptiveCamera_Horror/Showcase` to `DirectoriesToNeverCook` (`Config/DefaultGame.ini` `[/Script/UnrealEd.ProjectPackagingSettings]`) to exclude them from final game cooking while keeping `Content/Presets/`.

### Step 3 - Connect Blueprint Events, Runtime Overrides, Delegates & Animation Notifies
Call the `BlueprintCallable` / `BlueprintPure` functions on **`UAdaptiveCameraHorrorComponent`** from your gameplay triggers, darkness volumes, encounters, or stealth systems:

| Blueprint / C++ Function | Signature & Defaults | Gameplay Usage |
| :--- | :--- | :--- |
| **`SetFearLevel`** | `(float NewFear01)` | Normalized fear level (`0.0 = calm`, `1.0 = panic`). Accelerates heart BPM, breathing, handheld drift, focus hunting, stress vignette, and `PanicApertureFStop`. |
| **`SetSanityLevel`** | `(float NewSanity01)` | Normalized sanity level (`1.0 = lucid`, `0.0 = fractured`). Drives `SanityFOVDistortion`, oscillating Dutch tilt, low-sanity desaturation, and stress vignette. |
| **`SetAmbientLightFactor`** | `(float LightLevel01)` | Normalized ambient illumination (`1.0 = well lit`, `0.0 = pitch dark`). Amplifies `LowLightHuntingMultiplier` and VHS temporal noise in dark areas. |
| **`TriggerTensionSpike`** | `(float Intensity = 1.0f)` | Transient adrenaline spike accelerating breathing and pulse, tightening stress vignette, and increasing low-light focus hunting. |
| **`TriggerJumpScare`** | `(const FVector& ThreatOrigin, float Intensity = 1.0f)` | Directional camera shock impulse toward/away from `ThreatOrigin` with damped 2nd-order spring recovery, transient fear spike, and optional VHS burst. |
| **`TriggerVHSGlitch`** | `(float Intensity = 1.0f, float Duration = 0.45f)` | Temporary burst of horizontal VHS tracking loss, tape tearing, chroma bleed, and temporal noise. |
| **`SwitchProfile`** | `(UAdaptiveCameraHorrorProfileDataAsset* NewProfile, float BlendTime = -1.0f)` | Blends between two profiles at runtime (`BlendTime < 0` uses `DefaultProfileBlendTime`). |
| **`SetMovementVelocity`** | `(const FVector& NewVelocity)` | Optional manual velocity override (`cm/s`) for bodycam stride cadence when using custom movement without `UCharacterMovementComponent`. |
| **`ClearMovementVelocityOverride`** | `()` | Clears manual velocity override and resumes passive `SampleOwnerVelocity()` sampling each tick. |
| **`SetMovementGrounded`** | `(bool bGrounded)` | Optional manual grounded-state override for bodycam footstep stride cadence on custom locomotion rigs. |
| **`ClearMovementGroundedOverride`** | `()` | Clears manual grounded-state override and resumes automatic grounded detection. |
| **`GetCameraModifier`** *(pure)* | `() const -> UAdaptiveCameraHorrorCameraModifier*` | Returns the active `UAdaptiveCameraHorrorCameraModifier` managed by the component (or `nullptr`). |

**Runtime Overrides API (`Category = "Adaptive Camera|Runtime"` on `UAdaptiveCameraHorrorComponent` & `UAdaptiveCameraHorrorCameraModifier`):**
Runtime overrides layer on top of the active profile blend without mutating any `UAdaptiveCameraHorrorProfileDataAsset`. Overrides cached on `UAdaptiveCameraHorrorComponent` persist across `SwitchProfile(...)` transitions and camera modifier re-registration:

| Blueprint / C++ Function | Signature | Behavior |
| :--- | :--- | :--- |
| **`SetMasterIntensityOverride`** | `(float NewIntensity01)` | Replaces profile `MasterIntensity` with a clamped `[0..1]` override without mutating the Data Asset. |
| **`ClearMasterIntensityOverride`** | `()` | Clears the `MasterIntensity` override and restores the blended profile value. |
| **`SetRuntimeMasterIntensityScale`** | `(float Scale01)` | Applies a multiplicative `[0..1]` runtime scalar (default `1.0`) on top of effective `MasterIntensity`. |
| **`ClearRuntimeMasterIntensityScale`** | `()` | Resets `RuntimeMasterIntensityScale` back to `1.0`. |
| **`GetEffectiveMasterIntensity`** *(pure)* | `() const -> float` | Returns effective `MasterIntensity` after override and multiplicative runtime scaling. |
| **`GetRuntimeMasterIntensityScale`** *(pure)* | `() const -> float` | Returns the current multiplicative runtime `MasterIntensity` scalar `[0..1]`. |
| **`HasMasterIntensityOverride`** *(pure)* | `() const -> bool` | Returns `true` when a runtime `MasterIntensity` override is active. |
| **`SetAccessibilityOverrides`** | `(const FAdaptiveCameraHorrorAccessibilitySettings& NewSettings)` | Overrides all accessibility comfort fields (`MotionIntensityScale`, `CameraRollScale`, `ChromaticAberrationScale`, `bReduceFlashIntensity`). |
| **`ClearAccessibilityOverrides`** | `()` | Clears all runtime accessibility overrides and restores blended profile values. |
| **`SetMotionIntensityScale`** | `(float NewScale)` | Overrides `MotionIntensityScale` (`[0..1]`; `0.0` zeroes all procedural translation, pitch, yaw, stride, turn inertia, and FOV pulse). |
| **`SetCameraRollScale`** | `(float NewScale)` | Overrides `CameraRollScale` (`[0..1]`; `0.0` zeroes all procedural camera roll, stride roll, turn inertia roll, and Dutch tilt). |
| **`SetChromaticAberrationScale`** | `(float NewScale)` | Overrides `ChromaticAberrationScale` (`[0..1]`) for the HLSL pass. |
| **`SetReduceFlashIntensity`** | `(bool bReduce)` | Overrides `bReduceFlashIntensity` to attenuate sudden flash, vignette, and VHS glitch bursts. |
| **`GetEffectiveAccessibilitySettings`** *(pure)* | `() const -> FAdaptiveCameraHorrorAccessibilitySettings` | Returns effective accessibility comfort settings after runtime overrides are applied. |
| **`HasAccessibilityOverride`** *(pure)* | `() const -> bool` | Returns `true` when any runtime accessibility override is active. |
| **`SetNightVisionIntensityOverride`** | `(float NewIntensity01)` | Overrides HLSL IR `NightVisionIntensity` (`[0..1]`) at runtime without mutating any Data Asset. |
| **`ClearNightVisionIntensityOverride`** | `()` | Clears the runtime IR `NightVisionIntensity` override and restores the blended profile value. |
| **`GetEffectiveNightVisionIntensity`** *(pure)* | `() const -> float` | Returns the current effective IR `NightVisionIntensity` (`[0..1]`) after runtime overrides are applied. |
| **`HasNightVisionIntensityOverride`** *(pure)* | `() const -> bool` | Returns `true` when a runtime IR `NightVisionIntensity` override is active. |

**BlueprintAssignable Component Delegates (`UAdaptiveCameraHorrorComponent`):**
- `OnHeartbeatPulse(float HeartRateBPM)` - Broadcast on each cardiac systole cycle synced to the active horror pulse rate.
- `OnJumpScareTriggered(FVector ThreatOrigin, float Intensity)` - Broadcast when `TriggerJumpScare` executes on a local player.
- `OnVHSGlitchTriggered(float Intensity, float Duration)` - Broadcast when `TriggerVHSGlitch` executes on a local player.
- `OnFocusHuntStateChanged(bool bIsHuntingActive)` - Broadcast when low-light or panic focus hunting activates or deactivates.

**Animation Notify (`UAdaptiveCameraHorrorAnimNotify`):**
- Place **Adaptive Camera: Horror FX** on player or creature montages with `NotifyAction` (`JumpScare`, `TensionSpike`, `VHSGlitch`), clamped `Intensity`, `Duration`, `ThreatSocketName`, and `MaxAffectRadius`.

---

## 3. Built-In Preset Parameter Tables

All three presets default to `Profile.MasterIntensity = 0.65` with each category `MasterIntensity = 1.0` (`0.0` disables output, `1.0` applies full configured values) and can be applied in C++/Blueprint via `UAdaptiveCameraHorrorProfileDataAsset::ApplyPresetValues(EAdaptiveCameraHorrorPresetArchetype)` or from the **Quick Presets** bar in **Adaptive Camera: Horror**.

### 3.1 Breathing & Pulse (`FAdaptiveCameraHorrorMovementSettings`)

| Parameter | Range (`ClampMin..Max`) | `Found Footage VHS` | `Bodycam Paranormal` | `Psychological Dread` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `BaseHeartRateBPM` (BPM) | `45.0 .. 110.0` | `72.0` | `76.0` | `64.0` |
| `BreathingVerticalAmplitude` (cm) | `0.0 .. 1.5` | `0.42` | `0.38` | `0.30` |
| `HandheldPitchAmplitude` (deg) | `0.0 .. 1.0` | `0.22` | `0.18` | `0.12` |
| `PanicHeartRateBPM` *(Advanced)* | `85.0 .. 175.0` | `138.0` | `148.0` | `124.0` |
| `HandheldYawAmplitude` *(Advanced)* | `0.0 .. 1.0` | `0.18` | `0.14` | `0.10` |
| `HandheldRollAmplitude` *(Advanced)* | `0.0 .. 1.2` | `0.28` | `0.34` | `0.16` |
| `FearTremorAmplitude` *(Advanced)* | `0.0 .. 0.8` | `0.20` | `0.24` | `0.14` |
| `BreathingRatePerMinute` *(Advanced)* | `6.0 .. 32.0` | `15.0` | `17.0` | `13.0` |
| `BreathingPitchAmplitude` *(Advanced)* | `0.0 .. 1.0` | `0.18` | `0.16` | `0.12` |
| `BaseFOV` *(Advanced)* | `60.0 .. 110.0` | `80.0` | `98.0` | `82.0` |
| `bEnableBodycamStride` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `StrideVerticalAmplitude` *(Advanced)* | `0.0 .. 2.5` (cm) | `0.55` | `0.65` | `0.38` |
| `StridePitchAmplitude` *(Advanced)* | `0.0 .. 1.2` (deg) | `0.22` | `0.26` | `0.14` |
| `StrideRollAmplitude` *(Advanced)* | `0.0 .. 1.5` (deg) | `0.28` | `0.36` | `0.16` |
| `StrideCadenceHz` *(Advanced)* | `0.5 .. 4.0` (Hz) | `1.85` | `1.95` | `1.65` |
| `StrideReferenceSpeed` *(Advanced)* | `100.0 .. 1000.0` (cm/s) | `420.0` | `420.0` | `400.0` |
| `bEnableTurnInertia` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `TurnInertiaSensitivity` *(Advanced)* | `0.0 .. 0.08` (deg/(deg/s)) | `0.012` | `0.015` | `0.008` |
| `TurnInertiaMaxRollDegrees` *(Advanced)* | `0.0 .. 6.0` (deg) | `1.8` | `2.2` | `1.1` |
| `TurnInertiaMaxLagDegrees` *(Advanced)* | `0.0 .. 4.0` (deg) | `1.2` | `1.4` | `0.8` |
| `TurnInertiaInterpSpeed` *(Advanced)* | `2.0 .. 30.0` (1/s) | `10.0` | `11.5` | `8.5` |

### 3.2 Stress Vignette & Tension (`FAdaptiveCameraHorrorFearSettings`)

| Parameter | Range (`ClampMin..Max`) | `Found Footage VHS` | `Bodycam Paranormal` | `Psychological Dread` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `ClaustrophobicVignetteBoost` | `0.0 .. 0.45` | `0.22` | `0.24` | `0.26` |
| `SanityFOVDistortion` (deg) | `-8.0 .. 8.0` | `-2.8` | `-2.2` | `-3.8` |
| `LowSanityDutchTiltDegrees` (deg) | `0.0 .. 5.0` | `2.0` | `1.6` | `3.2` |
| `BaseVignetteIntensity` *(Advanced)* | `0.0 .. 0.30` | `0.08` | `0.10` | `0.06` |
| `LowSanityDesaturation` *(Advanced)* | `0.0 .. 0.65` | `0.24` | `0.28` | `0.36` |
| `PanicFOVPulseAmplitude` *(Advanced)* | `0.0 .. 1.8` | `0.60` | `0.70` | `0.75` |
| `JumpScareImpulseDegrees` *(Advanced)* | `0.0 .. 15.0` | `4.5` | `5.5` | `3.5` |
| `JumpScareFOVPunch` *(Advanced)* | `-12.0 .. 12.0` | `-5.5` | `-6.5` | `-4.5` |
| `JumpScareSpringStiffness` *(Advanced)* | `4.0 .. 40.0` | `18.0` | `22.0` | `14.0` |
| `JumpScareSpringDamping` *(Advanced)* | `0.2 .. 2.5` | `0.72` | `0.68` | `0.80` |

### 3.3 Focus Hunting DOF (`FAdaptiveCameraHorrorDOFSettings`)

| Parameter | Range (`ClampMin..Max`) | `Found Footage VHS` | `Bodycam Paranormal` | `Psychological Dread` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `HuntingAmplitudeCm` (cm) | `0.0 .. 280.0` | `80.0` | `70.0` | `95.0` |
| `LowLightHuntingMultiplier` | `1.0 .. 3.5` | `2.0` | `2.3` | `2.4` |
| `ApertureFStop` (base f-stop) | `1.2 .. 16.0` | `3.2` | `4.0` | `2.4` |
| `PanicApertureFStop` *(Advanced)* | `0.7 .. 32.0` | `2.2` | `2.4` | `1.6` |
| `DiaphragmBladeCount` *(Advanced)* | `4 .. 16` | `6` | `5` | `9` |
| `SensorWidthMM` *(Advanced)* | `4.0 .. 70.0` | `12.8` | `8.8` | `36.0` |

### 3.4 Bodycam / VHS HLSL Shader (`FAdaptiveCameraHorrorPostProcessSettings`)

| Parameter | Range (`ClampMin..Max`) | `Found Footage VHS` | `Bodycam Paranormal` | `Psychological Dread` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `FisheyeBarrelDistortion` (k1) | `-0.12 .. 0.15` | `0.022` | `0.065` | `0.012` |
| `VHSTemporalNoise` (HLSL) | `0.0 .. 0.40` | `0.12` | `0.10` | `0.04` |
| `VHSScanlineIntensity` (HLSL) | `0.0 .. 0.35` | `0.10` | `0.08` | `0.00` |
| `LensCubicDistortion` *(Advanced)* | `-0.10 .. 0.10` | `0.008` | `0.025` | `0.004` |
| `VHSChromaticBleed` *(Advanced)* | `0.0 .. 0.60` | `0.12` | `0.08` | `0.00` |
| `VHSTrackingGlitchIntensity` *(Advanced)* | `0.0 .. 0.50` | `0.04` | `0.03` | `0.00` |
| `ClaustrophobicEdgeDarkening` *(Advanced)* | `0.0 .. 0.35` | `0.08` | `0.14` | `0.10` |
| `NightVisionIntensity` *(Advanced)* | `0.0 .. 1.0` | `0.00` | `0.00` | `0.00` |
| `NightVisionOrVHSTint` *(Advanced)* | `FLinearColor` | `(0.96, 0.97, 0.94, 0.30)` | `(0.90, 0.96, 0.92, 0.35)` | `(0.94, 0.96, 1.00, 0.25)` |

### 3.5 Accessibility Comfort Scaling (`FAdaptiveCameraHorrorAccessibilitySettings`)

| Parameter | Range (`ClampMin..Max`) | Default | Effect |
| :--- | :--- | :---: | :--- |
| `MotionIntensityScale` | `0.0 .. 1.0` | `1.0` | Scales procedural camera translation, pitch, yaw, and FOV pulses (`0.0` removes all procedural motion). |
| `CameraRollScale` | `0.0 .. 1.0` | `1.0` | Scales procedural camera roll and low-sanity Dutch tilt (`0.0` zeroes all roll). |
| `ChromaticAberrationScale` | `0.0 .. 1.0` | `1.0` | Scales HLSL chromatic aberration and VHS chroma bleed. |
| `bReduceFlashIntensity` | `bool` | `false` | Attenuates sudden flash, vignette, and VHS tracking spikes during jump scares and bursts. |

---

## 4. Verification, Building & Packaging

### Automation Tests
Run the plugin's C++ automation suite (`AdaptiveCamera.Horror.SpringConvergenceAndBounds`, `AdaptiveCamera.Horror.ProfileLerpAndPresets`, `AdaptiveCamera.Horror.AccessibilityZeroMotion`, `AdaptiveCamera.Horror.RuntimeOverridesAndPersistence`, `AdaptiveCamera.Horror.PassiveVelocitySampling`, `AdaptiveCamera.Horror.BodycamStrideAndTurnInertia`):

```powershell
& "<UE_5.8>\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" "<PathToProject>.uproject" `
  -unattended -nop4 -nosplash -NullRHI `
  -ExecCmds="Automation RunTests AdaptiveCamera.Horror;Quit" `
  -TestExit="Automation Test Queue Empty"
```

### Fab Plugin Packaging
`Config/FilterPlugin.ini` includes `/Config/...`, `/Content/...`, `/Resources/...`, `/Shaders/...`, `/Source/...`, and `/README.md` under `[FilterPlugin]` so `BuildPlugin` packages source, icons, presets, and `Shaders/AdaptiveCameraHorrorFX.usf` without `Binaries/` or `Intermediate/`. Note that `Content/Showcase/` and `Content/Maps/` are optional demonstration assets and can be omitted from final game packaging if only the runtime camera system and presets are needed:

```powershell
# Ensure LINUX_MULTIARCH_ROOT points to Unreal Engine 5.8's v26_clang-20.1.8-rockylinux8 toolchain when compiling Linux targets
& "<UE_5.8>\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin `
  -Plugin="<PathToProject>\Plugins\AdaptiveCamera_Horror\AdaptiveCamera_Horror.uplugin" `
  -Package="<OutputDir>\AdaptiveCamera_Horror" `
  -TargetPlatforms=Win64+Linux -Rocket -StrictIncludes
```
