---
title: clads_radio
description: Production radio communication system for FiveM with jammer and GPS modules.
---

# clads_radio

A production-grade radio comms package for FiveM servers. Ships an LCD/CRT-style React UI, framework-agnostic logic via `community_bridge`, support for the three major voice backends (`pma-voice`, `saltychat`, `tokovoice`), restricted job channels, a placeable signal jammer, and a GPS broadcast module for emergency services.

---

## 1. Overview

`clads_radio` replaces the radio scripts that ship with most QB / ESX / standalone server templates with a single, polished package designed for daily roleplay use.

Highlights:

- **Handheld radio prop** with cellphone hand-pose animation while the menu is open.
- **LCD / CRT-style React UI** with scanline overlay, signal meter, battery gauge, in-game clock, and a draggable channel HUD that persists its position in `localStorage`.
- **Three voice backends supported** out of the box: `pma-voice`, `saltychat`, `tokovoice`. The voice abstraction lives in one place (`client/functions.lua`) so swapping backends is a one-line config change.
- **Frequency-based channels** from `1.00` through `Config.MaxFrequency` MHz, with two-decimal precision (e.g. `45.25 MHz`).
- **Restricted job channels** — any channel listed in `Config.RestrictedChannels` rejects players whose job is not on its allowlist. Notification routes through `community_bridge`'s notify adapter.
- **Signal jammer module** — placeable world prop (`sm_prop_smug_jammer` by default) that knocks any non-whitelisted player off their current channel while inside the jamming radius. Restricted jobs (police, sheriff, etc.) are immune and are the only ones who can place / pick up jammers.
- **GPS broadcast module** — emergency-service jobs can publish their callsign + live position to every other GPS-active player. Persisted per-citizen blip color and callsign (MySQL). Vehicle occupants merge into a single shared blip. Dead players' blips flash red / blue.
- **Framework agnostic** — works on Standalone, ESX, QBCore, and QBox via `community_bridge`. Inventory and target backends are auto-detected.
- **Locale-driven** — every UI string and notification routes through `_t(key)` and the active `locales/<code>.lua` file. Built-in: English, Turkish, German, French.

---

## 2. Requirements

| Dependency             | Required | Notes                                                                                  |
|------------------------|----------|----------------------------------------------------------------------------------------|
| `community_bridge`     | Yes      | Handles framework / inventory / target / notify abstraction.                           |
| `ox_lib`               | Yes      | Used for `lib.callback` and `lib.progressBar`.                                         |
| `oxmysql`              | Yes      | Server-side, only used when `Config.GPS.enabled = true` (table auto-created).          |
| Voice backend          | Yes      | Pick **one**: `pma-voice` **or** `saltychat` **or** `tokovoice`.                       |
| Inventory backend      | Optional | Auto-detected through `community_bridge`: `ox_inventory`, `qb-inventory`, `qs-inventory`, `origen_inventory`, `codem-inventory`, `tgiann-inventory`, etc. Only required if `Config.RequireItem = true`. |
| Target backend         | Optional | Required only for jammer pickup interaction: `ox_target`, `qb-target`, or `sleepless_interact`. Disable via `Config.TargetEnabled = false`. |

The resource manifest declares only the two hard dependencies; everything else is detected at runtime by `community_bridge`.

```lua
-- fxmanifest.lua (excerpt)
dependencies {
    'community_bridge',
    'ox_lib',
}
```

---

## 3. Installation

1. Drop the `clads_radio` folder into your `resources/` directory (or a `[clads]` group of your choice).
2. Make sure `community_bridge`, `ox_lib`, and `oxmysql` are present in your server and start **before** `clads_radio`.
3. Start the voice backend you intend to use (`pma-voice`, `saltychat`, or `tokovoice`).
4. Add the resource to your `server.cfg`:

   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure oxmysql
   ensure pma-voice          ## or saltychat / tokovoice
   ensure clads_radio
   ```

5. Open `shared/config.lua` and at minimum:
   - Set `Config.VoiceSystem` to your active voice backend.
   - Adjust `Config.RestrictedChannels` job names to match your server.
   - Adjust `Config.JammerSettings.restricted_jobs` and `Config.GPS.allowedJobs` to match your job names.
6. (Optional) Add the items `clads_radio` and `clads_jammer` to your inventory of choice. See [Section 6 — Items](#6-items).
7. Start the server. On first boot the GPS preferences table is auto-created via `oxmysql`:

   ```sql
   CREATE TABLE IF NOT EXISTS `clads_radio_gps` (
       identifier VARCHAR(64) NOT NULL PRIMARY KEY,
       callsign   VARCHAR(20) NOT NULL DEFAULT '',
       blip_color INT NOT NULL DEFAULT 38
   );
   ```

   The table name is configurable via `Config.DBTable` (default `'clads_radio_gps'`). The table is **only** created when `Config.GPS.enabled = true`.

---

## 4. Configuration

All configuration lives in `shared/config.lua`. The file is excluded from CFX escrow via `escrow_ignore`, so you can edit it freely after purchase. Each block below corresponds 1:1 with a section of that file.

### 4.1 Locale

```lua
Config.Locale = 'en'
```

| Key            | Type   | Default | Description                                                                 |
|----------------|--------|---------|-----------------------------------------------------------------------------|
| `Config.Locale`| string | `'en'`  | Locale code. Loads `locales/<code>.lua`. Falls back to `en` if missing.     |

Built-in locales: `en`, `tr`, `de`, `fr`. Drop a new file in `locales/<code>.lua` to add languages — the dictionary is shipped to the React UI on player load via the `setLocaleDict` NUI message, so your custom strings appear automatically.

### 4.2 Voice System

```lua
Config.VoiceSystem    = 'pma-voice'
Config.RadioEffect    = true
Config.RadioAnimation = true
Config.RadioKey       = 'CAPS'
```

| Key                     | Type     | Default       | Description                                                                                  |
|-------------------------|----------|---------------|----------------------------------------------------------------------------------------------|
| `Config.VoiceSystem`    | string   | `'pma-voice'` | One of `'pma-voice'`, `'saltychat'`, `'tokovoice'`. The matching resource must be running.   |
| `Config.RadioEffect`    | boolean  | `true`        | Enables the voice submix that gives speech the "real radio" treatment. (pma-voice/tokovoice) |
| `Config.RadioAnimation` | boolean  | `true`        | Plays the radio prop + cellphone animation when the menu opens.                              |
| `Config.RadioKey`       | string   | `'CAPS'`      | Default push-to-talk key. Saltychat ignores this and uses its own binding.                   |

The voice abstraction (in `client/functions.lua`) forwards three operations to whichever backend is active: `connectVoiceRadio(channel)`, `leaveVoiceRadio()`, `setVoiceRadioVolume(value)`. Mute toggles only forward when the backend is `pma-voice`.

### 4.3 Item Requirement

```lua
Config.RequireItem      = true
Config.RadioItem        = 'clads_radio'
Config.InventoryBackend = 'auto'
```

| Key                       | Type    | Default          | Description                                                                                  |
|---------------------------|---------|------------------|----------------------------------------------------------------------------------------------|
| `Config.RequireItem`      | boolean | `true`           | When `true`, the player must carry `Config.RadioItem` to open the radio. Losing it auto-disconnects. |
| `Config.RadioItem`        | string  | `'clads_radio'`  | Inventory item name.                                                                         |
| `Config.InventoryBackend` | string  | `'auto'`         | `'auto'` lets `community_bridge` auto-detect. May be forced to `'ox_inventory'`, `'qb-inventory'`, `'qs-inventory'`, `'codem-inventory'`, `'origen_inventory'`, etc. |

A 1-second client loop (in `client/loops.lua`) checks for the item every tick. If the player is on the radio, has the menu open, or is broadcasting GPS and the item disappears, everything is torn down: GPS is disabled, the radio leaves its channel, and the UI closes.

### 4.4 Input

```lua
Config.RadioToggleKey = 'Y'
```

| Key                       | Type   | Default | Description                                                                              |
|---------------------------|--------|---------|------------------------------------------------------------------------------------------|
| `Config.RadioToggleKey`   | string | `'Y'`   | Key that toggles the radio UI (when the player has the item). Bound via `RegisterKeyMapping` so end-users can rebind it from FiveM's keybind menu. |
| `Config.RadioKey`         | string | `'CAPS'`| Push-to-talk key forwarded to the voice backend. See [Voice System](#4-2-voice-system).  |

### 4.5 Frequency Rules

```lua
Config.MaxFrequency = 500

Config.RestrictedChannels = {
    [1]  = { police = true, ambulance = true, sahp = true, bcso = true },
    [2]  = { police = true, ambulance = true, sahp = true, bcso = true },
    -- … channels 3–10 follow the same pattern by default
}
```

| Key                          | Type   | Default      | Description                                                                                |
|------------------------------|--------|--------------|--------------------------------------------------------------------------------------------|
| `Config.MaxFrequency`        | number | `500`        | Highest frequency a player can tune to. Anything `<= 0` or `> MaxFrequency` is rejected.   |
| `Config.RestrictedChannels`  | table  | (channels 1–10) | `[<freq>] = { <jobName> = true, ... }`. Any channel **not** listed is open to everyone.   |

Restriction is enforced server-aware but client-checked at the `joinRadio` NUI callback (see `client/functions.lua`). When access is denied, the locale key `restricted_channel` is shown.

### 4.6 PTT Click Sound

```lua
Config.RadioClickRange = 5.0
```

| Key                       | Type   | Default | Description                                                                              |
|---------------------------|--------|---------|------------------------------------------------------------------------------------------|
| `Config.RadioClickRange`  | number | `5.0`   | Range (m) at which nearby non-radio players hear the radio click. Set to `0` to disable. |

The click is broadcast server-side from `clads_radio:pttClick` and re-emitted to nearby players through `clads_radio:nearbyPttClick` with distance-attenuated volume.

### 4.7 Jammer Settings

```lua
Config.JammerSettings = {
    available                    = true,
    item_name                    = 'clads_jammer',
    restricted_jobs              = { police = true, sahp = true, bcso = true },
    object                       = 'sm_prop_smug_jammer',
    min_distance_between_jammers = 15,
    range                        = 10.0,
}
```

| Key                                 | Type     | Default                  | Description                                                                                |
|-------------------------------------|----------|--------------------------|--------------------------------------------------------------------------------------------|
| `available`                         | boolean  | `true`                   | Master switch. When `false`, every jammer code path (events, loops, target zones) is skipped at startup. |
| `item_name`                         | string   | `'clads_jammer'`         | Useable item that triggers placement.                                                      |
| `restricted_jobs`                   | table    | police / sahp / bcso     | Jobs allowed to **place** and **pick up** jammers. They are also immune to jamming.        |
| `object`                            | string   | `'sm_prop_smug_jammer'`  | World prop spawned at the player's feet (offset 2m forward, snapped to ground).            |
| `min_distance_between_jammers`      | number   | `15`                     | Minimum spacing (m) between two jammers — prevents stacking.                               |
| `range`                             | number   | `10.0`                   | Effective jamming radius (m).                                                              |

### 4.8 Target Backend

```lua
Config.TargetEnabled = true
Config.TargetBackend = 'auto'
```

| Key                    | Type    | Default | Description                                                                                |
|------------------------|---------|---------|--------------------------------------------------------------------------------------------|
| `Config.TargetEnabled` | boolean | `true`  | When `false`, no target zones are added — jammer removal must use the admin command.       |
| `Config.TargetBackend` | string  | `'auto'`| `'auto'` lets `community_bridge` pick. Force with `'ox_target'`, `'qb-target'`, `'sleepless_interact'`. |

The pickup zone uses a 2.5m sphere zone with the icon `fas fa-tower-cell` and the locale label `jammer_remove_label`.

### 4.9 GPS Module

```lua
Config.GPS = {
    enabled        = true,
    allowedJobs    = { police = true, ambulance = true, bcso = true, sahp = true },
    updateInterval = 500,
    blip = {
        sprite        = 1,
        vehicleSprite = 225,
        color         = 38,
        deadColor     = 1,
        scale         = 0.85,
    },
    colorOptions = {
        { label = 'gps_color_blue',   id = 3,  hex = '#3B82F6' },
        { label = 'gps_color_brown',  id = 46, hex = '#92603D' },
        { label = 'gps_color_orange', id = 17, hex = '#F97316' },
        { label = 'gps_color_red',    id = 1,  hex = '#EF4444' },
        { label = 'gps_color_green',  id = 2,  hex = '#22C55E' },
        { label = 'gps_color_purple', id = 27, hex = '#A855F7' },
        { label = 'gps_color_white',  id = 0,  hex = '#FFFFFF' },
        { label = 'gps_color_yellow', id = 5,  hex = '#EAB308' },
    },
}
```

| Key                           | Type     | Default      | Description                                                                                |
|-------------------------------|----------|--------------|--------------------------------------------------------------------------------------------|
| `enabled`                     | boolean  | `true`       | Master switch. When `false`, the entire GPS subsystem is removed (no DB table, no callbacks, no UI tab). |
| `allowedJobs`                 | table    | police / ambulance / bcso / sahp | Jobs whose members can broadcast on the GPS network. Other jobs see the GPS tab hidden. |
| `updateInterval`              | number   | `500`        | Position broadcast interval (ms). Lower = smoother blips, higher = lighter on netcode.     |
| `blip.sprite`                 | number   | `1`          | On-foot blip sprite ID.                                                                    |
| `blip.vehicleSprite`          | number   | `225`        | Sprite shown when the GPS player is in a vehicle.                                          |
| `blip.color`                  | number   | `38`         | Fallback GTA blip color when no per-player color has been saved.                           |
| `blip.deadColor`              | number   | `1`          | Color the blip flashes to when the player is dead (alternates with blue).                  |
| `blip.scale`                  | number   | `0.85`       | Blip scale.                                                                                |
| `colorOptions`                | table[]  | 8 colors     | Color swatches shown in the GPS tab. Each entry has `id` (GTA blip color ID), `hex` (CSS), and `label` (translation key or literal). |

Selected color persists per-citizen via `Config.DBTable`. If `label` is a translation key the UI resolves it through the active locale; otherwise the literal string is shown.

### 4.10 Database Table

```lua
Config.DBTable = 'clads_radio_gps'
```

| Key                | Type   | Default              | Description                                                                          |
|--------------------|--------|----------------------|--------------------------------------------------------------------------------------|
| `Config.DBTable`   | string | `'clads_radio_gps'`  | Table name used to persist per-citizen GPS preferences (callsign + blip color). Auto-created on first boot when `Config.GPS.enabled = true`. |

Schema:

```sql
CREATE TABLE IF NOT EXISTS `clads_radio_gps` (
    identifier VARCHAR(64) NOT NULL PRIMARY KEY,
    callsign   VARCHAR(20) NOT NULL DEFAULT '',
    blip_color INT NOT NULL DEFAULT 38
);
```

Identifier resolution falls back to the player's `license:` identifier when `Framework.GetPlayerIdentifier(src)` returns nothing — so the table works on standalone setups too.

---

## 5. Commands

### `/cladsradio`

Client-side toggle for the radio UI. Bound via `RegisterKeyMapping` to `Config.RadioToggleKey` (default `Y`), so end-users can rebind it from the FiveM keybind menu without touching config. Honours the item check (`Config.RequireItem`) and refuses to open while the player is dead.

### `/cladsradio_clearjammers`

Admin-only console / chat command. Removes every placed jammer on the server in a single sweep. Useful when:

- `Config.TargetEnabled = false` and there is no in-world way to pick up jammers.
- A jammer was placed on inaccessible geometry.
- You need to reset state during testing.

Permission check uses `Framework.GetIsFrameworkAdmin(src)` from `community_bridge`. The command is also runnable from the server console (`src == 0` bypass).

```text
> cladsradio_clearjammers
[clads_radio] All jammers cleared.
```

---

## 6. Items

`clads_radio` registers its useable items through `Framework.RegisterUsableItem` from `community_bridge`. On standalone setups (no inventory) the call is silently skipped, but you can still trigger the events directly.

### 6.1 `clads_radio`

The radio item itself. Using it from the inventory fires:

```lua
TriggerClientEvent('clads_radio:use-radio', src)
```

which toggles the radio UI on the target client. Required only when `Config.RequireItem = true`.

Suggested `ox_inventory` entry:

```lua
['clads_radio'] = {
    label  = 'Radio',
    weight = 200,
    stack  = false,
    close  = true,
    description = 'Two-way radio. Press [Y] to open.',
    client = { image = 'clads_radio.png' },
},
```

### 6.2 `clads_jammer`

The signal jammer item. Using it from the inventory fires:

```lua
TriggerClientEvent('clads_radio:use-jammer', src)
```

which kicks off the placement flow: job check → minimum-distance check → 2.5-second progress bar → server creates a jammer entity and broadcasts it to every client. The item name is configurable via `Config.JammerSettings.item_name`.

Suggested `ox_inventory` entry:

```lua
['clads_jammer'] = {
    label  = 'Signal Jammer',
    weight = 1500,
    stack  = true,
    close  = true,
    description = 'Disrupts nearby radio communications.',
    client = { image = 'clads_jammer.png' },
},
```

---

## 7. Customization

### 7.1 UI Theme — dark_red palette

The LCD now ships with the **dark_red** palette. The CSS variable name kept its historical `--lcd-green` identifier for backwards compatibility, but its value is the dark-red accent (`#FF003C`).

```css
/* web/src/index.css */
:root {
  --lcd-green:      #FF003C;                    /* primary accent — dark red */
  --lcd-green-dim:  rgba(255, 0, 60, 0.06);
  --lcd-green-glow: rgba(255, 0, 60, 0.4);
  --lcd-amber:      #F59E0B;
  --lcd-amber-dim:  rgba(245, 158, 11, 0.06);
  --lcd-amber-glow: rgba(245, 158, 11, 0.35);
  --lcd-red:        #FF003C;                    /* alarm / disconnect */
  --lcd-red-glow:   rgba(255, 0, 60, 0.4);
}
```

Surface tokens used by the LCD panel itself:

```css
/* web/src/components/Radio/index.css */
.radio-screen {
  background:
    radial-gradient(ellipse at 50% 30%, rgba(255, 0, 60, 0.015) 0%, transparent 60%),
    linear-gradient(180deg, #0D0D0D 0%, #1A1A1A 50%, #0D0D0D 100%);
}
```

To re-skin the radio:

1. Edit the four `--lcd-*` accent variables in `web/src/index.css`.
2. Edit the surface gradient (`#0D0D0D` → `#1A1A1A`) in `web/src/components/Radio/index.css`.
3. Rebuild the web bundle (`web/build/**` is what the resource serves).

The draggable channel HUD and the scanline overlay both consume the same tokens, so a single palette change propagates everywhere.

### 7.2 Adding restricted channels

Add new entries to `Config.RestrictedChannels` keyed by frequency. Each value is a job allowlist:

```lua
Config.RestrictedChannels = {
    [1]  = { police = true, sahp = true, bcso = true },
    [2]  = { police = true, sahp = true, bcso = true, ambulance = true },

    -- New: a private dispatch channel for dispatchers and supervisors
    [25] = { dispatch = true, police_chief = true, sahp_chief = true },

    -- New: an EMS-only triage channel
    [40] = { ambulance = true },
}
```

Players whose `PlayerData.job.name` does not appear in the allowlist receive the `restricted_channel` notification and the connect attempt fails. Channels not listed in the table remain open to everyone.

### 7.3 Adding GPS color options

Each entry in `Config.GPS.colorOptions` becomes a swatch in the GPS tab:

```lua
{ label = 'gps_color_pink', id = 19, hex = '#FF6BCB' },
```

| Field   | Description                                                                                          |
|---------|------------------------------------------------------------------------------------------------------|
| `label` | A locale key (e.g. `'gps_color_pink'`) **or** a literal display string. Resolved via `_t(label)`.    |
| `id`    | A GTA V blip color ID. Used in-world via `SetBlipColour`.                                            |
| `hex`   | CSS color used for the UI swatch background.                                                         |

If you use a translation key, add the matching string in every `locales/*.lua` file. Otherwise the literal string is rendered as-is.

### 7.4 Jammer prop / range tuning

```lua
Config.JammerSettings.object                       = 'sm_prop_smug_jammer'
Config.JammerSettings.range                        = 10.0
Config.JammerSettings.min_distance_between_jammers = 15
```

- **`object`** — any valid prop hash or model name. The client snaps it to the ground via `PlaceObjectOnGroundProperly`, so most static props work.
- **`range`** — the radius (m) within which non-restricted jobs are forced off the radio. The client tests this every 1000ms.
- **`min_distance_between_jammers`** — denies placement when another jammer is within this many meters.

Keep `range` ≤ `min_distance_between_jammers` if you want overlapping coverage to require deliberate placement.

---

## 8. Exports & Events

### 8.1 Client exports

| Export                                | Returns / Notes                                                                              |
|---------------------------------------|----------------------------------------------------------------------------------------------|
| `exports.clads_radio:GetGPSState()`   | Returns `{ enabled = boolean, callsign = string, lastCallsign = string }`. Useful for radial menus. |
| `exports.clads_radio:ToggleGPS()`     | Toggles GPS on/off using the last-used callsign. Returns `true` on success, `false` if no callsign is stored or the job lacks GPS access. |

Example — wiring into a radial menu:

```lua
local state = exports.clads_radio:GetGPSState()
if state.enabled then
    exports.clads_radio:ToggleGPS()  -- disables
else
    exports.clads_radio:ToggleGPS()  -- re-enables with last callsign
end
```

### 8.2 Client net events

| Event                              | Direction       | Payload                                            | Purpose                                                  |
|------------------------------------|-----------------|----------------------------------------------------|----------------------------------------------------------|
| `clads_radio:use-radio`            | server → client | —                                                  | Toggles the radio UI. Fired by the useable item.         |
| `clads_radio:use-jammer`           | server → client | —                                                  | Begins the jammer placement flow. Fired by the useable item. |
| `clads_radio:createJammer`         | server → client | `objectId, model`                                  | Asks the placing client to spawn the prop locally.       |
| `clads_radio:syncJammers`          | server → client | `data` (table)                                     | Full jammer table broadcast — used on join / change.     |
| `clads_radio:removeJammer`         | server → client | `objectId`                                         | Removes a single jammer entity from clients.             |
| `clads_radio:refreshPlayerList`    | server → client | —                                                  | Asks the client to re-pull the channel roster.           |
| `clads_radio:nearbyPttClick`       | server → client | `isOn, distance, maxRange`                         | Plays the nearby PTT click sound with attenuated volume. |
| `clads_radio:syncGPSPositions`     | server → client | `blipData, playerList`                             | Per-tick GPS push to a single GPS-active client.         |

### 8.3 Server net events

| Event                              | Direction       | Payload                          | Purpose                                                  |
|------------------------------------|-----------------|----------------------------------|----------------------------------------------------------|
| `clads_radio:joinChannel`          | client → server | `channel, monicker?`             | Joins a frequency. Optional 20-char nickname.            |
| `clads_radio:leaveChannel`         | client → server | —                                | Leaves the current channel and clears the monicker.      |
| `clads_radio:setDeadState`         | client → server | `isDead`                         | Reports player death state — flips dead-blip flash.      |
| `clads_radio:spawnJammer`          | client → server | `model`                          | Reserves an `objectId` and asks the placer to spawn.     |
| `clads_radio:setJammerData`        | client → server | `objectId, { coords }`           | Confirms the placed jammer's final coordinates.          |
| `clads_radio:deleteJammer`         | client → server | `objectId`                       | Removes a jammer by id (after target pickup).            |
| `clads_radio:requestJammerSync`    | client → server | —                                | Asks for the current jammer table (fired on player load).|
| `clads_radio:setGPSColor`          | client → server | `colorId`                        | Persists the player's chosen blip color.                 |
| `clads_radio:disableGPS`           | client → server | —                                | Removes the player from the GPS broadcast list.          |
| `clads_radio:updateGPS`            | client → server | `{ coords, heading, vehicleNetId }`| Per-tick position update from a GPS-active player.       |
| `clads_radio:pttClick`             | client → server | `isOn`                           | Reports a PTT key-down/up so the server can relay clicks.|

### 8.4 Server callbacks (`lib.callback`)

| Name                                      | Returns                                                  | Purpose                                                  |
|-------------------------------------------|----------------------------------------------------------|----------------------------------------------------------|
| `clads_radio:getPlayerData`               | `{ identifier, job, locale, localeCode }`                | Initial payload sent on player load.                     |
| `clads_radio:getPlayersInChannel(channel)`| Array of `{ source, name, pid, isMuted, isDead }`        | Roster for the given channel.                            |
| `clads_radio:getGPSPrefs`                 | `{ callsign, blipColor }` or `nil`                       | Loads the saved GPS prefs for the caller.                |
| `clads_radio:canUseGPS`                   | `boolean`                                                | Server-side check that the caller's job is GPS-eligible. |
| `clads_radio:enableGPS(callsign)`         | `boolean`                                                | Activates GPS broadcasting for the caller.               |

### 8.5 Bridge events consumed

| Event                                            | Purpose                                                                         |
|--------------------------------------------------|---------------------------------------------------------------------------------|
| `community_bridge:Client:OnPlayerLoaded`         | Re-fetch player data and request a jammer sync.                                 |
| `community_bridge:Client:OnPlayerUnload`         | Disable GPS, leave the channel, reset client state.                             |
| `community_bridge:Client:OnPlayerJobUpdate`      | Refreshes `PlayerData.job`. Drops GPS broadcast if the new job lacks access.    |
| `pma-voice:setTalkingOnRadio`                    | Forwarded to the HUD as `radioTalkingState` so you can highlight the speaker.   |

---

## 9. Troubleshooting

### Voice does not transmit / receive

- Confirm `Config.VoiceSystem` matches the **running** voice resource name. The three accepted values are `pma-voice`, `saltychat`, `tokovoice`.
- Make sure the voice resource starts **before** `clads_radio` in `server.cfg` — the client tries to call its exports as soon as a channel is joined.
- For `saltychat`, the volume slider uses a 0.0–1.0 scale internally (the slider value is divided by 100). If you hear audio but volume seems frozen at the wrong level, that's normal for SaltyChat.
- The mute button only does something on `pma-voice` (and `tokovoice`, which mirrors the same API). On `saltychat` the button is hidden.

### "You are not authorized to join this channel."

- The frequency is listed in `Config.RestrictedChannels` and the player's job is not on its allowlist.
- Verify the **exact** job name returned by your framework. `community_bridge` exposes it via `Framework.GetPlayerJobData(src).jobName`. Capitalisation matters.
- If you renamed a job (e.g. `lspd` → `police`), update the table accordingly.

### Jammer is placed but radios are not blocked

- Confirm `Config.JammerSettings.available = true`.
- Check the player's job: members of `restricted_jobs` are immune by design. Test with a civilian.
- The jammer range check runs every 1000ms — there is up to a one-second delay between entering the radius and being kicked off the channel.
- The disconnect only fires when the affected player is currently on a channel (or had `lastChannel > 0`); a player with the radio closed will see no notification, that is expected.

### GPS blips do not appear

- `Config.GPS.enabled` must be `true` on **both** the broadcaster and the receiver — the entire subsystem is gated on this flag at startup.
- The player must have a valid identifier. On standalone, that comes from the `license:` identifier; if your server strips identifiers, GPS prefs cannot be saved or loaded.
- The receiver must also be GPS-active. Players who have not entered a callsign do not receive `clads_radio:syncGPSPositions` payloads.
- Check the database — the table is auto-created on first boot via `oxmysql`. If `oxmysql` is not running, the GPS table never exists and `lib.callback('clads_radio:getGPSPrefs')` will return `nil` for everyone.

### Target zone does not appear on a placed jammer

- `Config.TargetEnabled` must be `true`.
- A target backend must be running — `ox_target`, `qb-target`, or `sleepless_interact`. `community_bridge` auto-detects when `Config.TargetBackend = 'auto'`.
- Only members of `Config.JammerSettings.restricted_jobs` get the zone. Civilians cannot see the pickup option by design.
- Zones are added/removed in a 2-second loop — there is a brief delay after placement.
- If you cannot use any target system, use the admin command `/cladsradio_clearjammers` to wipe state.

### "No signal — connection refused."

The player is currently inside a jammer's radius and tried to join a channel. Move outside the jammer or remove it.

### Radio closes immediately after opening

The item is gone. The 1-second item watcher (in `client/loops.lua`) calls `toggleRadio(false)` and emits `resetRadio` to the UI when `hasRadioItem()` becomes false. Re-add the `clads_radio` item, or set `Config.RequireItem = false`.

---

## 10. Support

Reach out via the Creative Lads Discord.

