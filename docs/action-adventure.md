# Adaptive Camera: Action - Adventure - Third-Person Combat Camera & Impact FX (UE 5.8)

**Fab Title:** `Adaptive Camera: Action - Adventure | Lock-On Framing, Rack Focus & Impact FX (C++)`  
**Author:** Ricardo Pinto / Fractum Software  
**Support:** `fractumsoftware@gmail.com`  
**Target Engine:** Unreal Engine 5.8 (`Win64`, `Linux` - SM5/SM6 Desktop Renderers)

Procedural third-person camera modifier, diagonal duel lock-on framing, dialogue/execution rack focus DOF, continuous low-health and healing camera/shader states, accessibility motion controls, animation notifies, and post-tonemap HLSL parry/hit-stop impact shader pass for **Action - Adventure** games in **Unreal Engine 5.8**.

---

## 1. Architecture & Modules

**Adaptive Camera: Action - Adventure** (`Plugins/AdaptiveCamera_Action/`) is a standalone C++ plugin with no external plugin or game-module dependencies. It can be installed on its own or alongside the other plugins in the **Adaptive Camera** suite (`Adaptive Camera: Shooter`, `Adaptive Camera: Horror`, `Adaptive Camera: Strategy - City Building`) in the same project.

- **Public Symbol Prefix:** `UAdaptiveCameraAction*`, `FAdaptiveCameraAction*`, and `EAdaptiveCameraAction*`; primary asset type `AdaptiveCameraActionProfile`.
- **Feature Scope:** Third-person orbital inertia, translation lag, steadicam idle float, speed-driven FOV/cadence framing, diagonal low-angle duel lock-on framing with dynamic Dutch tilt and runtime side flipping, smart exploration autofocus, dialogue/execution rack focus DOF, continuous low-health camera sway and pulsing crimson edge shader, healing relief camera lift and luminous edge-to-center recovery shader, runtime intensity/accessibility overrides, multiplayer local-player isolation, `UAdaptiveCameraActionAnimNotify`, and a post-tonemap HLSL combat shader pass.
- **`MasterIntensity` Scaling:** `UAdaptiveCameraActionProfileDataAsset::MasterIntensity` defaults to `0.65`, while every category struct exposes `MasterIntensity` defaulting to `1.0` (clamped `[0.0 .. 1.0]`). Setting either scalar to `0.0` disables output and skips the RDG shader pass; `1.0` applies full configured values without double attenuation. Runtime overrides on `UAdaptiveCameraActionComponent` and `UAdaptiveCameraActionCameraModifier` layer on top without mutating `UAdaptiveCameraActionProfileDataAsset` assets.
- **Module Breakdown (`AdaptiveCamera_Action.uplugin`):**

| Module | Type | `LoadingPhase` | `PlatformAllowList` | Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **`AdaptiveCameraAction`** | `Runtime` | `PostConfigInit` | `Win64`, `Linux` | Mounts `/Plugin/AdaptiveCamera_Action` (`Shaders/AdaptiveCameraActionFX.usf`), registers `r.AdaptiveCameraAction.Enable` and `r.AdaptiveCameraAction.Debug` CVars, registers `FAdaptiveCameraActionViewExtension` after `FCoreDelegates::GetOnPostEngineInit()`, and provides the runtime classes (`UAdaptiveCameraActionProfileDataAsset`, `UAdaptiveCameraActionCameraModifier`, `UAdaptiveCameraActionComponent`, `UAdaptiveCameraActionAnimNotify`, `AAdaptiveCameraActionShowcasePawn`, and `FAdaptiveCameraAction*` structs). Zero dependencies on `UnrealEd`, `Slate`, `PropertyEditor`, `ToolMenus`, `LevelEditor`, or `CinematicCamera`. |
| **`AdaptiveCameraActionEditor`** | `Editor` | `PostEngineInit` | `Win64`, `Linux` | Registers `FAdaptiveCameraActionEditorStyle` (`Resources/Icon128.png`), registers the dockable **Adaptive Camera: Action - Adventure** panel (Tab ID: `AdaptiveCameraActionStudioTab`) via `FGlobalTabmanager`, adds the **Adaptive Camera: Action - Adventure** toolbar button (`AdaptiveCameraActionStudioToolbarButton`) to `LevelEditor.LevelEditorToolBar.PlayToolBar` (section `AdaptiveCameraActionToolbar`), and provides 6-tab editing, Live Preview, A/B Split Compare, Save as DataAsset, Revert Viewport, and Live PIE impulse/state testing. |

- **Render Thread & HLSL Pipeline (`/Plugin/AdaptiveCamera_Action`):**
  - The global pixel shader `FAdaptiveCameraActionFXPS` (`Shaders/AdaptiveCameraActionFX.usf`, entry point `MainPS`) runs at `EPostProcessingPass::Tonemap` via RDG (`AddDrawScreenPass`).
  - **Console Variables (Render-Thread Safe):**
    - `r.AdaptiveCameraAction.Enable` (default `1`): global kill-switch for the HLSL post-tonemap pass.
    - `r.AdaptiveCameraAction.Debug` (default `0`): draws a compact top-left telemetry meter inside the active `ViewRect` showing live Master Intensity, Radial Blur, Desaturation/Contrast, and Flash/Low-Health/Heal levels.
  - **Per-Player, Split-Screen, MRQ & Editor Isolation:**
    - Supports up to 4 local players (`MaxSupportedLocalPlayers = 4`) across 16 scene-keyed slots (`MaxRuntimeShaderSlots = 16`) via `SetRuntimePassParameters_GameThread(LocalPlayerIndex, Params)` / `ClearRuntimePassParametersForScene_GameThread(InScene, LocalPlayerIndex)` alongside an isolated editor preview slot (`SetEditorPreviewPassParameters_GameThread` / `ClearEditorPreviewPassParameters_GameThread`).
    - `BeginRenderViewFamily` snapshots the complete POD bundle (`RuntimeSlots`, `EditorPreview`, `bCVarEnabled`, `bCVarDebug`) via `ENQUEUE_RENDER_COMMAND`; `SubscribeToPostProcessingPass` rejects scene captures and reflections, routes game/MRQ views by `(Scene, LocalPlayerIndex)`, routes editor views to `EditorPreview`, and captures the resolved POD struct by value in the RDG pass lambda.
  - **Shader Features & `ViewRect` Handling:**
    - Maps `UVAndScreenPos.xy` through `Input_UVViewportMin` and `Input_UVViewportSizeInverse` and clamps all `Texture2DSampleLevel` UVs to `[Input_UVViewportBilinearMin, Input_UVViewportBilinearMax]`.
    - Includes a 12-tap weighted radial blur around `RadialBlurCenterUV`, peripheral R/B chromatic dispersion, luminance desaturation with `AccentTint`, HDR-preserving S-curve hit-stop contrast grading, impact vignette, Perfect Parry edge/ring flash (`ParryFlashIntensity`), continuous low-health pulsing crimson screen edges with progressive desaturation, luminous edge-to-center healing wave with clarity/saturation recovery, and interactive A/B Split Compare with a 1.5px neutral divider.

---

## 2. Installation & Integration

### Step 1 - Enable the Plugin & Add `UAdaptiveCameraActionComponent`
1. Copy `AdaptiveCamera_Action/` into `<YourProject>/Plugins/` and enable it under **Edit -> Plugins -> Camera -> Adaptive Camera: Action - Adventure** or in `<YourProject>.uproject`:
   ```json
   "Plugins": [
     { "Name": "AdaptiveCamera_Action", "Enabled": true }
   ]
   ```
2. Open your player `ACharacter` or `APawn` Blueprint/C++ class, click **Add Component**, and add **Adaptive Camera: Action - Adventure Component** (`UAdaptiveCameraActionComponent`).
3. Dedicated servers (`IsRunningDedicatedServer()`, `NM_DedicatedServer`) and non-local simulated proxies (`!IsLocallyControlled()`, `!IsLocalController()`) are bypassed automatically; only locally controlled players register `UAdaptiveCameraActionCameraModifier` on their `APlayerCameraManager` and push per-player HLSL state.

### Step 2 - Select, Edit, or Create a `UAdaptiveCameraActionProfileDataAsset`
1. **Included Presets (`Content/Presets/`):** Three Data Assets are included (enable **Show Plugin Content** in Content Browser settings):
   - `/AdaptiveCamera_Action/Presets/DA_Action_SoulsLikeLockOn`
   - `/AdaptiveCamera_Action/Presets/DA_Action_HackAndSlash`
   - `/AdaptiveCamera_Action/Presets/DA_Action_CinematicExecution`
2. **Adaptive Camera: Action - Adventure Panel:** Click **Adaptive Camera: Action - Adventure** in the Level Editor Play toolbar (or open **Tools -> Adaptive Camera: Action - Adventure**, Tab ID `AdaptiveCameraActionStudioTab`):
   - Pick a Data Asset from the Asset Registry selector, click **Temp Scratch** for in-memory experimentation, or click **Save as DataAsset...** to save a reusable `UAdaptiveCameraActionProfileDataAsset` into your project `/Game` folder.
   - Load any preset (**Souls-Like Lock-On**, **Hack & Slash**, **Cinematic Execution**) and edit parameters across all 6 tabs (**Third-Person Camera**, **Combat & Lock-On**, **DOF & Rack Focus**, **Impact / Parry Shader**, **Accessibility**, **Profiles**).
   - Enable **Live Preview** for automatic viewport updates on property changes, toggle **A/B Split Compare** with its `[0 .. 1]` divider slider, click **Apply to Viewport** to update `ACineCameraActor` and `APostProcessVolume` (`Ctrl+Z` supported), or click **Revert Viewport** to restore pre-apply actor values and clear editor shader preview.
   - While Play In Editor (PIE) is active, use the **Live PIE Tester** bar (**Test Parry**, **Test Critical Hit**, **Test Hit-Stop**, **Test Rack Focus**, **Test Low Health**, **Test Heal**) to trigger impulses and toggle health states directly on the local player's `UAdaptiveCameraActionComponent`.
3. Assign your profile to `UAdaptiveCameraActionComponent::DefaultProfile` (and set `DefaultProfileBlendTime`, default `0.30 s`).
4. **Included Showcase Level (`/AdaptiveCamera_Action/Maps/L_Showcase`):** Open `Content/Maps/L_Showcase` (with **Show Plugin Content** enabled) to inspect a golden-hour & twilight-indigo basalt duel courtyard with 3 `ACineCameraActor` rigs (`CAM_01_Hero_Showcase`, `CAM_02_Execution_RackFocus`, `CAM_03_Courtyard_Establishing`), dual-target OTS lock-on and rack-focus subjects, an exposure-locked `APostProcessVolume`, and a pre-wired `ShowcasePlayerPawn` (`AAdaptiveCameraActionShowcasePawn` with `USpringArmComponent`) ready for **Live Preview**, **A/B Split Compare**, **Apply / Revert Viewport**, and **PIE / Live PIE Tester**.
   - **Playable `L_Showcase` Controls (`AAdaptiveCameraActionShowcasePawn`):** `[WASD]` Move, `[Shift]` Sprint, `[Space]` Jump, `[Q]` Swap Shoulder (`CameraBoom->SocketOffset.Y`), `[1]` Toggle Lock-On (`LockOnTarget` actor/boss), `[2]` Toggle Rack Focus, `[3]` Critical Hit + Hit-Stop Pulse, `[4]` Parry Impact, `[5]` Flip Lock-On Duel Side (`+1` Right / `-1` Left), `[6]` Toggle Low Health (`0.14` / `1.0`), `[7]` Trigger Heal Pulse, `[Tab]` Cycle Presets, `[0]` Reset (Lock-On, Rack Focus, Duel Side, Health), `[H]` Toggle Controls Legend.
   - **`USpringArmComponent` & Camera Collision Note:** `UAdaptiveCameraActionCameraModifier` cooperates with `USpringArmComponent` setups (`bUsePawnControlRotation = true`) and sweeps the complete procedural offset (including duel lock-on pullback, lateral, and vertical framing offsets) against `ECC_Camera` (`bPreventCameraObstruction`, `CollisionProbeSize`) plus lock-on controller rotation syncing (`bSyncControllerRotation`) without breaking third-person probe sweeps.

> **Packaging Note (`DirectoriesToNeverCook`):** `Content/Showcase/` and `Content/Maps/` are optional demonstration assets included for testing and evaluation. Add `/AdaptiveCamera_Action/Maps` and `/AdaptiveCamera_Action/Showcase` to `DirectoriesToNeverCook` (`Config/DefaultGame.ini` `[/Script/UnrealEd.ProjectPackagingSettings]`) when packaging your final game build; only `Content/Presets/`, `Source/`, `Shaders/`, `Resources/`, and `Config/` are needed for runtime profiles.

### Step 3 - Connect Blueprint Events, Delegates & `UAdaptiveCameraActionAnimNotify`
Call the `BlueprintCallable` / `BlueprintPure` functions on **`UAdaptiveCameraActionComponent`** or place **`UAdaptiveCameraActionAnimNotify`** directly on Animation Montages and Sequences:

| Blueprint / C++ Function | Signature & Defaults | Gameplay Usage |
| :--- | :--- | :--- |
| **`SetLockOnTarget`** | `(AActor* TargetActor)` | Enables diagonal dual-target duel framing, low-angle offset, distance-based pullback, dynamic Dutch tilt, and midpoint DOF tracking (`LockOnMidpointFocusBias`). |
| **`ClearLockOnTarget`** | `()` | Disengages lock-on and returns to orbital exploration framing. |
| **`SetLockOnDuelSide`** | `(float SideSign)` | Flips or centers the lateral duel offset relative to the player-to-enemy axis (`-1.0` = left, `0.0` = centered, `+1.0` = right). Exposed on both Component and CameraModifier. |
| **`GetLockOnDuelSide`** *(pure)* | `() const -> float` | Returns the configured duel lateral side sign `[-1.0 .. 1.0]`. Exposed on both Component and CameraModifier. |
| **`SetHealthNormalized`** | `(float Health01)` | Updates normalized health `[0.0 .. 1.0]` (default `1.0`). Values below `LowHealthThreshold` ramp up low-health camera sway, crimson edge pulse, and desaturation. Exposed on both Component and CameraModifier. |
| **`GetHealthNormalized`** *(pure)* | `() const -> float` | Returns the current normalized health `[0.0 .. 1.0]`. Exposed on both Component and CameraModifier. |
| **`GetLowHealthAlpha`** *(pure)* | `() const -> float` | Returns the smoothed low-health severity alpha `[0.0 .. 1.0]` below `LowHealthThreshold`. Exposed on both Component and CameraModifier. |
| **`IsLowHealth`** *(pure)* | `() const -> bool` | Returns `true` when `bEnableLowHealthEffect` is active and normalized health is below `LowHealthThreshold`. Exposed on both Component and CameraModifier. |
| **`TriggerHealEffect`** | `(float Intensity = 1.0f, float DurationOverride = -1.0f)` | Triggers upward camera relief lift, damped FOV relaxation, temporary low-health sway attenuation, and a luminous edge-to-center recovery shader wave. Exposed on both Component and CameraModifier. |
| **`GetHealEffectAlpha`** *(pure)* | `() const -> float` | Returns the current healing pulse envelope alpha `[0.0 .. 2.0]`. Exposed on both Component and CameraModifier. |
| **`StartRackFocus`** | `(float TargetDistanceCm, float TargetApertureFStop = 2.2f, float Duration = -1.0f)` | Transitions focal distance and aperture (e.g., executions or dialogues; `Duration < 0` holds until `ClearRackFocus`). |
| **`ClearRackFocus`** | `()` | Releases manual rack focus and restores exploration or lock-on focus distance. |
| **`TriggerParryImpact`** | `(const FVector& WorldImpactLocation, float Intensity = 1.0f)` | Triggers parry FOV impulse (`ParryFOVPunch`), radial zoom blur centered at `WorldImpactLocation`, parry ring flash, desaturation, and HLSL accent tint. |
| **`TriggerCriticalHitImpact`** | `(const FVector& WorldImpactLocation, float Intensity = 1.0f)` | Triggers damped spring punch, S-curve contrast boost (`CriticalHitContrastBoost`), and radial impact blur. |
| **`TriggerHitStopPulse`** | `(float Duration = 0.18f, float Intensity = 1.0f)` | Short hit-stop pulse combining rotational micro-jolt and HLSL impact framing. |
| **`SwitchProfile`** | `(UAdaptiveCameraActionProfileDataAsset* NewProfile, float BlendTime = -1.0f)` | Interpolates between profiles (`BlendTime < 0` uses `DefaultProfileBlendTime`). |
| **`GetCameraModifier`** *(pure)* | `() const -> UAdaptiveCameraActionCameraModifier*` | Returns the active camera modifier managed by the component (or `nullptr`). |

**BlueprintAssignable Event Delegates on `UAdaptiveCameraActionComponent`:**
- `OnParryImpactTriggered(FVector WorldImpactLocation, float Intensity)`
- `OnCriticalHitTriggered(FVector WorldImpactLocation, float Intensity)`
- `OnHitStopTriggered(float Duration, float Intensity)`
- `OnLockOnTargetChanged(AActor* TargetActor)`
- `OnRackFocusChanged(bool bIsActive, float TargetDistanceCm, float TargetApertureFStop)`
- `OnLowHealthStateChanged(bool bIsLowHealth)`
- `OnHealEffectTriggered(float Intensity, float Duration)`

**Animation Notify (`UAdaptiveCameraActionAnimNotify`):**
- Listed in AnimNotify pickers as **Adaptive Camera: Action - Adventure FX** (`Action - Adventure FX: <Action>`).
- Actions (`EAdaptiveCameraActionNotifyAction`): `ParryImpact`, `CriticalHit`, `HitStopPulse`, `StartRackFocus`, `ClearRackFocus`, `HealEffect`.
- Supports optional `ImpactSocketName` lookup on the owning `USkeletalMeshComponent` for world-space radial blur centers, and optional `HealDurationOverride` (`-1.0` uses profile `HealEffectDuration`) for `HealEffect`.

### Runtime Overrides API (`Category = "Adaptive Camera|Runtime"`)

Exposed identically as `BlueprintCallable` / `BlueprintPure` on both **`UAdaptiveCameraActionComponent`** and **`UAdaptiveCameraActionCameraModifier`**. Runtime overrides apply non-destructively in memory without modifying `UAdaptiveCameraActionProfileDataAsset` assets, and component-side overrides persist across `SwitchProfile(...)` blends and modifier re-registration:

| Blueprint / C++ Function | Signature | Behavior |
| :--- | :--- | :--- |
| **`SetMasterIntensityOverride`** | `(float NewIntensity01)` | Replaces the profile `MasterIntensity` at runtime (`[0.0 .. 1.0]`). |
| **`ClearMasterIntensityOverride`** | `()` | Clears the absolute override and reverts to the active profile `MasterIntensity`. |
| **`SetRuntimeMasterIntensityScale`** | `(float Scale01)` | Applies a multiplicative scale `[0.0 .. 1.0]` (default `1.0`) on top of the base or overridden `MasterIntensity`. |
| **`ClearRuntimeMasterIntensityScale`** | `()` | Resets the multiplicative runtime scale back to `1.0`. |
| **`GetEffectiveMasterIntensity`** *(pure)* | `() const -> float` | Returns the resolved `MasterIntensity` after applying overrides and scale. |
| **`GetRuntimeMasterIntensityScale`** *(pure)* | `() const -> float` | Returns the active multiplicative runtime scale `[0.0 .. 1.0]`. |
| **`HasMasterIntensityOverride`** *(pure)* | `() const -> bool` | Returns `true` when `SetMasterIntensityOverride` is active. |
| **`SetAccessibilityOverrides`** | `(const FAdaptiveCameraActionAccessibilitySettings& NewSettings)` | Overrides all four accessibility settings at runtime. |
| **`ClearAccessibilityOverrides`** | `()` | Clears all runtime accessibility overrides and restores profile defaults. |
| **`SetMotionIntensityScale`** | `(float NewScale)` | Overrides procedural translation, pitch, yaw, roll, and FOV scaling (`[0.0 .. 1.0]`). |
| **`SetCameraRollScale`** | `(float NewScale)` | Overrides procedural camera roll scaling (`[0.0 .. 1.0]`). |
| **`SetChromaticAberrationScale`** | `(float NewScale)` | Overrides peripheral RGB chromatic dispersion scaling (`[0.0 .. 1.0]`). |
| **`SetReduceFlashIntensity`** | `(bool bReduce)` | Overrides peak flash and contrast reduction for photosensitivity comfort. |
| **`GetEffectiveAccessibilitySettings`** *(pure)* | `() const -> FAdaptiveCameraActionAccessibilitySettings` | Returns the resolved accessibility settings after runtime overrides. |
| **`HasAccessibilityOverride`** *(pure)* | `() const -> bool` | Returns `true` if any runtime accessibility override is active. |

---

## 3. Built-In Preset Parameter Tables

All three presets default to `Profile.MasterIntensity = 0.65` with each category `MasterIntensity = 1.0` and can be applied in C++/Blueprint via `UAdaptiveCameraActionProfileDataAsset::ApplyPresetValues(EAdaptiveCameraActionPresetArchetype)` or from the **Quick Presets** bar in **Adaptive Camera: Action - Adventure**.

### 3.1 Third-Person Camera (`FAdaptiveCameraActionThirdPersonSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Souls-Like Lock-On` | `Hack & Slash` | `Cinematic Execution` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `BaseFOV` (deg) | Visible | `55.0 .. 105.0` | `78.0` | `88.0` | `72.0` |
| `ShoulderHorizontalOffset` (cm) | Visible | `-75.0 .. 75.0` | `24.0` | `14.0` | `34.0` |
| `OrbitalYawInertiaSpeed` | Visible | `3.0 .. 25.0` | `9.5` | `14.0` | `7.0` |
| `bEnableOrbitalInertia` | Advanced | `bool` | `true` | `true` | `true` |
| `OrbitalPitchInertiaSpeed` | Advanced | `3.0 .. 25.0` | `10.5` | `15.0` | `8.0` |
| `FOVInterpSpeed` | Advanced | `1.0 .. 25.0` | `7.0` | `11.0` | `5.5` |
| `CameraLagTranslationSpeed` | Advanced | `2.0 .. 30.0` | `11.0` | `15.0` | `8.5` |
| `MaxLagOffsetDistance` (cm) | Advanced | `0.0 .. 75.0` | `24.0` | `32.0` | `28.0` |
| `MaxReferenceSpeed` (cm/s) | Advanced | `150.0 .. 1500.0` | `600.0` | `750.0` | `540.0` |
| `ShoulderVerticalOffset` (cm) | Advanced | `-35.0 .. 50.0` | `8.0` | `12.0` | `6.0` |
| `bEnableSteadicamIdleFloat` | Advanced | `bool` | `true` | `true` | `true` |
| `SteadicamFloatAmplitudeCm` (cm) | Advanced | `0.0 .. 6.0` | `1.15` | `0.85` | `1.45` |
| `SteadicamFloatPitchYawDeg` (deg) | Advanced | `0.0 .. 2.0` | `0.22` | `0.16` | `0.28` |
| `SteadicamFloatFrequencyHz` (Hz) | Advanced | `0.05 .. 2.0` | `0.28` | `0.32` | `0.24` |
| `SpeedFOVExpansion` (deg) | Advanced | `0.0 .. 12.0` | `2.5` | `3.5` | `2.0` |
| `RunCadenceWeight` | Advanced | `0.0 .. 1.0` | `0.45` | `0.35` | `0.55` |

### 3.2 Combat & Lock-On (`FAdaptiveCameraActionCombatLockOnSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Souls-Like Lock-On` | `Hack & Slash` | `Cinematic Execution` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `LockOnPitchBiasDegrees` (deg) | Visible | `-15.0 .. 18.0` | `2.5` | `-6.0` | `4.5` |
| `LockOnBasePullbackCm` (cm) | Visible | `0.0 .. 160.0` | `55.0` | `80.0` | `42.0` |
| `LockOnDistancePullbackFactor` | Visible | `0.0 .. 0.30` | `0.10` | `0.14` | `0.08` |
| `bEnableDualTargetFraming` | Advanced | `bool` | `true` | `true` | `true` |
| `bSyncControllerRotation` | Advanced | `bool` | `true` | `true` | `true` |
| `LockOnRotationInterpSpeed` | Advanced | `1.5 .. 25.0` | `8.0` | `11.0` | `6.5` |
| `LockOnFOVBoost` (deg) | Advanced | `-6.0 .. 8.0` | `2.5` | `4.0` | `1.5` |
| `ParryFOVPunch` (deg) | Advanced | `-6.0 .. 6.0` | `-2.8` | `-3.6` | `-4.0` |
| `LockOnDuelLateralOffsetCm` (cm) | Advanced | `-120.0 .. 120.0` | `52.0` | `24.0` | `68.0` |
| `LockOnVerticalOffsetCm` (cm) | Advanced | `-80.0 .. 80.0` | `-18.0` | `6.0` | `-26.0` |
| `LockOnDynamicDutchTiltDegrees` (deg) | Advanced | `0.0 .. 8.0` | `2.4` | `1.6` | `3.5` |

- **Diagonal Duel Lock-On Composition:** When `LockOnDuelLateralOffsetCm` or `LockOnVerticalOffsetCm` is non-zero, lock-on shifts the camera off the player-to-enemy axis along `DuelRight` (flippable via `SetLockOnDuelSide`) and lowers or raises camera height so both combatants read side-by-side without the player's back obstructing the opponent. Positive `LockOnPitchBiasDegrees` tilts the camera upward from a low angle to emphasize larger enemies and bosses, while `LockOnDynamicDutchTiltDegrees` adds damped lateral roll driven by duel stance and strafing velocity (strictly scaled by `MotionIntensityScale * CameraRollScale`). The full duel position offset passes through the `ECC_Camera` sphere sweep (`bPreventCameraObstruction`).

### 3.3 DOF & Rack Focus (`FAdaptiveCameraActionDOFSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Souls-Like Lock-On` | `Hack & Slash` | `Cinematic Execution` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `RackFocusApertureFStop` (f-stop) | Visible | `1.2 .. 16.0` | `2.2` | `2.8` | `1.8` |
| `ExecutionFocusDistance` (cm) | Visible | `50.0 .. 1500.0` | `240.0` | `280.0` | `210.0` |
| `DialogueFocusDistance` (cm) | Visible | `50.0 .. 1200.0` | `185.0` | `210.0` | `165.0` |
| `bEnableDynamicDOF` | Advanced | `bool` | `true` | `true` | `true` |
| `DefaultExplorationFocusDistance` (cm) | Advanced | `100.0 .. 20000.0` | `750.0` | `950.0` | `600.0` |
| `LockOnMidpointFocusBias` | Advanced | `0.0 .. 1.0` | `0.70` | `0.65` | `0.78` |
| `RackFocusInterpSpeed` | Advanced | `1.0 .. 25.0` | `8.5` | `11.0` | `7.0` |
| `MinFocusDistance` (cm) | Advanced | `15.0 .. 250.0` | `45.0` | `55.0` | `35.0` |
| `ApertureFStop` (f-stop) | Advanced | `1.4 .. 22.0` | `5.6` | `8.0` | `4.0` |
| `DiaphragmBladeCount` (blades) | Advanced | `4 .. 16` | `7` | `6` | `9` |
| `SensorWidthMM` (mm) | Advanced | `8.0 .. 50.0` | `24.576` | `24.576` | `36.0` |
| `bEnableExplorationAutofocus` | Advanced | `bool` | `true` | `true` | `true` |
| `bUseRaycastAutofocusInExploration` | Advanced | `bool` | `false` | `false` | `false` |
| `AutofocusTraceChannel` | Advanced | `ECollisionChannel` | `ECC_Visibility` | `ECC_Visibility` | `ECC_Visibility` |
| `ExplorationAutofocusMaxDistance` (cm) | Advanced | `200.0 .. 20000.0` | `4000.0` | `4500.0` | `3500.0` |

- **Smart Exploration Autofocus Behavior:** During free exploration (when neither lock-on nor manual rack focus is active), enabling `bEnableExplorationAutofocus` tracks the focal distance from the camera to the active `ViewTarget` actor (`ResolveActorFocalPoint`) so a third-person character remains in focus dynamically. Enabling `bUseRaycastAutofocusInExploration` instead casts a line trace along the camera forward vector on `AutofocusTraceChannel` up to `ExplorationAutofocusMaxDistance` (ignoring the camera owner and view target), falling back to the `ViewTarget` distance or `DefaultExplorationFocusDistance` when no blocking hit occurs. Disabling `bEnableExplorationAutofocus` uses `DefaultExplorationFocusDistance`.

### 3.4 Impact / Parry, Low-Health & Healing HLSL Shader (`FAdaptiveCameraActionPostProcessSettings`)

| Parameter | Visibility | Range (`ClampMin..Max`) | `Souls-Like Lock-On` | `Hack & Slash` | `Cinematic Execution` |
| :--- | :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | Visible | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `RadialBlurMaxStrength` | Visible | `0.0 .. 0.55` | `0.22` | `0.30` | `0.26` |
| `ParryDesaturation` | Visible | `0.0 .. 0.75` | `0.38` | `0.30` | `0.45` |
| `CriticalHitContrastBoost` | Visible | `0.0 .. 0.60` | `0.22` | `0.28` | `0.30` |
| `bEnableImpactShaderPass` | Advanced | `bool` | `true` | `true` | `true` |
| `ImpactEdgeChromaticShift` | Advanced | `0.0 .. 0.40` | `0.00` | `0.00` | `0.00` |
| `CombatImpactVignetteBoost` | Advanced | `0.0 .. 0.35` | `0.14` | `0.12` | `0.18` |
| `ImpactFlashDuration` (s) | Advanced | `0.05 .. 0.80` | `0.24` | `0.20` | `0.30` |
| `ParryAccentTint` (RGBA) | Advanced | `FLinearColor` | `(0.88, 0.94, 1.00, 1.0)` | `(1.00, 0.90, 0.72, 1.0)` | `(0.88, 0.94, 1.00, 1.0)` |
| `bEnableLowHealthEffect` | Advanced | `bool` | `true` | `true` | `true` |
| `LowHealthThreshold` | Advanced | `0.05 .. 0.60` | `0.30` | `0.30` | `0.30` |
| `LowHealthVignetteColor` (RGBA) | Advanced | `FLinearColor` | `(0.55, 0.03, 0.04, 1.0)` | `(0.62, 0.04, 0.05, 1.0)` | `(0.50, 0.02, 0.03, 1.0)` |
| `LowHealthVignetteStrength` | Advanced | `0.0 .. 1.0` | `0.58` | `0.50` | `0.66` |
| `LowHealthDesaturation` | Advanced | `0.0 .. 0.90` | `0.48` | `0.36` | `0.56` |
| `LowHealthPulseBPM` (BPM) | Advanced | `30.0 .. 160.0` | `72.0` | `84.0` | `66.0` |
| `LowHealthSwayAmplitudeDeg` (deg) | Advanced | `0.0 .. 4.0` | `0.85` | `0.60` | `1.10` |
| `LowHealthRollAmplitudeDeg` (deg) | Advanced | `0.0 .. 5.0` | `1.25` | `0.90` | `1.60` |
| `HealEffectTint` (RGBA) | Advanced | `FLinearColor` | `(0.32, 0.92, 0.58, 1.0)` | `(0.26, 0.94, 0.56, 1.0)` | `(0.96, 0.84, 0.42, 1.0)` |
| `HealEffectIntensity` | Advanced | `0.0 .. 1.5` | `0.70` | `0.65` | `0.80` |
| `HealEffectDuration` (s) | Advanced | `0.15 .. 4.0` | `1.10` | `0.90` | `1.35` |
| `HealCameraLiftPitchDeg` (deg) | Advanced | `0.0 .. 6.0` | `1.35` | `1.05` | `1.75` |
| `HealFOVRelaxationDeg` (deg) | Advanced | `0.0 .. 8.0` | `2.20` | `2.60` | `2.40` |

- **Low-Health & Healing Behavior:** Below `LowHealthThreshold`, `GetLowHealthAlpha()` ramps toward `1.0` at `0.0` health, driving organic pitch/yaw/roll sway (scaled by `MasterIntensity`, `MotionIntensityScale`, and `CameraRollScale`) alongside pulsing crimson edges and progressive desaturation in `AdaptiveCameraActionFX.usf` (attenuated by `bReduceFlashIntensity`). Calling `TriggerHealEffect` applies an upward relief pitch lift, damped FOV relaxation, temporary low-health sway attenuation, and a luminous edge-to-center recovery wave with clarity/saturation recovery.

### 3.5 Accessibility (`FAdaptiveCameraActionAccessibilitySettings`)

| Parameter | Range (`ClampMin..Max`) | Default | Effect |
| :--- | :--- | :---: | :--- |
| `MotionIntensityScale` | `0.0 .. 1.0` | `1.0` | Scales procedural translation, pitch, yaw, roll, Dutch tilt, low-health sway, heal camera lift, and FOV impulses (`0.0` removes all procedural motion while keeping voluntary framing). |
| `CameraRollScale` | `0.0 .. 1.0` | `1.0` | Scales procedural camera roll, duel Dutch tilt, and low-health roll sway (`0.0` zeroes all roll). |
| `ChromaticAberrationScale` | `0.0 .. 1.0` | `1.0` | Scales peripheral R/B chromatic dispersion in the HLSL pass. |
| `bReduceFlashIntensity` | `bool` | `false` | Attenuates peak parry ring/edge flash, desaturation pulse, radial blur, contrast bursts, low-health edge pulsing, and healing pulse brightness. |

---

## 4. Building, Automation Tests & Packaging

Built-in C++ automation tests (`WITH_DEV_AUTOMATION_TESTS`) cover:
- `AdaptiveCamera.ActionAdventure.SpringConvergenceAndBounds`
- `AdaptiveCamera.ActionAdventure.ProfileLerpAndPresets`
- `AdaptiveCamera.ActionAdventure.AccessibilityZeroMotion`
- `AdaptiveCamera.ActionAdventure.RuntimeOverridesPersistence`
- `AdaptiveCamera.ActionAdventure.ExplorationAutofocus`
- `AdaptiveCamera.ActionAdventure.ZeroScaleExactDeltaWithBaseFeel`
- `AdaptiveCamera.ActionAdventure.CinematicDuelLockOnFraming`
- `AdaptiveCamera.ActionAdventure.LowHealthStateAndEffects`
- `AdaptiveCamera.ActionAdventure.HealEffectAndSwayAttenuation`

`Config/FilterPlugin.ini` includes `/Config/...`, `/Content/...`, `/Resources/...`, `/Shaders/...`, `/Source/...`, and `/README.md` under `[FilterPlugin]` so `BuildPlugin` packages source, icon, presets, and `Shaders/AdaptiveCameraActionFX.usf` without `Binaries/` or `Intermediate/`:

```powershell
# Ensure LINUX_MULTIARCH_ROOT points to Unreal Engine 5.8's v26_clang-20.1.8-rockylinux8 toolchain when compiling Linux targets
& "<UE_5.8>\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin `
  -Plugin="<PathToProject>\Plugins\AdaptiveCamera_Action\AdaptiveCamera_Action.uplugin" `
  -Package="<OutputDir>\AdaptiveCamera_Action" `
  -TargetPlatforms=Win64+Linux -Rocket -StrictIncludes
```
