# Adaptive Camera: Strategy - City Building (UE 5.8)

**Fab Title:** `Adaptive Camera: Strategy - City Building | Smooth Pan, Tilt-Shift DOF & Map Scan (C++)`  
**Author:** Ricardo Pinto / Fractum Software  
**Support:** `fractumsoftware@gmail.com`  
**Target Engine:** Unreal Engine 5.8 (`Win64`, `Linux` - SM5/SM6 Desktop Renderers)

Procedural top-down camera modifier, zoom-to-pitch curve, non-invasive overlay mode, optional aerial diorama hover, altitude-coupled tilt-shift DOF, runtime overrides, accessibility scaling, true 3D world-space `SceneDepth` tactical scans, customizable directional perimeter map alerts, distance- and altitude-attenuated artillery impact shocks, low-altitude battlefield combat proximity rumble, and a single post-tonemap HLSL shader pass for **strategy, city-building, 4X, and isometric games** in **Unreal Engine 5.8**.

---

## 1. Architecture & Modules

**Adaptive Camera: Strategy - City Building** (`Plugins/AdaptiveCamera_Strategy/`) is a standalone C++ plugin with zero external plugin or game-module dependencies. It can be installed on its own or alongside the other plugins in the **Adaptive Camera** suite in the same project.

- **Public Symbol Prefix:** `UAdaptiveCameraStrategy*`, `FAdaptiveCameraStrategy*`, and `EAdaptiveCameraStrategy*`; primary asset type `AdaptiveCameraStrategyProfile`.
- **Feature Scope:**
  1. **Damped Top-Down Panning & Overlay Mode:** Pan inertia smoothing, momentum tilt, orbital yaw damping, map focus glide, optional non-invasive overlay mode (`bDriveAltitudeAndPitch = false`) that preserves an existing strategy pawn's Z altitude, pitch, and FOV, optional owning pawn XY sync (`bSyncOwningPawnXY`), and optional vertical aerial hover (`AerialHoverAmplitudeCm`, `AerialHoverFrequencyHz`).
  2. **Zoom-to-Pitch Curve:** Altitude zoom, near/far pitch and FOV interpolation, and optional `UCurveFloat`.
  3. **Altitude-Coupled Tilt-Shift DOF:** Dual f-stop altitude coupling (coupled to `CurrentZoomAlpha` or live camera Z in overlay mode), ground autofocus trace, and screen-space bokeh defocus band.
  4. **3D World-Space Tactical Scan (`SceneDepth` + LWC):** Reconstructs camera-relative `TranslatedWorld` coordinates from `SceneDepthTexture` and `View.ClipToTranslatedWorld` so the expanding radial wave (`ScanWorldMaxRadiusCm`), coordinate grid (`WorldGridCellSizeCm`), and reticle conform to terrain relief, buildings, units, and water without swimming while panning or orbiting. Automatically falls back to 2D screen-space scan when `bUseWorldSpace3DScan = false`, when scene depth is unavailable, or when viewing empty sky.
  5. **Customizable & Directional Perimeter Map Alerts:** Supports both profile-colored alerts (`TriggerMapAlert`) and per-event color and duration overrides (`TriggerMapAlertWithColor`) inside the single post-tonemap HLSL pass. Optional camera-space directional projection (`bDirectionalMapAlertEdge`, `DirectionalMapAlertEdgeBias`) biases perimeter glow toward off-screen alert directions, and both entry points broadcast `OnMapAlertTriggered` for audio hookups.
  6. **Distance- and Altitude-Attenuated Artillery Shocks:** `TriggerArtilleryShock` attenuates camera impulse, chromatic fringe, and border flash by 3D distance (`ArtilleryShockInnerRadiusCm` to `ArtilleryShockMaxDistanceCm`, exact `0.0` beyond max distance) and camera altitude/zoom (`bAttenuateArtilleryByAltitude`, `ArtilleryAltitudeFalloffExponent`).
  7. **Battlefield Proximity Rumble & Combat Zone Component:** Genre-agnostic combat micro-vibration system driven by registered world combat zones (`RegisterOrUpdateCombatZone`, `UnregisterCombatZone`, `ClearCombatZones`), global ambient intensity (`SetAmbientCombatIntensity`), or the zero-code `UAdaptiveCameraStrategyCombatZoneComponent` attached to squads, tanks, ships, or battle actors. Imperceptible at high map overview (`CombatRumbleMaxAltitudeCm`), generates non-repeating multi-frequency translation/rotation tremor plus irregular clash impulses when zoomed low near combat, and broadcasts `OnCombatProximityIntensityChanged(float EffectiveProximity01)` for ambient battle audio mixing.
  8. **Runtime Overrides, Accessibility & Multiplayer Isolation:** Non-mutating runtime `MasterIntensity` and accessibility overrides (`Category = "Adaptive Camera|Runtime"`), motion/roll/chromatic aberration scaling (`0.0` guarantees exact zero camera motion or roll), flash reduction, Dedicated Server guards, and 4-player split-screen / PIE / Editor Preview viewport isolation.
- **`MasterIntensity` Scaling:** Every profile asset exposes `MasterIntensity` (default `0.65`) and every feature struct exposes `MasterIntensity` (`[0.0 .. 1.0]`, default `1.0`), where `0.0` disables output and `1.0` applies full configured values without compounding attenuation.
- **Module Breakdown (`AdaptiveCamera_Strategy.uplugin`):**

| Module | Type | `LoadingPhase` | `PlatformAllowList` | Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **`AdaptiveCameraStrategy`** | `Runtime` | `PostConfigInit` | `Win64`, `Linux` | Mounts `/Plugin/AdaptiveCamera_Strategy` (`Shaders/AdaptiveCameraStrategyFX.usf`), registers `r.AdaptiveCameraStrategy.Enable` and `r.AdaptiveCameraStrategy.Debug` CVars, manages `FAdaptiveCameraStrategyViewExtension`, and provides runtime classes (`UAdaptiveCameraStrategyProfileDataAsset`, `UAdaptiveCameraStrategyCameraModifier`, `UAdaptiveCameraStrategyComponent`, `UAdaptiveCameraStrategyCombatZoneComponent`, `UAdaptiveCameraStrategyAnimNotify`, `AAdaptiveCameraStrategyShowcasePawn`, and `FAdaptiveCameraStrategy*` structs). Zero editor UI dependencies. |
| **`AdaptiveCameraStrategyEditor`** | `Editor` | `PostEngineInit` | `Win64`, `Linux` | Registers `FAdaptiveCameraStrategyEditorStyle` (`Resources/Icon128.png`), the dockable Studio panel (`AdaptiveCameraStrategyStudioTab`) via `FGlobalTabmanager`, and the Play toolbar button (`AdaptiveCameraStrategyStudioToolbarButton` under `AdaptiveCameraStrategyToolbar`), supporting 5-tab editing, Live Preview, A/B Split Compare, Save as DataAsset, Revert Viewport, and Live PIE Testing (3D Scan, Map Alert, Colored Alert, Near/Far Artillery, Combat Rumble). |

- **Render Thread & HLSL Pipeline (`/Plugin/AdaptiveCamera_Strategy`):**
  - `Shaders/AdaptiveCameraStrategyFX.usf` runs once at `EPostProcessingPass::Tonemap` via RDG (`AddDrawScreenPass`), clamping all color UV samples to `[Input_UVViewportBilinearMin, Input_UVViewportBilinearMax]` and all `SceneDepthTexture` UV samples via `View.ViewSizeAndInvSize`, `View.ViewRectMin`, `View.BufferSizeAndInvSize`, and `View.BufferBilinearUVMinMax` for Screen Percentage, split-screen, and Movie Render Queue compatibility.
  - Large World Coordinates (LWC) are handled on the Render Thread by converting `ScanOriginWorldLocation` into camera-relative `TranslatedWorld` space and wrapping grid world phase in `double` precision (`PreViewTranslation`) before uploading `float4` shader parameters.
  - Game Thread state is snapshotted into POD buffers per local player (`MaxSupportedLocalPlayers = 4`) and Editor Preview (`SetRuntimePassParameters_GameThread`, `SetEditorPreviewPassParameters_GameThread`), and routed per view in `SubscribeToPostProcessingPass` while ignoring scene captures and reflection captures.
  - Console variables (`ECVF_RenderThreadSafe`):
    - `r.AdaptiveCameraStrategy.Enable` (default `1`): global runtime toggle for the HLSL pass.
    - `r.AdaptiveCameraStrategy.Debug` (default `0`): draws a compact telemetry bar in the top-left corner of the active ViewRect.

---

## 2. Installation & Integration

### Step 1 - Enable the Plugin & Add `UAdaptiveCameraStrategyComponent`
1. Copy `AdaptiveCamera_Strategy/` into `<PathToProject>/Plugins/` and enable it under **Edit -> Plugins -> Camera -> Adaptive Camera: Strategy - City Building**.
2. Open your strategy or city-building camera `APawn` or `APlayerController`, click **Add Component**, and add **Adaptive Camera: Strategy - City Building Component** (`UAdaptiveCameraStrategyComponent`).
3. On local interactive clients (non-dedicated-server), the component automatically registers `UAdaptiveCameraStrategyCameraModifier` on the local `APlayerCameraManager` during `BeginPlay` and cleans it up in `EndPlay`.
4. *(Optional)* Attach **Adaptive Camera: Strategy Combat Zone** (`UAdaptiveCameraStrategyCombatZoneComponent`) to squads, ships, vehicles, or battle actors to drive low-altitude battlefield proximity rumble automatically with zero Blueprint or C++ code.

### Step 2 - Select, Tune, or Save a `UAdaptiveCameraStrategyProfileDataAsset`
1. **Included Presets (`Content/Presets/`):** Three Data Assets are included (enable **Show Plugin Content** in Content Browser settings):
   - `/AdaptiveCamera_Strategy/Presets/DA_Strategy_GrandTacticalRTS`
   - `/AdaptiveCamera_Strategy/Presets/DA_Strategy_DioramaTiltShift4X`
   - `/AdaptiveCamera_Strategy/Presets/DA_Strategy_IsometricSquadCommand`
2. **Studio Panel (`AdaptiveCameraStrategyStudioTab`):** Open from the Level Editor Play toolbar or **Tools -> Adaptive Camera: Strategy - City Building**:
   - Edit parameters across **5 tabs** (**Pan & Zoom**, **Tilt-Shift DOF**, **Tactical Scan**, **Accessibility**, **Profiles**), with 3 to 4 primary visible controls per tab and extended tuning under `AdvancedDisplay`.
   - Enable **Live Preview** for immediate HLSL shader updates (including live 3D `SceneDepth` scan preview) in the Editor Viewport, and use **A/B Split Compare** with the `[0..1]` divider slider to inspect raw vs. processed output.
   - Click **Save as DataAsset...** to save any profile (including **Temp Scratch** edits) to your project's Content Browser.
   - Click **Apply to Viewport** to update an `ACineCameraActor` / `APostProcessVolume` (with `Ctrl+Z` Undo/Redo) and **Revert Viewport** to restore snapshotted actor properties and clear the Editor shader preview.
   - Use the **Live PIE Tester** bar to trigger **Test 3D Scan**, **Test Map Alert**, **Colored Alert**, **Artillery (Near)**, **Artillery (Far)**, and **Combat Rumble** directly in an active Play-In-Editor session.
3. Assign your profile to `UAdaptiveCameraStrategyComponent::DefaultProfile`.
4. **Included Showcase Level (`/AdaptiveCamera_Strategy/Maps/L_Showcase`):** Open `Content/Maps/L_Showcase` (with **Show Plugin Content** enabled) to inspect a miniature architectural metropolis with a reflective canal, central command spire, container harbor, 3 `ACineCameraActor` rigs (`CAM_01_Hero_Showcase`, `CAM_02_Canal_Macro_Diorama`, `CAM_03_Grand_Strategy_Overview`), an exposure-locked `APostProcessVolume`, and a pre-wired `ShowcasePlayerPawn` (`AAdaptiveCameraStrategyShowcasePawn`) configured for **Live Preview**, **A/B Split Compare**, **Apply / Revert Viewport**, and **PIE / Live PIE Tester**.
   - **Playable `L_Showcase` Controls (`AAdaptiveCameraStrategyShowcasePawn`):** `[WASD / Arrow Keys]` Pan, `[Shift]` Fast Pan, `[Mouse Wheel / Z / X]` Zoom, `[Q / E or RMB + Drag]` Orbit Yaw, `[1]` 3D Tactical Scan Pulse, `[2]` Focus Map Glide, `[3]` Map Alert, `[4]` Near Artillery Shock, `[5]` Colored Directional Map Alert, `[6]` Far Artillery Shock, `[7]` Toggle Combat Zone Rumble, `[Tab]` Cycle Presets, `[0]` Reset, `[H]` Toggle Controls Legend.
   - **Owning Pawn XY Sync (`bSyncOwningPawnXY`):** Enable `PanZoomSettings.bSyncOwningPawnXY` when you want the strategy pawn's world X/Y location updated to match damped camera pan coordinates (such as when world streaming volumes or pawn collision bounds follow camera framing).
   > **Packaging Note (`DirectoriesToNeverCook`):** `Content/Showcase/` and `Content/Maps/` are optional demonstration assets used for the included showcase level. Add `/AdaptiveCamera_Strategy/Maps` and `/AdaptiveCamera_Strategy/Showcase` to `DirectoriesToNeverCook` (`Config/DefaultGame.ini` `[/Script/UnrealEd.ProjectPackagingSettings]`) to exclude both directories from final game cooking.

### Step 3 - Connect Blueprint Events, Runtime Overrides & Animation Notifies

| Blueprint / C++ Function | Signature & Defaults | Gameplay Usage |
| :--- | :--- | :--- |
| **`SetTargetZoomAlpha`** | `(float ZoomAlpha01)` | Sets normalized target altitude `[0..1]` (`0 = MinZoomAltitudeCm`, `1 = MaxZoomAltitudeCm`) and fires `OnZoomAltitudeChanged`. |
| **`AddZoomDelta`** | `(float ZoomDelta)` | Adds a zoom increment to target altitude and fires `OnZoomAltitudeChanged`. |
| **`SetPanVelocityInput`** | `(const FVector& WorldPanVelocity)` | Horizontal world pan velocity (cm/s); drives damped pan inertia and momentum tilt. |
| **`AddOrbitalYawInput`** | `(float DeltaYawDegrees)` | Adds smoothed orbital yaw rotation (degrees) via `UAdaptiveCameraStrategyComponent` (`Category = "Adaptive Camera|Input"`) or `UAdaptiveCameraStrategyCameraModifier`. |
| **`FocusMapLocation`** | `(const FVector& TargetWorldLocation)` | Glides camera pan focus to a world map coordinate (`FocusSnapInterpSpeed`). |
| **`TriggerTacticalScanPulse`** | `(const FVector& OriginWorldLocation, float Intensity = 1.0, float DurationOverride = -1.0)` | Launches an expanding 3D world-space (or 2D fallback) HLSL scan wave, world-anchored reticle, and coordinate grid from `OriginWorldLocation` and fires `OnTacticalScanTriggered`. |
| **`TriggerMapAlert`** | `(const FVector& AlertWorldLocation, float Intensity = 1.0)` | Triggers a perimeter HLSL tactical alert border pulse using profile `MapAlertColor` (with optional directional edge bias) and fires `OnMapAlertTriggered`. |
| **`TriggerMapAlertWithColor`** | `(const FVector& AlertWorldLocation, const FLinearColor& ColorOverride, float Intensity = 1.0, float DurationOverride = -1.0)` | Triggers a perimeter HLSL tactical alert pulse with a custom color and optional duration override (`<= 0` uses profile `MapAlertDuration`), applies directional edge bias when enabled, and fires `OnMapAlertTriggered`. |
| **`TriggerArtilleryShock`** | `(const FVector& ImpactWorldLocation, float Intensity = 1.0)` | Triggers a damped camera shock wave attenuated by 3D distance (`ArtilleryShockInnerRadiusCm..ArtilleryShockMaxDistanceCm`, exact `0.0` beyond max) and camera altitude/zoom, and fires `OnArtilleryShockTriggered`. |
| **`ComputeArtilleryShockAttenuation`** *(pure)* | `(const FVector& ImpactWorldLocation) const -> float` | Returns the combined 3D distance and camera altitude attenuation multiplier `[0..1]` for an artillery impact location. |
| **`RegisterOrUpdateCombatZone`** | `(FName ZoneId, const FVector& WorldLocation, float RadiusCm, float Intensity01 = 1.0)` | Registers or updates an active battlefield combat zone driving low-altitude proximity micro-vibrations. Available on both Component and Modifier. |
| **`UnregisterCombatZone`** | `(FName ZoneId)` | Removes a registered battlefield combat zone by ID. Available on both Component and Modifier. |
| **`ClearCombatZones`** | `()` | Clears all registered battlefield combat zones. Available on both Component and Modifier. |
| **`SetAmbientCombatIntensity`** | `(float Intensity01)` | Sets global ambient combat intensity `[0..1]` (altitude-gated by `CombatRumbleMaxAltitudeCm`). Available on both Component and Modifier. |
| **`GetCurrentCombatProximityIntensity`** *(pure)* | `() const -> float` | Returns the current effective combat proximity intensity `[0..1]` after spatial and altitude attenuation. |
| **`SwitchProfile`** | `(UAdaptiveCameraStrategyProfileDataAsset* NewProfile, float BlendTime = -1.0)` | Blends between profiles (`BlendTime < 0` uses `DefaultProfileBlendTime`) while preserving active runtime overrides. |
| **`GetCameraModifier`** *(pure)* | `() const -> UAdaptiveCameraStrategyCameraModifier*` | Returns the registered camera modifier (or `nullptr`). |

#### Delegates (`Category = "Adaptive Camera|Strategy|Events"`)
- `OnTacticalScanTriggered(FVector OriginWorldLocation, float Intensity, float Duration)` (Component)
- `OnMapAlertTriggered(FVector AlertWorldLocation, float Intensity)` (Component - emitted by both `TriggerMapAlert` and `TriggerMapAlertWithColor`)
- `OnArtilleryShockTriggered(FVector ImpactWorldLocation, float Intensity)` (Component)
- `OnZoomAltitudeChanged(float NewZoomAlpha01)` (Component)
- `OnCombatProximityIntensityChanged(float EffectiveProximity01)` (Both Component and Modifier - for driving MetaSounds, Wwise, or FMOD ambient battle audio mixes)

#### Helper Component: `UAdaptiveCameraStrategyCombatZoneComponent`
Attachable `UActorComponent` (`BlueprintSpawnableComponent`, display name `Adaptive Camera: Strategy Combat Zone`) for squads, vehicles, vessels, or battle emitters. Automatically registers its owner's world position with local player strategy cameras, pushes updates at `UpdateIntervalSeconds` (default `0.05s`), and unregisters on deactivation, `EndPlay`, or component teardown (no-op on Dedicated Servers).
- **Properties:** `ZoneId` (`NAME_None` auto-generates a unique ID), `RadiusCm` (default `1400.0`), `Intensity01` (default `1.0`), `bZoneActive` (default `true`), `UpdateIntervalSeconds` *(Advanced)* (default `0.05`).
- **Blueprint Functions:** `SetCombatZoneActive(bool bActive)`, `SetCombatZoneIntensity(float NewIntensity01)`, `SetCombatZoneRadius(float NewRadiusCm)`, `GetResolvedZoneId()`.

#### Runtime Overrides API (`Category = "Adaptive Camera|Runtime"`)
Available on both `UAdaptiveCameraStrategyComponent` and `UAdaptiveCameraStrategyCameraModifier`. Runtime overrides modify effective modifier output without mutating `UAdaptiveCameraStrategyProfileDataAsset` assets and persist across `SwitchProfile(...)` blends and modifier re-registration:

| Function | Signature / Return | Behavior |
| :--- | :--- | :--- |
| **`SetMasterIntensityOverride`** / **`ClearMasterIntensityOverride`** | `(float NewIntensity01)` / `()` | Overrides or restores `Profile.MasterIntensity` `[0..1]` at runtime. |
| **`SetRuntimeMasterIntensityScale`** / **`ClearRuntimeMasterIntensityScale`** | `(float Scale01)` / `()` | Applies or resets (`1.0`) a multiplicative `[0..1]` scale on top of effective `MasterIntensity`. |
| **`GetEffectiveMasterIntensity`** / **`GetRuntimeMasterIntensityScale`** / **`HasMasterIntensityOverride`** *(pure)* | `() const -> float` / `float` / `bool` | Inspects effective `MasterIntensity`, runtime scale, and override state. |
| **`SetAccessibilityOverrides`** / **`ClearAccessibilityOverrides`** | `(const FAdaptiveCameraStrategyAccessibilitySettings& NewSettings)` / `()` | Overrides or clears all accessibility fields at runtime. |
| **`SetMotionIntensityScale`** / **`SetCameraRollScale`** / **`SetChromaticAberrationScale`** | `(float NewScale)` | Individually overrides procedural motion, roll, or chromatic aberration scaling `[0..1]`. |
| **`SetReduceFlashIntensity`** | `(bool bReduce)` | Individually overrides photosensitive peak flash attenuation. |
| **`GetEffectiveAccessibilitySettings`** / **`HasAccessibilityOverride`** *(pure)* | `() const -> FAdaptiveCameraStrategyAccessibilitySettings` / `bool` | Inspects effective accessibility settings and override state. |

- **Animation Notify (`Adaptive Camera: Strategy - City Building FX` / `UAdaptiveCameraStrategyAnimNotify`):** Add to Animation Sequences or Montages to fire `TacticalScanPulse`, `MapAlert`, or `ArtilleryShock` from an optional `OriginSocketName`.

---

## 3. Built-In Preset Parameter Tables

All three presets default to `Profile.MasterIntensity = 0.65` with each category `MasterIntensity = 1.0` and can be applied in C++/Blueprint via `UAdaptiveCameraStrategyProfileDataAsset::ApplyPresetValues(EAdaptiveCameraStrategyPresetArchetype)` or from the **Quick Presets** bar.

### 3.1 Pan & Zoom (`FAdaptiveCameraStrategyPanZoomSettings`)

| Parameter | Range (`ClampMin..Max`) | `Grand Tactical Strategy` | `Diorama Tilt-Shift 4X` | `Isometric Squad Command` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `MaxZoomAltitudeCm` (cm) | `500.0 .. 40000.0` | `7500.0` | `5500.0` | `4200.0` |
| `FarZoomPitchDegrees` (deg) | `-85.0 .. -25.0` | `-68.0` | `-62.0` | `-66.0` |
| `PanMomentumTiltDegrees` (deg) | `0.0 .. 3.5` | `1.4` | `1.1` | `1.8` |
| `MinZoomAltitudeCm` *(Advanced)* | `150.0 .. 8000.0` | `650.0` | `500.0` | `450.0` |
| `NearZoomPitchDegrees` *(Advanced)* | `-80.0 .. -15.0` | `-40.0` | `-36.0` | `-42.0` |
| `NearZoomFOV` / `FarZoomFOV` *(Advanced)* | `30..85` / `25..80` | `60.0` / `50.0` | `52.0` / `44.0` | `58.0` / `50.0` |
| `PanDampingSpeed` *(Advanced)* | `1.0 .. 20.0` | `7.0` | `5.5` | `10.0` |
| `FocusSnapInterpSpeed` *(Advanced)* | `1.0 .. 18.0` | `6.5` | `5.0` | `9.5` |
| `bDriveAltitudeAndPitch` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `AerialHoverAmplitudeCm` *(Advanced)* | `0.0 .. 40.0` | `0.0` | `2.5` | `0.0` |
| `AerialHoverFrequencyHz` *(Advanced)* | `0.05 .. 2.0` | `0.28` | `0.24` | `0.32` |

- **Overlay Mode (`bDriveAltitudeAndPitch = false`):** Preserves the strategy pawn's own Z altitude, pitch, and FOV while continuing to apply pan momentum tilt, orbital yaw offset (`AddOrbitalYawInput`), Tilt-Shift DOF (computing camera-to-ground distance and altitude f-stop coupling from live camera Z and pitch), 3D Tactical Scan, Directional Map Alerts, Artillery Shock, and Combat Proximity Rumble.

### 3.2 Tilt-Shift DOF (`FAdaptiveCameraStrategyTiltShiftDOFSettings`)

| Parameter | Range (`ClampMin..Max`) | `Grand Tactical Strategy` | `Diorama Tilt-Shift 4X` | `Isometric Squad Command` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `FarAltitudeFStop` | `1.4 .. 22.0` | `4.0` | `2.0` | `4.8` |
| `TiltShiftBandFocusWidth` | `0.22 .. 0.88` | `0.50` | `0.38` | `0.58` |
| `TiltShiftEdgeBlurStrength` | `0.0 .. 0.65` | `0.28` | `0.42` | `0.20` |
| `NearAltitudeFStop` *(Advanced)* | `1.4 .. 22.0` | `5.6` | `2.8` | `6.3` |
| `ApertureFStop` *(Advanced)* | `1.4 .. 22.0` | `4.5` | `2.4` | `5.6` |
| `DiaphragmBladeCount` *(Advanced)* | `4 .. 16` | `7` | `9` | `6` |
| `SensorWidthMM` *(Advanced)* | `12.0 .. 65.0` | `36.0` | `44.0` | `24.576` |

### 3.3 Tactical Scan, Alerts, Artillery & Combat Rumble (`FAdaptiveCameraStrategyTacticalScanSettings`)

| Parameter | Range (`ClampMin..Max`) | `Grand Tactical Strategy` | `Diorama Tilt-Shift 4X` | `Isometric Squad Command` |
| :--- | :--- | :---: | :---: | :---: |
| `MasterIntensity` | `0.0 .. 1.0` | `1.00` | `1.00` | `1.00` |
| `ScanPulseIntensity` | `0.0 .. 0.85` | `0.45` | `0.38` | `0.50` |
| `ScanPulseDuration` (s) | `0.3 .. 5.0` | `1.5` | `1.8` | `1.2` |
| `TacticalScanColor` (RGBA) | `FLinearColor` | `(0.14, 0.72, 0.92, 1.0)` | `(0.88, 0.72, 0.30, 1.0)` | `(0.16, 0.82, 0.64, 1.0)` |
| `bUseWorldSpace3DScan` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `ScanWorldMaxRadiusCm` *(Advanced)* | `500.0 .. 50000.0` | `5500.0` | `4200.0` | `3200.0` |
| `WorldGridCellSizeCm` *(Advanced)* | `25.0 .. 5000.0` | `300.0` | `250.0` | `175.0` |
| `ScanGridOverlayIntensity` *(Advanced)* | `0.0 .. 0.60` | `0.22` | `0.18` | `0.28` |
| `GridCellDensity` *(Advanced)* | `4.0 .. 64.0` | `16.0` | `12.0` | `24.0` |
| `ScanRingSharpness` *(Advanced)* | `4.0 .. 36.0` | `16.0` | `12.0` | `20.0` |
| `MapAlertBorderIntensity` *(Advanced)* | `0.0 .. 1.0` | `0.65` | `0.50` | `0.75` |
| `bDirectionalMapAlertEdge` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `DirectionalMapAlertEdgeBias` *(Advanced)* | `0.0 .. 1.0` | `0.75` | `0.70` | `0.80` |
| `ArtilleryShockAmplitudeDegrees` *(Advanced)* | `0.0 .. 6.0` | `1.8` | `1.2` | `2.4` |
| `ArtilleryShockInnerRadiusCm` *(Advanced)* | `0.0 .. 25000.0` | `1200.0` | `1000.0` | `1000.0` |
| `ArtilleryShockMaxDistanceCm` *(Advanced)* | `100.0 .. 100000.0` | `9000.0` | `7000.0` | `6500.0` |
| `bAttenuateArtilleryByAltitude` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `ArtilleryAltitudeFalloffExponent` *(Advanced)* | `0.25 .. 5.0` | `1.6` | `1.8` | `1.4` |
| `bEnableCombatProximityRumble` *(Advanced)* | `bool` | `true` | `true` | `true` |
| `CombatRumbleMaxAltitudeCm` *(Advanced)* | `300.0 .. 15000.0` | `2600.0` | `2200.0` | `2400.0` |
| `CombatRumbleTranslationAmplitudeCm` *(Advanced)* | `0.0 .. 40.0` | `5.5` | `4.0` | `7.0` |
| `CombatRumbleRotationAmplitudeDegrees` *(Advanced)* | `0.0 .. 3.0` | `0.45` | `0.32` | `0.60` |
| `CombatRumbleClashRateHz` *(Advanced)* | `0.0 .. 8.0` | `1.35` | `1.00` | `1.75` |
| `CombatRumbleClashStrength` *(Advanced)* | `0.0 .. 2.0` | `0.65` | `0.50` | `0.80` |

### 3.4 Accessibility (`FAdaptiveCameraStrategyAccessibilitySettings`)

| Parameter | Range (`ClampMin..Max`) | Default | Behavior |
| :--- | :--- | :---: | :--- |
| `MotionIntensityScale` | `0.0 .. 1.0` | `1.0` | Scales procedural translation, pitch, and yaw across pan momentum tilt, aerial hover, artillery shock, and combat proximity rumble (`0.0` disables all procedural camera motion while keeping `OnCombatProximityIntensityChanged` active for audio mixing). |
| `CameraRollScale` | `0.0 .. 1.0` | `1.0` | Scales procedural roll from pan momentum, artillery shocks, and combat rumble (`0.0` guarantees zero roll). |
| `ChromaticAberrationScale` | `0.0 .. 1.0` | `1.0` | Scales shock chromatic fringe (`0.0` disables fringe override). |
| `bReduceFlashIntensity` | `bool` | `false` | Attenuates scan wave and map alert peak brightness for photosensitive players. |

---

## 4. Automation Tests & Packaging

### Automation Tests (`AdaptiveCameraStrategyAutomationTests.cpp`)
Under `#if WITH_DEV_AUTOMATION_TESTS`, ten tests run under the `AdaptiveCamera.Strategy.*` namespace:
- `AdaptiveCamera.Strategy.SpringConvergenceAndBounds`
- `AdaptiveCamera.Strategy.ProfileLerpAndPresets`
- `AdaptiveCamera.Strategy.AccessibilityZeroMotion`
- `AdaptiveCamera.Strategy.RuntimeOverridesAndOrbitalInput`
- `AdaptiveCamera.Strategy.ArtilleryShockZeroGating`
- `AdaptiveCamera.Strategy.OverlayModeAndAerialHover`
- `AdaptiveCamera.Strategy.WorldSpace3DScanParamsAndPOD`
- `AdaptiveCamera.Strategy.ColoredAndDirectionalMapAlert`
- `AdaptiveCamera.Strategy.ArtilleryDistanceAndAltitudeAttenuation`
- `AdaptiveCamera.Strategy.CombatProximityRumbleAndZoneComponent`

Run headless via `UnrealEditor-Cmd.exe`:
```powershell
& "<UE_5.8>\Engine\Binaries\Win64\UnrealEditor-Cmd.exe" "<PathToProject>.uproject" `
  -unattended -nop4 -nosplash -NullRHI `
  -ExecCmds="Automation RunTests AdaptiveCamera.Strategy;Quit" -TestExit="Automation Test Queue Empty"
```

### Packaging (`BuildPlugin`)
`Config/FilterPlugin.ini` includes `/Config/...`, `/Content/...`, `/Resources/...`, `/Shaders/...`, `/Source/...`, and `/README.md` under `[FilterPlugin]` so `BuildPlugin` packages source, presets, icon resources, and `Shaders/AdaptiveCameraStrategyFX.usf` without `Binaries/` or `Intermediate/`. Note that `Content/Showcase/` and `Content/Maps/` are optional demonstration assets and can be omitted from final packaged builds via `DirectoriesToNeverCook` while retaining `Content/Presets/` if your project references the included preset Data Assets:

```powershell
# Ensure LINUX_MULTIARCH_ROOT points to Unreal Engine 5.8's v26_clang-20.1.8-rockylinux8 toolchain when compiling Linux targets
& "<UE_5.8>\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin `
  -Plugin="<PathToProject>\Plugins\AdaptiveCamera_Strategy\AdaptiveCamera_Strategy.uplugin" `
  -Package="<OutputDir>\AdaptiveCamera_Strategy" `
  -TargetPlatforms=Win64+Linux -Rocket -StrictIncludes
```
