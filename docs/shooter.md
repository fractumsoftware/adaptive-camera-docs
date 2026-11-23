# Adaptive Camera: Shooter - First-Person & Third-Person Camera & Optical FX (UE 5.8)

**Fab Title:** `Adaptive Camera: Shooter | Dynamic Motion, DOF & Post-Process (C++)`  
**Author:** Ricardo Pinto / Fractum Software  
**Support:** `fractumsoftware@gmail.com`  
**Target Engine:** Unreal Engine 5.8 (`Win64`, `Linux` - SM5/SM6 Desktop Renderers)

Procedural first-person and third-person shooter camera modifier, raycast autofocus DOF, accessibility scaling, animation notifies, and post-tonemap HLSL optical shader pass (`Tactical Mil-Sim`, `Arcade FPS`, `Sniper ADS`) in **Unreal Engine 5.8**.

---

## 1. Architecture & Modules

**Adaptive Camera: Shooter** (`Plugins/AdaptiveCamera_Shooter/`) is a standalone C++ plugin with no external plugin or game-module dependencies. It can be installed on its own or alongside the other plugins in the **Adaptive Camera** suite (`Adaptive Camera: Horror`, `Adaptive Camera: Action - Adventure`, `Adaptive Camera: Strategy - City Building`) in the same project.

- **Public Symbol Prefix:** `UAdaptiveCameraShooter*`, `FAdaptiveCameraShooter*`, and `EAdaptiveCameraShooter*`; primary asset type `AdaptiveCameraShooterProfile`.
- **Feature Scope:**
  - **Native First-Person & Third-Person Support:** Profile-level `EAdaptiveCameraShooterPerspectiveMode` (`FirstPerson`, `ThirdPerson`) plus runtime overrides (`SetPerspectiveMode`, `SetShoulderSide`, `ToggleShoulderSide`). Third-Person mode provides configurable over-the-shoulder right/up framing, shoulder-swap interpolation, ADS forward zoom and lateral tightening (unattenuated by `MasterIntensity` or `MotionIntensityScale`), full `ECC_Camera` sphere collision sweeps over all 3P translation offsets, lateral shoulder lean without head roll, moderate 3P scaling for walk stride, idle breathing, and recoil kickback, and recursive character/weapon exclusion during raycast autofocus.
  - **Combat & Locomotion Dynamics:** Spring-damper recoil, ADS zoom and dual hip/ADS aperture DOF, grounded landing impact, idle breathing sway, grounded figure-8 walk stride cadence, bounded look turn inertia, optional auto-sprint FOV threshold, explosion shock, and **First-Person-only grenade throw kinematics** (wind-up pullback/roll/yaw, release forward whip + micro FOV pulse, damped recovery, strictly blocked to exact `0.0` delta in `ThirdPerson` mode).
  - **Separated Suppression vs. Damage Impact:** `TriggerSuppression` applies directional flinch, micro-shake, FOV narrowing, and a dark peripheral tunnel vignette + desaturation + optional peripheral edge blur in HLSL (`SuppressionTint`, `SuppressionDesaturation`, `SuppressionEdgeBlur`), never mixed with red damage tint. `TriggerDamageImpact` applies a fast damped-spring directional whiplash and an independent directional crimson edge pulse pointing toward the attacker (`DirectionalDamageTint`, `DamageVignetteStrength`, `DamageImpulseDuration`).
  - **Overrides, Accessibility & Notifies:** Non-destructive runtime overrides persisting across `SwitchProfile` and camera modifier re-registration, accessibility scaling (including Dark Flashbang mode and flash/pulse reduction), `UAdaptiveCameraShooterAnimNotify`, and a post-tonemap HLSL optical shader with A/B split compare.
- **`MasterIntensity` Scaling:** The profile asset (`UAdaptiveCameraShooterProfileDataAsset::MasterIntensity`, default `0.65`) and every category struct (`MasterIntensity`, default `1.0`, range `[0.0 .. 1.0]`) expose independent intensity scalars, where `0.0` disables procedural output and `1.0` applies full configured values without compounding attenuation. Voluntary Third-Person shoulder/ADS framing remains active regardless of `MasterIntensity` or `MotionIntensityScale`.
- **Module Breakdown (`AdaptiveCamera_Shooter.uplugin`):**

| Module | Type | `LoadingPhase` | `PlatformAllowList` | Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **`AdaptiveCameraShooter`** | `Runtime` | `PostConfigInit` | `Win64`, `Linux` | Mounts `/Plugin/AdaptiveCamera_Shooter` (`Shaders/AdaptiveCameraShooterFX.usf`), registers CVars `r.AdaptiveCameraShooter.Enable` and `r.AdaptiveCameraShooter.Debug`, registers `FAdaptiveCameraShooterViewExtension` after `FCoreDelegates::OnPostEngineInit`, and provides `UAdaptiveCameraShooterProfileDataAsset`, `UAdaptiveCameraShooterCameraModifier`, `UAdaptiveCameraShooterComponent`, `UAdaptiveCameraShooterAnimNotify`, `AAdaptiveCameraShooterShowcasePawn`, and `FAdaptiveCameraShooter*` structs. Zero editor dependencies. |
| **`AdaptiveCameraShooterEditor`** | `Editor` | `PostEngineInit` | `Win64`, `Linux` | Registers `FAdaptiveCameraShooterEditorStyle` (`Resources/Icon128.png`), the dockable **Adaptive Camera: Shooter** panel (Tab ID: `AdaptiveCameraShooterStudioTab`) via `FGlobalTabmanager`, and the Play toolbar button (`AdaptiveCameraShooterStudioToolbarButton`) in `LevelEditor.LevelEditorToolBar.PlayToolBar` (`AdaptiveCameraShooterToolbar`). Provides Live Preview (ADS, Suppression, Damage, Flashbang), A/B Split Compare, Save as DataAsset, Undo/Redo Viewport apply/revert, and Live PIE Impulse Testing. |

- **Render Thread, Isolation & HLSL Pipeline (`/Plugin/AdaptiveCamera_Shooter`):**
  - The global pixel shader `FAdaptiveCameraShooterFXPS` (`Shaders/AdaptiveCameraShooterFX.usf`) runs at `EPostProcessingPass::Tonemap` via RDG (`AddDrawScreenPass`).
  - **Per-Viewport & Split-Screen Isolation:** `FAdaptiveCameraShooterViewExtension` isolates runtime slots keyed by `(Scene, LocalPlayerIndex)` (`MaxSupportedLocalPlayers = 4`, `MaxRuntimeShaderSlots = 16`) via `SetRuntimePassParameters_GameThread` from the editor viewport preview slot (`SetEditorPreviewPassParameters_GameThread`). Game/PIE/MRQ views route to matching runtime slots, editor views route to `EditorPreview`, and scene/reflection/planar captures are skipped.
  - **Thread Safety:** `BeginRenderViewFamily` snapshots the full POD bundle (`RuntimeSlots`, `EditorPreview`, `bCVarEnabled`, `bCVarDebug`) via `ENQUEUE_RENDER_COMMAND`, and `SubscribeToPostProcessingPass` captures the resolved POD struct by value inside the pass lambda.
  - **ViewRect Handling:** Maps `UVAndScreenPos.xy` through `Input_UVViewportMin` and `Input_UVViewportSizeInverse` and clamps all sample UVs (including peripheral suppression blur taps and chromatic shift taps) to `[Input_UVViewportBilinearMin, Input_UVViewportBilinearMax]` for Screen Percentage, split-screen, and Movie Render Queue compatibility.
  - **Console Variables:**
    - `r.AdaptiveCameraShooter.Enable` (default `1`, `ECVF_RenderThreadSafe`): globally enables or disables the HLSL post-tonemap pass.
    - `r.AdaptiveCameraShooter.Debug` (default `0`, `ECVF_RenderThreadSafe`): draws a top diagnostic telemetry bar inside the active `ViewRect` showing MasterIntensity, Optics, Suppression, Damage Impact, and Flashbang levels.

---

## 2. Installation & Integration

### Step 1 - Enable the Plugin & Add `UAdaptiveCameraShooterComponent`
1. Copy `AdaptiveCamera_Shooter/` into `<YourProject>/Plugins/` and enable it under **Edit -> Plugins -> Camera -> Adaptive Camera: Shooter** or in `<YourProject>.uproject`:
   ```json
   "Plugins": [
     { "Name": "AdaptiveCamera_Shooter", "Enabled": true }
   ]
   ```
2. Open your player `ACharacter` or `APawn` Blueprint/C++ class, click **Add Component**, and add **Adaptive Camera: Shooter Component** (`UAdaptiveCameraShooterComponent`).
3. The component guards against dedicated servers (`!IsRunningDedicatedServer()`, `GetNetMode() != NM_DedicatedServer`) and non-local pawns (`IsLocallyControlled()`, `IsLocalController()`), automatically registering `UAdaptiveCameraShooterCameraModifier` on the owning local player's `APlayerCameraManager` during `BeginPlay` and unregistering it in `EndPlay`.

### Step 2 - Select, Edit, or Save a `UAdaptiveCameraShooterProfileDataAsset`
1. **Included Presets (`Content/Presets/`):** Three Data Assets are included (enable **Show Plugin Content** in Content Browser settings):
   - `/AdaptiveCamera_Shooter/Presets/DA_Shooter_TacticalMilSim`
   - `/AdaptiveCamera_Shooter/Presets/DA_Shooter_ArcadeFPS`
   - `/AdaptiveCamera_Shooter/Presets/DA_Shooter_SniperADS`
   You can also create new assets via **Right-Click -> Miscellaneous -> Data Asset -> AdaptiveCameraShooterProfileDataAsset**.
2. **Adaptive Camera: Shooter Panel:** Click **Adaptive Camera: Shooter** in the Level Editor Play toolbar (or open **Window / Tools -> Adaptive Camera: Shooter**, Tab ID `AdaptiveCameraShooterStudioTab`):
   - Pick a Data Asset from the Asset Registry dropdown, or click **Temp Scratch** to iterate in memory, then click **Save as DataAsset...** to persist a new asset to Content Browser.
   - Load any preset (**Tactical Mil-Sim**, **Arcade FPS**, **Sniper ADS**) and edit parameters across the 6 tabs (**Movement**, **Combat & ADS**, **DOF & Optics**, **Optics & Flashbang**, **Accessibility**, **Profiles**). Primary parameters (4 per category tab) are shown by default; secondary, Third-Person, Damage Impact, and Grenade Throw tuning is grouped under **Advanced**.
   - Enable **Live Preview** and adjust **Preview ADS**, **Suppression**, **Damage**, **Flashbang**, and **A/B Split Compare** (with the `0.0 .. 1.0` divider slider) to inspect raw vs. processed HLSL output in the Editor Viewport without touching PIE.
   - Use **Apply to Viewport** to apply camera/DOF overrides to `ACineCameraActor` / `APostProcessVolume` (`Ctrl+Z` supported) and **Revert Viewport** to restore pre-apply actor values and clear the shader preview.
   - Use **Live PIE Tester** (**Test Recoil**, **Test Suppression**, **Test Damage**, **Test Explosion**, **Test Grenade**, **Test Flashbang**, **Test Landing**) while Play-In-Editor is running to trigger impulses on the active local player component.
3. Assign your profile to `UAdaptiveCameraShooterComponent::DefaultProfile` (and set `DefaultProfileBlendTime`, default `0.25 s`).
4. **Included Showcase Level (`/AdaptiveCamera_Shooter/Maps/L_Showcase`):** Open `Content/Maps/L_Showcase` (with **Show Plugin Content** enabled) to inspect a dark tactical killhouse & optics range pre-populated with 3 `ACineCameraActor` rigs (`CAM_01_Hero_Showcase`, `CAM_02_Sniper_ADS_Range`, `CAM_03_Trailer_Establishing`), an exposure-locked `APostProcessVolume`, and a pre-wired `ShowcasePlayerPawn` (`AAdaptiveCameraShooterShowcasePawn`) ready for **Live Preview**, **A/B Split Compare**, **Apply / Revert Viewport**, and **PIE / Live PIE Tester**.
   - **Playable `L_Showcase` Controls (`AAdaptiveCameraShooterShowcasePawn`):** `[WASD]` Move, `[Shift]` Sprint, `[Space]` Jump, `[C / Ctrl]` Slide Impact, `[Q / E]` Lean, `[RMB]` Hold ADS, `[LMB]` Fire Recoil, `[V]` Toggle First-Person / Third-Person, `[X]` Swap Third-Person Shoulder (Left/Right), `[1]` Suppression, `[2]` Explosion, `[3]` Flashbang, `[4]` Landing, `[5]` Damage Impact, `[G]` Grenade Throw (First-Person only), `[F]` Hold/Release Grenade Wind-Up (First-Person only), `[Tab]` Cycle Presets, `[0]` Reset, `[H]` Toggle Controls Legend.
   - **First-Person Viewmodel & Shallow DOF Note:** Built-in `ApertureFStop` / `ADSApertureFStop` values use Unreal Engine's cinematic DOF pass. When using shallow ADS apertures (`1.8` - `2.8`) with close first-person weapon/arms meshes, raise `MinFocusDistance`, keep `ADSApertureFStop >= 2.8`, or render viewmodel weapons separately so barrel/iron sights remain sharp during ADS.
   - **Packaging Exclusion (`DirectoriesToNeverCook`):** `Content/Showcase/` and `Content/Maps/` are optional demonstration assets. Add `/AdaptiveCamera_Shooter/Maps` and `/AdaptiveCamera_Shooter/Showcase` to `DirectoriesToNeverCook` under Project Settings -> Packaging (`Config/DefaultGame.ini` `[/Script/UnrealEd.ProjectPackagingSettings]`) to omit showcase maps and test meshes from final cooked builds.

### Step 3 - Connect Blueprint Events, Delegates & Animation Notifies
Call the `BlueprintCallable` functions on **`UAdaptiveCameraShooterComponent`** (or directly on **`UAdaptiveCameraShooterCameraModifier`**) from weapon, movement, combat, or animation Montage logic:

| Blueprint / C++ API | Signature & Defaults | Gameplay Usage |
| :--- | :--- | :--- |
| **`TriggerRecoil`** | `(float Intensity = 1.0, float HorizontalBias = 0.0)` | Spring-damper recoil impulse on fire (Pitch, Yaw, Roll, `-X` kickback scaled in 3P by `ThirdPersonRecoilKickbackScale`, and FOV punch). |
| **`TriggerSuppression`** | `(FVector ThreatOrigin, float Intensity = 1.0)` | Directional flinch, FOV narrowing, micro-shake, and HLSL dark tunnel vignette + desaturation + optional peripheral edge blur (never mixed with red damage tint). |
| **`TriggerDamageImpact`** | `(FVector DamageOrigin, float Intensity = 1.0)` | Fast damped-spring directional whiplash and independent HLSL directional red edge pulse pointing toward `DamageOrigin`. |
| **`TriggerExplosionShock`** | `(FVector ExplosionOrigin, float Intensity = 1.0)` | Directional damped-spring blast roll/pitch/translation jolt, FOV punch, and HLSL chromatic/vignette shock pulse. |
| **`TriggerFlashbang`** | `(FVector FlashOrigin, float Intensity = 1.0, float DurationOverride = -1.0)` | HLSL blinding flash with highlight persistence knee burn-in, desaturation, and bright or dark slate veil (`DurationOverride < 0` uses `FlashbangMaxDuration`). |
| **`TriggerLandingImpact`** | `(float ImpactSpeedZ = 600.0)` | Damped-spring vertical dip and pitch kick on heavy landings (also triggered automatically on `ACharacter` grounded transitions above `LandingImpactThresholdSpeed`). |
| **`TriggerGrenadeThrow`** | `(float Intensity = 1.0)` | **First-Person only:** plays full wind-up pullback/roll/yaw -> forward release whip + micro FOV pulse -> damped spring recovery. Ignored/cancelled with exact `0.0` delta in `ThirdPerson`. |
| **`StartGrenadeWindup`** | `()` | **First-Person only:** ramps and holds grenade wind-up pose for cooked/held throws. Ignored/cancelled in `ThirdPerson`. |
| **`ReleaseGrenadeThrow`** | `(float Intensity = 1.0)` | **First-Person only:** transitions from held wind-up into forward release whip, counter-roll, micro FOV pulse, and spring recovery. Ignored/cancelled in `ThirdPerson`. |
| **`CancelGrenadeWindup`** | `()` | Smoothly relaxes an active grenade wind-up back to neutral without firing a release whip (resets immediately in `ThirdPerson`). |
| **`SetAimingState`** | `(bool bNewAiming)` | Enables/disables **ADS**: zoom transition, `ADSApertureFStop` switch, optic barrel curvature, and Third-Person ADS forward/lateral framing. |
| **`SetSprintState`** | `(bool bNewSprinting)` | Enables/disables sprint FOV expansion (`SprintFOVOffset`). |
| **`SwitchProfile`** | `(UAdaptiveCameraShooterProfileDataAsset* NewProfile, float BlendTime = -1.0)` | Blends between two profiles at runtime (`BlendTime < 0` uses `DefaultProfileBlendTime`). |
| **`SetLeanInput`** *(optional)* | `(float NewLeanInput)` | Normalized lean input in `[-1.0 .. +1.0]`: swept lateral offset + roll in First-Person; lateral over-the-shoulder shift with `0.0` roll in Third-Person. |
| **`GetCameraModifier`** *(pure)* | `() const -> UAdaptiveCameraShooterCameraModifier*` | Returns the active `UAdaptiveCameraShooterCameraModifier` managed by the component (or `nullptr`). |

- **BlueprintAssignable Delegates on `UAdaptiveCameraShooterComponent`:** `OnRecoilTriggered`, `OnSuppressionTriggered`, `OnDamageImpactTriggered`, `OnExplosionShockTriggered`, `OnFlashbangTriggered`, `OnLandingImpactTriggered`, `OnGrenadeThrowTriggered`.
- **Animation Notify (`UAdaptiveCameraShooterAnimNotify`):** Add **Adaptive Camera: Shooter FX** to Animation Sequences or Montages and select `EAdaptiveCameraShooterNotifyAction` (`Recoil`, `Suppression`, `DamageImpact`, `ExplosionShock`, `Flashbang`, `LandingImpact`, `GrenadeThrow`).

### Runtime Overrides & Perspective API (`Category = "Adaptive Camera|Runtime"`)

Both **`UAdaptiveCameraShooterComponent`** and **`UAdaptiveCameraShooterCameraModifier`** expose identical runtime override controls so player settings menus and gameplay states can switch perspective, swap shoulders, scale master intensity, or adjust accessibility options without mutating source `UAdaptiveCameraShooterProfileDataAsset` assets. Overrides cached on `UAdaptiveCameraShooterComponent` persist across `SwitchProfile(...)` transitions and camera modifier re-registration.

| Runtime Override API | Signature | Behavior |
| :--- | :--- | :--- |
| **`SetPerspectiveMode`** | `(EAdaptiveCameraShooterPerspectiveMode NewMode)` | Overrides active perspective (`FirstPerson` or `ThirdPerson`) without mutating the source Data Asset; switching to `ThirdPerson` cancels any active grenade wind-up/throw immediately. |
| **`ClearPerspectiveModeOverride`** | `()` | Clears the runtime perspective override and resumes using the active profile's `PerspectiveMode`. |
| **`GetPerspectiveMode`** *(pure)* | `() const -> EAdaptiveCameraShooterPerspectiveMode` | Returns the effective active perspective mode. |
| **`HasPerspectiveModeOverride`** *(pure)* | `() const -> bool` | Returns `true` when a runtime perspective mode override is active. |
| **`SetShoulderSide`** | `(float SideSign)` | Sets the Third-Person over-the-shoulder side (`< 0` = left shoulder `-1`, `>= 0` = right shoulder `+1`). |
| **`ToggleShoulderSide`** | `()` | Smoothly swaps Third-Person over-the-shoulder framing between right (`+1`) and left (`-1`). |
| **`GetShoulderSide`** *(pure)* | `() const -> float` | Returns the target Third-Person shoulder side sign (`-1.0` or `+1.0`). |
| **`SetMasterIntensityOverride`** | `(float NewIntensity01)` | Overrides profile `MasterIntensity` in `[0.0 .. 1.0]` without editing the Data Asset. |
| **`ClearMasterIntensityOverride`** | `()` | Clears the master intensity override and resumes using the active profile value. |
| **`SetRuntimeMasterIntensityScale`** | `(float Scale01)` | Sets a multiplicative runtime scale `[0.0 .. 1.0]` (default `1.0`) applied on top of master intensity. |
| **`ClearRuntimeMasterIntensityScale`** | `()` | Resets the multiplicative runtime master intensity scale back to `1.0`. |
| **`GetEffectiveMasterIntensity`** *(pure)* | `() const -> float` | Returns effective master intensity after override and multiplicative runtime scale. |
| **`GetRuntimeMasterIntensityScale`** *(pure)* | `() const -> float` | Returns the current multiplicative runtime master intensity scale `[0.0 .. 1.0]`. |
| **`HasMasterIntensityOverride`** *(pure)* | `() const -> bool` | Returns `true` when a runtime master intensity override is active. |
| **`SetAccessibilityOverrides`** | `(const FAdaptiveCameraShooterAccessibilitySettings& NewSettings)` | Overrides all accessibility settings at runtime without modifying the source Data Asset. |
| **`ClearAccessibilityOverrides`** | `()` | Clears all runtime accessibility overrides and restores profile defaults. |
| **`SetMotionIntensityScale`** | `(float NewScale)` | Overrides `MotionIntensityScale` in `[0.0 .. 1.0]`. |
| **`SetCameraRollScale`** | `(float NewScale)` | Overrides `CameraRollScale` in `[0.0 .. 1.0]`. |
| **`SetChromaticAberrationScale`** | `(float NewScale)` | Overrides `ChromaticAberrationScale` in `[0.0 .. 1.0]`. |
| **`SetReduceFlashIntensity`** | `(bool bReduce)` | Overrides `bReduceFlashIntensity` (scales flashbang peak and directional damage edge pulse). |
| **`SetUseDarkFlashbang`** | `(bool bEnableDark)` | Overrides `bUseDarkFlashbang` for dark slate flashbang mode. |
| **`GetEffectiveAccessibilitySettings`** *(pure)* | `() const -> FAdaptiveCameraShooterAccessibilitySettings` | Returns effective accessibility settings with active runtime overrides applied. |
| **`HasAccessibilityOverride`** *(pure)* | `() const -> bool` | Returns `true` when any runtime accessibility override is active. |

---

## 3. Built-In Preset Parameter Tables

All three presets default to `Profile.MasterIntensity = 0.65` with each category `MasterIntensity = 1.0` and `PerspectiveMode = FirstPerson`, and can be applied via `UAdaptiveCameraShooterProfileDataAsset::ApplyPresetValues(EAdaptiveCameraShooterPresetArchetype)` or from the **Quick Presets** bar in **Adaptive Camera: Shooter**.

### 3.1 Movement (`FAdaptiveCameraShooterMovementSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Tactical Mil-Sim` | `Arcade FPS` | `Sniper ADS` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `BaseFOV` (deg) | Visible | `60.0 .. 115.0` | `88.0` | `100.0` | `85.0` |
| `SprintFOVOffset` (deg) | Visible | `0.0 .. 8.0` | `3.5` | `4.5` | `2.8` |
| `MaxStrafeRollDegrees` (deg) | Visible | `0.0 .. 3.0` | `1.0` | `0.6` | `0.8` |
| `MaxReferenceSpeed` (cm/s) | Advanced | `150.0 .. 1500.0` | `550.0` | `750.0` | `520.0` |
| `StrafeRollInterpSpeed` | Advanced | `1.0 .. 25.0` | `9.0` | `16.0` | `8.0` |
| `LeanAngleDegrees` (deg) | Advanced | `0.0 .. 12.0` | `6.0` | `4.5` | `5.5` |
| `LeanHorizontalOffset` (cm) | Advanced | `0.0 .. 35.0` | `16.0` | `12.0` | `15.0` |
| `LeanInterpSpeed` | Advanced | `1.0 .. 25.0` | `11.0` | `18.0` | `10.0` |
| `SprintFOVInterpSpeed` | Advanced | `1.0 .. 20.0` | `7.0` | `12.0` | `6.5` |
| `AutoSprintSpeedThreshold` (cm/s) | Advanced | `0.0 .. 1500.0` | `0.0` | `0.0` | `0.0` |
| `bEnableIdleBreathing` | Advanced | `bool` | `true` | `true` | `true` |
| `IdleBreathingPitchDegrees` (deg) | Advanced | `0.0 .. 2.0` | `0.14` | `0.08` | `0.18` |
| `IdleBreathingYawDegrees` (deg) | Advanced | `0.0 .. 2.0` | `0.09` | `0.05` | `0.12` |
| `IdleBreathingVerticalOffset` (cm) | Advanced | `0.0 .. 5.0` | `0.25` | `0.15` | `0.28` |
| `IdleBreathingFrequency` (Hz) | Advanced | `0.05 .. 2.0` | `0.28` | `0.32` | `0.24` |
| `ADSIdleSwayAttenuation` | Advanced | `0.0 .. 1.0` | `0.30` | `0.35` | `0.20` |
| `bEnableWalkStride` | Advanced | `bool` | `true` | `true` | `true` |
| `WalkStrideVerticalOffset` (cm) | Advanced | `0.0 .. 6.0` | `1.10` | `0.75` | `1.20` |
| `WalkStrideLateralOffset` (cm) | Advanced | `0.0 .. 4.0` | `0.70` | `0.45` | `0.75` |
| `WalkStrideRollDegrees` (deg) | Advanced | `0.0 .. 2.5` | `0.35` | `0.22` | `0.40` |
| `WalkStridePitchDegrees` (deg) | Advanced | `0.0 .. 2.5` | `0.25` | `0.15` | `0.28` |
| `WalkStrideFrequency` (Hz) | Advanced | `0.5 .. 5.0` | `1.85` | `2.20` | `1.70` |
| `ADSStrideAttenuation` | Advanced | `0.0 .. 1.0` | `0.25` | `0.30` | `0.20` |
| `bEnableTurnInertia` | Advanced | `bool` | `true` | `true` | `true` |
| `TurnInertiaSensitivity` | Advanced | `0.0 .. 3.0` | `1.00` | `0.85` | `0.90` |
| `TurnInertiaMaxYawDegrees` (deg) | Advanced | `0.0 .. 4.0` | `0.80` | `0.50` | `0.55` |
| `TurnInertiaMaxPitchDegrees` (deg) | Advanced | `0.0 .. 4.0` | `0.50` | `0.35` | `0.35` |
| `TurnInertiaMaxRollDegrees` (deg) | Advanced | `0.0 .. 5.0` | `1.00` | `0.65` | `0.85` |
| `TurnInertiaDamping` | Advanced | `1.0 .. 30.0` | `10.0` | `16.0` | `12.0` |
| `bEnableLandingImpact` | Advanced | `bool` | `true` | `true` | `true` |
| `LandingImpactPitchDegrees` (deg) | Advanced | `0.0 .. 8.0` | `1.8` | `1.2` | `2.0` |
| `LandingImpactDropDistance` (cm) | Advanced | `0.0 .. 15.0` | `3.5` | `2.4` | `3.8` |
| `LandingImpactThresholdSpeed` (cm/s) | Advanced | `100.0 .. 1500.0` | `380.0` | `420.0` | `360.0` |
| `LandingImpactReferenceSpeed` (cm/s) | Advanced | `200.0 .. 2500.0` | `900.0` | `950.0` | `880.0` |
| `PerspectiveMode` | Advanced | `FirstPerson \| ThirdPerson` | `FirstPerson` | `FirstPerson` | `FirstPerson` |
| `ThirdPersonShoulderOffsetRight` (cm) | Advanced | `0.0 .. 150.0` | `45.0` | `40.0` | `48.0` |
| `ThirdPersonShoulderOffsetUp` (cm) | Advanced | `-60.0 .. 100.0` | `15.0` | `12.0` | `16.0` |
| `ThirdPersonShoulderInterpSpeed` | Advanced | `1.0 .. 30.0` | `10.0` | `14.0` | `9.0` |
| `ThirdPersonADSZoomOffsetForward` (cm) | Advanced | `0.0 .. 200.0` | `55.0` | `45.0` | `65.0` |
| `ThirdPersonADSOffsetRight` (cm) | Advanced | `0.0 .. 150.0` | `32.0` | `30.0` | `28.0` |
| `ThirdPersonCollisionProbeRadius` (cm) | Advanced | `2.0 .. 30.0` | `12.0` | `12.0` | `12.0` |
| `ThirdPersonWalkStrideScale` | Advanced | `0.0 .. 1.0` | `0.45` | `0.40` | `0.45` |
| `ThirdPersonIdleBreathingScale` | Advanced | `0.0 .. 1.0` | `0.50` | `0.45` | `0.45` |
| `ThirdPersonRecoilKickbackScale` | Advanced | `0.0 .. 1.0` | `0.40` | `0.35` | `0.45` |

### 3.2 Combat & ADS (`FAdaptiveCameraShooterCombatSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Tactical Mil-Sim` | `Arcade FPS` | `Sniper ADS` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `RecoilVerticalKick` (deg) | Visible | `0.0 .. 4.5` | `1.25` | `0.85` | `2.20` |
| `ADSFOV` (deg) | Visible | `20.0 .. 95.0` | `65.0` | `78.0` | `32.0` |
| `SuppressionFlinchDegrees` (deg) | Visible | `0.0 .. 3.0` | `1.00` | `0.60` | `1.20` |
| `RecoilHorizontalSpread` (deg) | Advanced | `0.0 .. 2.0` | `0.40` | `0.25` | `0.35` |
| `RecoilRollKick` (deg) | Advanced | `0.0 .. 2.0` | `0.40` | `0.25` | `0.70` |
| `RecoilKickbackDistance` (cm) | Advanced | `0.0 .. 4.5` | `1.40` | `0.85` | `2.20` |
| `RecoilSpringStiffness` | Advanced | `60.0 .. 500.0` | `220.0` | `320.0` | `160.0` |
| `RecoilSpringDamping` | Advanced | `6.0 .. 50.0` | `20.0` | `26.0` | `16.0` |
| `RecoilRecoverySpeed` (1/s) | Advanced | `2.0 .. 25.0` | `11.0` | `18.0` | `7.5` |
| `ADSRecoilMultiplier` | Advanced | `0.1 .. 1.0` | `0.55` | `0.70` | `0.45` |
| `RecoilFOVPunch` (deg) | Advanced | `0.0 .. 2.0` | `0.50` | `0.30` | `0.80` |
| `ADSInterpInSpeed` / `OutSpeed` | Advanced | `2.0 .. 25.0` | `11.0` / `9.5` | `18.0` / `16.0` | `8.0` / `11.0` |
| `SuppressionFOVNarrowing` (deg) | Advanced | `0.0 .. 6.0` | `2.5` | `1.5` | `2.2` |
| `bEnableExplosionShock` | Advanced | `bool` | `true` | `true` | `true` |
| `ExplosionShockPitchDegrees` (deg) | Advanced | `0.0 .. 10.0` | `2.5` | `1.6` | `2.8` |
| `ExplosionShockRollDegrees` (deg) | Advanced | `0.0 .. 12.0` | `3.2` | `2.0` | `3.6` |
| `ExplosionShockOffsetDistance` (cm) | Advanced | `0.0 .. 20.0` | `4.0` | `2.5` | `4.5` |
| `bEnableDamageImpact` | Advanced | `bool` | `true` | `true` | `true` |
| `DamageImpactPitchDegrees` (deg) | Advanced | `0.0 .. 10.0` | `2.2` | `1.6` | `2.4` |
| `DamageImpactRollDegrees` (deg) | Advanced | `0.0 .. 12.0` | `2.8` | `2.0` | `3.0` |
| `DamageImpactYawDegrees` (deg) | Advanced | `0.0 .. 10.0` | `1.8` | `1.4` | `2.0` |
| `DamageImpactSpringStiffness` | Advanced | `80.0 .. 600.0` | `320.0` | `380.0` | `300.0` |
| `DamageImpactSpringDamping` | Advanced | `6.0 .. 60.0` | `24.0` | `28.0` | `22.0` |
| `bEnableGrenadeThrowMotion` | Advanced | `bool` | `true` | `true` | `true` |
| `GrenadeWindupDuration` (s) | Advanced | `0.05 .. 1.0` | `0.24` | `0.16` | `0.28` |
| `GrenadeWindupPullbackDistance` (cm) | Advanced | `0.0 .. 15.0` | `4.0` | `3.0` | `4.5` |
| `GrenadeWindupRollDegrees` (deg) | Advanced | `0.0 .. 10.0` | `3.2` | `2.4` | `3.5` |
| `GrenadeWindupYawDegrees` (deg) | Advanced | `0.0 .. 10.0` | `2.4` | `1.8` | `2.6` |
| `GrenadeReleasePitchDegrees` (deg) | Advanced | `0.0 .. 12.0` | `3.6` | `2.8` | `3.8` |
| `GrenadeReleaseRollDegrees` (deg) | Advanced | `0.0 .. 12.0` | `4.0` | `3.0` | `4.2` |
| `GrenadeReleaseForwardOffset` (cm) | Advanced | `0.0 .. 18.0` | `5.5` | `4.2` | `5.8` |
| `GrenadeReleaseFOVPulse` (deg) | Advanced | `0.0 .. 8.0` | `2.0` | `1.6` | `2.2` |
| `GrenadeThrowSpringStiffness` | Advanced | `40.0 .. 450.0` | `190.0` | `260.0` | `170.0` |
| `GrenadeThrowSpringDamping` | Advanced | `4.0 .. 45.0` | `18.0` | `22.0` | `17.0` |

### 3.3 DOF & Optics (`FAdaptiveCameraShooterDOFSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Tactical Mil-Sim` | `Arcade FPS` | `Sniper ADS` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `bEnableRaycastAutofocus` | Visible | `bool` | `true` | `true` | `true` |
| `ApertureFStop` (Hip f-stop) | Visible | `1.2 .. 22.0` | `6.3` | `11.0` | `7.1` |
| `ADSApertureFStop` (ADS f-stop) | Visible | `1.2 .. 16.0` | `2.8` | `4.8` | `1.8` |
| `MaxFocusTraceDistance` (cm) | Advanced | `200.0 .. 60000.0` | `15000.0` | `20000.0` | `50000.0` |
| `MinFocusDistance` (cm) | Advanced | `15.0 .. 300.0` | `45.0` | `50.0` | `60.0` |
| `DefaultHipFocusDistance` (cm) | Advanced | `100.0 .. 30000.0` | `2200.0` | `3500.0` | `4000.0` |
| `DiaphragmBladeCount` (blades) | Advanced | `4 .. 16` | `7` | `6` | `9` |
| `SensorWidthMM` (mm) | Advanced | `8.0 .. 50.0` | `24.576` | `24.576` | `36.0` |

### 3.4 Optics & Flashbang HLSL Shader (`FAdaptiveCameraShooterPostProcessSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Tactical Mil-Sim` | `Arcade FPS` | `Sniper ADS` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `ADSLensBarrelDistortion` (k1 ADS) | Visible | `-0.08 .. 0.08` | `0.024` | `0.012` | `0.045` |
| `DirectionalVignetteStrength` (Suppression tunnel) | Visible | `0.0 .. 0.65` | `0.28` | `0.22` | `0.30` |
| `FlashbangMaxDuration` (s) | Visible | `0.3 .. 6.0` | `2.5` | `2.0` | `2.8` |
| `LensBarrelDistortion` (k1 Hip) | Advanced | `-0.06 .. 0.06` | `0.0` | `0.0` | `0.0` |
| `LensCubicDistortion` (k2) | Advanced | `-0.05 .. 0.05` | `0.004` | `0.002` | `0.010` |
| `OpticalEdgeChromaticShift` (HLSL) | Advanced | `0.0 .. 0.35` | `0.0` | `0.0` | `0.0` |
| `OpticalEdgeVignette` (HLSL) | Advanced | `0.0 .. 0.35` | `0.06` | `0.05` | `0.10` |
| `SuppressionTint` (RGBA) | Advanced | `FLinearColor` | `(0.03, 0.035, 0.045, 1.0)` | `(0.035, 0.04, 0.05, 1.0)` | `(0.025, 0.03, 0.04, 1.0)` |
| `SuppressionDesaturation` | Advanced | `0.0 .. 1.0` | `0.38` | `0.28` | `0.42` |
| `SuppressionEdgeBlur` | Advanced | `0.0 .. 1.0` | `0.25` | `0.15` | `0.30` |
| `DirectionalDamageTint` (RGBA) | Advanced | `FLinearColor` | `(0.62, 0.08, 0.06, 1.0)` | `(0.75, 0.10, 0.07, 1.0)` | `(0.60, 0.07, 0.05, 1.0)` |
| `DamageVignetteStrength` | Advanced | `0.0 .. 1.0` | `0.48` | `0.52` | `0.50` |
| `DamageImpulseDuration` (s) | Advanced | `0.1 .. 2.5` | `0.55` | `0.45` | `0.60` |
| `FlashbangBurnInStrength` | Advanced | `0.0 .. 1.0` | `0.55` | `0.45` | `0.55` |
| `FlashbangDesaturation` | Advanced | `0.0 .. 1.0` | `0.60` | `0.48` | `0.60` |

### 3.5 Accessibility (`FAdaptiveCameraShooterAccessibilitySettings`)

| Parameter | Range (`ClampMin..Max`) | Default | Behavior |
| :--- | :---: | :---: | :--- |
| `MotionIntensityScale` | `0.0 .. 1.0` | `1.0` | Scales all involuntary procedural translation, pitch, yaw, and motion FOV punches (`0.0` disables involuntary camera motion while preserving voluntary lean and Third-Person framing). |
| `CameraRollScale` | `0.0 .. 1.0` | `1.0` | Scales camera roll/tilt from strafe, lean, recoil, suppression, damage whiplash, grenade throw, and explosion shock (`0.0` zeroes roll). |
| `ChromaticAberrationScale` | `0.0 .. 1.0` | `1.0` | Scales HLSL RGB edge chromatic dispersion (`0.0` disables chromatic shift). |
| `bReduceFlashIntensity` | `bool` | `false` | Scales flashbang peak intensity, burn-in persistence, and directional damage edge pulse down for visual comfort. |
| `bUseDarkFlashbang` | `bool` | `false` | Replaces bright white flashbang overexposure with a dark slate veil `float3(0.05, 0.055, 0.065)`. |

---

## 4. Automation Tests & Fab Packaging

- **C++ Automation Tests (`AdaptiveCamera.Shooter.*`):**
  - `AdaptiveCamera.Shooter.SpringConvergenceAndBounds`
  - `AdaptiveCamera.Shooter.ProfileLerpAndPresets`
  - `AdaptiveCamera.Shooter.AccessibilityZeroMotion`
  - `AdaptiveCamera.Shooter.RuntimeOverridesPersistence`
  - `AdaptiveCamera.Shooter.PassiveBaseFeelAndZeroScale`
  - `AdaptiveCamera.Shooter.ThirdPersonFramingAndAdaptation`
  - `AdaptiveCamera.Shooter.GrenadeThrowFirstPersonOnly`
  - `AdaptiveCamera.Shooter.SuppressionVsDamageSeparation`
- **Packaging (`Config/FilterPlugin.ini`):** Includes `/Config/...`, `/Content/...`, `/Resources/...`, `/Shaders/...`, `/Source/...`, and `/README.md` under `[FilterPlugin]` so `BuildPlugin` packages without `Binaries/` or `Intermediate/`. Note that `Content/Showcase/` and `Content/Maps/` are optional demonstration assets that may be excluded from final game cooking via `DirectoriesToNeverCook`:

```powershell
# Ensure LINUX_MULTIARCH_ROOT points to Unreal Engine 5.8's v26_clang-20.1.8-rockylinux8 toolchain when compiling Linux targets
& "<UE_5.8>\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin `
  -Plugin="<PathToProject>\Plugins\AdaptiveCamera_Shooter\AdaptiveCamera_Shooter.uplugin" `
  -Package="<OutputDir>\AdaptiveCamera_Shooter" `
  -TargetPlatforms=Win64+Linux -Rocket -StrictIncludes
```
