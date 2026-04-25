# Creative Lads — Crafting (`clads_crafting`)

A full-screen crafting system for FiveM. Players walk up to a workbench (static
or player-placed), open a NUI panel with category tabs, search, and a 3D item
preview, queue up recipes, watch a progress bar with a looping anim, and
receive the crafted output in their inventory. Bench items can be placed
anywhere in the world via a ghost-prop placement mode and persist across
restarts in MySQL.

The resource is framework-agnostic and is wired through `community_bridge`,
so it runs on QBCore, QBox, ESX, or in standalone mode without forks. Every
inventory, target, notification, and skill call is routed through the bridge —
swap inventories tomorrow and crafting keeps working.

---

## 1. Overview

`clads_crafting` ships four bench archetypes (workbench, chemistry table,
gunsmith bench, kitchen), a starter pack of nine recipes spanning tools,
medicine, food, drinks, and ammo, a placement / pickup loop that owns benches
to the placing player, and a category-driven React/Vite NUI with a 3D
preview pane.

**Who it's for**

- Roleplay servers that need a real crafting loop — placeable benches,
  profession gating, XP rewards — without writing the system from scratch.
- Survival / DayZ-style servers that want recipe categories tied to specific
  benches (medicine on chemistry tables, ammo on gunsmith benches, etc.).
- Any server that wants a single crafting UI driven entirely from a config
  file rather than database edits.

**Feature highlights**

- Four bench archetypes shipped with hardcoded static spawns + spawnable
  inventory items — drop the workbench in `Config.Benches` and the placement
  flow is wired automatically.
- Ghost-prop placement mode with raycast cursor, scroll-wheel rotation,
  ground snap, and outline color matched to the UI primary.
- Persistent placed benches in `clads_crafting_benches` (MySQL) — owners
  can pick their bench back up via the target prompt, admins can wipe a
  rogue bench by id with `/clearbench`.
- Recipe queue (configurable max 5) with progress-bar crafts, batch sizes
  up to a configurable cap (default 10), inventory-full / level / job
  guards, and an animation loop on the player ped.
- Profession gating + XP rewards via `community_bridge.Skills`
  (pickle_xp / evolent_skills / ot_skills auto-detected). Falls back
  silently when no skills backend is wired up.
- Server-side re-validation on every craft: ingredients, profession level,
  inventory space, and job allowlist are all re-checked even if the NUI
  payload is tampered with.
- Theme block that pushes every accent color into the NUI's CSS variables
  at runtime — no Vite rebuild required.
- JSON locales (en/tr/de/fr/es) with `clads_locale` → `ox:locale` →
  `qb_locale` → `lang` resolution and per-bench/recipe label overrides.
- Three exports (`OpenCrafting`, `CloseCrafting`, `IsUIOpen`) plus
  `GetNearestBench(benchType)` for other resources to pipe into.

---

## 2. Requirements

### Hard dependencies

Pulled directly from `fxmanifest.lua:49-53`:

```lua
dependencies {
    'community_bridge',
    'ox_lib',
    'oxmysql',
}
```

| Dependency         | Purpose                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `community_bridge` | Framework, inventory, target, notify, and skills abstraction.                            |
| `ox_lib`           | Callbacks, progress bar, notifications, text UI, ped streaming, and `lib.addCommand`.    |
| `oxmysql`          | Persistent storage for placed benches (`clads_crafting_benches` table, auto-created).    |

### Frameworks (auto-detected)

`Config.Framework = 'auto'` defers to `community_bridge`. The bridge probes
for `qbx_core`, `qb-core`, then `es_extended`; otherwise it returns standalone.
Override with `'qb'`, `'qbox'`, `'esx'`, or `'standalone'` if you need to pin
the choice (`config.lua:25`).

| Framework    | Identifier source                                       | Notes                                                                  |
| ------------ | ------------------------------------------------------- | ---------------------------------------------------------------------- |
| `qbox` / `qb` | `Framework.GetPlayerIdentifier`                         | Used as the `owner` column on every placed bench.                      |
| `esx`        | `Framework.GetPlayerIdentifier`                         | Same.                                                                  |
| `standalone` | First `license:` row from `GetPlayerIdentifiers(src)`   | Profession gates always pass; XP rewards become no-ops.                |

### Optional integrations (auto-routed by community_bridge)

- **Inventories:** `ox_inventory`, `qb-inventory`, `qs-inventory`,
  `codem-inventory`, `origen_inventory`, `core_inventory`, `ps-inventory`,
  `tgiann-inventory`, and any backend the bridge supports. The resource
  registers an `ox_inventory` `usingItem` hook directly when present, and
  falls back to a `clads_crafting:server:useBenchItem` net event for any
  other inventory wired through the bridge (`server/main.lua:165-182`).
- **Targets:** `ox_target`, `qb-target`, `sleepless_interact` — auto-routed
  via `Bridge.Target.AddLocalEntity` on every spawned bench.
- **Skills / XP:** `pickle_xp`, `evolent_skills`, `ot_skills`. If no backend
  is wired, both the level gate and the XP award degrade silently
  (`server/main.lua:111-124`).
- **Notifications:** `ox_lib`, `okok`, `ps-ui`, `lation_ui`, `qb`, `esx`.
  Falls back to `lib.notify` if the bridge has no notify provider.
- **Locales:** ships with `en`, `tr`, `de`, `fr`, `es` JSON files; English
  is the source of truth and the fallback for missing keys.

---

## 3. Installation

1. **Download** the resource and copy the `clads_crafting/` folder into your
   server's `resources/` directory (commonly
   `resources/[scripts]/clads_crafting`).
2. **Install dependencies** so they start before this resource:
   - `community_bridge`
   - `ox_lib`
   - `oxmysql`
3. **Add the bench items to your inventory.** Each bench in `Config.Benches`
   that has an `item` field requires a matching inventory item so players
   can place it. The default config ships:

   | Item key                    | Bench it places   |
   | --------------------------- | ----------------- |
   | `crafting_workbench`        | `workbench`       |
   | `crafting_chemistry_table`  | `chemistry_table` |
   | `crafting_gunsmith_bench`   | `gunsmith_bench`  |
   | `crafting_kitchen`          | `kitchen`         |

   Add these to your inventory's items list (e.g. `ox_inventory/data/items.lua`)
   with whatever label and weight you prefer. If you skip an item, that bench
   becomes static-only — place it via `locations` in `Config.Benches`.

4. **Start the resource.** Add to `server.cfg` after dependencies:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure oxmysql
   ensure clads_crafting
   ```
5. **Test.**
   - Drive or walk to one of the static workbench spawns — by default
     `vec4(870.40, -2312.70, 30.57, 88.15)` (LSC tools area) or
     `vec4(156.31, 3130.07, 43.58, 190.46)` (Sandy Shores).
   - The target should show a `Use Workbench` prompt.
   - Open it and queue a `Lockpick` craft (4 metalscrap + 4 plastic) to
     verify the bridge inventory hooks resolve.
   - Give yourself a `crafting_workbench` item, use it, place it somewhere,
     then re-log to confirm persistence (the bench should respawn at the
     same coords).

> The SQL table is created automatically on first start
> (`server/main.lua:33-48`). `sql/install.sql` is provided as a manual
> reference only — buyers who prefer to inspect the schema before booting
> the resource can run it by hand.

---

## 4. Configuration

`config.lua` is split into seven commented sections. Headings below mirror
the order in the file.

### 4.1 Framework

Defined in `config.lua:25`.

```lua
Config.Framework = 'auto'   -- 'auto' | 'esx' | 'qb' | 'qbox' | 'standalone'
```

`auto` is recommended. Pin it only if you run multiple cores side by side or
want to force standalone for testing. With `'standalone'` every recipe is
free of profession checks and XP awards become no-ops
(`server/main.lua:111-124`).

### 4.2 Locale

Defined in `config.lua:34`.

```lua
Config.Locale = 'auto'   -- 'auto' | 'en' | 'tr' | 'de' | 'fr' | 'es'
```

`auto` resolves through convars in this order: `clads_locale` → `ox:locale` →
`qb_locale` → `lang` → `en` (`shared/locale.lua`). Drop additional files into
`locales/<code>.json` to add new languages — English remains the hard fallback
for any missing key.

### 4.3 Debug

Defined in `config.lua:39`.

```lua
Config.Debug = false
```

When `true`, `[clads_crafting]` lines print to the server console on every
craft and the client `/clads_crafting:open <benchType>` debug command becomes
usable (`client/main.lua:338-346`).

### 4.4 UI Theme

Defined in `config.lua:49-65`. Every value is a 6-digit hex color with a
leading `#`. The web bundle reads `Config.UI` on menu open and injects the
values into `:root` CSS variables — restyles take effect on resource restart,
no Vite rebuild required.

```lua
Config.UI = {
    primary       = '#9D4EDD',
    primaryLight  = '#C77DFF',
    primaryDark   = '#7B2CBF',
    action        = '#FFEA00',
    actionDark    = '#CCBB00',
    secondary     = '#00E5FF',
    success       = '#00E676',
    warning       = '#F59E0B',
    danger        = '#FF2A6D',
    bgCore        = '#0D0B14',
    bgSurface     = '#1F1B29',
    bgElevated    = '#2A2438',
    border        = '#3A3050',
    textPrimary   = '#FFFFFF',
    textMuted     = '#B5B5B5',
}
```

| Key             | Used for                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `primary*`      | Selected card border, active tab, progress fill, brand glow                 |
| `action*`       | Confirm CTA accent (Craft button)                                           |
| `secondary`     | Info / secondary accents                                                    |
| `success`       | "Have enough" pill, success notifications                                   |
| `warning`       | Locked / "level required" badge                                             |
| `danger`        | Missing-materials labels and error states                                   |
| `bg*`           | Layered backgrounds (core / surface / elevated)                             |
| `border`        | Card outlines                                                               |
| `textPrimary`   | Default text colour                                                         |
| `textMuted`     | Subtitle / muted labels                                                     |

> The default placement-outline colour `r=157, g=78, b=221, a=255` matches
> `Config.UI.primary`. If you change the primary, also update
> `Config.PlacementOutline` (`config.lua:72`) so the ghost prop stays on
> brand.

### 4.5 Settings

Defined in `config.lua:77-132`.

```lua
Config.Settings = {
    interactionDistance = 2.0,
    maxQueueSize        = 5,
    xpMultiplier        = 1.0,
    durationMultiplier  = 1.0,
    showLockedRecipes   = true,
    allowBatchCrafting  = true,
    maxBatchSize        = 10,
    sounds              = { ... },
    animation           = { dict = 'mini@repair', anim = 'fixing_a_ped' },
    blips               = { enabled = false, sprite = 566, color = 27, scale = 0.7, shortRange = true },
    categoryIcons       = { tools = 'wrench', misc = 'box', food = 'utensils', ... },
}
```

| Key                    | Type    | Default                                  | Description                                                                            |
| ---------------------- | ------- | ---------------------------------------- | -------------------------------------------------------------------------------------- |
| `interactionDistance`  | number  | `2.0`                                    | Distance (m) at which the target prompt appears on a bench.                            |
| `maxQueueSize`         | number  | `5`                                      | Maximum recipes that can be queued at once. The 6th queue add returns `error_queue_full`. |
| `xpMultiplier`         | number  | `1.0`                                    | Global multiplier on every recipe's XP reward (`server/main.lua:385`).                  |
| `durationMultiplier`   | number  | `1.0`                                    | Global multiplier on every recipe duration. Lower = faster crafts.                      |
| `showLockedRecipes`    | bool    | `true`                                   | Show recipes the player can't yet craft, greyed out. Set `false` to hide them.          |
| `allowBatchCrafting`   | bool    | `true`                                   | Show the +/- batch selector in the UI. Set `false` to lock every craft to count = 1.    |
| `maxBatchSize`         | number  | `10`                                     | Caps the +/- selector and the server-side count clamp (`server/main.lua:351`).          |
| `sounds.craftStart`    | table   | `Hack_Success` (`DLC_HEIST_BIOLAB...`)   | Played on menu open.                                                                   |
| `sounds.craftComplete` | table   | `PICK_UP` (`HUD_FRONTEND_DEFAULT...`)    | Played on successful craft.                                                            |
| `sounds.craftFail`     | table   | `ERROR` (`HUD_FRONTEND_DEFAULT...`)      | Played on failed craft.                                                                |
| `animation.dict/anim`  | strings | `mini@repair` / `fixing_a_ped`           | Looped on the player ped while a recipe is processing.                                 |
| `blips.enabled`        | bool    | `false`                                  | Set `true` to draw a blip on every static bench in `Config.Benches[*].locations`.      |
| `categoryIcons`        | table   | `wrench / box / utensils / coffee / ...` | Glyph keys matched by name in the React UI's category tabs.                            |

### 4.6 Workbenches

Defined in `config.lua:149-192` (`Config.Benches`). Each entry is a flat table:

```lua
workbench = {
    label      = 'bench_workbench',
    model      = 'prop_tool_bench02',
    item       = 'crafting_workbench',
    categories = { 'tools', 'misc' },
    profession = 'crafting',
    minLevel   = 0,
    locations  = {
        vec4(870.40, -2312.70, 30.57,  88.15),
        vec4(156.31, 3130.07,  43.58, 190.46),
    },
},
```

| Field        | Type     | Description                                                                                                                                                                |
| ------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `label`      | string   | Translation key (e.g. `bench_workbench`) or a literal string. Resolved via `Locale.resolve`.                                                                               |
| `model`      | string   | GTA prop name spawned for the bench entity and the placement ghost.                                                                                                        |
| `item`       | string?  | Inventory item that lets a player place this bench. Omit to make the bench **static-only** — it will only spawn at the coords listed in `locations`.                       |
| `categories` | string[] | Recipe categories shown when this bench is opened. Recipes whose `category` is not in this list are filtered out (`server/main.lua:281-283`).                              |
| `profession` | string?  | Skill key checked against the player's level via `Bridge.Skills.GetCurrentLevel`. Set `nil` to skip the gate.                                                              |
| `minLevel`   | number   | Minimum skill level surfaced in `getBenchInfo` (used by the UI header). Recipe-level gating is enforced separately by each recipe's `requiredLevel`.                       |
| `groups`     | string[]?| Optional ox_target groups (job/gang) that gate the bench's target prompt — passed straight into `Target.AddLocalEntity` (`client/main.lua:50`).                            |
| `locations`  | vec4[]   | Hardcoded spawn coords for static benches. Pass an empty list `{}` if the bench should only exist as a player-placed entity.                                               |

Four benches ship by default:

| Key               | Item                          | Categories            | Profession   | Static spawns                                                                          |
| ----------------- | ----------------------------- | --------------------- | ------------ | -------------------------------------------------------------------------------------- |
| `workbench`       | `crafting_workbench`          | `tools`, `misc`       | `crafting`   | `(870.40, -2312.70, 30.57, 88.15)`, `(156.31, 3130.07, 43.58, 190.46)`                 |
| `chemistry_table` | `crafting_chemistry_table`    | `medicine`            | `chemist`    | None — placement-only                                                                  |
| `gunsmith_bench`  | `crafting_gunsmith_bench`     | `weapons`, `ammo`     | `gunsmith`   | None — placement-only                                                                  |
| `kitchen`         | `crafting_kitchen`            | `food`, `drinks`      | `cooking`    | None — placement-only                                                                  |

### 4.7 Recipes

Defined in `config.lua:215-357` (`Config.Recipes`). Each entry is a flat table:

```lua
lockpick = {
    label         = 'recipe_lockpick',
    category      = 'tools',
    duration      = 5000,
    output        = { item = 'lockpick', count = 1 },
    ingredients   = {
        { item = 'metalscrap', count = 4 },
        { item = 'plastic',    count = 4 },
    },
    profession    = 'crafting',
    xp            = 15,
    requiredLevel = 0,
    model         = 'prop_tool_screwdvr01',
},
```

**Required fields**

| Field         | Type            | Description                                                                                |
| ------------- | --------------- | ------------------------------------------------------------------------------------------ |
| `label`       | string          | Translation key OR literal string shown in the UI.                                         |
| `category`    | string          | Must match one of the categories listed on a bench (otherwise the recipe never appears).    |
| `duration`    | number          | Milliseconds the progress bar spins for. Multiplied by `Config.Settings.durationMultiplier`. |
| `output`      | table           | `{ item = '<inv item>', count = <integer> }`. The crafted item awarded to the player.       |
| `ingredients` | list of tables  | `{ { item = '...', count = ... }, ... }`. Re-validated server-side on every craft.          |

**Optional fields**

| Field           | Type      | Default | Description                                                                                                       |
| --------------- | --------- | ------- | ----------------------------------------------------------------------------------------------------------------- |
| `profession`    | string?   | `nil`   | Skill key the recipe gates on. `nil` skips the gate.                                                              |
| `xp`            | number    | `0`     | XP awarded on success. Multiplied by `Config.Settings.xpMultiplier × count`.                                      |
| `requiredLevel` | number    | `0`     | Minimum skill level required. The level gate is skipped when no skills backend is wired up.                       |
| `model`         | string?   | `nil`   | GTA prop name shown in the 3D preview pane when the recipe is selected.                                           |
| `requiredJob`   | string[]? | `nil`   | List of job/gang names allowed to craft (e.g. `{ 'mechanic' }`). Bridge resolves the player's job server-side.    |

Nine recipes ship as a starter pack:

| Recipe        | Bench               | Category   | Duration | Output         | Profession  | Lvl |
| ------------- | ------------------- | ---------- | -------- | -------------- | ----------- | --- |
| `lockpick`    | `workbench`         | `tools`    | 5 s      | `1× lockpick`  | `crafting`  | 0   |
| `repairkit`   | `workbench`         | `tools`    | 8 s      | `1× repairkit` | `crafting`  | 4   |
| `radio`       | `workbench`         | `misc`     | 12 s     | `1× radio`     | `crafting`  | 8   |
| `bandage`     | `chemistry_table`   | `medicine` | 3 s      | `1× bandage`   | `chemist`   | 0   |
| `firstaid`    | `chemistry_table`   | `medicine` | 8 s      | `1× firstaid`  | `chemist`   | 5   |
| `sandwich`    | `kitchen`           | `food`     | 4 s      | `1× sandwich`  | `cooking`   | 0   |
| `coffee`      | `kitchen`           | `drinks`   | 3 s      | `1× coffee`    | `cooking`   | 0   |
| `pistol_ammo` | `gunsmith_bench`    | `ammo`     | 6 s      | `12× pistol_ammo` | `gunsmith` | 3   |
| `rifle_ammo`  | `gunsmith_bench`    | `ammo`     | 10 s     | `12× rifle_ammo`  | `gunsmith` | 8   |

These are intentionally sparse — they cover every category and bench so the
resource feels alive on first boot, but you should replace them with your
server's own economy once the table is finalised.

---

## 5. Commands

| Command          | Source file               | Permission           | Description                                                                                            |
| ---------------- | ------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------ |
| `/clearbench <id>` | `server/main.lua:426-439` | `group.admin` (Ace)  | Force-removes a placed bench by its database id (`clads_crafting_benches.id`). Use when a bench is stuck inside geometry or a player is offline and needs cleanup. |

The command is registered through `lib.addCommand` so it inherits ox_lib's
chat suggestions and parameter help (`cmd_clearbench_help`,
`cmd_clearbench_arg`). Owners of a placed bench can pick it up via the target
prompt without admin help — the `/clearbench` command exists for the rare
case where ownership is broken.

The `/clads_crafting:open <benchType>` debug command in
`client/main.lua:338-346` is gated by `Config.Debug` and is only useful for
testing UI changes without standing next to a bench.

---

## 6. Customization

### Theme

Edit `Config.UI` in `config.lua:49-65`. Values are pushed into the NUI's
`:root` CSS variables on menu open via `buildUiTheme()` in
`client/utils.lua:71-73`, so changing a colour just needs a resource restart
— no Vite rebuild. To match the placement-mode outline to your primary, also
update `Config.PlacementOutline` (`config.lua:72`).

### Locales

Five locales ship in `locales/`: `en.json`, `tr.json`, `de.json`, `fr.json`,
`es.json`. To add a new language, copy `en.json`, translate the values, save
as `locales/<code>.json`, and either set `Config.Locale = '<code>'` or rely
on the convar chain (`clads_locale`, `ox:locale`, `qb_locale`, `lang`).

Both bench and recipe `label` fields can be either a locale key (e.g.
`recipe_lockpick`) or a literal string — `Locale.resolve` (`shared/locale.lua`)
returns the value if it exists, otherwise the key itself. So you can quickly
override one label with `label = 'My Custom Lockpick'` without touching JSON.

### Adding a new bench

Append a new key to `Config.Benches`:

```lua
Config.Benches.electronics = {
    label      = 'Electronics Bench',
    model      = 'prop_tool_bench02',
    item       = 'crafting_electronics_bench',
    categories = { 'misc' },
    profession = 'crafting',
    minLevel   = 5,
    locations  = {},
}
```

Add the matching `crafting_electronics_bench` item to your inventory's items
list, then start the resource. The `usingItem` hook auto-registers the new
item on resource start (`server/main.lua:23-28`).

If you want the bench to also exist as a static spawn, add `vec4` entries
to `locations` — they spawn on `initBenches` and on every
`community_bridge:Client:OnPlayerLoaded` (`client/events.lua:20-22`).

### Adding a new recipe

Append a new key to `Config.Recipes`. The minimal viable recipe is:

```lua
Config.Recipes.duct_tape = {
    label       = 'Duct Tape',
    category    = 'misc',                        -- must match a bench category
    duration    = 4000,
    output      = { item = 'duct_tape', count = 1 },
    ingredients = {
        { item = 'plastic', count = 2 },
        { item = 'cloth',   count = 1 },
    },
}
```

Add the locale string for the label if you used a key. The recipe shows up
on every bench whose `categories` list contains `misc`. Add `profession`,
`requiredLevel`, and `xp` to gate it; add `requiredJob = { 'mechanic' }` to
restrict it to a job; add `model = 'prop_meridian_box'` to render a 3D
preview when the recipe card is selected.

### Adding a new category

Three steps:

1. Add the category key to one or more benches in `Config.Benches[*].categories`.
2. Add a glyph entry in `Config.Settings.categoryIcons` so the UI tab gets
   an icon. Supported glyph keys come from the React component's icon map —
   pick a Lucide / FA icon name; falls back to `box` if unknown.
3. Add a translation in every locale file under `cat_<key>` (e.g. `cat_explosives`).
4. Add a recipe with `category = '<key>'`.

### Replacing the placement outline

`Config.PlacementOutline` (`config.lua:72`) is an RGBA table consumed by
`SetEntityDrawOutlineColor` in `client/placement.lua:54-55`. Set
`a = 0` to disable the outline entirely; the ghost prop's transparent
alpha (`SetEntityAlpha(.., 150)`) is what makes it readable as a placement
preview, so leave that alone.

### Asset paths

The web bundle lives in `web/dist/`. The logo at `web/dist/logo.png` is one
of four files in `escrow_ignore` (`fxmanifest.lua:57-62`), so you can swap
it for your own logo without touching the encrypted JS. The other escrow-
ignored files are `config.lua`, `locales/**.json`, and `sql/install.sql`.

---

## 7. Exports & Events

### Client exports

Defined in `client/main.lua:352-378` and declared in `fxmanifest.lua`.

| Export                                  | Purpose                                                                                                                                          |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `exports.clads_crafting:OpenCrafting(benchType)` | Opens the NUI panel for the given bench key. Useful for binding to your own keybind, target system, or dialog.                          |
| `exports.clads_crafting:CloseCrafting()` | Closes the NUI and cancels any in-flight progress bar. Safe to call when the UI is already closed.                                              |
| `exports.clads_crafting:IsUIOpen()`     | Returns `true` while the crafting panel is open. Use to gate other UI from opening on top of it.                                                  |
| `exports.clads_crafting:GetNearestBench(benchType)` | Returns `(coords, distance)` of the nearest bench of the given type — searches both `Config.Benches[*].locations` and live placed benches. |

### Notable client net events

All under the `clads_crafting:client:` namespace.

| Event                                    | Purpose                                                                                                  |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `clads_crafting:client:startPlacement`   | Server tells the client the bench item was used; client switches into placement mode.                     |
| `clads_crafting:client:benchPlaced`      | Broadcast to everyone after a successful place — every client spawns the bench prop locally.              |
| `clads_crafting:client:benchRemoved`     | Broadcast on pickup or `/clearbench` — every client deletes the matching prop locally.                    |

### Notable server net events / callbacks

| Name                                        | Type     | Purpose                                                                                                  |
| ------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------- |
| `clads_crafting:server:useBenchItem`        | Event    | Fallback path for non-ox inventories — fired when the player uses a bench item.                          |
| `clads_crafting:server:placeBench`          | Callback | Validates the pending placement, removes the inventory item, inserts the row into MySQL.                  |
| `clads_crafting:server:cancelPlacement`     | Callback | Drops the pending placement state without consuming the item.                                            |
| `clads_crafting:server:pickupBench`         | Callback | Owner-only — deletes the bench row and gives the item back to the player.                                |
| `clads_crafting:server:getPlacedBenches`    | Callback | Returns every persisted bench. Called on `initBenches` and on every player load.                         |
| `clads_crafting:server:getRecipes`          | Callback | Returns the filtered recipe list for a bench, including per-recipe `canCraft` / `hasIngredients` flags.  |
| `clads_crafting:server:getBenchInfo`        | Callback | Returns `{ label, profession, playerLevel, minLevel }` for the UI header.                                |
| `clads_crafting:server:craftItem`           | Callback | Re-validates everything, charges ingredients, awards output and XP, returns `(ok, message)`.             |
| `clads_crafting:server:itemCrafted`         | Event    | Fired server-side after a successful craft — `(source, item, totalCount, recipeId)`. Use this from your own resources to log analytics, run economy hooks, etc. |

---

## 8. Troubleshooting

### Target prompt won't appear on a bench

- Confirm `community_bridge`, `ox_lib`, and `oxmysql` all started without
  errors in the server console.
- Check that the bench's `model` is a valid GTA prop name (the resource
  silently bails on `CreateObject` failure, see `client/main.lua:32-36`).
- For static benches, verify the `locations` `vec4` actually puts the prop
  somewhere reachable — `PlaceObjectOnGroundProperly` shifts it down to the
  ground but not across geometry.
- For player-placed benches, re-log to trigger the
  `getPlacedBenches` callback (also fired by
  `community_bridge:Client:OnPlayerLoaded`).

### Bench item doesn't trigger placement

- `ox_inventory`: the resource registers a `usingItem` hook on resource
  start (`server/main.lua:165-177`). If you added the bench item to
  `Config.Benches` *after* the resource was already running, restart the
  resource so the hook re-registers with the new filter.
- Other inventories: the bridge needs to fire
  `clads_crafting:server:useBenchItem` from your inventory's "use item"
  pipeline. Check the bridge release notes for your specific inventory —
  most ship a built-in usable-item hook.
- Verify the item exists in your inventory's items list with the exact key
  used in `Config.Benches[*].item`.

### Recipe shows in the UI but `Craft` is greyed out

The card is greyed when **any** of these are true (server enforces them all):

- Player level < `recipe.requiredLevel` and a skills backend is wired up.
- Player's job is not in `recipe.requiredJob` (when `requiredJob` is set).
- Player's inventory is missing one or more ingredients in the required
  count × batch size.

The first two return `error_level_required` / `error_no_permission` from
`craftItem`; the third returns `error_missing_ingredients`. Open the
recipe card — the per-ingredient `Have: X / Required: Y` lines show
exactly what's short.

### "Failed to remove materials" on craft

The server tried to consume an ingredient and the inventory backend
rejected the call (`server/main.lua:371-375`). This usually means another
resource removed the item between the recipe list refresh and the craft —
the queue is short-lived enough that it's rare, but it can happen if you
have an auto-loot or rapid trade resource. Re-open the menu to refresh
the recipe list and retry.

### Recipe XP doesn't award

`awardXp` (`server/main.lua:120-124`) is a no-op when the bridge has no
`Skills.AddXp` hook. Confirm:

- A skills resource is started before `clads_crafting` (`pickle_xp`,
  `evolent_skills`, or `ot_skills`).
- The bridge release notes for your skills backend show it as supported.
- The recipe has a non-zero `xp` value — `xp = 0` is treated as "no XP".

The level *gate* shares the same fallback rule: when no skills backend
exists, the gate passes silently. If your players can craft above their
"level", that's why.

### Placed bench survives `/clearbench`

The command deletes the row and broadcasts a removal event, but the local
prop is despawned via `despawnBench('db_' .. id)` (`client/main.lua:319-321`).
If the prop persists, the client missed the event — usually because the
player was streamed out at the time. The bench will despawn on next
`initBenches` (resource restart or player reload). To force it now, run
`/clearbench <id>` again on every shard.

### Placed bench falls through the world

`PlaceObjectOnGroundProperly` uses a downward raycast. On unmapped IPL
interiors or custom MLOs, the raycast can miss the floor. Workarounds:

- Pick a `Config.Benches[*].model` whose pivot is at the base (e.g.
  `prop_tool_bench02`) — top-pivoted props amplify the issue.
- Disable the ground snap by replacing the
  `PlaceObjectOnGroundProperly(placementObj)` call in
  `client/placement.lua:75` with a no-op for the affected MLO and place
  on a flat surface manually.

### UI opens but is empty (no recipes)

- The bench's `categories` list may not match any recipe's `category`.
  Check that at least one recipe in `Config.Recipes` has a matching
  category for the bench you opened.
- `Config.Settings.showLockedRecipes = false` hides every recipe the
  player can't currently craft. Set it back to `true` to confirm whether
  recipes exist but are filtered out.
- Inspect the server console with `Config.Debug = true` and re-open the
  menu — the `getRecipes` callback runs on every open and any error in
  the loop shows as a Lua trace.

---

## 9. Support

Reach out via the Creative Lads Discord.
