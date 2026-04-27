# CLads Multichar

Creative Lads character selection and multicharacter system for FiveM. Built on `community_bridge` for framework, clothing, and locale abstraction. Ships with a glassmorphic React UI styled with a Dark Red palette.

---

## 1. Overview

`clads_multichar` is a drop-in character selection screen that replaces the default multicharacter UI from your framework. It presents the player with a configurable number of slots, an animated 3D preview of the selected ped, and a TAB-toggled settings panel for tuning the look of the selection scene.

### Feature highlights

- **Multi-slot character selection** with per-player unlock progression (`MaxSlots`, `DefaultUnlockedSlots`, plus per-license extra slots stored in MySQL).
- **Framework auto-detection through `community_bridge`** — supports QBCore, QBox (`qbx_core`), ESX, and a fully self-contained Standalone mode.
- **Per-framework character bridge** (`bridge/qb_characters.lua`, `bridge/qbox_characters.lua`, `bridge/esx_characters.lua`, `bridge/standalone_characters.lua`) so the same UI works everywhere.
- **Live ped preview** rendered with a custom scripted camera, depth-of-field, and tutorial-session isolation.
- **Settings panel (TAB)** with toggles and dropdowns for:
  - Streamer Mode (mask names)
  - Focus Mode (DoF blur on background)
  - Filter Mode + Filter Type (timecycle modifiers, e.g. `NG_filmic01`, `cinema`)
  - Particle Effect mode + Particle Type (configurable ptfx assets)
  - Scenario Mode + Scenario Type (idle animations such as `WORLD_HUMAN_SMOKING`)
  - Weather override (Sunny, Rain, Foggy, Snow, Christmas, etc.)
  - Time override (06:00 through 03:00 buckets)
  - Background location picker (Random / City / Vinewood)
- **Settings persist per client** — saved on the player's local FiveM install.
- **Intro cutscene** (`MP_INTRO_CONCAT`) plays automatically on first character creation.
- **Loadscreen integration** — sends `customEvent: showContinue` to the configured loadscreen and waits for `loadscreen:continueClicked` (with a configurable timeout).
- **Clothing abstraction** through `community_bridge:Bridge().Clothing` — supports qb-clothing, illenium-appearance, fivem-appearance, esx_skin, and rcore_clothing out of the box, with a configurable fallback export.
- **Localized UI** — English, Turkish, German, and French locales bundled in `locales/`.
- **HUD hide hook** — fires a configurable event so your HUD resource can hide itself during selection.
- **Glassmorphic in-game UI** with subtle motion.

---

## 2. Requirements

| Requirement | Notes |
|---|---|
| FiveM artifact | `cerulean` fxmanifest (`fxmanifest.lua` line 1) |
| Lua | 5.4 (`lua54 'yes'`) |
| `ox_lib` | Loaded as shared script and used for callbacks, commands, notifications |
| `community_bridge` | Hard dependency. Provides framework, clothing, and language modules |
| `oxmysql` | Required server-side for `clads_charslots` and `clads_characters` tables |
| MySQL / MariaDB | Schema is auto-created on resource start |
| Spawnmanager | Built-in FiveM resource (`exports.spawnmanager:spawnPlayer`) |

### Supported frameworks (auto-detected)

- **QBox** (`qbx_core`)
- **QBCore** (`qb-core`) using `qb-multicharacter` callbacks
- **ESX** (`es_extended`) using direct MySQL on the `characters` table
- **Standalone** (no framework) using its own `clads_characters` table

### Supported clothing systems

Anything `community_bridge`'s Clothing module supports. This includes:

- `qb-clothing`
- `illenium-appearance`
- `fivem-appearance`
- `esx_skin`
- `rcore_clothing`

Anything else can be wired in via `Config.ClothingFallback`.

### Optional integrations

- A loadscreen resource (default expected name: `loadscreen`) that listens for `SendLoadingScreenMessage` and triggers `loadscreen:continueClicked`.
- A HUD resource that listens to a custom event for a hide signal.

---

## 3. Installation

1. Stop your server.
2. Place `clads_multichar` into your `resources/` directory.
3. Ensure the dependencies are present and started **before** this resource:
   - `ox_lib`
   - `community_bridge`
   - `oxmysql`
4. Add to your `server.cfg` after the dependencies:
   ```cfg
   ensure ox_lib
   ensure oxmysql
   ensure community_bridge
   ensure clads_multichar
   ```
5. (Optional) If you have an existing multicharacter resource (`qb-multicharacter`, `esx_multicharacter`, etc.), make sure its UI is disabled — this resource replaces it.
6. Open `clads_multichar/config.lua` and adjust the values described in section 4.
7. Start the server. On first start the resource will:
   - Create `clads_charslots` (license2 → extra slot count).
   - Create `clads_characters` (only when running in standalone mode).
   - Print the detected framework: `[clads_multichar] Detected framework: <qbox|qb|esx|standalone>`.

---

## 4. Configuration

All configuration lives in `clads_multichar/config.lua`. The file is `escrow_ignore`d so edits are safe across updates.

### Framework

Controls which character bridge is loaded by `bridge/init.lua`.

```lua
Config.Framework = 'auto'   -- 'auto' | 'qbox' | 'qb' | 'esx' | 'standalone'
```

When set to `'auto'`, `bridge/init.lua` checks resource state for `qbx_core`, then `qb-core`, then `es_extended`, falling back to `'standalone'`.

### Language / Locale

UI text lives in `locales/<lang>.json`. The client loader (`getLocale()` in `client/main.lua`) follows this order:

1. If `Config.Language ~= 'auto'`, load `locales/<Config.Language>.json`.
2. Otherwise read the `lang`, `ox:locale`, then `qb_locale` convars and load the matching file.
3. Always falls back to `locales/en.json`.

```lua
Config.Language = 'auto'    -- 'auto' | 'en' | 'tr' | 'de' | 'fr' | <your_locale>
```

Bundled translations: `en.json`, `tr.json`, `de.json`, `fr.json`.

### Character slots

```lua
Config.MaxSlots             = 5   -- Total possible slots in the UI
Config.DefaultUnlockedSlots = 1   -- How many slots every player has by default
```

Players can be granted additional slots via `/charslotver` (see Commands). The grant is keyed by `license2:` and stored in `clads_charslots`.

### UI appearance

```lua
Config.UI = {
    LogoSrc     = './logo.png',
    FontUrl     = 'https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Rajdhani:wght@400;500;600;700&display=swap',
    FontPrimary = "'Inter', -apple-system, BlinkMacSystemFont, sans-serif",
    FontDisplay = "'Rajdhani', sans-serif",
}
```

| Key | Meaning |
|---|---|
| `LogoSrc` | URL or filename of the logo. Defaults to the shipped `./logo.png` — drop a replacement at the same path or point this at any HTTPS URL. |
| `FontUrl` | A Google Fonts (or any CSS) stylesheet URL. The React app injects a `<link>` tag at runtime. |
| `FontPrimary` | CSS `font-family` string for body text — written to `--font-primary`. |
| `FontDisplay` | CSS `font-family` for headings/labels — written to `--font-display`. |

### Spawn

```lua
Config.SpawnMode    = 'defaultSpawn'   -- 'defaultSpawn' | 'lastLocation'
Config.DefaultSpawn = { x = -1037.73, y = -2737.88, z = 20.17, w = 330.0 }
```

| Mode | Behavior |
|---|---|
| `defaultSpawn` | Always spawn at `Config.DefaultSpawn` (the airport). |
| `lastLocation` | Read `player.PlayerData.position` from `Bridge.Framework.GetPlayer(serverId)` and spawn there. Falls back to `DefaultSpawn` if the position is missing. |

### Default settings

These seed the settings panel for new clients. Once the player toggles anything, their values are saved on their client and override the defaults on next login.

```lua
Config.DefaultSettings = {
    streamerMode    = false,
    focusPlayer     = false,
    filterMode      = false,
    filterType      = 'default',
    particleMode    = false,
    particleType    = 'none',
    scenarioMode    = false,
    scenarioType    = 'none',
    weatherMode     = false,
    weatherType     = 'CLEAR',
    timeMode        = false,
    timeType        = '12',
    backgroundType  = 'random',
}
```

| Key | Type | Description |
|---|---|---|
| `streamerMode` | bool | Mask character names and money values with `Config.StreamerMaskChar`. |
| `focusPlayer` | bool | Tightens the camera depth-of-field around the ped. |
| `filterMode` | bool | Enables the timecycle color filter. |
| `filterType` | string | Timecycle modifier name. Examples: `NG_filmic01`, `NG_filmic02`, `NG_filmic03`, `NG_filmic08`, `NG_filmic25`, `cinema`, `spectator1`. |
| `particleMode` | bool | Enables atmospheric particles around the previewed ped. |
| `particleType` | string | Key from `Config.ParticleEffects` (e.g. `leaves`, `petals`). |
| `scenarioMode` | bool | Plays an idle animation on the previewed ped. |
| `scenarioType` | string | Scenario hash name. Examples: `WORLD_HUMAN_SMOKING`, `WORLD_HUMAN_LEANING`, `WORLD_HUMAN_STAND_MOBILE`, `WORLD_HUMAN_AA_COFFEE`. |
| `weatherMode` | bool | Override the weather during selection. |
| `weatherType` | string | Weather hash. Examples: `EXTRASUNNY`, `CLEAR`, `CLOUDS`, `OVERCAST`, `RAIN`, `THUNDER`, `FOGGY`, `SNOW`, `XMAS`. |
| `timeMode` | bool | Override the in-game clock. |
| `timeType` | string | Hour as a string. Bundled options: `'0'`, `'3'`, `'6'`, `'9'`, `'12'`, `'15'`, `'18'`, `'21'`. |
| `backgroundType` | string | Key from `Config.BackgroundLocations`. Built in: `RandomPool`, `City`, `Vinewood` (the UI label `'random'` maps to `RandomPool`). |

### Intro cutscene

```lua
Config.PlayIntroCutscene = true
```

When `true`, the `MP_INTRO_CONCAT` cutscene plays once after the player creates their first character. The client picks the male or female playback based on `IsPedMale(playerId)`. Set to `false` to skip.

The cutscene is triggered by the server-side `clads_multichar:client:firstCharacterDone` net event.

### Loadscreen integration

```lua
Config.LoadScreenResource = 'loadscreen'   -- or false to disable
Config.LoadScreenTimeout  = 60000          -- ms to wait for continueClicked
```

When set, the client (after preview camera setup but before showing the selection UI) does:

```lua
SendLoadingScreenMessage(json.encode({ customEvent = 'showContinue' }))
-- Waits for AddEventHandler('loadscreen:continueClicked', ...)
-- or for Config.LoadScreenTimeout milliseconds
ShutdownLoadingScreenNui()
```

Your loadscreen resource should listen for the `showContinue` custom event and emit `loadscreen:continueClicked` (client event) when the user clicks Continue.

Set `Config.LoadScreenResource = false` to skip the wait entirely.

### Character creation limits

```lua
Config.MinNameLength    = 2
Config.MaxNameLength    = 20
Config.MinBirthYear     = 1960
Config.MaxBirthYear     = 2006
Config.MinHeight        = 160      -- cm
Config.MaxHeight        = 200      -- cm
Config.DefaultBirthdate = '1990-01-15'
```

The character creator UI uses these for client-side input validation and slider ranges.

### HUD control

```lua
Config.HUDHideEvent = false   -- or 'your_hud:toggle'
```

When set to a string, the client triggers `TriggerEvent(Config.HUDHideEvent, true)` when the selection UI opens and `TriggerEvent(Config.HUDHideEvent, false)` when it closes. Leave at `false` to skip.

The selection screen also calls `HideHudAndRadarThisFrame()` and `HideHudComponentThisFrame(...)` for components 1–22 every frame while the UI is open, so a custom hook is only necessary for resources with their own NUI HUD.

### FrameworkEvents (QB legacy compatibility)

```lua
Config.FrameworkEvents = {
    fireQBLegacyEvents      = true,
    triggerQBHouseReset     = true,
    triggerQBClothesCreator = true,
}
```

These flags only fire when the active framework is detected as `qb-core` or `qbx_core`. ESX and standalone always skip them.

| Flag | Effect |
|---|---|
| `fireQBLegacyEvents` | After spawn, fires `QBCore:Server:OnPlayerLoaded` (server) and `QBCore:Client:OnPlayerLoaded` (client) for legacy QB-era resources. |
| `triggerQBHouseReset` | Fires `qb-houses:server:SetInsideMeta` and `qb-apartments:server:SetInsideMeta` to clear stale "inside" metadata. |
| `triggerQBClothesCreator` | After the first spawn, fires `qb-clothes:client:CreateFirstCharacter` to open qb-clothing's first-time appearance editor. |

Always-fired events (regardless of framework or these flags):

- `community_bridge:Client:OnPlayerLoaded`
- `clads_multichar:client:characterSpawned`

Disable any of the QB flags if your QB stack already triggers them on its own, or if you do not use the targeted resource.

### Clothing fallback

```lua
Config.ClothingFallback = {
    enabled  = false,
    resource = 'qb-clothing',
    export   = 'SetPlayerAppearance',
}
```

`setAppearance(data)` in `client/main.lua` first tries `Bridge.Clothing.SetAppearance(serverId, data, false, false)`. If `community_bridge` does not expose a Clothing module on your server (or you want to override it), enable the fallback and the client will call:

```lua
exports[Config.ClothingFallback.resource][Config.ClothingFallback.export](
    exports[Config.ClothingFallback.resource], PlayerPedId(), data
)
```

The fallback only fires when the configured resource is `started`.

### Background locations

Each entry is a `{ pedCoords = vec4(x,y,z,heading), camCoords = vec4(x,y,z,camHeading) }` pair. The keys of `Config.BackgroundLocations` (e.g. `RandomPool`, `City`, `Vinewood`) must match the option values exposed in the UI dropdown — keep `Config.BackgroundLocations` and the `options.background` block of your locale file in sync.

```lua
Config.BackgroundLocations = {
    RandomPool = {
        { pedCoords = vec4(-75.21, -818.81, 326.18, 0.0),   camCoords = vec4(-75.21, -815.81, 327.18, 180.0) },
        { pedCoords = vec4(112.27, -1715.87, 30.11, 140.0), camCoords = vec4(110.27, -1718.37, 31.11, 320.0) },
        { pedCoords = vec4(-682.15, 592.81, 145.39, 180.0), camCoords = vec4(-682.15, 589.31, 146.39, 0.0) },
    },
    City     = { ... },
    Vinewood = { ... },
}
```

The selected pool is consulted on first open and every time `backgroundType` changes; one entry is picked at random.

### Streamer mode

```lua
Config.StreamerMaskChar = '•'
```

Used to mask character names and money values in the UI when `streamerMode` is on.

### Particle effects

The catalogue is fully data-driven. Any registered GTA V ptfx asset works.

```lua
Config.ParticleEffects = {
    leaves = { dict = 'cut_family5',           name = 'cs_fam5_leaf_fall' },
    petals = { dict = 'scr_mp_bunkerbusiness', name = 'scr_bnkr_cfm_confetti' },
    -- snow    = { dict = 'core',           name = 'ent_amb_snow_fall' },
    -- sparks  = { dict = 'scr_jewel_heist', name = 'scr_jewel_heist_drill_spark' },
}
```

The keys here must match the option values listed in `locales/<lang>.json` under `options.particles`. For example, `petals` is shown to the player as `"Flower Petals"` in `en.json`.

The client requests the dict, waits for `HasNamedPtfxAssetLoaded`, then loops `StartNetworkedParticleFxNonLoopedAtCoord` every 800 ms with a small random offset around the previewed ped.

---

## 5. Commands

### `/charslotver <target> <amount>`

Defined in `server/main.lua` via `lib.addCommand('charslotver', ...)`. Restricted to `group.admin` (ace permission).

| Argument | Type | Notes |
|---|---|---|
| `target` | playerId | The server ID of the player to grant slots to. |
| `amount` | number | Must be `1` or `2` — the number of **extra** slots above `DefaultUnlockedSlots`. |

Behavior:

1. Reads the target's `license2:` identifier.
2. `INSERT ... ON DUPLICATE KEY UPDATE` into `clads_charslots (license2, slots)`.
3. Notifies admin: "Gave a total of N slots to <name>".
4. Notifies target: "You have N character slots unlocked! Reconnect to use them."

The grant is read on next character selection by the `clads_multichar:server:getUnlockedSlots` callback (`ox_lib`), which returns `Config.DefaultUnlockedSlots + slots`.

There are no other in-game commands shipped by this resource.

---

## 6. Customization

### UI theme (Dark Red palette)

The character selector ships with the **Dark Red** palette — primary `#FF003C` on a black/dark-grey surface. The colour scheme is fixed for this version. If you need a custom palette to match your server's brand, open a ticket and we can ship a recoloured build. Future releases will move this resource onto the same runtime-theme override used by the newer `clads_*` resources.

### Changing the logo

- Replace the shipped `logo.png` with your own (recommended size: square, transparent PNG, same filename).
- Or set `Config.UI.LogoSrc` to a remote URL — it is rendered directly in the UI.
- The logo also appears on the splash overlay and the creator menu.

### Adding a background location

Append to `Config.BackgroundLocations` in `config.lua`:

```lua
Config.BackgroundLocations.Beach = {
    { pedCoords = vec4(-1605.45, -1024.74, 13.05, 120.0),
      camCoords = vec4(-1607.45, -1027.24, 14.05, 300.0) },
}
```

Then add the matching label in your locale file(s) so it appears in the dropdown:

```json
// locales/en.json
"options": {
  "background": {
    "random":   "Random",
    "city":     "City",
    "vinewood": "Vinewood",
    "Beach":    "Beach"
  }
}
```

The dropdown value the player picks is sent back as `settings.backgroundType`, which is used as the table key in `Config.BackgroundLocations`. The default `'random'` value resolves to `RandomPool` via the fallback in `applyBackground()` (`client/main.lua`).

### Adding a particle effect

```lua
Config.ParticleEffects.snow = { dict = 'core', name = 'ent_amb_snow_fall' }
```

```json
// locales/en.json
"options": {
  "particles": {
    "none":   "Off",
    "leaves": "Leaves",
    "petals": "Flower Petals",
    "snow":   "Snow"
  }
}
```

Restart the resource. The dropdown will show the new entry, and selecting it streams the dict on demand.

### Adding a language

1. Copy `locales/en.json` to `locales/<code>.json` (for example `locales/es.json`).
2. Translate every leaf string. Keep the keys and the `options.*` map keys identical to the English file.
3. Add the file to `fxmanifest.lua`:
   ```lua
   files {
       ...
       'locales/es.json',
   }
   ```
4. Restart the resource. With `Config.Language = 'auto'`, set the convar:
   ```cfg
   set ox:locale "es"
   ```
   Or pin it explicitly:
   ```lua
   Config.Language = 'es'
   ```

---

## 7. Exports & Events

### Events fired by this resource (client)

| Event | When | Notes |
|---|---|---|
| `community_bridge:Client:OnPlayerLoaded` | After every spawn (default and last-location) | Always fired. Standardized by community_bridge. |
| `clads_multichar:client:characterSpawned` | After every spawn | Always fired. Use this in your own resources to react to a character entering the world. |
| `QBCore:Client:OnPlayerLoaded` | After spawn, QB/QBox only | Gated by `Config.FrameworkEvents.fireQBLegacyEvents`. |
| `qb-clothes:client:CreateFirstCharacter` | After first spawn, QB/QBox only | Gated by `Config.FrameworkEvents.triggerQBClothesCreator`. |
| `clads_multichar:client:firstCharacterDone` | Net event consumed by the client | Triggers the `MP_INTRO_CONCAT` intro cutscene when `Config.PlayIntroCutscene` is true. |

### Events fired by this resource (server)

| Event | When |
|---|---|
| `QBCore:Server:OnPlayerLoaded` | After spawn, QB/QBox only, when `fireQBLegacyEvents` is true. |
| `qb-houses:server:SetInsideMeta` | After spawn, QB/QBox only, when `triggerQBHouseReset` is true. |
| `qb-apartments:server:SetInsideMeta` | After spawn, QB/QBox only, when `triggerQBHouseReset` is true. |

### Events listened to (client)

The character chooser is reopened automatically when any of these fire (defensive cross-framework support):

- `community_bridge:Client:OnPlayerUnload`
- `qbx_core:client:playerLoggedOut`
- `QBCore:Client:OnPlayerUnload`
- `esx:onPlayerLogout`

### `ox_lib` callbacks (server)

| Callback | Purpose |
|---|---|
| `clads_multichar:server:getUnlockedSlots` | Returns `Config.DefaultUnlockedSlots + extra` for the calling player. Backed by the `clads_charslots` table. |

### Bridge functions (shared `Characters` global)

The `Characters` global is set by `bridge/init.lua` after loading the framework-specific bridge file. All four bridges expose the same shape:

| Function | Signature | Purpose |
|---|---|---|
| `Characters.GetAll(source)` | → `characters, maxSlots, unlockedSlots` | List slots for the calling player. |
| `Characters.Create(source, slot, data)` | → `citizenid, error?` | Create a character. `data = { firstname, lastname, birthdate, gender, nationality }`. |
| `Characters.Load(source, citizenid)` | → `boolean` | Switch the active character. |
| `Characters.GetPreviewData(source, citizenid)` | → `string` (JSON) or `nil` | Returns ped appearance for live preview. |
| `Characters.GetUnlockedSlots(source)` | → `number` | Convenience wrapper over the server callback. |

---

## 8. Troubleshooting

### Slots not unlocking after `/charslotver`

- Confirm the target's `license2:` identifier exists. The command warns "Player license2 not found." if missing.
- Slots are read on the **next** character selection load. Tell the player to disconnect and reconnect (the in-game notification says "Reconnect to use them.").
- Inspect the table:
  ```sql
  SELECT * FROM clads_charslots WHERE license2 = 'license2:...';
  ```
- Verify `Config.MaxSlots` is at least `DefaultUnlockedSlots + amount`. Slots above `MaxSlots` are not surfaced in the UI.

### Wrong framework detected

Check the server console on resource start:

```
[clads_multichar] Detected framework: <name>
```

If detection picked the wrong framework, hard-pin it:

```lua
Config.Framework = 'qbox'   -- or 'qb' | 'esx' | 'standalone'
```

The detection order in `bridge/init.lua` is `qbx_core` → `qb-core` → `es_extended` → `standalone`. If multiple framework cores are started simultaneously, manual pinning is required.

### Clothing not applying to the previewed ped

1. Confirm `community_bridge` is started **before** `clads_multichar`.
2. Verify your clothing resource is one of the supported set (qb-clothing, illenium-appearance, fivem-appearance, esx_skin, rcore_clothing). If yes, `Bridge.Clothing.SetAppearance` should resolve.
3. If your clothing system is custom, enable `Config.ClothingFallback`:
   ```lua
   Config.ClothingFallback = {
       enabled  = true,
       resource = 'my-clothing',
       export   = 'ApplyAppearance',
   }
   ```
   The export receives `(playerPed, appearanceData)`.
4. For QB stacks where qb-clothing's first-time creator never opens, ensure `Config.FrameworkEvents.triggerQBClothesCreator = true`.

### Background scene black or empty

- The selection routine relies on `NetworkStartSoloTutorialSession`. If the tutorial session never starts within 10 s the client continues anyway, which can leave the world un-streamed. Verify your population streaming is healthy.
- Confirm `Config.BackgroundLocations.RandomPool` (or whichever pool matches `settings.backgroundType`) has at least one entry — `applyBackground()` falls back to `RandomPool` if the requested key is missing, so emptying both will leave the camera floating.
- If the camera stays at the player's last position, the script likely failed to acquire model `mp_m_freemode_01` — check the server console for "[clads_multichar] Model ... failed (attempt N/3)" lines (logged by `loadModelWithRetry`).

### Intro cutscene stuck

- Make sure no other resource calls `StartCutscene` at the same time.
- The cutscene is gated by `Config.PlayIntroCutscene`. Set it to `false` to bypass entirely.
- The cutscene only triggers if a server-side script fires `clads_multichar:client:firstCharacterDone`. If you have a custom new-player flow, you may need to fire this event from your own onboarding logic.
- The cutscene has hard-coded waits (`Wait(24520)`); if a hardware stutter causes desync, the script still runs `StopCutsceneImmediately()` on completion.

### Loadscreen never finishes / Continue button does nothing

- Set `Config.LoadScreenResource = false` to bypass the wait.
- Or shorten `Config.LoadScreenTimeout` (default 60000 ms). After the timeout, the script continues to show the character UI regardless.
- Your loadscreen must:
  1. React to the NUI message `{"customEvent": "showContinue"}` (delivered via `SendLoadingScreenMessage`).
  2. Emit `loadscreen:continueClicked` (a regular client event, not a NUI callback) when the player clicks Continue.

### Settings reset every time

Settings are saved on the player's client. If the player wipes their FiveM data (or moves to another machine) the settings reset. There is no server-side persistence for settings by design.

### Character selection UI never appears

- The UI assets did not load — re-download the resource from your Tebex account and replace the folder. The bundle ships pre-built.
- Check the client console (F8) for missing locale errors. The loader requires at least `locales/en.json`.
- If the issue persists after a clean re-download, open a ticket.

---

## 9. Support

Reach out via the Creative Lads Discord.
