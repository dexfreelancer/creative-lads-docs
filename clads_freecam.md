# CLads Freecam

Cinematic freecam system for FiveM with timecycle filters, camera presets, distance limits, and an anti-exploit wall-detection blackout.

---

## Overview

CLads Freecam is a self-contained, hold-to-toggle cinematic camera designed for screenshot photography, content creation, and roleplay scenes. It is built around a small, predictable control set and ships with security features that prevent the freecam from being misused as a wall-hack.

Core features:

- **Hold-to-toggle activation.** Players hold the activation key (default `V`) for 500ms to enter or exit freecam. The same key handles both directions; a debounce ensures the activation press cannot immediately trigger deactivation.
- **WASD/RF movement.** Strafe with `WASD` and rise/descend with `R` (up) and `F` (down). Hold `SHIFT` for a fast-move multiplier or `CTRL` for slow, precise movement.
- **Mouse look.** Pitch and yaw are driven by mouse input, with pitch clamped to `[-89, +89]` to prevent the camera from flipping.
- **Scroll-wheel zoom.** The camera's field-of-view (FOV) is reduced or expanded via the scroll wheel, clamped to configurable min/max.
- **Q/E tilt.** `Q` and `E` apply a roll axis (camera tilt), clamped to `[-45, +45]` for a controlled cinematic effect.
- **TAB filter menu.** A NUI menu opens on `TAB`, listing every configured visual filter. Hovering a filter previews it live; clicking commits the selection.
- **Camera presets.** A configurable list of named angles (pitch / yaw / roll / FOV) can be cycled programmatically through an export, with smooth ease-out cubic transitions.
- **Distance limits.** The camera cannot drift more than a configurable number of metres from the player on either the horizontal or vertical axis. A throttled notification fires when the limit is reached.
- **Wall-detection blackout (anti-exploit).** A per-frame raycast from the player to the camera detects line-of-sight blockers (walls, vehicles, world objects). When the camera goes behind cover, a fullscreen NUI blackout overlay with a branded warning hides the view.
- **HUD integration.** Optional integration block hides the server's HUD/minimap while freecam is active and restores it on exit, supporting both export-mode and event-mode HUDs.
- **Locale support.** Every player-facing string lives in `locale/<lang>.lua`. English, Turkish, German, and French ship by default; new languages drop in by adding a file.

---

## Requirements

- **FiveM server** (`fx_version 'cerulean'`, `game 'gta5'`, Lua 5.4).
- **ox_lib** — required dependency, used for notifications and the `cache.serverId` helper.

Optional:

- A HUD resource (e.g. `wais-hud`, `okokHUD`, `qb-hud`, or any HUD that exposes an export or net event) for the `HudIntegration` block. The resource works fine without it.

---

## Installation

1. Drop the `clads_freecam` folder into your server's `resources/` directory (or any subfolder included by your `server.cfg`).
2. Ensure `ox_lib` starts before this resource:
   ```cfg
   ensure ox_lib
   ensure clads_freecam
   ```
3. Open `config.lua` and review the settings — at minimum, choose a `DefaultLanguage` and decide whether you want `WallDetection.enabled` on (recommended) or off.
4. (Optional) If your server uses a custom HUD that must be hidden during freecam, configure the `HudIntegration` block (see [Configuration › HudIntegration](#hudintegration)).
5. Restart the server, or `refresh; ensure clads_freecam` from the live console.
6. In-game, hold `V` for 500ms to enter freecam.

---

## Configuration

All settings live in `config.lua`. The file is escrow-ignored, so edits survive any future updates.

### DefaultLanguage

Selects which file is loaded from `locale/`. Must match a filename without the `.lua` extension. Falls back to English if the file is missing or fails to compile.

```lua
DefaultLanguage = 'en',
```

Bundled locales: `en`, `tr`, `de`, `fr`.

### Keys

GTA V control IDs used to activate freecam and open the filter menu. Reference: [FiveM controls list](https://docs.fivem.net/docs/game-references/controls/).

```lua
Keys = {
    activate   = 0,    -- V (INPUT_NEXT_CAMERA)
    filterMenu = 37,   -- TAB (INPUT_SELECT_WEAPON)
},
```

Common alternatives for `activate`:

| Control ID | Default Key | Input name |
|------------|-------------|------------|
| `0`  | V | INPUT_NEXT_CAMERA |
| `47` | G | INPUT_DETONATE |
| `74` | H | INPUT_VEH_HEADLIGHT |
| `51` | E | INPUT_CONTEXT |
| `58` | T | INPUT_THROW_GRENADE |
| `29` | B | INPUT_SPECIAL_ABILITY_SECONDARY |
| `303`| U | INPUT_PICKUP |
| `310`| X | INPUT_VEH_DUCK |
| `311`| K | INPUT_REPLAY_START_STOP_RECORDING |

### HoldDuration

Time in milliseconds the activation key must be held to toggle freecam on or off. Lower values feel snappier but increase the chance of an accidental toggle.

```lua
HoldDuration = 500,
```

### Movement

Per-axis movement tuning. `MoveSpeed` is the base; the multipliers stack with `SHIFT` and `CTRL`.

```lua
MoveSpeed          = 1.0,   -- Base speed for WASD
FastMoveMultiplier = 2.5,   -- Hold SHIFT
SlowMoveMultiplier = 0.3,   -- Hold CTRL
VerticalSpeed      = 0.8,   -- R (up) and F (down)
```

### Rotation

Look and tilt sensitivity.

```lua
RotateSpeed   = 5.0,        -- Mouse look
QERotateSpeed = 2.0,        -- Q / E roll (tilt)
```

Pitch is clamped to `[-89, +89]`, roll to `[-45, +45]`, and yaw wraps at 360.

### Zoom

Scroll-wheel zoom (controls the camera's FOV).

```lua
ZoomSpeed = 3.0,            -- Per scroll tick
CameraFOV = 55.0,           -- Default FOV on enable
MinZoom   = 15.0,           -- Most zoomed in (smallest FOV)
MaxZoom   = 90.0,           -- Most zoomed out (largest FOV)
```

### Distance limits

Caps how far the camera can drift from its initial position when freecam was enabled. Both axes are enforced independently.

```lua
MaxHorizontalDistance = 15.0,   -- Metres on X/Y plane
MaxVerticalDistance   = 15.0,   -- Metres on Z axis
DistanceLimitNotify   = true,   -- Notify the player when blocked
DistanceLimitCooldown = 5000,   -- Min ms between notifications
```

When the player tries to push past the limit, the position update is rejected and (optionally) a throttled error notification fires.

### Camera presets

A list of named angles the player can cycle through programmatically (see [Exports & Events](#exports--events)). Each preset places the camera behind the player at a configurable distance and pitch and looks at the player's chest.

```lua
Presets = {
    { name = "Normal",    pitch = 0.0,  yaw = 0.0, roll = 0.0, fov = 55.0 },
    -- { name = "Close-Up",  pitch = 15.0, yaw = 0.0, roll = 0.0, fov = 35.0 },
    -- { name = "Wide",      pitch = -10.0,yaw = 0.0, roll = 0.0, fov = 75.0 },
    -- { name = "Cinematic", pitch = 5.0,  yaw = 0.0, roll = 2.0, fov = 45.0 },
},
```

Per-preset optional fields: `distance` (metres behind player), `height` (metres above player). Top-down presets (`pitch < -20`) automatically rise higher and pull closer for a clean overhead shot.

### Visual filters

Timecycle modifiers shown in the TAB menu. Each entry needs a unique `id` and a `timecycle` (a GTA V timecycle modifier name). Display names are pulled from `locale/<lang>.lua` under the `filters` table, keyed by `id`.

```lua
DefaultFilter = 'default',
Filters = {
    { id = 'default',    timecycle = 'default' },
    { id = 'vintage',    timecycle = 'CAMERA_BW' },
    { id = 'blackwhite', timecycle = 'blackNwhite' },
    { id = 'hdr',        timecycle = 'Bloom' },
    { id = 'warm',       timecycle = 'hud_def_desat_Trevor' },
    { id = 'contrast',   timecycle = 'hud_def_desatcrunch' },
},
```

Suggested timecycle modifiers (drop these into the `timecycle` field of any new filter):

| Timecycle modifier | Effect |
|--------------------|--------|
| `default`                   | No effect (passthrough) |
| `CAMERA_BW`                 | Sepia / vintage |
| `blackNwhite`               | Black and white |
| `Bloom`                     | Bloom / HDR look |
| `hud_def_desat_Trevor`      | Warm, desaturated |
| `hud_def_desatcrunch`       | High contrast, desaturated |
| `Nightvision`               | Night-vision green |
| `phone_cam`                 | Phone camera look |
| `Prologue_ending_Franklin`  | Dark cinematic |
| `secret_camera`             | Surveillance camera |

The full timecycle list is built into the base game; thousands of modifiers exist. The names above are tested-known-good defaults.

### UI

Filter menu and easing animation toggles.

```lua
ShowFilterMenu = true,      -- TAB opens the filter picker
EnableEasing   = true,      -- Smooth transitions on enable/disable/preset
EasingDuration = 500,       -- Easing duration in ms
```

### WallDetection

Per-frame raycast from the player to the camera; if an obstacle blocks line of sight, a NUI blackout overlay hides the view and a throttled warning notification fires.

```lua
WallDetection = {
    enabled              = true,
    checkInterval        = 0,             -- 0 = every frame
    raycastFlags         = 1 + 16 + 256,  -- 1=world, 16=objects, 256=vehicles
    blackoutFadeDuration = 200,           -- ms
    showWarning          = true,
    warningCooldown      = 3000,          -- ms between warnings
},
```

Raycast flag reference (combine with `+`):

| Flag value | Hits |
|------------|------|
| `1`   | World geometry (terrain, buildings) |
| `2`   | Vehicles (legacy) |
| `4`   | Pedestrians and ragdolls |
| `8`   | Other entities |
| `16`  | Map objects (props) |
| `32`  | Other objects |
| `256` | Vehicles |

The default mask of `1 + 16 + 256` blocks vision through walls, props, and vehicles, but does not trigger from teammates or pedestrians passing in front of the camera.

### Logo

Image shown in the centre of the blackout overlay. Empty by default — the overlay falls back to text only.

```lua
Logo = {
    url    = '',     -- '' = no logo, only warning text
    width  = 120,
    height = 120,
},
```

Two URL formats are supported:

- **NUI path** — `nui://<resource>/path/to/logo.png`. Best for self-hosted assets. Drop the file into `clads_freecam/html/` and add it to the `files {}` block in `fxmanifest.lua`:
  ```lua
  url = 'nui://clads_freecam/html/logo.png'
  ```
- **CDN URL** — `https://your.cdn.com/logo.png`. No manifest changes needed; the player's CEF browser fetches the image directly.

### HudIntegration

Optional. Calls into your HUD resource(s) so the minimap / status bars / compass disappear during freecam and reappear on exit.

```lua
HudIntegration = {
    enabled = false,
    actions = {
        -- ...
    },
},
```

Two integration modes — pick whichever your HUD exposes.

**Export mode** (recommended for modern HUDs):

```lua
{ resource = 'wais-hud',  export = 'showHud',      hideArg = false, showArg = true  },
{ resource = 'okokHUD',   export = 'OpenHUD',      hideArg = false, showArg = true  },
{ resource = 'qb-hud',    export = 'ToggleAirHud', hideArg = false, showArg = true  },
```

This calls `exports['<resource>']:<export>(arg)` with `hideArg` on freecam start and `showArg` on freecam stop. If the listed resource is not running, the entry is skipped silently.

**Event mode** (for HUDs that listen on net events):

```lua
{ event = 'hud:client:Toggle', hideArgs = { false }, showArgs = { true } },
```

This calls `TriggerEvent('<event>', table.unpack(hideArgs))` on start and `TriggerEvent('<event>', table.unpack(showArgs))` on stop. Multiple actions can be combined freely — for example, an export call to hide the minimap plus an event to hide the compass.

### Fonts

Google Fonts URL plus the CSS family strings used in the NUI. The blackout title and menu headings use `displayFamily`; everything else uses `primaryFamily`.

```lua
Fonts = {
    url           = 'https://fonts.googleapis.com/css2?family=Rajdhani:wght@400;500;600;700&family=Inter:wght@300;400;500;600&display=swap',
    primaryFamily = "'Inter', 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif",
    displayFamily = "'Rajdhani', sans-serif",
},
```

To use custom fonts:

1. Replace the `url` with a Google Fonts CSS URL that loads your chosen families.
2. Update `primaryFamily` / `displayFamily` to match the family names returned by the CSS.

---

## Controls Reference

| Input | Action |
|-------|--------|
| **V** (hold 500ms) | Toggle freecam on / off |
| **W A S D**         | Move forward / left / backward / right (relative to camera facing) |
| **R / F**           | Move up / down (world-space vertical) |
| **Mouse**           | Look (pitch and yaw) |
| **Scroll wheel**    | Zoom in / out (FOV) |
| **SHIFT**           | Fast-move multiplier (default `2.5x`) |
| **CTRL**            | Slow-move multiplier (default `0.3x`) |
| **Q / E**           | Tilt camera (roll axis, clamped `+/- 45`) |
| **TAB**             | Open the visual-filter menu |
| **ESC**             | Close the filter menu / open the pause menu |

While freecam is active, all default game controls are blocked except `ESC` (pause menu) and `N` (push-to-talk), so accidental sprinting, weapon-switching, or vehicle entry cannot occur.

---

## Customization

### UI theme

The NUI uses CSS custom properties exposed at `:root` in `html/style.css`. The default palette is `dark_red`:

```css
:root {
    --bg-core:           #0D0D0D;
    --bg-surface:        #1A1A1A;
    --bg-elevated:       #2E2E2E;
    --bg-glass:          rgba(26,26,26,0.85);
    --color-primary:     #FF003C;
    --color-primary-light:#FF3D63;
    --color-primary-dark:#CC0030;
    --color-action:      #FF003C;
    --color-secondary:   #888888;
    --color-success:     #22C55E;
    --color-danger:      #FF003C;
    --text-primary:      rgba(255,255,255,0.95);
    --text-secondary:    rgba(255,255,255,0.7);
    --text-muted:        rgba(255,255,255,0.5);
    --border-light:      rgba(255,255,255,0.1);
    --border-active:     var(--color-primary);
    --shadow-md:         0 4px 16px rgba(0,0,0,0.4);
    --shadow-lg:         0 8px 32px rgba(0,0,0,0.5);
    --glow-primary:      0 0 20px rgba(255,0,60,0.45);
    --glow-danger:       0 0 30px rgba(255,0,60,0.45);
    --radius-md:         8px;
    --radius-lg:         12px;
}
```

Re-skin the entire UI by editing only the `--color-primary`, `--color-primary-light`, `--color-primary-dark`, and `--color-danger` variables — the gradient on filter cards, the glow on the active card, the blackout text colour, and the breathing animation on the blackout logo all derive from those.

### Adding camera presets

Append entries to `Config.Presets`. Each preset is a single Lua table:

```lua
Presets = {
    { name = "Normal",    pitch =  0.0,  yaw = 0.0, roll = 0.0, fov = 55.0 },
    { name = "Top-Down",  pitch = -45.0, yaw = 0.0, roll = 0.0, fov = 60.0, distance = 4.0, height = 6.0 },
    { name = "Hero Shot", pitch = 10.0,  yaw = 0.0, roll = 1.5, fov = 35.0, distance = 3.5 },
},
```

Cycle through them at runtime by binding a key to the export:

```lua
exports.clads_freecam:CycleNextPreset()
```

### Adding visual filters

Pick a timecycle name (see the [Visual filters](#visual-filters) reference table) and add it to `Config.Filters`, then add a matching display name to every locale file you ship.

```lua
-- config.lua
Filters = {
    -- ...existing...
    { id = 'noir', timecycle = 'Prologue_ending_Franklin' },
},
```

```lua
-- locale/en.lua
filters = {
    -- ...existing...
    noir = "Noir",
},
```

To browse every available timecycle in-game, use a timecycle picker tool such as the OpenIV `timecycle.dat` browser or any community-maintained list. The `id` is your internal handle — the `timecycle` is the GTA-side modifier name.

### Adding language files

1. Copy `locale/en.lua` to `locale/<your-code>.lua` (e.g. `locale/es.lua`).
2. Translate every string value in the file. Keep the keys identical.
3. Add the new file to `fxmanifest.lua` under `files { ... }`:
   ```lua
   files {
       'html/index.html',
       'html/style.css',
       'html/script.js',
       'locale/en.lua',
       'locale/es.lua',   -- new
       -- ...
   }
   ```
4. Set `DefaultLanguage = 'es'` in `config.lua`.
5. Restart the resource.

If the configured language fails to load, the resource logs a warning and falls back to English silently.

### Custom blackout logo

Drop a `.png` (transparent background recommended, square aspect ratio) into `html/`, register it in `fxmanifest.lua`, and point `Config.Logo.url` at it:

```lua
-- fxmanifest.lua
files {
    'html/index.html',
    'html/style.css',
    'html/script.js',
    'html/logo.png',   -- new
    'locale/en.lua',
    -- ...
}
```

```lua
-- config.lua
Logo = {
    url    = 'nui://clads_freecam/html/logo.png',
    width  = 160,
    height = 160,
},
```

The CSS applies a `grayscale(30%)` and a red drop-shadow that matches the primary palette colour, plus a slow `breathe` animation. To remove those effects, edit `.blackout-logo` in `html/style.css`.

---

## Exports & Events

The resource exposes three client exports for external resources (admin tools, photo modes, ad-hoc keybinds, etc.).

| Export | Returns | Description |
|--------|---------|-------------|
| `exports.clads_freecam:IsFreecamActive()` | `boolean` | `true` while freecam is currently rendering. |
| `exports.clads_freecam:EnableFreecam()`   | `boolean` | Activates freecam. Returns `false` if blocked (in vehicle, dead). |
| `exports.clads_freecam:DisableFreecam()`  | `nil`     | Deactivates freecam (no-op if not active). |
| `exports.clads_freecam:CycleNextPreset()` | `nil`     | Advances to the next entry in `Config.Presets` and re-frames the player. |

State bag flag (set on the local player while freecam is on):

```lua
LocalPlayer.state.clads_freecamActive  -- boolean
```

Other resources can read this to suppress their own UI, prevent interactions, etc., without depending on this resource directly.

State bag input (this resource listens for it):

```lua
LocalPlayer.state.isDead  -- boolean
```

If another resource (typically an ambulance/medic system) sets `isDead = true` while freecam is active, freecam exits automatically.

---

## Troubleshooting

**Freecam does not open when I hold V.**
- Confirm `Config.Keys.activate` matches the key you are pressing. The default `0` is GTA's `INPUT_NEXT_CAMERA` (`V` on keyboard). If you remapped your in-game `V` to a different key, the freecam follows the remap.
- Verify you are not in a vehicle and not flagged as dead. Both are blocked by design.
- Check `HoldDuration` — at the default `500ms`, releasing the key early cancels the activation.
- Ensure `ox_lib` is started before this resource.

**Wall detection is too aggressive (blackout flickers in open areas).**
- Lower the raycast mask — for example, drop vehicles by setting `raycastFlags = 1 + 16` (world + objects only).
- Set `WallDetection.checkInterval` to a small value like `50` (ms) to reduce the per-frame raycast count and smooth out brief overlaps.
- Disable warnings with `WallDetection.showWarning = false` if the notification spam is the issue but the visual blackout is fine.
- As a last resort, disable wall detection entirely with `WallDetection.enabled = false` (not recommended on public servers).

**The HUD is still visible while freecam is active.**
- `HudIntegration.enabled` defaults to `false`. Set it to `true`.
- Confirm the `resource` name in your action exactly matches your HUD's resource name (case-sensitive).
- For export mode, confirm the export name and argument values are correct for your HUD's API. The action is silently skipped if the resource is not running, so a typo in `resource` will look like nothing happens.
- For event mode, check the event name and the argument shape (`hideArgs` / `showArgs` are arrays unpacked into the event).

**Filters do not switch when I press TAB.**
- Verify `Config.ShowFilterMenu = true`.
- Confirm `Config.Keys.filterMenu` matches the key (default `37` = `TAB`).
- Ensure your `Config.Filters` array is non-empty and each entry has a unique `id`.
- If the menu opens but a filter has no display name, add the filter `id` to the `filters` table in your active locale file.

**The camera will not move past a certain distance.**
- This is the [distance-limit](#distance-limits) anti-abuse system working as intended. Increase `MaxHorizontalDistance` and/or `MaxVerticalDistance` in `config.lua` if you need more roam.
- Disable the on-screen notification with `DistanceLimitNotify = false` if you only want the silent cap.

---

## Support

Reach out via the Creative Lads Discord.
