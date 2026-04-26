# CLads Spawn

Cinematic free-camera spawn selector with bird's-eye navigation, live population tiers, last-location resume, and a buyer-extensible hook API for clothing, HUD, and force-respawn integrations.

---

## Overview

CLads Spawn is a framework-agnostic spawn point selector for FiveM. When a player joins (or when a multichar resource hands off a chosen character), the screen fades into a free-roaming bird's-eye camera. The player drags or WASD-pans the camera over the map, hovers over any of the configured spawn points to preview it, and confirms to teleport in with a cinematic zoom-in transition.

Core features:

- **Bird's-eye free-roam camera.** WASD pans the camera, mouse drag scrolls it, scroll-wheel adjusts the height (zoom), `SHIFT` boosts speed. The camera is clamped to the GTA V map bounds so it can never roam off the world.
- **Cinematic preview transitions.** Selecting a spawn from the bottom strip flies the camera to the new location with a context-aware easing — a near-spawn smooth pan, or a zoom-out / fly / zoom-in three-stage interpolation when the new point is far away.
- **Cinematic spawn drop.** Confirming a spawn fades the camera to ground level, teleports the player, and renders the script camera back to first-person over a 3-second fade — there is no ground "pop" mid-transition.
- **Live population tiers.** A server thread samples player positions every 3 minutes and reports a tier (`sakin` / `normal` / `yogun`) per spawn point based on the percentage of online players within `400m` of that point. The NUI shows the tier as a coloured dot + percentage.
- **"Last Location" resume.** When a player disconnects, their position is cached for 20 minutes by default. On reconnect, a synthetic "Last Location" entry appears at the top of the selector so they can pick up exactly where they left off. Persists in memory by default; switch to `oxmysql` for restart-durable storage.
- **Five language locales out of the box.** English, Turkish, German, French, and Spanish ship in `locales/`. Locale auto-detection runs through the `clads_locale` → `ox:locale` → `qb_locale` → `lang` convar chain.
- **Modern NUI.** A 1700+ line custom UI built on Inter and Oswald, with screen-projected markers, callout SVG paths, a hud card, hero intro, and a horizontal location strip with population dots. Obfuscated for shipping; CLADS_RAW=1 toggles raw output for local debugging.
- **Buyer extension hooks.** Five optional Lua functions in `config.lua` plug in clothing systems, HUD hide/show, jail / wanted force-respawn, character-load hand-off, and post-spawn callbacks — without forking the resource.
- **Public events for cross-resource integration.** Other resources can subscribe to `clads_spawn:client:selectorOpened`, `selectorClosed`, `characterLoaded`, and `spawnComplete` to react to selector state changes.

---

## Requirements

| Resource | Why |
|---|---|
| [`community_bridge`](https://github.com/The-Order-Of-The-Sacred-Flame/community_bridge) | Framework / notify abstraction. Auto-detects ESX, qb-core, qbx_core, or runs standalone. |
| [`ox_lib`](https://github.com/overextended/ox_lib) | Server callbacks, NUI focus, ped streaming, and the locale loader. |

Optional:

| Resource | When |
|---|---|
| `oxmysql` | Only if you set `Config.LastLocation.persistMode = 'oxmysql'`. Default `'memory'` mode needs no SQL layer. |

The resource works on a vanilla FiveM server with just the two hard dependencies above; framework integration is automatic when one of `es_extended`, `qb-core`, or `qbx_core` is present.

---

## Installation

1. Drop the `clads_spawn` folder into your server's `resources/` directory.
2. Ensure the dependencies start before this resource:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure clads_spawn
   ```
3. Open `config.lua` and review the spawn list, hooks, and last-location settings.
4. (Optional) Wire integrations through `Config.Hooks` — clothing, HUD hide/show, force-respawn checks. Leave them as empty functions if you don't need them.
5. (Optional) If you want last-location to survive server restarts, set `Config.LastLocation.persistMode = 'oxmysql'` and ensure `oxmysql` is running. The `clads_spawn_lastloc` table is created automatically on first save.
6. Restart the server, or `refresh; ensure clads_spawn` from the live console.
7. Test the flow: connect, run the LoadAndSelect export from your multichar (or set `Config.Multicharacters = false` to auto-open on every spawn), and confirm the selector renders.

---

## Configuration

All settings live in `config.lua`. The file is escrow-ignored so edits survive future updates.

### Framework

```lua
Config.Framework = 'auto'   -- 'auto' | 'esx' | 'qb' | 'qbox' | 'standalone'
```

`auto` lets `community_bridge` detect the running framework. Pin a specific value only if you have multiple framework cores loaded and need to force one.

### Locale

```lua
Config.Locale = 'auto'      -- 'auto' | 'en' | 'tr' | 'de' | 'fr' | 'es' | <other>
```

When `'auto'`, the loader resolves the active language from the first non-empty convar in this order:

1. `clads_locale`
2. `ox:locale`
3. `qb_locale`
4. `lang`
5. fallback `en`

To add a new language, drop a `locales/<code>.json` file with the same key set as `locales/en.json`. The English file is the source of truth and the fallback when a key is missing.

### Debug

```lua
Config.Debug = false
```

Enables verbose console output for locale fallbacks, hook errors, and adapter mismatches. Leave `false` in production.

### SelectorMode

```lua
Config.Multicharacters = true
```

| Value | Behaviour |
|---|---|
| `true` | Selector opens **only** through the `LoadAndSelect` export. Use this with multichar resources. |
| `false` | Selector opens automatically on every `playerSpawned` event. Use this for single-character flows. |

### Camera

Free-roam camera tuning. The defaults match the reference implementation — only edit if you want a different feel.

| Key | Default | Description |
|---|---|---|
| `speed` | `2.5` | WASD movement speed (world units per frame) |
| `sprintMult` | `5.0` | Multiplier when holding `SHIFT` |
| `dragScale` | `80.0` | Mouse drag sensitivity |
| `height` | `1000.0` | Starting camera height (max zoom out) |
| `minHeight` | `200.0` | Closest zoom |
| `maxHeight` | `1000.0` | Furthest zoom |
| `zoomStep` | `100.0` | Height change per scroll tick |
| `zoomSmooth` | `0.08` | Lerp factor per frame (0–1) |
| `pitch` | `-75.0` | Camera pitch (degrees) |
| `fov` | `80.0` | Camera FOV |
| `proximityEnter` | `200.0` | Distance to auto-select a spawn |
| `proximityLeave` | `280.0` | Distance to deselect (hysteresis) |

### Population

Live population sampler.

```lua
Config.Population = {
    enabled  = true,
    radius   = 400.0,
    interval = 180000,    -- 3 minutes in ms
    sakin    = 10,        -- ≤ 10% of online → 'sakin' (calm)
    normal   = 25,        -- ≤ 25% of online → 'normal'
                          -- > 25% of online → 'yogun' (busy)
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Master toggle. Disables the sampler thread when `false`. |
| `radius` | number | `400.0` | Units around each spawn that count as "near". |
| `interval` | number | `180000` | Milliseconds between server samples. |
| `sakin` | number | `10` | Percent threshold for the calm tier. |
| `normal` | number | `25` | Percent threshold for the normal tier. Anything above is `yogun`. |

### LastLocation

```lua
Config.LastLocation = {
    enabled     = true,
    ttlSeconds  = 1200,      -- 20 minutes
    persistMode = 'memory',  -- 'memory' | 'oxmysql'
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` | Toggle the "Last Location" entry on the selector. |
| `ttlSeconds` | number | `1200` | Drop the cached position after this many seconds. |
| `persistMode` | string | `'memory'` | `'memory'` keeps the cache in RAM (lost on restart). `'oxmysql'` persists to a `clads_spawn_lastloc` table. |

### Spawns

Each entry needs:

- `name` — internal id; must match the filename of an image in `web/build/images/<name>.png`
- `labelKey` — locale key for the card title (passed through `Locale.resolve`)
- `infoKey` — locale key for the card description
- `coord` — `vector4(x, y, z, heading)`

```lua
Config.Spawns = {
    {
        name     = 'pillbox',
        labelKey = 'spawn_pillbox_label',
        infoKey  = 'spawn_pillbox_info',
        coord    = vector4(246.86, -567.63, 43.28, 242.27),
    },
    -- ...
}
```

`labelKey` and `infoKey` accept either a locale key (preferred — gets translated) **or** a literal string (returned verbatim). Drop in custom locations without touching the JSON files.

To add a new spawn:

1. Append the entry to `Config.Spawns`.
2. Drop a 500x500 PNG image at `web/build/images/<name>.png`.
3. Add `spawn_<name>_label` and `spawn_<name>_info` keys to every `locales/<code>.json` file.

---

## Hooks

Buyer extension points live under `Config.Hooks`. Every hook ships as an empty `function() end` — wire only the ones you need.

### loadCharacter (server)

Server-side character loader called from the `LoadAndSelect` export. Receives the citizenid you pass in, runs **before** any client-side hook, and is awaited.

```lua
loadCharacter = function(source, citizenid)
    lib.callback.await('qbx_core:server:loadCharacter', source, citizenid)
end
```

Leave empty if you call `Selector()` directly from your own multichar resource — the framework's own loader has already run by that point.

### onCharacterLoaded (client)

Runs after the framework character load, before the selector opens. The right place to apply clothing, accessories, or any character-specific state.

```lua
onCharacterLoaded = function(citizenid)
    lib.callback('my_clothing:server:getAppearance', false, function(app)
        if app then exports['my_clothing']:setPlayerAppearance(app) end
    end)
end
```

### forceRespawn (server)

Server-side check before the selector opens. Return a table to **skip the selector** and teleport the player; return `nil` to open it normally.

```lua
forceRespawn = function(source, citizenid)
    if exports['my_jail']:isPlayerJailed(citizenid) then
        return {
            dead       = false,
            coord      = vec4(1690.0, 2566.0, 45.5, 270.0),
            messageKey = 'force_respawn_jail',
        }
    end
    return nil
end
```

Return-table fields:

| Field | Type | Description |
|---|---|---|
| `dead` | bool | `true` calls `NetworkResurrectLocalPlayer`; `false` just teleports. |
| `coord` | vec4 | Spawn position + heading. |
| `messageKey` | string | Locale key for the on-screen notification (`force_respawn_dead`, `force_respawn_wanted`, `force_respawn_jail`, `force_respawn_generic`, or your own). |

### onSelectorOpen / onSelectorClose (client)

Fire when the selector opens / closes. Use them to hide your custom HUD, radar, blip system, NUI overlays, etc. The built-in radar is already hidden for you.

```lua
onSelectorOpen = function()
    TriggerEvent('my_hud:client:hideHud', true)
end,
onSelectorClose = function()
    TriggerEvent('my_hud:client:hideHud', false)
end,
```

### onSpawnComplete (client)

Fires after the player lands at the chosen spawn (post-fade-in). Useful for last-second setup: insurance prompts, intro cutscene, timed events.

```lua
onSpawnComplete = function(spawnName, coord)
    if spawnName == 'pillbox' then
        TriggerServerEvent('my_hospital:server:onArrival')
    end
end
```

---

## Locale

The locale loader (`shared/locale.lua`) auto-detects the active language and exposes:

| Function | Purpose |
|---|---|
| `_t(key, ...)` | Translate `key`, optionally with `string.format` args. Returns the key verbatim if missing. |
| `Locale.dict()` | Raw dict (used to ship translations to the NUI). |
| `Locale.code()` | Resolved locale code (e.g. `'en'`). |
| `Locale.resolve(key)` | Translate `key` if it exists, otherwise return `key` literally. Use this in `config.lua` for spawn labels so authors can drop in literal strings without breaking the loader. |

To add a new language:

1. Copy `locales/en.json` to `locales/<code>.json` (e.g. `locales/it.json`).
2. Translate every value. Keep the keys identical to `en.json`.
3. Set `Config.Locale = 'it'`, or set `setr clads_locale "it"` in `server.cfg` for runtime auto-detection.

If a configured locale fails to load, the loader falls back to English and emits a single console warning when `Config.Debug = true`.

---

## Controls

While the selector is open:

| Input | Action |
|---|---|
| **Mouse drag** | Pan the bird's-eye camera |
| **W A S D** | Pan the camera in cardinal directions |
| **SHIFT** (held) | Speed boost (default `5x`) |
| **Scroll wheel** | Zoom in / out (height) |
| **← / →** (in strip) | Cycle previous / next spawn point |
| **ESC** | Reset zoom |
| **ENTER / click** | Confirm the highlighted spawn |

The default game controls (sprint, jump, weapon-switch, vehicle entry, etc.) are blocked while the selector is active so accidental input cannot leak into the world.

---

## Exports & Events

### Client exports

| Export | Returns | Description |
|---|---|---|
| `exports.clads_spawn:Selector(coord, options)` | `string` \| `nil` | Open the selector. `options` is an optional list of extra spawn entries to append to the configured set. Resolves with the chosen spawn `name` (or `nil` if cancelled). |
| `exports.clads_spawn:LoadAndSelect(citizenid)` | `nil` | Full multichar flow: runs `loadCharacter` hook → fires `characterLoaded` event → runs `onCharacterLoaded` hook → runs `forceRespawn` hook → opens the selector. |
| `exports.clads_spawn:CloseSelector()` | `nil` | Cancel the selector without spawning. Resolves the pending `Selector()` promise with `nil`. |

### Public events

Other resources can subscribe to react to selector state:

```lua
AddEventHandler('clads_spawn:client:selectorOpened', function()
    -- selector just appeared
end)

AddEventHandler('clads_spawn:client:selectorClosed', function()
    -- selector closed (spawned or cancelled)
end)

AddEventHandler('clads_spawn:client:characterLoaded', function(citizenid)
    -- LoadAndSelect just loaded a character (server load complete)
end)

AddEventHandler('clads_spawn:client:spawnComplete', function(spawnName, coord)
    -- player has landed at the chosen spawn
end)
```

### Net events fired by the resource

| Event | Direction | Payload |
|---|---|---|
| `clads_spawn:client:populationUpdate` | server → client | `{ [name] = { count, tier, percent }, ... }` |
| `clads_spawn:server:requestPopulation` | client → server | none — request a fresh population snapshot |

---

## Customization

### Logo

Replace `web/build/logo.png` with your own 256x256 transparent PNG. The path is escrow-ignored so the swap survives updates. The same logo is used for the hero intro and the title block.

### UI palette

The NUI styles live in `web/src/styles.css` and are bundled into `web/build/assets/index.css` at build time. The default `:root` block exposes the Purple Haze palette:

```css
:root{
    --primary:#9D4EDD;
    --primary-light:#B76EF5;
    --primary-dark:#7B2CBF;
    --primary-glow:rgba(157,78,221,0.4);
    --action:#FFEA00;
    /* ... */
}
```

To re-skin: edit `--primary*` and `--action` in `web/src/styles.css`, run `npm run build` from `web/`, commit the new `web/build/`. The card glow, callout strokes, and strip highlights all derive from those variables.

### Adding spawn points

1. Append a new entry to `Config.Spawns` (see [Spawns](#spawns)).
2. Drop a 500x500 PNG at `web/build/images/<name>.png` (kept identical to the `name` field).
3. Add `spawn_<name>_label` and `spawn_<name>_info` to every `locales/<code>.json` file. Use the same keys across all locales (run `diff` on the parsed key sets before release).
4. Restart the resource.

The selector picks up the new entry on the next open — no rebuild needed unless you also edited `web/src/`.

---

## Troubleshooting

**The selector never opens.**
- Verify `Config.Multicharacters`. With `true`, you must call `exports.clads_spawn:LoadAndSelect(citizenid)` explicitly. With `false`, it opens on every `playerSpawned` — confirm the event is firing.
- Check the F8 console for locale-load errors. `locales/en.json` is required; if it's missing or malformed, the resource refuses to start.

**Spawn images don't render.**
- The `name` field in `Config.Spawns` must match the image filename exactly (case-sensitive on Linux). `bennys` ↔ `bennys.png`, not `Bennys.png`.
- Confirm the image lives in `web/build/images/`, not `web/src/images/`. The shipped folder is `build/`; `src/` is developer-only.
- The fallback gradient (named in `THUMB_MAP` inside the bundled JS) renders when an image is missing — if you see a coloured gradient instead of a photo, the image lookup failed.

**The "Last Location" entry never appears.**
- Confirm `Config.LastLocation.enabled = true`.
- The position is cached on `playerDropped` only. Crashing or force-killing FiveM does **not** trigger `playerDropped`, so a hard crash will not save a position.
- With `persistMode = 'memory'`, a server restart wipes the cache. Switch to `'oxmysql'` for restart-durable storage.
- Check the configured `ttlSeconds`. The default `1200` (20 minutes) drops anything older.

**Population dots all show as `sakin`.**
- The sampler runs every `Config.Population.interval` ms (default 3 minutes). On a brand-new server start, allow one full interval before checking.
- Tiers are computed against the **percentage of online players**, not raw counts. With 2 players online and both at one spawn, that's 100% → `yogun`. With 100 players online and 2 at one spawn, that's 2% → `sakin`.
- Increase `Config.Population.radius` if your spawn points are widely spread; `400` units is the default and may not capture players spread thin across the map.

**Force-respawn fires but the player still sees the selector.**
- The `forceRespawn` hook must return a table with a `coord` field. Returning `true` or any non-table value is treated as "no override" → the selector opens.
- Server-side errors inside the hook are caught with `pcall`. Enable `Config.Debug = true` to see the hook's stack trace in the console.

**The buyer's HUD stays visible while the selector is open.**
- Wire `Config.Hooks.onSelectorOpen` to whatever event/export your HUD exposes. The built-in radar is already hidden, but custom NUI HUDs need an explicit hide call.

**oxmysql adapter errors on a non-DB server.**
- The adapter checks `GetResourceState('oxmysql') == 'started'` before every query and silently skips when oxmysql is absent. If you are still seeing errors, confirm the value of `Config.LastLocation.persistMode`. Setting it to `'oxmysql'` on a server that does not start oxmysql results in last-location simply not persisting (no crash).

---

## Support

Open a ticket via the panel in **#create-ticket** on the [Creative Lads Discord](https://discord.gg/creativelads). For documentation issues, edit the page directly through GitBook or open a PR against `dexfreelancer/creative-lads-docs`.
