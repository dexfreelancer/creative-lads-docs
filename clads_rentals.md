# clads_rentals

A framework-agnostic vehicle rental system for FiveM. Players approach a rental clerk, open a tablet-style React UI, browse a category-based vehicle catalog, and drive away in a freshly spawned vehicle. Returning the vehicle to the same location refunds a configurable percentage of the rental price.

---

## Overview

`clads_rentals` is designed as the "starter car" loop for new players on a roleplay server. It is built around a small set of interactions:

- **Tablet UI** — A React + Tailwind front end is loaded as a NUI page. The layout is a sidebar of categories on the left and a vehicle grid on the right, with hero images pulled from a public GTA V vehicle image CDN and a graceful icon fallback when an image is missing.
- **Framework-agnostic** — All framework, target, notification, and vehicle-key calls go through `community_bridge`. The same install runs on Standalone, ESX, QBCore, and QBox without code changes. On Standalone every rental is treated as free because there is no economy to charge.
- **Multi-location support** — Four rental locations ship by default: Los Santos International Airport, Legion Square, Sandy Shores, and Paleto Bay. Each location can override the global vehicle catalog and define one or many spawn slots.
- **Category-based catalog** — Vehicles are grouped into `economy`, `compact`, and `motorcycle` categories out of the box. Categories drive the sidebar navigation, accent colors, and icons shown in the React UI.
- **Per-rental pricing** — Each vehicle has a flat price that is charged on rental and partially refunded on return. Prices are validated server-side from the catalog so a tampered NUI payload cannot lower the cost.
- **Refund on return** — Players take the vehicle back to a rental ped (within `Config.ReturnDistance` meters) and receive a configurable percentage of the original price back to the same account they paid from.
- **Vehicle keys integration** — When a supported vehicle-keys resource is present, keys are granted automatically through the `community_bridge` `VehicleKey` adapter. Standalone or no-keys setups skip the call silently.

The system also enforces a one-rental-per-player limit (configurable), persists active rentals in memory keyed by player identifier, and cleans them up on player drop, framework unload, or when the vehicle no longer exists.

---

## Requirements

| Dependency | Required | Purpose |
| --- | --- | --- |
| `community_bridge` | Yes | Framework, target, notification, and vehicle-key abstraction. |
| `ox_lib` | Yes | Callbacks, alert dialogs, text UI, ped streaming, and command registration. |
| Framework | Optional (auto-detected) | `es_extended`, `qb-core`, or `qbx_core`. Falls back to standalone (free rentals) when none is detected. |
| Target backend | Optional (auto-detected) | `ox_target`, `qb-target`, or `sleepless_interact`. The rental ped is unreachable without one of these. |
| Vehicle keys backend | Optional (auto-detected) | `qbx_vehiclekeys`, `qb-vehiclekeys`, `qs-vehiclekeys`, `mrnewbvehiclekeys`, `wasabi_carlock`, or any other backend `community_bridge` exposes through its `VehicleKey` adapter. |

If no vehicle-keys backend is installed, rentals still work — the player simply owns the spawned vehicle without an explicit key entry and any hotwire blocker your server runs is bypassed for that vehicle.

---

## Installation

1. Drop the `clads_rentals` folder into your `resources/` directory (commonly under `[clads]/` or any bracket folder of your choice).
2. Install the required dependencies if you do not have them already:
   - `community_bridge`
   - `ox_lib`
3. Make sure your chosen target resource (`ox_target`, `qb-target`, or `sleepless_interact`) is started **before** `clads_rentals`.
4. Add the resource to your `server.cfg`:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure ox_target          # or qb-target / sleepless_interact
   ensure clads_rentals
   ```
5. Open `clads_rentals/config.lua` and adjust the locations, prices, theme, and payment accounts to match your server.
6. (Optional) Translate `locales/en.json` into another language by copying it to `locales/<code>.json` and changing `Config.Locale` to that code, or leave `Config.Locale = 'auto'` to follow your server's existing locale convars.
7. Restart the server.

The rental peds, target options, and (optionally) blips spawn automatically on resource start and again when a player loads in via `community_bridge:Client:OnPlayerLoaded`.

---

## Configuration

All configuration lives in `clads_rentals/config.lua`, which remains editable after Tebex/CFX escrow encryption (see `escrow_ignore` in `fxmanifest.lua`).

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

Built-in locales: `en`, `tr`, `de`, `fr`, `es`. Add your own by dropping a `locales/<code>.json` next to the existing files.

When `Config.Locale = 'auto'`, the loader picks the first non-empty value from the following convars and falls back to `en`:

1. `clads_locale`
2. `ox:locale`
3. `qb_locale`
4. `lang`

### Debug

```lua
Config.Debug = false
```

Reserved flag for future verbose logging. Leave at `false` for production.

### UI Theme

The React tablet's color scheme is injected into `:root` at runtime. Defaults below are the shipping **Dark Red** theme.

```lua
Config.UI = {
    primary       = '#FF003C',
    primaryLight  = '#FF3D63',
    primaryDark   = '#CC0030',
    action        = '#FF003C',
    secondary     = '#888888',
    success       = '#22C55E',
    warning       = '#F59E0B',
    danger        = '#FF003C',
    bgCore        = '#0D0D0D',
    bgSurface     = '#1A1A1A',
    bgElevated    = '#2E2E2E',
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `primary` | `#FF003C` | Main brand color (selected card border, active nav). |
| `primaryLight` | `#FF3D63` | Hover shade. |
| `primaryDark` | `#CC0030` | Gradient start. |
| `action` | `#FF003C` | Confirm CTA accent. |
| `secondary` | `#888888` | Economy category accent. |
| `success` | `#22C55E` | Success / "free" pill. |
| `warning` | `#F59E0B` | Motorcycle category accent. |
| `danger` | `#FF003C` | Cancel / error states. |
| `bgCore` | `#0D0D0D` | Darkest background layer. |
| `bgSurface` | `#1A1A1A` | Tablet background. |
| `bgElevated` | `#2E2E2E` | Card surfaces. |

Every value must be a 6-digit hex color with a leading `#`.

### Payment

```lua
Config.PaymentAccounts   = { 'bank', 'cash' }
Config.ReturnRefundPercent = 50
Config.OneRentalPerPlayer  = true
```

| Key | Default | Purpose |
| --- | --- | --- |
| `PaymentAccounts` | `{ 'bank', 'cash' }` | Order in which accounts are checked. The first account with enough balance is charged. `community_bridge` maps `'money'` to `'cash'` on QB/QBox automatically. |
| `ReturnRefundPercent` | `50` | Percentage of the rental price refunded when the vehicle is returned (0–100). The refund goes back to the same account the player paid from. |
| `OneRentalPerPlayer` | `true` | When `true`, a player cannot start a new rental until their current one is returned. |

On standalone (no framework), all rentals are treated as free regardless of catalog price.

### Target Backend

```lua
Config.TargetBackend = 'auto'
Config.ReturnDistance = 20.0
```

| Key | Values | Purpose |
| --- | --- | --- |
| `TargetBackend` | `'auto'`, `'ox_target'`, `'qb-target'` | Mirrors the `community_bridge` target adapters. `'auto'` picks the first installed one. |
| `ReturnDistance` | Number (meters) | Maximum distance from the rental ped at which a player may return their vehicle. |

### Vehicle Keys

```lua
Config.GiveKeys = true
```

When `true`, the client calls `community_bridge`'s `VehicleKey.GiveKeys` after the rental spawns. If no keys backend is installed (or the resolved backend resource name is `'default'`), the call is skipped silently.

### Plate Format

```lua
Config.PlatePrefix  = 'RENT'
Config.PlatePadding = 4
```

The server generates plates as `<Prefix><zero-padded-id>`. With the defaults this produces `RENT0001`, `RENT0002`, and so on, wrapping back to `0000` after 9999. Keep the total length at 8 characters or fewer.

### Rental NPC

```lua
Config.PedModel      = 's_m_m_autoshop_02'
Config.PedScenario   = 'WORLD_HUMAN_CLIPBOARD'
Config.PedFreeze     = true
Config.PedInvincible = true
```

| Key | Default | Purpose |
| --- | --- | --- |
| `PedModel` | `s_m_m_autoshop_02` | Model used for every rental clerk. Any valid ped model works. |
| `PedScenario` | `WORLD_HUMAN_CLIPBOARD` | Scenario played in place. Set to `''` to skip the scenario. |
| `PedFreeze` | `true` | Locks the ped's position so it cannot be pushed. |
| `PedInvincible` | `true` | Makes the ped invulnerable to damage. |

### Blip

```lua
Config.Blip = {
    enabled    = false,
    sprite     = 225,
    color      = 5,
    scale      = 0.7,
    shortRange = true,
    label      = 'blip_label',
}
```

| Key | Default | Purpose |
| --- | --- | --- |
| `enabled` | `false` | Set to `true` to draw a map blip at every rental location. |
| `sprite` | `225` | Standard GTA V blip sprite ID. |
| `color` | `5` | GTA V blip color index. |
| `scale` | `0.7` | Blip size on the map. |
| `shortRange` | `true` | When `true`, the blip is hidden when zoomed out. |
| `label` | `'blip_label'` | Translation key resolved against the active locale. Falls back to the literal string if no translation exists. |

### Interaction Icons & Labels

```lua
Config.Interaction = {
    rent = {
        icon  = 'fas fa-car',
        label = 'target_rent',
    },
    ['return'] = {
        icon  = 'fas fa-undo',
        label = 'target_return',
    },
}
```

`icon` accepts any Font Awesome class supported by your target resource. `label` is a translation key — pass any literal string to override.

### Locations

```lua
Config.Locations = {
    {
        id     = 'airport',
        label  = 'loc_airport',
        ped    = vec4(-1037.73, -2732.98, 20.17, 238.13),
        spawns = {
            vec4(-1027.28, -2733.51, 20.09, 239.55),
            vec4(-1036.06, -2728.81, 20.08, 237.94),
            vec4(-1043.15, -2724.65, 20.08, 238.82),
        },
        vehicles = {},
    },
    {
        id       = 'legion',
        label    = 'loc_legion',
        ped      = vec4(215.56, -808.97, 30.74, 339.85),
        spawn    = vec4(227.42, -798.64, 30.58, 158.0),
        vehicles = {},
    },
    {
        id       = 'sandy',
        label    = 'loc_sandy',
        ped      = vec4(1944.51, 3764.21, 32.21, 268.34),
        spawn    = vec4(1953.05, 3761.01, 31.78, 29.47),
        vehicles = {},
    },
    {
        id       = 'paleto',
        label    = 'loc_paleto',
        ped      = vec4(-210.12, 6218.35, 31.49, 266.66),
        spawn    = vec4(-205.09, 6222.55, 31.07, 224.38),
        vehicles = {},
    },
}
```

| Key | Type | Purpose |
| --- | --- | --- |
| `id` | string | Unique identifier used in callbacks, exports, and target names. |
| `label` | string | Translation key (e.g. `'loc_airport'`) or hardcoded literal shown in the tablet header. |
| `ped` | `vec4(x, y, z, w)` | Where the rental clerk is spawned. The `z` coordinate is automatically lowered by 1.0 to ground the ped. |
| `spawn` | `vec4(x, y, z, w)` | Single fallback spawn slot for the vehicle. |
| `spawns` | `vec4[]` | Optional list of spawn slots. The server picks the first slot not already occupied by another vehicle within 3 meters. |
| `vehicles` | table | Per-location catalog override. Leave empty (`{}`) to use `Config.DefaultVehicles`. |

A location must define at least one of `spawn` or `spawns`.

### Default Vehicles

Used by every location whose `vehicles` table is empty.

```lua
Config.DefaultVehicles = {
    { model = 'asea',       label = 'Asea',       price = 63,  category = 'economy'    },
    { model = 'emperor',    label = 'Emperor',    price = 63,  category = 'economy'    },
    { model = 'prairie',    label = 'Prairie',    price = 125, category = 'economy'    },
    { model = 'blista',     label = 'Blista',     price = 188, category = 'economy'    },
    { model = 'dilettante', label = 'Dilettante', price = 250, category = 'compact'    },
    { model = 'issi2',      label = 'Issi',       price = 313, category = 'compact'    },
    { model = 'faggio',     label = 'Faggio',     price = 94,  category = 'motorcycle' },
    { model = 'sanchez',    label = 'Sanchez',    price = 250, category = 'motorcycle' },
}
```

| Field | Purpose |
| --- | --- |
| `model` | Vehicle spawn name. |
| `label` | Display name shown on the card. |
| `price` | Flat rental price in the active currency. |
| `category` | Must match a key in `Config.Categories`. |

### Categories

```lua
Config.Categories = {
    economy    = { label = 'cat_economy',    icon = 'car'       },
    compact    = { label = 'cat_compact',    icon = 'car-front' },
    motorcycle = { label = 'cat_motorcycle', icon = 'bike'      },
}
```

`label` is a translation key resolved against the active locale. `icon` is a glyph name resolved by the React UI:

| Icon value | Lucide icon |
| --- | --- |
| `car` | `Car` |
| `car-front` | `CarFront` |
| `bike` | `Bike` |
| `gift` | `Gift` |

Any unknown icon value falls back to `Car`.

---

## Commands

Players never type a command — all interactions happen through the rental ped's target options. The resource ships two admin commands restricted to `group.admin`.

| Command | Restricted | Description |
| --- | --- | --- |
| `/rentals` | `group.admin` | Lists every active rental: identifier, model, plate, and minutes since rental start. |
| `/clearrental <identifier>` | `group.admin` | Force-deletes the active rental tied to the given player identifier (citizenid or license). Useful for clearing stuck entries. |

Both commands are registered through `lib.addCommand` and respect ox_lib's permission system.

---

## Customization

### UI Theme

Tweak the colors in `Config.UI`. Every value is shipped as part of the NUI `open` payload, which calls `applyTheme` to set CSS variables on `:root`. No rebuild is required — restart the resource and the new theme takes effect on the next open.

If you want a lighter theme, raise `bgCore`, `bgSurface`, and `bgElevated` to lighter greys and pick a softer `primary`. Because the success/warning/danger keys drive the "free" pill, motorcycle accent, and error states respectively, keep them visually distinct from `primary`.

### Adding Rental Locations

Append a new entry to `Config.Locations`:

```lua
table.insert(Config.Locations, {
    id     = 'mirror_park',
    label  = 'Vehicle Rental — Mirror Park',
    ped    = vec4(1138.50, -752.40, 57.78, 100.0),
    spawns = {
        vec4(1141.20, -748.10, 57.50, 100.0),
        vec4(1144.30, -745.20, 57.50, 100.0),
    },
    vehicles = {}, -- inherit the default catalog
})
```

If you want a location-specific catalog (for example, motorbikes only at a dirt-bike rental), pass a non-empty `vehicles` table that follows the same schema as `Config.DefaultVehicles`.

### Adding Vehicles

Add new entries to `Config.DefaultVehicles` (or a per-location `vehicles` table):

```lua
{ model = 'panto', label = 'Panto', price = 80, category = 'compact' },
```

The hero image is loaded from a public CDN at `https://raw.githubusercontent.com/MericcaN41/gta5carimages/main/images/<model>.png`. If the image is missing, the card falls back to a Lucide icon (`Car`, or `Bike` for motorcycles).

### Changing Categories

Define new categories in `Config.Categories`:

```lua
Config.Categories.suv = { label = 'cat_suv', icon = 'car' }
```

Then add a `cat_suv` translation to every locale JSON file (or hardcode a literal string in `label`), and set `category = 'suv'` on whichever vehicles should appear under it. The sidebar will pick up the new category automatically.

### Swapping the NPC Model

Change `Config.PedModel` to any valid ped model (e.g. `'a_m_y_business_03'`). The same model is reused at every location. To use different models per location you would need to extend the spawn loop in `client/functions.lua`, which is outside the scope of plain configuration.

### Custom Blip

Set `Config.Blip.enabled = true`, then adjust `sprite`, `color`, and `scale` to match your map style. The blip label is a translation key by default — point it at a literal like `'Vehicle Rentals'` to skip the locale dictionary.

---

## Exports & Events

### Client exports

| Export | Returns | Description |
| --- | --- | --- |
| `exports.clads_rentals:HasActiveRental()` | `boolean` | `true` if the local player currently has a tracked rental vehicle. |
| `exports.clads_rentals:GetRentalVehicle()` | `number\|nil` | Entity handle of the active rental vehicle, or `nil`. |

### Server exports

| Export | Returns | Description |
| --- | --- | --- |
| `exports.clads_rentals:GetActiveRentals()` | `table` | The full `activeRentals` map keyed by player identifier. Each value contains `netId`, `plate`, `model`, `price`, `account`, `rentedAt`, `locationId`. |
| `exports.clads_rentals:HasActiveRental(identifier)` | `boolean` | `true` if the given identifier has an active rental. |
| `exports.clads_rentals:GetPlayerRental(identifier)` | `table\|nil` | The rental record for the given identifier, or `nil`. |

### Net events / callbacks

The resource does not register any public net events. All client/server traffic flows through two `lib.callback` handlers:

| Callback | Side | Purpose |
| --- | --- | --- |
| `clads_rentals:server:processRental` | Server callback | Validates the request against the catalog, charges the player, spawns the vehicle, sets the plate, and returns `{ success, netId, plate }`. |
| `clads_rentals:server:returnRental` | Server callback | Verifies the rental belongs to the caller, refunds `Config.ReturnRefundPercent` to the original account, deletes the vehicle, and clears the entry. |

Externally consumed events (for lifecycle hooks):

| Event | Side | Purpose |
| --- | --- | --- |
| `community_bridge:Client:OnPlayerLoaded` | Client | Re-spawns rental peds after the player loads in. |
| `community_bridge:Client:OnPlayerUnload` | Client | Clears local rental state and closes the UI. |
| `community_bridge:Server:OnPlayerUnload` | Server | Deletes the player's active rental vehicle. |
| `playerDropped` | Server | Same as the unload event, but for hard disconnects. |

---

## Troubleshooting

### Rental NPC does not spawn

- Confirm `community_bridge` and `ox_lib` are started **before** `clads_rentals` in your `server.cfg`.
- Check `F8` for `[clads_rentals]` errors. A missing `locales/en.json` raises a fatal error at boot.
- Verify the ped model in `Config.PedModel` exists. Invalid models fail to load and no ped is created.
- Standalone servers spawn peds via `onResourceStart`. Framework servers re-spawn them on `community_bridge:Client:OnPlayerLoaded`. If you joined before that event fired, restart the resource or rejoin.

### Target option does not appear

- Make sure one of `ox_target`, `qb-target`, or `sleepless_interact` is started and that `community_bridge` recognises it.
- Pin the backend explicitly: `Config.TargetBackend = 'ox_target'` (or `'qb-target'`).
- Stand within range of the ped — the rental option uses local-entity targeting, so you must be looking directly at the clerk.
- Check that no other resource is overlaying its own target option on the same ped model.

### "Rental failed. Check your funds." on a standalone server

- This usually means `community_bridge` resolved to a framework that returns `0` for both `bank` and `cash` balances. Force standalone explicitly:
  ```lua
  Config.Framework = 'standalone'
  ```
- On a real framework, confirm the player has at least one of the accounts in `Config.PaymentAccounts` and that the balance is greater than or equal to the vehicle price.
- Reorder `Config.PaymentAccounts` so the account most likely to have funds is checked first.

### Vehicle keys are not granted

- Confirm a supported keys backend is installed and started: `qbx_vehiclekeys`, `qb-vehiclekeys`, `qs-vehiclekeys`, `mrnewbvehiclekeys`, `wasabi_carlock`, etc.
- Check `Config.GiveKeys = true`. The client only attempts to grant keys when this flag is enabled and `community_bridge`'s `VehicleKey.GetResourceName()` returns something other than `'default'`.
- Some keys backends require the vehicle to belong to a specific persistent owner. Rental vehicles are spawned without an owner record on purpose; if your backend rejects un-owned vehicles, you will need to extend `giveVehicleKeys` in `client/utils.lua`.

### Vehicle will not return

- The player must be within `Config.ReturnDistance` meters of the rental ped's coordinates. Increase the distance if your spawn slots sit far from the clerk.
- The return option only appears in the target menu when `gCurrentRental` is non-`nil`. If the vehicle was destroyed or despawned, the watchdog clears the local state and the option disappears — return is no longer needed.
- The server validates the plate. If you somehow re-plated the vehicle (admin tools, mechanic resource, etc.), the return is rejected. Either restore the original plate or use `/clearrental <identifier>` to drop the entry manually.
- On a standalone server the refund is `0` because no money was charged in the first place. The vehicle still deletes; the "Deposit refunded" notification is a literal translation, not a guarantee that funds moved.

---

## Support

Reach out via the Creative Lads Discord.
