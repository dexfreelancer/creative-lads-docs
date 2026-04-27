# Creative Lads — Crosshair

A parametric crosshair overlay for FiveM. Crosshairs are drawn live on an HTML5 canvas from a small set of numeric parameters, not pre-rendered from PNG sprites. Players tweak any parameter through an in-game editor with live preview, and the overlay redraws instantly.

---

## 1. Overview

`clads_crosshair` ships a fully customisable on-screen crosshair that any player on your server can shape, color and save to taste. Where most crosshair resources rely on a static image atlas, this resource generates the reticle dynamically:

- **Canvas-rendered, not PNG-based.** Every shape (cross, dot, target, diamond, hex, brackets, and more) is drawn at runtime by a small JavaScript renderer that consumes a parameter table — `style`, `color`, `alpha`, `thickness`, `length`, `gap`, `size`, `outline`, `centerDot`, `rotation`. Changing any value redraws the canvas immediately, so there is no scaling, no compression, and no need to ship asset bundles for new looks.
- **In-game editor with live preview.** Players open a NUI panel via a chat command or a configurable keybind, drag sliders, pick colors, and the overlay updates in real time as they edit. Hitting **Apply** persists the choice, hitting **Save as preset** stores the combination as a named custom preset.
- **20+ shape styles built in.** `dot`, `cross`, `tshape`, `plus`, `circle`, `target`, `diamond`, `square`, `hexagon`, `triangle`, `bracket`, `parens`, `chevron`, `arrow`, `x`, `equals`, `hash`, `vertical`, `horizontal`, `multidot` — combined with the parametric controls this covers thousands of unique reticles.
- **Per-player saved presets.** Each player has their own private preset library, saved on the player's client. Custom presets survive resource restarts, server restarts, and reconnects. A configurable cap (`Config.MaxCustomPresets`) prevents unbounded growth.
- **Framework-agnostic.** Notifications and framework detection go through `community_bridge`, which means the resource runs unchanged on standalone, ESX, QBCore and QBox servers.
- **Smart visibility logic.** The overlay can be configured to appear only while free-aiming, only while a weapon is equipped, or always on. It can also auto-hide while in vehicles, while the player is dead, and while the pause menu / map is open.

---

## 2. Requirements

| Dependency | Purpose | Required |
|------------|---------|----------|
| `community_bridge` | Framework abstraction (ESX / QBCore / QBox / Standalone), notification routing | Yes |
| `ox_lib` | Keybind registration, NUI utility helpers | Yes |
| Framework | Auto-detected via `community_bridge`. Standalone, ESX, QBCore, QBox all supported. | Auto |

The resource declares both dependencies explicitly in its `fxmanifest.lua` and will refuse to start without them.

---

## 3. Installation

1. Drop the `clads_crosshair` folder into your server's `resources/` directory.
2. Make sure `community_bridge` and `ox_lib` are present and are started **before** `clads_crosshair`.
3. Add the resource to your `server.cfg`:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure clads_crosshair
   ```
4. (Optional) Open `config.lua` and tune the values described in section 4.
5. Start (or restart) the server. Join the game and either type `/crosshair` or press the default keybind (`F7`) to open the editor.

No database tables are required. All persistence is handled per-player on the client, so there is nothing to migrate.

---

## 4. Configuration

All settings live in `config.lua` at the resource root. The file is left out of escrow encryption (`escrow_ignore`), so it stays editable after purchase.

### Framework

```lua
-- 'auto' | 'esx' | 'qb' | 'qbox' | 'standalone'
Config.Framework = 'auto'
```

`community_bridge` auto-detects which core is running. Pin a specific value here only if you run multiple cores side by side or want to force standalone mode while testing.

### Locale

```lua
Config.Locale = 'auto'
```

Built-in locales: `en`, `tr`, `de`, `fr`, `es`. When set to `'auto'`, the resource resolves the language from the following convars in order:

1. `clads_locale`
2. `ox:locale`
3. `qb_locale`
4. `lang`
5. fallback `en`

To add a new language, drop a `locales/<code>.json` file alongside the existing ones. Use `en.json` as a template — every key it contains must be present in the new file (missing keys silently fall back to the literal key string).

### Debug

```lua
Config.Debug = false
```

Enables verbose console output. Leave off in production.

### Command

```lua
Config.Command = 'crosshair'
```

The chat command players use to open the editor. Renaming it (for example to `xhair`) updates both the slash command and the keybind label registered with FiveM's key map.

### Keybind

```lua
Config.Keybind = {
    enabled     = true,
    key         = 'F7',           -- default key — players can rebind
    description = 'keybind_open', -- locale key, falls back to literal
}
```

The keybind is registered through `ox_lib` as a standard FiveM `RegisterKeyMapping`. Each player can rebind it from **Settings → Key Bindings → FiveM**. The default is `F7`. Set `enabled = false` (or `key = ''`) to ship without a default keybind — the chat command still works.

### Display Mode

```lua
-- 'aiming' | 'weapon' | 'always'
Config.ShowMode = 'aiming'
```

Controls when the crosshair is visible:

| Value | Behaviour |
|-------|-----------|
| `aiming` | Only while the player is free-aiming. Matches the GTA reticle behaviour. |
| `weapon` | Whenever a weapon other than `WEAPON_UNARMED` is equipped. |
| `always` | Always visible, regardless of weapon or aim state. |

The visibility loop polls at 5 Hz when idle and 20 Hz while in `aiming` mode, so it stays responsive without burning frames.

### Hide Conditions

```lua
Config.HideInVehicle    = true
Config.HideWhenDead     = true
Config.HideInPauseMenu  = true
```

| Flag | Effect |
|------|--------|
| `HideInVehicle` | Hides the crosshair while the player is in any vehicle (driver or passenger). Most servers want this on so the in-car aim reticle remains the game's own. |
| `HideWhenDead` | Hides the crosshair while the local ped is dead. |
| `HideInPauseMenu` | Hides the crosshair while the pause menu / map is open. |

### Theme

```lua
Config.Theme = {
    background     = '#1A1A1A',
    backgroundDark = '#0D0D0D',
    elevated       = 'rgba(255, 255, 255, 0.04)',
    elevatedHi     = 'rgba(255, 255, 255, 0.08)',
    text           = '#FFFFFF',
    textMuted      = '#888888',
    textFaint      = '#5A5A5A',
    border         = 'rgba(255, 255, 255, 0.06)',
    primary        = '#FF003C',
    primaryHover   = '#FF3D63',
    danger         = '#FF003C',
    backdrop       = 'rgba(13, 13, 13, 0.6)',
    radius         = 14,
}
```

The defaults are the dark-red Creative Lads theme. Every value is a CSS color string — hex, `rgb()`, `rgba()` and named colors all work. See section 6 for a complete variable reference and worked examples.

### Default Preset

```lua
Config.DefaultPreset = 'classic-cross'
```

The preset id applied to a player who has never opened the editor. Must match an `id` in `Config.Presets`.

### Max Custom Presets

```lua
Config.MaxCustomPresets = 20
```

Hard cap on how many custom presets a single player can save on their client.

### Built-in Presets

The resource ships 24 ready-to-use presets covering every supported shape. Each entry is a parametric definition consumed directly by the canvas renderer.

| ID | Style | Color | Notes |
|----|-------|-------|-------|
| `classic-cross` | cross | `#00FF7F` | Default preset. Thin green four-line cross. |
| `tactical-dot` | dot | `#FF2A6D` | Pink center dot, no outline strokes. |
| `cs-tshape` | tshape | `#00E5FF` | T-shape (top line removed), CS-style. |
| `plus-thick` | plus | `#FFEA00` | Thick yellow plus, no gap. |
| `circle-thin` | circle | `#9D4EDD` | Thin purple ring. |
| `bullseye` | target | `#FF9100` | Outer ring + cross combo. |
| `diamond-frame` | diamond | `#00E676` | Rotated square outline. |
| `corner-brackets` | bracket | `#B76EF5` | Four corner brackets. |
| `sniper-x` | x | `#FFFFFF` | Diagonal X with center dot. |
| `down-chevron` | chevron | `#FF2A6D` | Downward V. |
| `up-arrow` | arrow | `#00E5FF` | Upward V. |
| `square-frame` | square | `#FFEA00` | Axis-aligned square outline. |
| `hex-frame` | hexagon | `#9D4EDD` | Pointy-top hex outline. |
| `spear-tip` | triangle | `#FF9100` | Upward triangle outline. |
| `parentheses` | parens | `#00E676` | Two facing arcs. |
| `equals-pair` | equals | `#FFFFFF` | Two parallel horizontal lines. |
| `hashgrid` | hash | `#B76EF5` | Four parallel lines (# sign). |
| `vertical-bars` | vertical | `#FF2A6D` | Top + bottom lines only. |
| `horizontal-bars` | horizontal | `#00FF7F` | Left + right lines only. |
| `compass-dots` | multidot | `#00E5FF` | Four small dots at compass positions. |
| `micro-dot` | dot | `#FFFFFF` | Single 1px white dot. |
| `precision-cross` | cross | `#FFFFFF` | Tight white cross with center dot. |
| `shotgun-spread` | circle | `#FF9100` | Wide translucent ring. |
| `rotated-x-cross` | plus | `#9D4EDD` | Plus rotated 45° (diagonal cross). |

Every preset can be edited and re-saved by players as a custom preset; built-ins themselves cannot be deleted.

---

## 5. Commands & Keybinds

### Chat Command

```
/crosshair
```

Opens the crosshair editor with NUI focus. The command name is configurable via `Config.Command`. The command is also added to the chat suggestion list, so it appears in the chat autocomplete dropdown.

### Keybind

The default keybind is **F7**, registered through FiveM's standard key mapping system. Players can rebind it themselves at any time:

1. Open the FiveM pause menu.
2. Go to **Settings → Key Bindings → FiveM**.
3. Find the entry labelled **Open crosshair editor** (or the localized label if you ship a different locale).
4. Click and assign any key.

The label shown in that menu is taken from the locale key in `Config.Keybind.description` (default `keybind_open`). If you want to ship without a default key but still let players bind one from the menu, leave `enabled = true` and set `key = ''`.

---

## 6. Customization

### Theme Variables

Every entry under `Config.Theme` is forwarded to the NUI editor as a CSS variable. Setting a value to `nil` (or removing the line) keeps the default.

| Variable | Description |
|----------|-------------|
| `background` | Main panel background color. |
| `backgroundDark` | Gradient end (lower-right corner of the panel). |
| `elevated` | Subtle layer for cards, buttons and inputs. Use `rgba()` for translucency. |
| `elevatedHi` | Slightly stronger elevated layer (slider tracks, hover states). |
| `text` | Primary text color. |
| `textMuted` | Secondary text (labels, section titles). |
| `textFaint` | Least-prominent text (empty-state hints). |
| `border` | Divider / border color. Use `rgba()` for translucency. |
| `primary` | Accent color (active tab, primary buttons, slider thumb). |
| `primaryHover` | Accent color on hover. Usually a lighter `primary`. |
| `danger` | Destructive action color (delete buttons, close hover). |
| `backdrop` | Modal overlay color behind the panel. Use `rgba()`. |
| `radius` | Corner radius scale in px. The panel uses this directly. |

#### Worked example — light theme

```lua
Config.Theme = {
    background     = '#FFFFFF',
    backgroundDark = '#F4F2F8',
    elevated       = 'rgba(0,0,0,0.04)',
    elevatedHi     = 'rgba(0,0,0,0.08)',
    text           = '#1B1525',
    textMuted      = '#6B6378',
    textFaint      = '#9089A0',
    border         = 'rgba(0,0,0,0.08)',
    primary        = '#7B2CBF',
    primaryHover   = '#9D4EDD',
    danger         = '#E11D48',
    backdrop       = 'rgba(0,0,0,0.45)',
    radius         = 14,
}
```

#### Worked example — minimal mono with cyan accent

```lua
Config.Theme = {
    background     = '#0F0F12',
    backgroundDark = '#08080A',
    primary        = '#00E5FF',
    primaryHover   = '#5DEAFF',
    danger         = '#FF5577',
    radius         = 8,
}
```

### Adding New Presets

To ship additional built-in presets, add entries to `Config.Presets`. Every entry is a flat table that the canvas renderer reads directly:

```lua
Config.Presets[#Config.Presets + 1] = {
    id            = 'my-server-default',
    name          = 'My Server Default',   -- literal string OR locale key
    style         = 'cross',
    color         = '#00FF00', alpha = 1.0,
    thickness     = 2, length = 8, gap = 4,
    size          = 0,
    outline       = true, outlineColor = '#000000', outlineWidth = 1,
    centerDot     = true, centerDotSize = 1,
    rotation      = 0,
}
```

The `name` field can be a literal string or a locale key. If the value matches a key in the active locale JSON, it is translated automatically before being shown in the editor; otherwise it is rendered verbatim.

To set this as the default for new players, point `Config.DefaultPreset` at its `id`.

### Supported Shape Styles

| Style | Description |
|-------|-------------|
| `dot` | Center dot only. |
| `cross` | Classic 4-line cross with a configurable gap in the middle. |
| `tshape` | T-shape — like `cross` with the top line removed (CS-style). |
| `plus` | Cross with no gap (lines meet in the middle). |
| `circle` | Open ring outline. |
| `target` | Bullseye — outer ring plus a cross. |
| `diamond` | Rotated square outline. |
| `square` | Axis-aligned square outline. |
| `hexagon` | Pointy-top hexagon outline. |
| `triangle` | Upward-pointing triangle outline. |
| `bracket` | Four corner brackets framing the center. |
| `parens` | Two facing arcs ( ). |
| `chevron` | Downward-pointing V. |
| `arrow` | Upward-pointing V. |
| `x` | Diagonal X. |
| `equals` | Two parallel horizontal lines (= sign). |
| `hash` | Four parallel lines (# sign). |
| `vertical` | Only the top and bottom lines of a cross. |
| `horizontal` | Only the left and right lines of a cross. |
| `multidot` | Four small dots at compass positions (N/E/S/W). |

### Per-Style Parameters

Every preset is a flat parameter table. The renderer ignores parameters that do not apply to the chosen style (for example `length` does nothing for a pure `dot`).

| Field | Type | Range | Used by | Description |
|-------|------|-------|---------|-------------|
| `style` | string | enum (above) | all | Shape selector. |
| `color` | hex string | `#RRGGBB` | all | Stroke / fill color of the shape. |
| `alpha` | number | 0.05 – 1.0 | all | Opacity of the shape. |
| `thickness` | number | 0 – 10 | most | Stroke width in px. |
| `length` | number | 0 – 40 | cross, tshape, plus, x, chevron, arrow, equals, hash, vertical, horizontal | Line length in px. |
| `gap` | number | 0 – 30 | cross, tshape, x, chevron, arrow, equals, hash, vertical, horizontal, multidot | Empty pixels between center and each line / dot. |
| `size` | number | 0 – 40 | dot, circle, target, diamond, square, hexagon, triangle, bracket, parens | Radius / half-extent in px. |
| `outline` | boolean | true / false | all | Draws a 1–2 px outline behind the shape (improves contrast on busy backgrounds). |
| `outlineColor` | hex string | `#RRGGBB` | when `outline` | Outline color. |
| `outlineWidth` | number | 0 – 4 | when `outline` | Outline thickness in px. |
| `centerDot` | boolean | true / false | all | Draws a small dot in the middle on top of the shape. |
| `centerDotSize` | number | 0 – 8 | when `centerDot` | Center dot radius in px. |
| `rotation` | number | -180 – 180 | all | Degrees, applied to the whole shape. |

These ranges are enforced server-side by the `sanitisePreset` helper before any preset is persisted, so out-of-range values cannot be smuggled in through the NUI bridge.

---

## 7. Exports & Events

The resource is intentionally self-contained and does not expose public exports. Integration with other resources happens through the standard FiveM lifecycle events.

### NUI Callbacks (internal)

These callbacks are registered on the client and used by the NUI panel. They are listed here for transparency / auditing, not as a public API.

| Callback | Payload | Purpose |
|----------|---------|---------|
| `close` | — | Closes the editor and releases NUI focus. |
| `apply` | `{ id }` | Applies a preset by id (built-in or custom) and persists it. |
| `preview` | `{ preset }` | Live preview while the user drags sliders / picks a color. Does **not** persist. |
| `save` | `{ preset }` | Validates a preset, enforces `Config.MaxCustomPresets`, and saves it on the client. |
| `delete` | `{ id }` | Deletes a custom preset. Built-ins cannot be deleted. |

### Persistence

Saved data is local to each player's client and does not require any server-side database. Two pieces of data are stored:

- The active preset id.
- The player's custom preset list.

If the player wipes their FiveM data (or moves to another machine) the saved presets reset to the defaults.

### Lifecycle

The resource hooks `onResourceStop` to release NUI focus cleanly if the editor is open at the moment the resource is stopped. There are no other server-bound events.

---

## 8. Troubleshooting

### Crosshair never appears

1. Confirm `Config.ShowMode` matches what you expect:
   - `'aiming'` — you must hold the right mouse button (or the equivalent free-aim input) for the crosshair to show.
   - `'weapon'` — you must have a weapon other than `WEAPON_UNARMED` equipped.
   - `'always'` — set this temporarily to confirm the overlay itself is rendering.
2. Make sure no hide flag is firing: `HideInVehicle`, `HideWhenDead`, `HideInPauseMenu`.
3. Check the F8 console for `[clads_crosshair]` lines and confirm the resource started without errors.

### Crosshair invisible inside vehicles

This is the default behaviour. The in-car aim reticle is the game's own. If you want the parametric crosshair to show inside vehicles too, set:

```lua
Config.HideInVehicle = false
```

### Custom presets not saving

1. The player may have hit the `Config.MaxCustomPresets` cap. The editor surfaces a notification when this happens (`limit_reached` locale key). Raise the cap or have the player delete an old preset.
2. Saved data can fail to write if the resource was renamed mid-session. Make sure the folder name on disk matches the name you `ensure` in `server.cfg`. A restart fixes this.
3. If a payload is rejected as `invalid_payload`, the preset's `style` was not in the allowed list. This usually means the resource was edited by hand — restore from a fresh copy.

### Keybind not registering

1. The keybind only appears in the FiveM Key Bindings menu **after** the player has joined the server at least once with the resource running. Have them join, then check the menu again.
2. Make sure `Config.Keybind.enabled = true` and `Config.Keybind.key` is a non-empty string (use FiveM key names like `F7`, `HOME`, `INSERT`).
3. `ox_lib` must be started before `clads_crosshair`. Check your `ensure` order in `server.cfg`.

### Editor opens but is blank

1. The UI assets did not load — re-download the resource from your Tebex account and replace the folder. The bundle ships pre-built; there is nothing to install on your end.
2. If your server runs a strict asset proxy or CDN in front of `cfx-nui-*`, make sure it isn't blocking the resource's static files.
3. If the issue persists after a clean re-download, open a ticket.

---

## 9. Support

Reach out via the Creative Lads Discord.
