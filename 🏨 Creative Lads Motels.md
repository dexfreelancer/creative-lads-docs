# clads_motels

A framework-agnostic motel rental and ownership system for FiveM. Players rent rooms from a manager NPC, get teleported into a per-tenant persistent shell (or an MLO interior), control who can come in via a public/doorbell/private access mode, and run the whole motel as a business if they buy it.

---

## Overview

`clads_motels` covers the full hotel-room loop on a roleplay server, from short-term rentals to long-term business management. It is built around a small set of layered systems:

- **Manager NPC + tablet UI** — Each motel spawns a clerk at a configurable coord. Targeting the clerk opens a React + Tailwind tablet UI: a room grid for renters, a stat panel for buyers, and a five-tab dashboard for owners and staff.
- **TP shells *or* MLO motels** — Set `Mlo = false` per motel to teleport renters into a streamed shell (one per citizenid, persisted to a far-away grid so contents survive disconnects). Set `Mlo = true` to keep players in the world and use real GTA interior doors.
- **Per-tenant shell instances** — Every tenant who has ever rented a room gets a permanent grid slot in `clads_motels_instances`. Re-renting any room reuses the same slot, so stash items and the room layout never reset.
- **Three access modes** — Each room renter sets `public` (anyone walks in), `doorbell` (visitors ring, owner approves once or grants permanent access), or `private` (closed). The Room Management panel inside the shell exposes the queue.
- **Police breach** — Players in `Config.BreachJobs` can shoot a locked room door; the resource raycasts the bullet path against every door in range and forces the matching room back to public access.
- **Lockpick path** — A `lockpick` item gates a `lib.skillCheck` minigame for MLO doors. TP shells use the access-mode flow instead.
- **Owner business** — When `Config.Business = true`, a player can buy an unowned motel for `purchasePrice`, set the rate, hire staff (employees see the rental dashboard too), withdraw revenue, send invoices, transfer ownership, or sell the motel back at half price.
- **Invoicing** — Owners and employees can send a one-line invoice to any online player. The target sees a popup with cash/bank toggle and a 60-second window to pay; revenue lands in the motel account.
- **Persistent rent expiry** — A 60-second server loop checks every room's `duration`. Expired tenants are auto-evicted (configurable via `Config.AutoKickOnExpire`) and their room data is cleared.
- **Crash recovery** — The client persists a small KVP record while inside a shell. On crash/respawn the client re-enters the same shell at the same offset.
- **Five languages out of the box** — `en`, `tr`, `de`, `fr`, `es`. The locale dictionary is sent to the NUI on every open so the React UI re-renders in the active language without a rebuild.
- **Configurable theme** — `Config.UI` ships six ready-to-copy color presets (Purple Haze, Dark Red, Cyber Cyan, Forest Green, Sunset Orange, Mono Slate) with guidance for which suits TP shells vs. MLO motels. Any 6-digit hex value is valid; CSS vars are applied at runtime.
- **Framework-agnostic** — Every framework / inventory / notify / target / clothing call goes through `community_bridge`. The same install runs on Standalone, ESX, QBCore, and QBox without code changes.

A small `bridge/` folder ships outside the escrow encryption so server owners can extend the wardrobe adapter (e.g. add their own clothing resource) and tweak persistence helpers without forking the locked code.

---

## Requirements

| Dependency | Required | Purpose |
| --- | --- | --- |
| `community_bridge` | Yes | Framework, target, notify, inventory, and clothing abstraction. |
| `ox_lib` | Yes | Callbacks, notifications, progress bar, skill-check minigame, and `lib.zones.sphere`. |
| `oxmysql` | Yes | Persistence for motel state (`clads_motels_motels`) and shell instances (`clads_motels_instances`). |
| Framework | Optional (auto-detected) | `es_extended`, `qb-core`, or `qbx_core`. Falls back to standalone (free rentals, no money checks) when none is detected. |
| Target backend | Optional (auto-detected) | `ox_target`, `qb-target`, or `sleepless_interact`. Required for the manager NPC and door zones. |
| Inventory backend | Optional (auto-detected) | `ox_inventory`, `qb-inventory`, `qs-inventory`, `origen_inventory`, `tgiann-inventory`. Required for room stashes and the motel-key item. |
| Clothing backend | Optional (auto-detected) | `illenium-appearance`, `qb-clothing`, `fivem-appearance`, `esx_skin`. The wardrobe target inside the shell is hidden when none is installed (or `Config.WardrobeBackend = 'standalone'`). |

If no inventory backend is installed, the rental flow still works — the stash and the motel-key item just won't function. Server owners are expected to be on at least one of the supported inventory resources.

---

## Installation

1. Drop the `clads_motels` folder into your `resources/` directory (commonly under `[clads]/`).
2. Install the required dependencies if you do not have them already:
   - `community_bridge`
   - `ox_lib`
   - `oxmysql`
3. Make sure your chosen target and inventory resources start **before** `clads_motels`.
4. Add the resource to your `server.cfg`:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure oxmysql
   ensure ox_target          # or qb-target / sleepless_interact
   ensure ox_inventory       # or qb-inventory / qs-inventory / etc.
   ensure clads_motels
   ```
5. The `clads_motels_motels` and `clads_motels_instances` tables are auto-created on first launch. If you prefer to install the schema manually, run `sql/install.sql` against your database first.
6. Open `clads_motels/config.lua` and adjust the motel definitions, prices, theme, and payment accounts to match your server.
7. (Optional) Translate `locales/en.json` into another language by copying it to `locales/<code>.json` and changing `Config.Locale` to that code, or leave `Config.Locale = 'auto'` to follow your server's existing locale convars.
8. Restart the server.

The manager NPC, blip, and motel-zone target sphere spawn on resource start and re-attach when the player loads in via `community_bridge:Client:OnPlayerLoaded`.

---

## Configuration

All configuration lives in `clads_motels/config.lua`, which remains editable after Tebex/CFX escrow encryption (see `escrow_ignore` in `fxmanifest.lua`). The bridge adapters under `bridge/**.lua` are also escrow-ignored so server owners can extend them without forking the locked code.

### Framework

```lua
Config.Framework = 'auto'
```

| Value | Behavior |
| --- | --- |
| `'auto'` | Let `community_bridge` pick the active framework (recommended). |
| `'esx'` | Force `es_extended`. |
| `'qb'` | Force `qb-core`. |
| `'qbox'` | Force `qbx_core`. |
| `'standalone'` | No framework. Every rental is free, no money checks are performed. |

### Locale

```lua
Config.Locale = 'auto'
```

Built-in locales: `en`, `tr`, `de`, `fr`, `es`. Add your own by dropping a `locales/<code>.json` next to the existing files — keys must mirror `en.json` exactly.

When `Config.Locale = 'auto'`, the loader picks the first non-empty value from the following convars and falls back to `en`:

1. `clads_locale`
2. `ox:locale`
3. `qb_locale`
4. `lang`

### Debug

```lua
Config.Debug = false
```

Enables verbose `[clads_motels]` console prints during boot, rent, and shell allocation. Leave at `false` for production.

### UI Theme

The React panels' color scheme is injected into `:root` at runtime — every `SendNUIMessage` carries the active `Config.UI` table, which `applyTheme` writes to CSS variables. **No rebuild needed:** edit hex values, restart the resource, the new theme takes effect on the next open.

```lua
Config.UI = {
    primary       = '#9D4EDD',
    primaryLight  = '#B76EF5',
    primaryDark   = '#7B2CBF',
    action        = '#FFEA00',
    secondary     = '#00E5FF',
    success       = '#00E676',
    warning       = '#FF9100',
    danger        = '#FF2A6D',
    bgCore        = '#0D0B14',
    bgSurface     = '#1F1B29',
    bgElevated    = '#2A2635',
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `primary` | `#9D4EDD` | Main brand color (selected card border, focus outline, accents). |
| `primaryLight` | `#B76EF5` | Hover shade. |
| `primaryDark` | `#7B2CBF` | Gradient start. |
| `action` | `#FFEA00` | Main CTA button (Rent / Buy / Pay). High-attention color. |
| `secondary` | `#00E5FF` | Stat numbers, info pills. |
| `success` | `#00E676` | Success notifications, "Public access" badge. |
| `warning` | `#FF9100` | Breach panel border, "Doorbell" badge. |
| `danger` | `#FF2A6D` | Error states, close-button hover, "Private" badge. |
| `bgCore` | `#0D0B14` | Darkest background layer (overlay backdrop). |
| `bgSurface` | `#1F1B29` | Panel background. |
| `bgElevated` | `#2A2635` | Card and list-item background. |

`config.lua` ships **six ready-to-copy presets** in a comment block above the active table:

| Preset | Mood | Best for |
| --- | --- | --- |
| **Purple Haze** *(default)* | Modern, neon, cyberpunk | TP shell motels (Vespucci shell, etc.) |
| **Dark Red** | Warm, luxury hotel | MLO motels (Pillbox, Vinewood, Eclipse Tower) |
| **Cyber Cyan** | Neon blue, nightclub | Beach / Vespucci TP shells |
| **Forest Green** | Calm, rural roadside | Paleto / Sandy Shores MLO motels |
| **Sunset Orange** | Warm retro neon-sign | Desert / Sandy Shores TP shells |
| **Mono Slate** | Colorless, corporate | Premium MLO motels (Eclipse Tower, Richman) |

To switch presets, copy the desired block out of the comment, paste it over the active `Config.UI = { ... }` at the bottom of the section, and `restart clads_motels`.

### Feature Toggles

```lua
Config.Business         = true
Config.AutoKickOnExpire = true
Config.PaymentAccounts  = { 'bank', 'cash' }
Config.CurrencySymbol   = '$'
```

| Key | Default | Purpose |
| --- | --- | --- |
| `Business` | `true` | Allow players to purchase motels and earn rent income. When `false`, motels are owned by the server and revenue is voided. |
| `AutoKickOnExpire` | `true` | Auto-evict tenants the moment their `duration` ticks below the current time. When `false`, the room flags as expired but the tenant can stay until they leave manually (rent will not auto-renew). |
| `PaymentAccounts` | `{ 'bank', 'cash' }` | Order in which accounts are checked for rent / purchase. The first account with enough balance is charged. `community_bridge` maps `'money'` to `'cash'` on QB/QBox automatically. |
| `CurrencySymbol` | `'$'` | Display-only currency prefix shown in the UI. Money math runs through the framework bridge regardless. |

### Break-In (Police)

```lua
Config.BreachJobs = {
    ['police'] = true,
    ['swat']   = true,
    ['sheriff'] = true,
}
```

Job names match the active framework's job table. Set to `{}` to disable the breach feature entirely.

### Wardrobe Backend

```lua
Config.WardrobeBackend = 'auto'
```

| Value | Behavior |
| --- | --- |
| `'auto'` | Auto-detect installed clothing resource (recommended). |
| `'illenium-appearance'` / `'qb-clothing'` / `'fivem-appearance'` / `'esx_skin'` | Force a specific bridged backend. |
| `'standalone'` | No clothing resource: the wardrobe target is hidden inside the shell. |

The bridge layer at `bridge/client/wardrobe.lua` also exposes a `BUYER_OVERRIDES` table — drop any non-bridged clothing resource (e.g. `betterv_clothing`) into it without touching the rest of the code.

### Target / Inventory

```lua
Config.TargetBackend    = 'auto'
Config.InventoryBackend = 'auto'
```

Backends mirror `community_bridge` adapter naming. `'auto'` picks whichever is installed.

### Stash

```lua
Config.Stash = {
    slots     = 70,
    weightKg  = 70,
    perPlayer = true,
    blacklist = {
        -- ['water'] = true,
    },
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `slots` | `70` | Stash slot count per room. |
| `weightKg` | `70` | Max weight in kilograms. |
| `perPlayer` | `true` | When `true`, every renter gets their own private stash even when sharing a room. When `false`, the stash is shared among everyone with key access. |
| `blacklist` | `{}` | Map of `item_name = true` items blocked from being stored in any motel stash. Useful for preventing bulk-water stash-stacking. |

### Interaction Icons & Labels

```lua
Config.Interaction = {
    rent     = { icon = 'fas fa-key',         label = 'target_rent'     },
    door     = { icon = 'fas fa-door-open',   label = 'target_door'     },
    stash    = { icon = 'fas fa-box',         label = 'target_stash'    },
    wardrobe = { icon = 'fas fa-tshirt',      label = 'target_wardrobe' },
    bed      = { icon = 'fas fa-bed',         label = 'target_bed'      },
    exit     = { icon = 'fas fa-sign-out-alt',label = 'target_exit'     },
}
```

`icon` accepts any Font Awesome class supported by your target resource. `label` is a translation key — pass any literal string to override.

### Blip

```lua
Config.Blip = {
    enabled    = true,
    sprite     = 475,
    color      = 4,
    scale      = 0.7,
    shortRange = true,
}
```

Blip placement follows each motel's `rentNpc.coord`. The label is composed from the motel `label` translation key plus the `blip_label` locale entry (defaults to `"Motel"`).

### Database

```lua
Config.DBPrefix = 'clads_motels_'
```

Table names are built as `<prefix>motels` and `<prefix>instances`. Change only if you need to namespace differently (e.g. multi-server shared DB).

### Shell Definitions (TP mode)

Shell models referenced by `Config.Motels[*].shell`. Each shell needs per-room offsets relative to the shell origin (bed, stash, wardrobe, exit). MLO motels (`Mlo = true`) ignore this table and use the door coords directly.

```lua
Config.Shells = {
    ['tihulu_kafi_motel'] = {
        model = `tihulu_kafi_motel`,
        offsets = {
            exit     = vec3(-0.11, -4.41, 1.16),
            stash    = vec3(-3.99,  4.16, 1.16),
            wardrobe = vec3( 4.28, -2.50, 1.16),
            bed      = vec3(-4.14,  0.64, 0.80),
        },
    },
}
```

| Key | Type | Purpose |
| --- | --- | --- |
| `model` | model hash | The shell prop loaded with `RequestModel`. Use backticks for compile-time hashing. |
| `offsets.exit` | `vec3` | Where the player spawns on entry and exits to the world from. |
| `offsets.stash` | `vec3` | Stash target zone offset. |
| `offsets.wardrobe` | `vec3` | Wardrobe target zone offset. |
| `offsets.bed` | `vec3` | Bed target zone offset (sleep notification). |

### Motels

Each entry in `Config.Motels` defines one rental location. Two modes:

- `Mlo = false` → TP / shell mode. Each room teleports the renter into the named shell instance. Set `shell` to a key from `Config.Shells`.
- `Mlo = true` → MLO mode. Each room is a fixed door in a real GTA interior. `shell` is ignored and the per-room `stash`, `wardrobe`, `bed`, `exit` coords are used directly.

```lua
Config.Motels = {
    {
        id              = 'vespucci',
        label           = 'motel_vespucci',
        Mlo             = false,
        shell           = 'tihulu_kafi_motel',
        rentalPeriod    = 'week',
        rate            = 800,
        purchasePrice   = 800000,
        paymentAccount  = 'bank',
        maxOccupants    = 50,
        radius          = 150.0,
        center          = vec3(-1473.68, -659.35, 28.98),
        rentNpc = {
            coord   = vec3(-1477.07, -674.46, 29.04),
            heading = 37.81,
            model   = 's_m_y_armymech_01',
        },
        rooms = {
            [1]  = { door = { { coord = vec3(-1493.74, -668.39, 29.01), model = `v_ilev_fh_door01` } } },
            -- … 38 rooms in the shipping config …
        },
    },
}
```

| Key | Type | Purpose |
| --- | --- | --- |
| `id` | string | Unique key. Stored in DB and used by every callback. |
| `label` | string | Translation key (e.g. `'motel_vespucci'`) or hardcoded literal shown in panel headers and the blip. |
| `Mlo` | boolean | TP shell mode (`false`) or MLO interior mode (`true`). |
| `shell` | string | Key into `Config.Shells`. Ignored when `Mlo = true`. |
| `rentalPeriod` | string | `'day'` / `'week'` / `'month'`. Maps to `Config.RentPeriods` for duration math. |
| `rate` | integer | Per-period rent. |
| `purchasePrice` | integer | Business price. Returned at half value when sold back via the dashboard. |
| `paymentAccount` | string | Default account to charge for rent and purchase. Falls through `Config.PaymentAccounts` if balance is short. |
| `maxOccupants` | integer | Per-room cap. |
| `radius` | number | Sphere radius around `center` that triggers the door target zones to attach. |
| `center` | `vec3` | Geometric center of the motel — door zones only spawn while a player is inside this sphere. |
| `rentNpc.coord` | `vec3` | Manager NPC spawn coord (z is lowered by 1.0 to ground the ped). |
| `rentNpc.heading` | number | Initial heading. |
| `rentNpc.model` | string\|hash | Ped model for the manager. |
| `rooms[index]` | table | One entry per room. `door` is a list of door entities (single-door rooms have a one-element list). |

For MLO motels, every room entry should also carry `stash`, `wardrobe`, `bed`, and `exit` `vec3` coords — the resource will skip the shell offset math and use these directly.

### Rent Periods

```lua
Config.RentPeriods = {
    day   = 86400,
    week  = 604800,
    month = 2592000,
}
```

Maps `Config.Motels[*].rentalPeriod` to a duration in seconds. Override values to shorten or lengthen one period across all motels.

---

## Commands

The resource ships no public commands. All player interactions happen through the manager NPC, the door target zones around each motel, and the room-management target inside the shell.

Server owners can register their own admin tools on top of the server callbacks if needed — see the **Exports & Events** section.

---

## Customization

### Switching theme presets

The fastest way to re-skin the UI is to copy one of the six commented presets in `config.lua` over the active `Config.UI` block and restart the resource. No NUI rebuild required — colors are CSS variables injected at runtime.

If you want a custom theme, edit the hex values directly. Each one is documented inline. Keep `success` / `warning` / `danger` distinct from `primary` so the access-mode badges remain readable.

### Adding a new motel

Append a record to `Config.Motels` following the schema above. The minimum viable entry needs `id`, `label`, `Mlo`, `rentalPeriod`, `rate`, `purchasePrice`, `maxOccupants`, `radius`, `center`, `rentNpc`, and at least one entry under `rooms`. On the next restart the server auto-creates the row in `clads_motels_motels` and the manager NPC appears.

### Adding rooms to an existing motel

Add new keys to the `rooms` table:

```lua
[39] = { door = { { coord = vec3(-1500.00, -670.00, 29.01), model = `v_ilev_fh_door01` } } },
```

The room is empty-state by default. It will appear in the rental UI on the next motel panel open.

### Adding a new shell

Add an entry to `Config.Shells` keyed by your shell prop name. Calibrate the `exit` offset against the door inside the shell, then `stash`, `wardrobe`, and `bed` against the matching props. Reference `tihulu_kafi_motel` as a starting point — those offsets are tuned for the model that ships in `stream/`.

### Switching to an MLO motel

Set `Mlo = true` on the motel and define each room's interior coords:

```lua
[1] = {
    door     = { { coord = vec3(...), model = `v_ilev_fh_door01` } },
    stash    = vec3(...),
    wardrobe = vec3(...),
    bed      = vec3(...),
    exit     = vec3(...),  -- where the renter is teleported to inside the MLO
},
```

The TP shell will not load and the renter stays in the world after entering.

### Adding a non-bridged clothing system

Open `bridge/client/wardrobe.lua` (escrow-ignored — editable post-encryption) and add an entry to `BUYER_OVERRIDES`:

```lua
local BUYER_OVERRIDES = {
    ['betterv_clothing'] = function()
        TriggerEvent('betterv_clothing:client:openOutfitMenu')
    end,
    ['my_clothing'] = function()
        exports.my_clothing:openMenu()
    end,
}
```

When `Config.WardrobeBackend = 'auto'`, the override is preferred over the bridged adapter if its resource is started. Pin the name explicitly via `Config.WardrobeBackend = 'my_clothing'` to override auto-detection.

### Adjusting payment fallback order

`Config.PaymentAccounts` controls which accounts are tried for rent and purchase. The first one with enough balance is charged. To force every charge through bank only, set `{ 'bank' }`. To prefer cash, swap the order to `{ 'cash', 'bank' }`. Standalone always returns `'free'`.

### Disabling the police breach

Set `Config.BreachJobs = {}` to drop the raycast listener. The lockpick path remains active for MLO motels.

### Custom blip

Adjust `sprite`, `color`, and `scale`. The blip label is composed from the motel `label` plus the `blip_label` translation key — replace `blip_label` in your locale file (or override per motel by pointing `motelCfg.label` at a literal string).

---

## Exports & Events

### Net events

The resource exposes two server-side net events for use cases that need to fire from outside the standard NUI flow:

| Event | Side | Payload | Purpose |
| --- | --- | --- | --- |
| `clads_motels:server:toggleDoor` | Client → Server | `{ motel, index, doorindex, coord, Mlo, citizenid }` | Server-side door state toggle for MLO motels. The server re-validates that `data.citizenid` matches the requester. |
| `clads_motels:server:breachDoor` | Client → Server | `{ motel, index, coord, Mlo, citizenid }` | Police breach. The server re-validates the requester's job against `Config.BreachJobs` before flipping the room to public access. |

The matching `clads_motels:client:doorState` event is broadcast back to all clients for visual feedback hooks.

### Server callbacks (`lib.callback.register`)

All player ↔ server traffic flows through `lib.callback`. Names mirror the event surface so external tools can reuse them.

| Callback | Purpose |
| --- | --- |
| `clads_motels:server:getMotels` | Returns the full `GlobalState.Motels` snapshot + `os.time()`. |
| `clads_motels:server:rentRoom` | Charges the player and writes the new room player record. Returns `true \| 'has_room' \| 'no_money' \| 'already_rented' \| 'invalid_duration'`. |
| `clads_motels:server:payRent` | Extends the player's `duration` by `amount / rate` periods. Returns `true \| 'no_money' \| 'invalid_amount'`. |
| `clads_motels:server:createKey` | Adds a `keys` item with `{ type, serial, owner }` metadata via `Bridge.Inventory.AddItem`. |
| `clads_motels:server:buyMotel` | Sets `motel.owner` to the buyer citizenid. Returns boolean. |
| `clads_motels:server:sellMotel` | Clears owner + employees, refunds half the purchase price. Returns boolean. |
| `clads_motels:server:editRate` | Owner-only. Updates `motel.hourRate`. |
| `clads_motels:server:withdrawFunds` | Owner-only. Debits `motel.revenue`, credits the player's cash. |
| `clads_motels:server:transferMotel` | Owner-only. Hands the motel to the target source. |
| `clads_motels:server:addEmployee` / `removeEmployee` | Owner-only. Maintains `motel.employees`. |
| `clads_motels:server:addOccupant` / `removeOccupant` | Owner can add or remove tenants from a room directly. |
| `clads_motels:server:sendInvoice` | Pushes a `clads_motels:client:invoice` event to the target. Waits 60 seconds for payment, returns `true` if paid. |
| `clads_motels:server:payInvoice` | Charges the invoice amount, credits motel revenue. |
| `clads_motels:server:canEnterRoom` | Returns `{ allowed, reason, mode }`. Reasons: `owner`, `staff`, `key`, `permanent_access`, `doorbell_once`, `public`, `doorbell`, `private`, `not_found`. |
| `clads_motels:server:registerShellPresence` | Records the player as inside a specific room (used by doorbell broadcast). |
| `clads_motels:server:getRoomManagement` | Builds the Room Management panel payload for the requesting renter. |
| `clads_motels:server:setAccessMode` | Switches a room's access mode between `public`, `doorbell`, and `private`. |
| `clads_motels:server:addPermAccess` / `removePermAccess` | Manages the room's permanent allow-list. |
| `clads_motels:server:clearDoorbell` | Removes a doorbell request from the queue. |
| `clads_motels:server:approveDoorbell` | Grants a one-time entry to the requester. |
| `clads_motels:server:kickRoomPlayer` | Force-exits a visitor from the shell. Refuses during a police breach. |
| `clads_motels:server:getShellCoords` | Allocates / returns the persistent shell coord for a citizenid. |
| `clads_motels:server:clearShellInstance` | Unregisters the player's shell presence on exit. |
| `clads_motels:server:openStash` | Validates access and opens the room stash via `Bridge.Inventory.OpenStash`. |
| `clads_motels:server:getRoomKeys` | Returns the list of key holders the player owns a key for in a given room. |

### Server-pushed client events

| Event | Payload | Purpose |
| --- | --- | --- |
| `clads_motels:client:doorbellRing` | `{ id, citizenid, name, motel, index, targetCitizenid, createdAt }` | Notifies players inside the room that someone rang the doorbell. |
| `clads_motels:client:invoice` | `{ motel, amount, description, id, payment, sender }` | Pops up the invoice payment UI on the target. |
| `clads_motels:client:forceExitRoom` | `{ motel, index }` | Owner kicked the player out — client fades and teleports them back to `lastloc`. |
| `clads_motels:client:doorState` | `{ motel, index, ... }` | Broadcast door state change for visual feedback. |

### Lifecycle events consumed

| Event | Side | Purpose |
| --- | --- | --- |
| `community_bridge:Client:OnPlayerLoaded` | Client | Re-fetch `gPlayerData.identifier` and `job`. |
| `community_bridge:Client:OnPlayerUnload` | Client | Clear local identifier. |
| `community_bridge:Client:OnPlayerJobUpdate` | Client | Update `gPlayerData.job` for the breach gate. |
| `playerDropped` | Server | Unregister shell presence to keep the doorbell broadcast list clean. |

---

## Troubleshooting

### Manager NPC does not spawn

- Confirm `community_bridge`, `ox_lib`, and `oxmysql` are started **before** `clads_motels` in your `server.cfg`.
- Check `F8` for `[clads_motels]` errors. A missing `locales/en.json` raises a fatal error at boot.
- Verify the ped model in `Config.Motels[*].rentNpc.model` exists. Invalid models fail to load and no ped is created.
- The boot loop waits for `GlobalState.Motels` to populate. If you joined before the server finished its initial `State.FetchAllMotels()` call, the spawn happens on the `community_bridge:Client:OnPlayerLoaded` event — restart the resource or rejoin if it's missing.

### Door target options do not appear

- Make sure one of `ox_target`, `qb-target`, or `sleepless_interact` is started and that `community_bridge` recognises it.
- Pin the backend explicitly: `Config.TargetBackend = 'ox_target'` (or `'qb-target'`).
- Door zones only attach while the player is inside the motel sphere defined by `Config.Motels[*].center` and `radius`. Stand close enough to enter the sphere and the zones will pop in.
- The "Enter your room" option is gated by `IsRoomMine` (ownership check). The "Room management" option is gated identically. Until you actually rent the room, only "Enter room" (visit someone else) and "Force lock" (with a `lockpick` item) are visible.

### "You have no active rental in this room" when targeting your own door

- The room state lives in `GlobalState.Motels[motelKey].rooms[index].players[citizenid]`. Confirm via the F8 console:
  ```
  print(json.encode(GlobalState.Motels.vespucci.rooms['1'].players))
  ```
- If the entry is missing, your rent expired and `Config.AutoKickOnExpire` cleared it. Pay rent before the timer hits zero, or set `Config.AutoKickOnExpire = false` to keep expired tenants in place.

### Stash does not open inside the shell

- The stash open call goes through `Bridge.Inventory.OpenStash` server-side. Confirm a supported inventory backend is started (`ox_inventory`, `qb-inventory`, etc.) and that `community_bridge` is bound to it.
- The server re-runs the access check when handling the open. If your access mode was just downgraded to `private` and you don't have a key or permanent access, the stash refuses to open. Set the mode to `public` from the Room Management panel, or have the owner grant you permanent access.
- `Config.Stash.perPlayer = true` (default) means each renter has their own stash even when sharing a room. To share a single stash, set this to `false`.

### Wardrobe target is missing or does nothing

- `Config.WardrobeBackend = 'auto'` requires one of the bridged clothing resources (`illenium-appearance`, `qb-clothing`, `fivem-appearance`, `esx_skin`) or a buyer-override entry in `bridge/client/wardrobe.lua`.
- Set `Config.WardrobeBackend = 'standalone'` to hide the wardrobe target altogether on servers without a clothing system.

### Police breach does not work

- The shooter's job (read via `community_bridge:Client:OnPlayerJobUpdate`) must match a key in `Config.BreachJobs`.
- The server re-validates the job server-side, so rejecting the breach silently usually means the bridge returned an unexpected job name. Print `Bridge.Framework.GetPlayerJobData()` in F8 to see what the framework reports.
- The bullet must land within 2 meters of a configured `door.coord`. Walk closer to the door before firing.
- The targeted room must have at least one occupant whose `accessMode` is `doorbell` or `private`. Public rooms cannot be breached because they're already open.

### Shell loads but the player falls through the floor

- Collision needs to load before the freeze releases. The client requests collision for up to 4 seconds before spawning the shell, then up to 5 more seconds after. On heavily-modded maps that may not be enough — increase the `timeout` values in `EnterShell` (`client/main.lua`) if your server consistently falls through.
- Verify the shell `model` actually streams. Add the shell `.ydr` to a streaming resource, or keep using the bundled `tihulu_kafi_motel` model that ships in `stream/`.

### Tenants don't see locale changes

- The locale dictionary is sent on every `SendNUIMessage` open call. If a panel was already on screen when the convar changed, close and reopen it.
- For Lua-side strings (notifications, target labels), the locale loader picks the value at resource boot. Restart `clads_motels` to refresh after changing `Config.Locale`.

---

## Support

Reach out via the Creative Lads Discord.
