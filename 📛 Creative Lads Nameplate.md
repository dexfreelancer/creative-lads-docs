# Creative Lads — Nameplate (`clads_nameplate`)

A DUI-based player nameplate that floats above every nearby player, showing
a customizable name, title, level orb, wanted stars, optional badge, and
four cosmetic overlay slots (flair, frame, name color, name effect). The
resource ships with a brand-aligned cosmetic menu so players can equip what
they own, and a public export surface so any other resource — skill systems,
admin panels, ticket systems, donator shops — can drive the values shown
above each player.

Like every Creative Lads resource, it is framework-agnostic through
`community_bridge` and works on QBCore, QBox, ESX, or standalone without
forks. Every visual element is independently togglable: server owners can
pin a static title to the entire population, hide the level orb, replace
the badge with a server logo image, or strip the cosmetic menu entirely.

---

## 1. Overview

`clads_nameplate` is a UI overlay. It does not award titles or levels on
its own — that's the job of your skill / wanted / admin / shop systems.
It exposes a clean public API those systems call into, and it draws what
you tell it to draw above every player at render distance.

**Who it's for**

- Roleplay servers that want a polished nameplate beyond `pma-voice`'s
  built-in HUD, with prestige titles, level visibility, and cosmetic flex.
- Servers running custom skill or progression systems that need a place
  to surface levels and unlocked titles.
- Any server that wants to monetize cosmetics — flair emojis, frame
  palettes, name colours and animated effects — without writing a custom
  rendering layer.

**Feature highlights**

- DUI nameplate with `lib.dui` — one offscreen browser per nearby player,
  drawn via `DrawSprite` with a distance-based alpha fade.
- Three render loops: a slow scan (lifecycle), a per-frame draw (sprites),
  and a periodic data refresh — so a hundred players nearby costs you
  one fetch per plate per ~2 seconds, not one per frame.
- Four cosmetic categories — **flair**, **frame**, **name colour**,
  **name effect** — with rarity tints, owned/locked states, and live
  in-menu nameplate preview.
- Modern in-game UI on the Creative Lads "Purple Haze + Liquid Glass"
  design system.
- 12 public exports for buyer-side integration: `SetPlayerLevel`,
  `SetWantedStars`, `SetBadge`, `SetPlayerTitle`, `SetCustomTitle`,
  `SetPoliceMode`, `GrantCosmetic`, `RevokeCosmetic`, `EquipCosmetic`,
  `UnequipCosmetic`, `GetEquippedCosmetics`, `BuildRenderData`.
- Per-element display modes (`dynamic` / `static` / `hidden`) so admins
  can pin or remove any single field from `config.lua`.
- Static badge supports image filenames — drop `web/build/logo.png` and
  the entire population renders your server logo as the left ornament.
- JSON locales (en/tr/de/fr/es) with the standard `clads_locale` →
  `ox:locale` → `qb_locale` → `lang` resolution chain.
- SQL persistence for owned + equipped cosmetics + active title — the
  player picks once and the choice survives restarts.

---

## 2. Requirements

### Hard dependencies

Pulled directly from `fxmanifest.lua:48-52`:

```lua
dependencies {
    'community_bridge',
    'ox_lib',
    'oxmysql',
}
```

| Dependency        | Purpose                                                        |
| ----------------- | -------------------------------------------------------------- |
| `community_bridge` | Framework lifecycle (`OnPlayerLoaded`/`OnPlayerUnload`), notify, identifier resolution. |
| `ox_lib`          | `lib.dui`, `lib.callback`, `lib.notify`, `lib.addRadialItem`.  |
| `oxmysql`         | Persists owned + equipped cosmetics and the active title.      |

### SQL

Run `sql/install.sql` once before starting the resource:

```sql
CREATE TABLE clads_nameplate_owned     (identifier, category, cosmetic, granted_at);
CREATE TABLE clads_nameplate_equipped  (identifier, category, cosmetic);
CREATE TABLE clads_nameplate_titles    (identifier, title_id);
```

Three tables, all `utf8mb4`, primary keys on `(identifier, category[, cosmetic])`.

### Frameworks (auto-detected)

`Config.Framework = 'auto'` defers to `community_bridge`. Override with
`'qb'`, `'qbox'`, `'esx'`, or `'standalone'` if you need to pin the
choice (`config.lua:21`).

### Optional integrations

Everything beyond name + plate chrome is optional and disabled by default
unless another resource calls the corresponding export:

| Field        | Off by default unless a buyer script calls…   |
| ------------ | --------------------------------------------- |
| `level`      | `exports.clads_nameplate:SetPlayerLevel(src, n)` |
| `wanted`     | `exports.clads_nameplate:SetWantedStars(src, 0..5)` |
| `badge`      | `exports.clads_nameplate:SetBadge(src, glyph_or_image)` |
| `policeMode` | `exports.clads_nameplate:SetPoliceMode(src, true)` |
| `title`      | Defaults to `Config.Behaviour.defaultTitleId`; change with `SetPlayerTitle` or `SetCustomTitle`. |

If no integration is wired, the plate gracefully shows just the name +
title + level (level defaults to `1`).

---

## 3. Installation

1. **Download** the resource and copy the `clads_nameplate/` folder into
   your server's `resources/` directory.
2. **Run `sql/install.sql`** against your server database.
3. **Install dependencies** so they start before this resource:
   - `community_bridge`
   - `ox_lib`
   - `oxmysql`
4. **Start the resource.** Add to `server.cfg` after dependencies:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure oxmysql
   ensure clads_nameplate
   ```
5. **Test.**
   - Have two players join. Each should see the other's nameplate above
     their head with their player name and the default `Newcomer` title.
   - Type `/cosmetics` (or open the radial menu) to confirm the cosmetic
     menu opens and shows four locked categories (no cosmetics owned by
     default).
   - From the server console, run
     `exports.clads_nameplate:GrantCosmetic(<src>, 'flair', 'crown')` and
     reopen the menu — the Crown card should now show as owned.
6. **Wire your skill / wanted / admin systems** by calling the exports
   listed in §7. Until you do, the plate just shows name + title + level.

---

## 4. Configuration

`config.lua` is split into nine sections. Headings below mirror the order
in the file.

### 4.1 Framework

```lua
Config.Framework = 'auto'   -- 'auto' | 'esx' | 'qb' | 'qbox' | 'standalone'
```

`auto` is recommended; pin it only if you run multiple cores side by side
or want to force standalone for testing.

### 4.2 Locale

```lua
Config.Locale = 'auto'      -- 'auto' resolves through convars
```

Resolution order: `clads_locale` → `ox:locale` → `qb_locale` → `lang` →
`en` (`shared/locale.lua:20-30`). Drop additional files into
`locales/<code>.json` to add new languages.

### 4.3 Debug

```lua
Config.Debug = false
```

Enables verbose console output. Leave OFF on production servers.

### 4.4 Render tuning

Defined in `Config.Render`. These are the knobs that control how DUI
nameplates appear in-world.

```lua
Config.Render = {
    distance       = 30.0,   -- max metres at which a plate is shown
    fadeStart      = 20.0,   -- metres at which the plate starts fading
    duiWidth       = 512,    -- DUI texture resolution
    duiHeight      = 80,
    spriteWidth    = 0.243,  -- screen-relative sprite size
    headBoneOffset = 0.35,   -- vertical offset above the head bone
    scanInterval   = 500,    -- ms between create/destroy scans
    dataInterval   = 2000,   -- ms between server-side data refreshes
}
```

| Key             | Type   | Default | Description                                                       |
| --------------- | ------ | ------- | ----------------------------------------------------------------- |
| `distance`      | float  | 30.0    | Maximum visible range. Beyond this, the DUI is destroyed.        |
| `fadeStart`     | float  | 20.0    | Linear alpha fade between this distance and `distance`.          |
| `duiWidth/Height` | int  | 512×80  | Texture resolution. Increase for sharper text on 4K clients.     |
| `spriteWidth`   | float  | 0.243   | Sprite width as a fraction of the screen. Lower = tighter plate. |
| `headBoneOffset` | float | 0.35    | Vertical offset above bone 31086 (head).                         |
| `scanInterval`  | int    | 500ms   | How often to walk `GetActivePlayers()` and update visibility.    |
| `dataInterval`  | int    | 2000ms  | How often to ask the server for fresh title/level/wanted data.   |

### 4.5 Behaviour

```lua
Config.Behaviour = {
    hideOwnPlate    = false,
    hideWhileTyping = true,
    defaultTitleId  = 'newcomer',
}
```

| Key              | Type   | Default     | Description                                                      |
| ---------------- | ------ | ----------- | ---------------------------------------------------------------- |
| `hideOwnPlate`   | bool   | `false`     | Whether to hide the local player's own nameplate.                |
| `hideWhileTyping`| bool   | `true`      | Hides a target's plate while they're typing in chat. Requires a chat resource that fires `clads_nameplate:client:setTyping`. |
| `defaultTitleId` | string | `newcomer`  | Initial title for players with no persisted choice. Must match a `Config.Titles` id. |

### 4.6 Display — per-element visibility

This is the layer server owners reach for first. Every nameplate element
is independently configurable via `Config.Display`. Each section supports
three modes:

| Mode      | Behaviour                                                              |
| --------- | ---------------------------------------------------------------------- |
| `dynamic` | Render whatever the runtime API sets. Hides when no value is present.  |
| `static`  | Always render the `static` value below; runtime exports are ignored.    |
| `hidden`  | Element is never rendered, regardless of runtime.                      |

```lua
Config.Display = {
    title = {
        mode   = 'dynamic',
        static = { label = 'Creative Lads', color = '#9D4EDD', glow = false },
    },
    level       = { mode = 'dynamic', static = 1 },
    wantedStars = { mode = 'dynamic', hideWhenZero = true },
    badge       = { mode = 'dynamic', static = 'logo' },
    policeMode  = { mode = 'dynamic' },
    flair       = { mode = 'dynamic' },
    frame       = { mode = 'dynamic' },
    nameColor   = { mode = 'dynamic' },
    nameEffect  = { mode = 'dynamic' },
}
```

**Common recipes**

| Outcome                                            | Config change                                                            |
| -------------------------------------------------- | ------------------------------------------------------------------------ |
| Hide the title row entirely                        | `Config.Display.title.mode = 'hidden'`                                   |
| Pin every player's title to "MyServer"             | `title = { mode = 'static', static = { label = 'MyServer', color = '#9D4EDD' } }` |
| No level orb at all                                | `Config.Display.level.mode = 'hidden'`                                   |
| Use server logo as the badge for everyone          | drop `web/build/logo.png` + `badge = { mode = 'static', static = 'web/build/logo.png' }` |
| Disable wanted stars                               | `Config.Display.wantedStars.mode = 'hidden'`                              |
| Disable cosmetics on the plate (and from the menu) | set the four cosmetic categories to `mode = 'hidden'`                     |

The `badge.static` value can be a glyph id (`admin`/`vip`/`donator`/`logo`),
an image filename relative to the resource root (`web/build/logo.png`), or a
full URL (`https://cdn.example.com/badge.png`). The renderer detects images
by extension and switches from a `<span>` glyph to an `<img>`.

### 4.7 Cosmetic menu

```lua
Config.Menu = {
    enabled  = true,
    command  = 'cosmetics',
    radial   = { enabled = true, id = 'clads_nameplate_open', icon = 'palette' },
    categories = {
        flair       = true,
        frame       = true,
        name_color  = true,
        name_effect = true,
    },
    showPreview = true,
}
```

| Key              | Type    | Default        | Description                                                      |
| ---------------- | ------- | -------------- | ---------------------------------------------------------------- |
| `enabled`        | bool    | `true`         | Master switch. Disables command + radial + the entire NUI.       |
| `command`        | string  | `'cosmetics'`  | Slash command. Set to `nil` to skip registration.                |
| `radial.enabled` | bool    | `true`         | Whether to add an `ox_lib` radial menu item.                     |
| `radial.icon`    | string  | `'palette'`    | FontAwesome icon name for the radial item.                       |
| `categories.*`   | bool    | `true`         | Per-category UI surface toggle. Setting any to `false` hides the tab. |
| `showPreview`    | bool    | `true`         | Live nameplate preview at the top of the menu. Disable on small screens. |

A category is shown in the menu only if **both** `Config.Menu.categories[cat]`
is true **and** the matching `Config.Display.<cat>.mode` is `dynamic`.
Static or hidden display modes drop the tab automatically — the menu
never shows controls that won't take effect.

### 4.8 Titles catalog

```lua
Config.Titles = {
    { id = 'newcomer', label = 'title_newcomer', color = '#FFFFFF' },
    { id = 'veteran',  label = 'title_veteran',  color = '#9D4EDD' },
    { id = 'elite',    label = 'title_elite',    color = '#00E5FF' },
    { id = 'prestige', label = 'title_prestige', color = '#FFEA00', glow = true },
    { id = 'legend',   label = 'title_legend',   color = '#FF8C00', glow = true },
    { id = 'staff',    label = 'title_staff',    color = '#D4AAFF', staffOnly = true, glow = true },
}
```

| Field      | Type   | Description                                                        |
| ---------- | ------ | ------------------------------------------------------------------ |
| `id`       | string | Stable identifier persisted to the DB.                             |
| `label`    | string | Locale key (resolved at runtime) or literal English string.        |
| `color`    | string | `#RRGGBB` rendered next to the player's name.                      |
| `glow`     | bool   | Optional pulsing text-shadow for prestige tiers.                   |
| `staffOnly`| bool   | Refuses `SetPlayerTitle` unless the target has a non-empty badge.  |

Six titles ship by default. Add as many rows as your progression system
needs — they're pure data and validated against `Config.Titles` only when
`SetPlayerTitle` is called.

### 4.9 Cosmetics catalog

```lua
Config.Cosmetics = {
    flair = {
        { id = 'star',    label = 'flair_star',    value = '⭐', rarity = 'common' },
        { id = 'fire',    label = 'flair_fire',    value = '🔥', rarity = 'common' },
        { id = 'crown',   label = 'flair_crown',   value = '👑', rarity = 'rare' },
        { id = 'diamond', label = 'flair_diamond', value = '💎', rarity = 'epic' },
        ...
    },
    frame = {
        { id = 'cyber', label = 'frame_cyber', value = 'cyber', rarity = 'rare' },
        ...
    },
    name_color  = { ... },
    name_effect = { ... },
}
```

Four categories. Each row has:

| Field    | Description                                                                  |
| -------- | ---------------------------------------------------------------------------- |
| `id`     | Unique within the category. Persisted in the DB.                             |
| `label`  | Locale key or literal string shown in the menu.                              |
| `value`  | Renderer payload: emoji for flair; palette id for frame; hex/`'rainbow'` for name_color; `'glow'`/`'pulse'` for name_effect. |
| `rarity` | UI tint only — `common`/`rare`/`epic`/`legendary`. No gameplay effect.        |

Edit, drop, or extend rows freely. The seed catalog is meant as a starting
point — most servers will swap the flair emojis for their own brand glyphs.

### 4.10 UI theme

```lua
Config.UI = {
    accent     = '#9D4EDD',
    accentSoft = 'rgba(157, 78, 221, 0.15)',
    danger     = '#FF2A6D',
    bg         = 'rgba(13, 11, 20, 0.92)',
    text       = '#FFFFFF',
    textDim    = 'rgba(255, 255, 255, 0.65)',
}
```

Edit the values, restart the resource, and the new theme takes effect on
the next menu open.

---

## 5. Commands

| Command       | Permission | Description                                       |
| ------------- | ---------- | ------------------------------------------------- |
| `/cosmetics`  | none       | Opens the cosmetic menu for the calling player.   |

Set `Config.Menu.command = nil` to remove the slash command. Set
`Config.Menu.enabled = false` to disable command + radial + the NUI
entirely.

---

## 6. Customization

### Theme

Edit `Config.UI` in `config.lua` and restart the resource — the new
theme takes effect on the next menu open. The cosmetic menu colors,
plate accents, and rarity tints are all driven from this block. The
in-world nameplate's static styling is fixed; everything that's
buyer-tweakable is exposed through `Config.UI` and `Config.Display`.

### Locales

Five locales ship in `locales/`: `en.json`, `tr.json`, `de.json`,
`fr.json`, `es.json`. To add a new language, copy `en.json`, translate the
44 keys, save as `locales/<code>.json`, and either set
`Config.Locale = '<code>'` or rely on the convar chain.

Title and cosmetic `label` values can be either a locale key (`title_veteran`,
`flair_crown`) or a literal string. `Locale.resolve` returns the value if it
exists, otherwise the key itself — so quick overrides are possible without
touching JSON.

### Server logo as badge

1. Drop a 64×64 PNG at `web/build/logo.png` (the path is in `escrow_ignore`,
   so buyers can swap it after install).
2. Set:
   ```lua
   Config.Display.badge = {
       mode   = 'static',
       static = 'web/build/logo.png',
   }
   ```
3. Restart the resource. Every player's nameplate now renders your logo
   as the left ornament.

### Pinning a server-wide title

```lua
Config.Display.title = {
    mode   = 'static',
    static = { label = 'MyRP — Season 3', color = '#FFEA00', glow = true },
}
```

Now no matter what `SetPlayerTitle` is called with, every plate renders
the static label. Useful during launch events or while a progression
system is being rebuilt.

### Adding a new cosmetic

Append a row to the relevant `Config.Cosmetics[<category>]` table:

```lua
Config.Cosmetics.flair[#Config.Cosmetics.flair + 1] = {
    id = 'rocket', label = 'flair_rocket', value = '🚀', rarity = 'epic',
}
```

Add a matching `flair_rocket` key to every locale file and you're done —
the menu picks it up on the next open.

---

## 7. Exports & Events

### Server exports

All return values are documented inline. Most return a `(ok, errKey)`
tuple where `errKey` matches a translation in `locales/*.json`.

| Export                                        | Args                                  | Returns                              | Purpose                                                                |
| --------------------------------------------- | ------------------------------------- | ------------------------------------ | ---------------------------------------------------------------------- |
| `SetPlayerLevel(src, n)`                       | `src`, `n: integer`                   | —                                    | Sets the level shown in the right-side orb.                            |
| `GetPlayerLevel(src)`                          | `src`                                 | `integer`                            | Reads the cached level.                                                 |
| `SetWantedStars(src, n)`                       | `src`, `n: 0..5`                      | —                                    | Sets the wanted-stars row (clamped 0..5).                               |
| `GetWantedStars(src)`                          | `src`                                 | `0..5`                               | Reads the cached wanted count.                                          |
| `SetBadge(src, badge)`                         | `src`, `badge: string?`               | —                                    | Sets the left badge. Pass `nil`/empty to hide.                          |
| `GetBadge(src)`                                | `src`                                 | `string?`                            | Reads the badge.                                                        |
| `SetPoliceMode(src, active)`                   | `src`, `active: bool`                 | —                                    | Tints the entire plate cyan when on.                                    |
| `GetPoliceMode(src)`                           | `src`                                 | `bool`                               | Reads the flag.                                                         |
| `SetPlayerTitle(src, titleId)`                 | `src`, `titleId: string`              | `(ok, errKey)`                       | Persists a `Config.Titles` id. Refuses unknown ids and `staffOnly` titles for non-staff. |
| `GetPlayerTitle(src)`                          | `src`                                 | `string`                             | Returns the active title id (or `'__custom__'` when a custom title is set). |
| `SetCustomTitle(src, label, color, glow?)`     | `src`, `label`, `#RRGGBB`, `bool?`    | `bool`                               | Bypasses the catalog with a one-off label/color.                        |
| `ClearCustomTitle(src)`                        | `src`                                 | —                                    | Reverts to the persisted catalog title.                                 |
| `GrantCosmetic(src, category, id)`             | `src`, `'flair'\|...`, cosmetic id   | `(ok, errKey)`                       | Persists ownership for the cosmetic.                                    |
| `RevokeCosmetic(src, category, id)`            | …                                     | `bool`                               | Removes ownership; auto-unequips if currently equipped.                 |
| `EquipCosmetic(src, category, id)`             | …                                     | `(ok, errKey)`                       | Sets the active cosmetic for that category. Requires ownership.         |
| `UnequipCosmetic(src, category)`               | …                                     | `bool`                               | Removes the equipped cosmetic for a category.                           |
| `GetEquippedCosmetics(src)`                    | `src`                                 | `{ flair?, frame?, name_color?, name_effect? }` | Returns the *resolved values* (emoji / palette id / hex / effect id) for each equipped category. Useful for other resources that want to mirror the rendering. |
| `GetOwnedCosmetics(src)`                       | `src`                                 | `{ category = { id, ... } }`         | All cosmetic ids the player owns.                                       |
| `BuildRenderData(src)`                         | `src`                                 | full render payload                  | Returns the same payload the DUI consumes — useful for HUDs/overlays that want to replicate the plate. |

### Client export

| Export                                        | Purpose                                                |
| --------------------------------------------- | ------------------------------------------------------ |
| `exports.clads_nameplate:OpenCosmeticMenu()`   | Opens the cosmetic menu programmatically.              |
| `exports.clads_nameplate:SetNameplatesEnabled(state)` | Master toggle for the local rendering loops. |

### Client net events

All under the `clads_nameplate:client:` namespace.

| Event                                          | Purpose                                                                |
| ---------------------------------------------- | ---------------------------------------------------------------------- |
| `clads_nameplate:client:setTyping`             | `(serverId, typing)` — call from your chat resource to hide a plate while the target is composing a message. |
| `clads_nameplate:client:refreshAll`            | Forces every visible plate to re-fetch its server data. Useful after a global event (season unlock, server-wide title grant). |

### Server callbacks

Internal — listed for completeness; you don't normally call these directly.

| Callback                                       | Purpose                                              |
| ---------------------------------------------- | ---------------------------------------------------- |
| `clads_nameplate:server:getPlayerData`         | Used by the DUI client to fetch render data per plate. |
| `clads_nameplate:server:getMenuPayload`        | Used by the cosmetic menu on open.                    |
| `clads_nameplate:server:equip`                 | Backend for the menu's equip button.                  |
| `clads_nameplate:server:unequip`               | Backend for the menu's unequip button.                |

---

## 8. Troubleshooting

### Nameplates don't appear

- Confirm `community_bridge`, `ox_lib`, `oxmysql`, and `clads_nameplate`
  all started without errors.
- Check that you and the test player are within `Config.Render.distance`
  (default `30.0` metres).
- Verify the `community_bridge:Client:OnPlayerLoaded` event fires for
  your framework — `client/events.lua` calls `SetNameplatesEnabled(true)`
  in response to it. Without it, the loops are dormant.
- Force-enable from console: `exports.clads_nameplate:SetNameplatesEnabled(true)`.

### Cosmetic menu opens but is empty

- The menu only shows categories where **both**
  `Config.Menu.categories[cat]` is true **and**
  `Config.Display.<cat>.mode` is `dynamic`. If you set the display mode
  to `'static'` or `'hidden'`, the matching category disappears from the menu.
- The grid renders cards for each row in `Config.Cosmetics[<category>]`.
  If the catalog is empty, the empty-state message shows. Add rows or
  switch categories.

### "You don't own this cosmetic yet" toast on every click

`EquipCosmetic` requires ownership. Either:

- Manually grant from console:
  `exports.clads_nameplate:GrantCosmetic(<src>, 'flair', 'crown')`
- Or wire your shop / battle-pass / unlock system to call `GrantCosmetic`
  when the player earns the cosmetic.

### Title doesn't change after `SetPlayerTitle`

- Check the title id matches a row in `Config.Titles`. Unknown ids return
  `(false, 'err_unknown_title')` and the cache is unchanged.
- If the title is marked `staffOnly = true`, it requires the target to
  have a non-empty badge. Pre-call `SetBadge(src, 'admin')` or remove
  the `staffOnly` flag.
- The server caches title ids per `src`. If the player relogged, their
  source id changed — make sure your script is using the current `src`.

### Server logo image doesn't load

- Confirm the file exists at the path you set
  (`web/build/logo.png` is the convention).
- The DUI URL is `nui://clads_nameplate/...`. Relative paths in
  `Config.Display.badge.static` are resolved against the resource root,
  so `'web/build/logo.png'` works; a leading slash is also supported.
- Confirm the file exists at the configured path. If you placed it
  somewhere outside the resource root, use a full URL instead of a
  relative path.

### Police-mode tint not appearing

- `Config.Display.policeMode.mode` must be `dynamic` (the default) or
  `static`. With `'hidden'`, the flag is dropped server-side regardless
  of `SetPoliceMode`.
- The runtime hook fires when another resource calls
  `exports.clads_nameplate:SetPoliceMode(src, true)` — there's no
  internal trigger.

### Plate appears for one second then vanishes

This is almost always the scan loop destroying the plate because the
target ped left `Config.Render.distance`. Check distance numbers; a
short fade range (e.g. `fadeStart = 28.0` with `distance = 30.0`)
exaggerates the effect. Widen the window or increase `distance`.

### Nameplates persist after a player disconnects

The scan loop reaps plates whose `playerId` is no longer in
`GetActivePlayers()`. If you see ghost plates, check if your client
is misreporting active players — restart the resource as a workaround.

---

## 9. Support

Reach out via the Creative Lads Discord.
