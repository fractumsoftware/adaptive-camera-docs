# Adaptive Camera Suite - Official Documentation (Unreal Engine 5.8)

**Publisher:** Fractum Software (`Ricardo Pinto / Fractum Software`)  
**Engine Version:** Unreal Engine `5.8`  
**Supported Platforms:** Windows (`Win64`), Linux (SM5/SM6 Desktop Renderers)  
**Architecture:** Standalone C++ Runtime (`PostConfigInit`) + Editor (`PostEngineInit`) Modules, Physical Depth of Field, and Post-Tonemap RDG HLSL Shaders

---

## Product Line Overview

The **Adaptive Camera** line consists of four standalone Unreal Engine 5.8 plugins. Every plugin compiles and runs independently with zero shared-core plugins, zero cross-plugin dependencies, isolated UHT symbols, dedicated shader mounts (`/Plugin/AdaptiveCamera_*`), non-colliding Console Variables, and its own dockable Editor Studio panel. You can enable a single plugin or install all four side-by-side in the same project.

| Plugin | Perspective Scope | Factory Presets (`3` each) | Documentation |
| :--- | :--- | :--- | :--- |
| **Adaptive Camera: Shooter** (`AdaptiveCamera_Shooter`) | First-Person & Third-Person *(Grenade Throw kinematics First-Person only)* | `Tactical Mil-Sim`, `Arcade FPS`, `Sniper ADS` | [Shooter Documentation](docs/shooter.md) |
| **Adaptive Camera: Horror** (`AdaptiveCamera_Horror`) | **First-Person Only** *(bodycam / found-footage / survival horror rigs)* | `Found Footage VHS`, `Bodycam Paranormal`, `Psychological Dread` | [Horror Documentation](docs/horror.md) |
| **Adaptive Camera: Action - Adventure** (`AdaptiveCamera_Action`) | Third-Person *(cooperates with `USpringArmComponent` rigs)* | `Souls-Like Lock-On`, `Hack & Slash`, `Cinematic Execution` | [Action - Adventure Documentation](docs/action-adventure.md) |
| **Adaptive Camera: Strategy - City Building** (`AdaptiveCamera_Strategy`) | Top-Down / Isometric / Tactical Aerial *(Full Driving or Non-Invasive Overlay Mode)* | `Grand Tactical Strategy`, `Diorama Tilt-Shift 4X`, `Isometric Squad Command` | [Strategy - City Building Documentation](docs/strategy-city-building.md) |

- **Bundle:** `Adaptive Camera: Complete Suite` includes all four plugins above on Epic Games Fab.
- **Release History:** See [CHANGELOG.md](CHANGELOG.md).

---

## 5-Step Quick Start (All Plugins)

1. **Install & Enable:** Install the plugin via Fab / Epic Games Launcher (or copy `Plugins/AdaptiveCamera_<Genre>/` into `<PathToProject>/Plugins/`), open your UE 5.8 project, enable the plugin in **Edit -> Plugins**, and restart the Editor.
2. **Attach Component:** Add the plugin's Actor Component (`UAdaptiveCamera<Genre>Component`) to your Player Character, Pawn, or `APlayerController`. It automatically registers its `UCameraModifier` on local players and skips dedicated servers.
3. **Assign a Preset:** Select one of the 3 included `UDataAsset` presets (`Content/Presets/`) in the component's **Default Profile** slot, or create your own from the Studio panel. All factory presets ship with `Profile.MasterIntensity = 0.65` and category `MasterIntensity = 1.0`.
4. **Tune Live in the Editor Studio Panel:** Open the dockable Studio tab from `PlayToolBar` to adjust primary controls, toggle **Live Preview**, inspect **A/B Split Compare**, save presets as `UDataAsset`, apply/revert viewport previews with `Ctrl+Z`, and test impulses live in PIE.
5. **Explore `L_Showcase`:** Open `/AdaptiveCamera_<Genre>/Maps/L_Showcase` and press Play (`PIE`) to test every mechanic via the self-contained `ShowcasePawn` hotkeys (`[H]` toggles the on-screen controls legend). Exclude `Content/Maps/` and `Content/Showcase/` via `DirectoriesToNeverCook` when packaging your final game.

---

## Shared Architecture & Runtime Guarantees

- **Strict Runtime / Editor Split:** Runtime modules carry zero `UnrealEd`, `Slate`, `ToolMenus`, `LevelEditor`, or `CinematicCamera` dependencies and build cleanly for `Win64` and `Linux` (`Development` and `Shipping`).
- **ViewRect-Safe RDG Post-Process (`EPostProcessingPass::Tonemap`):** Every HLSL shader pass clamps texture UV sampling inside `[Input_UVViewportBilinearMin, Input_UVViewportBilinearMax]` (plus `View.BufferBilinearUVMinMax` for `SceneDepthTexture` in Strategy) for Screen Percentage scaling, up to 4-player local split-screen, PIE, Editor preview (`G` Game View), and Movie Render Queue offline renders (`bIsOfflineRender`).
- **Non-Destructive Runtime & Accessibility Overrides (`Adaptive Camera|Runtime`):**
  - `SetMasterIntensityOverride` / `SetRuntimeMasterIntensityScale`
  - `SetMotionIntensityScale` *(setting `0.0` removes involuntary bob, hover, and screen shake while preserving voluntary framing, ADS, lean, pan, zoom, and lock-on controls)*
  - `SetCameraRollScale` *(setting `0.0` guarantees exact `0.0` camera roll)*
  - `SetChromaticAberrationScale`
  - `SetReduceFlashIntensity` *(plus `SetUseDarkFlashbang` in Shooter)*
- **Console Variables (`ECVF_RenderThreadSafe`):**
  - `r.AdaptiveCameraShooter.Enable` (`1`) / `.Debug` (`0`)
  - `r.AdaptiveCameraHorror.Enable` (`1`) / `.Debug` (`0`)
  - `r.AdaptiveCameraAction.Enable` (`1`) / `.Debug` (`0`)
  - `r.AdaptiveCameraStrategy.Enable` (`1`) / `.Debug` (`0`)

---

## Frequently Asked Questions (FAQ)

**Can I install all four plugins in the same Unreal Engine 5.8 project?**  
Yes. All four plugins use isolated modules, UHT symbol prefixes, `FPrimaryAssetType` identifiers, toolbar tabs, CVars, and shader virtual paths (`/Plugin/AdaptiveCamera_*`).

**Does Adaptive Camera require C++ in my project?**  
You can use every plugin directly in Blueprint-only projects via components, `UDataAsset` presets, `UAnimNotify` classes, and BlueprintCallable functions, or integrate directly in C++ using the exported `*_API` classes.

**Are macOS, Android, or Console platforms supported?**  
Currently verified and supported target platforms are **Windows (`Win64`)** and **Linux (SM5/SM6 Desktop Renderers)** on Unreal Engine `5.8`.

**Does Adaptive Camera: Horror support third-person cameras?**  
No. `Adaptive Camera: Horror` is strictly **First-Person Only** (bodycam, found-footage, and first-person survival horror rigs).

**Can Adaptive Camera: Strategy - City Building work with an existing custom strategy or city-building camera controller?**  
Yes. Set `bDriveAltitudeAndPitch = false` (Overlay Mode) to preserve your existing pawn altitude, pitch, and FOV while layering pan momentum tilt, altitude-coupled tilt-shift DOF, 3D `SceneDepth` tactical scans, perimeter map alerts, artillery shocks, and combat proximity rumble.

---

## Support & Community

- **Discord Support & Community:** https://discord.gg/3ZYSpCJwb
- **Email Support (Studio & Professional Licenses):** `fractumsoftware@gmail.com`
- **X (Updates & Release Notes):** `https://x.com/FractumSoftware`
- **Fab Storefront:** https://www.fab.com/sellers/Fractum%20Software

When reporting an issue, please include your Unreal Engine version (`5.8.x`), target platform (`Win64` or `Linux`), plugin name, reproduction steps, and relevant lines from `Saved/Logs/`.
