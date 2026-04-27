# Clads Garages

Framework-agnostic vehicle storage for FiveM. Forty-seven ship-ready garages out of the box, with public, faction, gang, depot, and business types, optional mechanic integration for stance and tuning data, an HTTP photo CDN with four provider modes, shared keys support, cross-garage retrieval fees, and a customisable showroom NUI.

---

## Overview

Clads Garages is a complete garage replacement designed to drop into existing servers without dictating which framework, vehicle keys, or mechanic resource you run. The shipping configuration includes 47 fully placed garages spanning every borough of Los Santos and Blaine County:

- **Public garages** parked across the city (Motel, Pillbox, Beach, Hotel, Megamall, etc.).
- **Faction garages** gated by job for Police, SAHP, EMS, and Sheriff units, including dedicated helipads.
- **Gang garages** for La Familia, Lost MC, and Cartel.
- **Depot / impound lots** that pull seized and abandoned vehicles back for a calculated fee, including dedicated air and sea depots.
- **Business / utility garages** for taxi stands, business centres, terminals, and pier locations.

Highlights:

- **Framework-agnostic.** QBox, QB-Core, ESX, and standalone servers are all supported through a single bridge layer powered by `community_bridge`. The adapter is selected once at startup based on `Config.Framework` (or auto-detected).
- **No vehicle table lock-in.** On QBox, Clads adds four extension columns directly onto `player_vehicles`. On QB / ESX / standalone, an overlay table (`clads_garages_state`) stores the new fields without touching your existing schema.
- **Mechanic integration.** Optional hooks let your mechanic resource own vehicle props, statebags (stance, nitrous, tuning), and stance-table reads so retrieved vehicles spawn with their full custom configuration intact.
- **Vehicle photo CDN.** Captured client-side via a green-screen scene and uploaded to one of four backends: disabled, FiveManage, a generic custom HTTP endpoint (imgbb / imgur / Discord webhook / S3), or a buyer-side local resource using token-issued uploads.
- **Shared keys.** Optional integration that reads any "phone vehicle keys" SQL table and surfaces shared vehicles to the recipient inside their garage menu.
- **Cross-garage retrieval.** Retrieve a vehicle parked at any other garage for a configurable flat fee.
- **Impound fees via callback.** `Config.calculateImpoundFee(vehicleId, modelName)` is a Lua function so you can read class, value, or any state you want when computing the depot price.
- **Vehicle transfer.** A `/transfervehicle` command hands ownership to another player and routes the vehicle into your default garage.
- **Customisable showroom NUI.** A web build with vehicle preview camera, body / engine / fuel bars, photo thumbnails, favorites, custom-name rename, and a brandable colour theme injected at runtime.

---

## Requirements

| Dependency         | Purpose                                                                                  | Required |
| ------------------ | ---------------------------------------------------------------------------------------- | -------- |
| `community_bridge` | Framework, notify, fuel, vehicle key, target abstraction.                                | Yes      |
| `ox_lib`           | Callbacks, vehicle props, locale loader, progress bars, JSON locale files.               | Yes      |
| `oxmysql`          | Database access for the overlay and standalone vehicle tables.                           | Yes      |
| Target backend     | `ox_target`, `qb-target`, or `sleepless_interact` — picked automatically by the bridge.  | Yes      |
| Framework          | `qbx_core` / `qb-core` / `es_extended` — or run pure standalone.                          | Optional |
| Vehicle keys       | `qbx_vehiclekeys`, `qb-vehiclekeys`, or `community_bridge.VehicleKey` provider.           | Optional |
| Weather sync       | `Renewed-Weathersync`, `vSync`, `qb-weathersync`, or `qbx_weathersync` — paused for capture. | Optional |
| FiveManage / CDN   | When using the photo capture flow with a remote backend.                                  | Optional |
| Mechanic resource  | Custom mechanic / tuning resource with props + statebag exports.                          | Optional |

The resource ships SQL migrations under `sql/install.sql`. They are also re-applied automatically at first startup through each adapter's `EnsureSchema()` call, so a manual import is only required if you prefer running migrations by hand.

---

## Installation

1. **Drop the resource folder** into your server resources tree, for example `resources/[clads]/clads_garages`.

2. **Install dependencies** in your server.cfg, ensuring `community_bridge`, `ox_lib`, and `oxmysql` start before `clads_garages`:

   ```cfg
   ensure oxmysql
   ensure ox_lib
   ensure community_bridge
   ensure clads_garages
   ```

3. **Run the SQL migration** appropriate to your stack from `sql/install.sql`. Choose one of the sections:

   - **QBox** — adds `custom_name`, `favorite`, `image_url`, `visual_hash` columns to the existing `player_vehicles` table.
   - **QB-Core / ESX / Standalone** — creates the overlay table `clads_garages_state` so the resource never modifies your existing vehicle table.
   - **Standalone-only** — additionally creates `clads_garages_vehicles` to act as the primary vehicle table.

   You can skip this step if you prefer the resource to run the equivalent statements automatically on first start.

4. **Open `config.lua`** and adjust the framework, theme, photo provider, mechanic integration, and garage list to match your server. Every value is documented inline.

5. **Restart your server** (or `start clads_garages` if you've ensured deps are already running). On the first start the bridge prints the detected framework slug, e.g. `[clads_garages] bridge ready (framework: qbx)`.

---

## Configuration

All settings live in `config.lua`, which is loaded as a `shared_script` so the global `Config` table is visible from both client and server. Values are grouped into framework, debug, UI theme, client toggles, server toggles, the impound fee callback, mechanic integration, the photo CDN, shared keys, the optional vehicle model fallback, and the garage definitions themselves.

### Framework

```lua
Config.Framework = 'auto'
```

| Value          | Behaviour                                                                          |
| -------------- | ---------------------------------------------------------------------------------- |
| `'auto'`       | Detect via `community_bridge` (recommended).                                        |
| `'qbox'`       | Force the `qbx_vehicles` adapter.                                                   |
| `'qb'`         | Force the `qb-core` adapter (uses `player_vehicles` plus the overlay table).         |
| `'esx'`        | Force the `es_extended` adapter (uses `owned_vehicles` plus the overlay table).      |
| `'standalone'` | Force the standalone adapter (uses `clads_garages_vehicles`).                       |

### Debug

```lua
Config.Debug = false
```

When `true`, `lib.print.*` output and `ox_target` debug polys are enabled. Leave disabled in production.

### UI Theme

The showroom NUI reads colours from `Config.UI` and injects them into `:root` at runtime. There is no rebuild required for theme changes — re-enter the garage and the new colours apply.

```lua
Config.UI = {
    primary       = '#FF003C',
    primaryLight  = '#FF3D63',
    primaryDark   = '#CC0030',
    action        = '#FF003C',
    success       = '#22C55E',
    warning       = '#F59E0B',
    danger        = '#FF003C',
    secondary     = '#888888',
    bgCore        = '#0D0D0D',
    bgSurface     = '#1A1A1A',
    bgElevated    = '#2E2E2E',
}
```

| Key            | Used For                                          |
| -------------- | ------------------------------------------------- |
| `primary`      | Active card border, focused button.               |
| `primaryLight` | Hover state, gradient end stop.                   |
| `primaryDark`  | Gradient start stop.                              |
| `action`       | "Take Out" call-to-action button.                 |
| `success`      | Body / engine / fuel "good" range on stat bars.   |
| `warning`      | Stat-bar warning range.                           |
| `danger`       | Close button, impound and seized state badges.     |
| `secondary`    | Info accents.                                     |
| `bgCore`       | Darkest background.                               |
| `bgSurface`    | Panel background.                                 |
| `bgElevated`   | Card background.                                  |

The shipping defaults form the "Dark Red" theme. Every value must be a six-digit hex with leading `#`.

### Client Toggles

```lua
Config.enableClient = true
Config.engineOn     = true
Config.debugPoly    = false
```

| Key             | Effect                                                                     |
| --------------- | -------------------------------------------------------------------------- |
| `enableClient`  | Disable to skip the bundled NUI (use this if you ship your own front-end). |
| `engineOn`      | Start the engine after a vehicle is retrieved.                             |
| `debugPoly`     | Render `ox_target` debug shapes around every garage zone.                  |

### Server Toggles

```lua
Config.autoRespawn       = true
Config.warpInVehicle     = true
Config.doorsLocked       = true
Config.distanceCheck     = 5.0
Config.remoteRetrieveFee = 63
Config.defaultGarage     = 'motelgarage'
```

| Key                  | Type     | Default        | Effect                                                                                                                |
| -------------------- | -------- | -------------- | --------------------------------------------------------------------------------------------------------------------- |
| `autoRespawn`        | boolean  | `true`         | On resource start, every vehicle currently in the `OUT` state is moved back into its garage.                          |
| `warpInVehicle`      | boolean  | `true`         | Place the player into the driver seat after the vehicle spawns.                                                       |
| `doorsLocked`        | boolean  | `true`         | Lock the doors after retrieval (uses `qbx_vehiclekeys` / `qb-vehiclekeys` event when present, native fallback otherwise). |
| `distanceCheck`      | number   | `5.0`          | Minimum clearance in metres at the spawn point. If a vehicle is detected nearby, retrieval is blocked.                |
| `remoteRetrieveFee`  | integer  | `63`           | Flat fee charged when retrieving a vehicle that is parked at a different garage. Set to `0` to disable.               |
| `defaultGarage`      | string   | `'motelgarage'` | Garage that newly transferred or orphaned vehicles get routed into.                                                   |

### `calculateImpoundFee` Callback

```lua
---@param vehicleId integer
---@param modelName string
---@return integer fee
Config.calculateImpoundFee = function(vehicleId, modelName)
    return 120
end
```

Called server-side every time a depot price needs to be set — typically when a vehicle moves into the `OUT` state from a garage and when a player retrieves a vehicle from another garage. Replace the body with whatever logic you want (vehicle class, brand-specific multipliers, time-of-day pricing, etc.).

### Mechanic Integration

Wire your mechanic resource here so the garage uses its rich props (stance, nitrous, tuning) instead of plain `ox_lib` props. Leave `enabled = false` to fall back to `lib.getVehicleProperties`.

```lua
Config.Mechanic = {
    enabled              = false,
    resource             = '',
    propsExport          = 'getVehicleProperties',
    statebagExport       = 'getVehicleStatebagProperties',
    statebagSaveCallback = '',
    statebagApplyEvent   = '',
    stanceTable          = nil,
}
```

| Key                    | Type     | Description                                                                                                                                          |
| ---------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`              | boolean  | Master switch. When `false`, every Mechanic call is a no-op.                                                                                          |
| `resource`             | string   | Resource name to read exports from (e.g. `'mymechanic'`).                                                                                             |
| `propsExport`          | string   | Export name returning a props table for a vehicle entity. Called as `exports[<resource>]:<propsExport>(vehicle)`.                                     |
| `statebagExport`       | string   | Export name returning a statebag table (or `nil`) for a vehicle entity.                                                                              |
| `statebagSaveCallback` | string   | `ox_lib` server callback name that persists statebag data, called with `(plate, data)`.                                                              |
| `statebagApplyEvent`   | string   | Server event fired with `(vehicle, plate)` after the vehicle spawns, so the mechanic resource can reapply effects.                                   |
| `stanceTable`          | string \| nil | Optional MySQL table read for an `enableStance` override. When set, the spawn flow strips `modSuspension` before applying props if the row enables stance. |

When a vehicle is parked, the bridge calls `propsExport` to capture rich props and `statebagExport` for stance / nitrous. The plate-keyed statebag is then forwarded to `statebagSaveCallback`. On spawn, `statebagApplyEvent` is fired so the mechanic resource can re-apply its visual effects.

### Vehicle Photo CDN

Vehicle thumbnails are captured client-side via a bundled green-screen scene and uploaded to a backend you choose. The provider is selected with `Config.Photo.provider`.

```lua
Config.Photo = {
    provider = 'disabled',

    fiveManage = { apiKey = '' },

    custom = {
        url             = '',
        headers         = {},
        bodyFormat      = 'multipart',
        fieldName       = 'image',
        responseUrlPath = 'url',
    },

    ['local'] = { resource = '' },
}
```

| Provider     | Behaviour                                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `disabled`   | Skip the capture flow entirely. The `image_url` column stays `NULL` and the showroom shows a placeholder.                                 |
| `fivemanage` | Convenience preset for [fivemanage.com](https://fivemanage.com). Set `Photo.fiveManage.apiKey`. POSTs multipart `image`, expects `{ url }`. |
| `custom`     | Generic HTTP endpoint. Configure `url`, `headers`, `bodyFormat`, `fieldName`, and `responseUrlPath`.                                      |
| `local`      | Use a buyer-side resource that issues short-lived upload tokens. Configure `Photo['local'].resource`.                                     |

#### `custom` field reference

| Field             | Type           | Default       | Description                                                                                |
| ----------------- | -------------- | ------------- | ------------------------------------------------------------------------------------------ |
| `url`             | string         | `''`          | Full upload endpoint. Required.                                                            |
| `headers`         | table          | `{}`          | HTTP headers (e.g. `Authorization`).                                                       |
| `bodyFormat`      | string         | `'multipart'` | One of `'multipart'`, `'json-base64'`, `'binary'`.                                          |
| `fieldName`       | string         | `'image'`     | Form-data field name when `bodyFormat = 'multipart'`.                                      |
| `responseUrlPath` | string         | `'url'`       | Dot-path into the JSON response containing the public URL (e.g. `data.url`).               |

#### Examples

**imgbb.com**

```lua
Config.Photo = {
    provider = 'custom',
    custom = {
        url             = 'https://api.imgbb.com/1/upload?key=YOUR_API_KEY',
        bodyFormat      = 'multipart',
        fieldName       = 'image',
        responseUrlPath = 'data.url',
    },
}
```

**Imgur (anonymous)**

```lua
Config.Photo = {
    provider = 'custom',
    custom = {
        url             = 'https://api.imgur.com/3/image',
        headers         = { ['Authorization'] = 'Client-ID YOUR_CLIENT_ID' },
        bodyFormat      = 'multipart',
        fieldName       = 'image',
        responseUrlPath = 'data.link',
    },
}
```

**Discord webhook**

```lua
Config.Photo = {
    provider = 'custom',
    custom = {
        url             = 'https://discord.com/api/webhooks/.../',
        bodyFormat      = 'multipart',
        fieldName       = 'file',
        responseUrlPath = 'attachments.0.url',
    },
}
```

**Self-hosted CDN expecting raw bytes (e.g. an S3 presigned PUT)**

```lua
Config.Photo = {
    provider = 'custom',
    custom = {
        url             = 'https://cdn.myserver.com/upload',
        headers         = { ['X-API-Key'] = 'secret' },
        bodyFormat      = 'binary',
        responseUrlPath = 'url',
    },
}
```

**FiveManage**

```lua
Config.Photo = {
    provider = 'fivemanage',
    fiveManage = { apiKey = 'fmk_...' },
}
```

**Local token-issued resource**

```lua
Config.Photo = {
    provider = 'local',
    ['local'] = { resource = 'my_upload_resource' },
}
```

The local provider expects the buyer-side resource to expose:

```lua
exports['<resource>']:GenerateUploadToken(citizenid, plate, ttlSeconds)
    -> token, base_url
```

The NUI then POSTs the captured image to `<base_url>/upload/image` with an `X-Upload-Token` header.

### Shared Keys

If your server tracks shared vehicle keys in its own SQL table (e.g. a phone "give my friend a key" feature), wire it up here so the garage shows shared vehicles to the recipient.

```lua
Config.SharedKeys = {
    enabled      = false,
    tableName    = '',
    plateColumn  = 'plate',
    targetColumn = 'target_citizenid',
}
```

| Field          | Description                                                                                  |
| -------------- | -------------------------------------------------------------------------------------------- |
| `enabled`      | Master switch. When `false`, no shared-key SQL is ever issued.                                |
| `tableName`    | Name of the shared-key table, e.g. `'phone_vehicle_keys'`.                                    |
| `plateColumn`  | Column holding the vehicle plate.                                                             |
| `targetColumn` | Column holding the recipient's citizenid (the player who has shared access).                  |

When enabled, shared vehicles surface in the recipient's garage menu (subject to the garage's vehicle type filter) and can be retrieved using the same flow as their own vehicles. On vehicle transfer (`/transfervehicle`), all shared key rows for that plate are wiped automatically.

### Vehicles by Name (ESX / Standalone fallback)

QBox and QB-Core expose a global `Vehicles` table keyed by model name. The garage uses that to read each model's brand, label, and category (helicopters / planes / boats). On ESX or pure standalone, no such table exists — populate `Config.VehiclesByName` manually if you want labels and AIR / SEA garage typing to work.

```lua
Config.VehiclesByName = {
    sultanrs = { brand = 'Karin', name = 'Sultan RS', category = 'sports' },
    maverick = { brand = 'Buckingham', name = 'Maverick', category = 'helicopters' },
    dinghy   = { brand = 'Nagasaki', name = 'Dinghy', category = 'boats' },
}
```

The categories that influence garage routing are `'helicopters'` and `'planes'` (treated as `VehicleType.AIR`) and `'boats'` (treated as `VehicleType.SEA`). Anything else is treated as `VehicleType.CAR`.

### Garages Table

The `Config.garages` table holds 47 entries spanning five categories: public, faction, gang, impound, and business. Add, remove, or re-coordinate entries at will — the resource auto-registers everything on start through `createGarages()`.

#### Schema

```lua
---@field label            string                       Display name shown in the menu and on the blip
---@field type             GarageType?                  Currently only DEPOT supported (omit for normal garage)
---@field vehicleType      VehicleType                  CAR | AIR | SEA | ALL
---@field groups           string|string[]?             Job / gang gating, e.g. 'police' or { 'police', 'sahp' }
---@field shared           boolean?                     Lets all members of the group see all vehicles
---@field states           VehicleState|VehicleState[]? Limits which states are retrievable
---@field skipGarageCheck  boolean?                     Returns vehicles regardless of which garage they're in
---@field canAccess        fun(source: number): boolean Custom access predicate
---@field accessPoints     AccessPoint[]                Coords + spawn + drop point + optional NPC + blip
```

`AccessPoint` schema:

```lua
{
    blip      = { name = 'Public Parking', sprite = 357, color = 3, coords = vec3?, scale = number? },
    coords    = vec4(x, y, z, h),    -- where the interaction zone is
    spawn     = vec4(x, y, z, h),    -- where the vehicle appears
    dropPoint = vec3(x, y, z),       -- optional second "store vehicle" zone
    npc       = { model = 's_m_m_paramedic_01', scenario = 'WORLD_HUMAN_CLIPBOARD' },
}
```

#### Enum reference

```lua
VehicleState = { OUT = 0, GARAGED = 1, IMPOUNDED = 2, SEIZED = 3 }
VehicleType  = { CAR = 'car', AIR = 'air', SEA = 'sea', ALL = 'all' }
GarageType   = { DEPOT = 'depot' }
```

#### Categories shipped

| Category   | Examples                                                       | Notes                                                                                     |
| ---------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Public     | `motelgarage`, `pillboxgarage`, `beachp`, `airportp`           | Default `vehicleType = CAR`. Anyone can park and retrieve their own vehicles.              |
| Public Air | `intairport`, `higginsheli`, `airsshores`                      | `vehicleType = AIR`.                                                                       |
| Public Sea | `lsymc`, `paleto`, `millars`                                   | `vehicleType = SEA`.                                                                       |
| Faction    | `police`, `policeair`, `sahp`, `sahpair`, `ambulance`, `sheriff` | Gated with `groups`. Includes helipads typed as AIR. EMS helipad demonstrates an `npc` block. |
| Gang       | `families`, `lostmc`, `cartel`                                 | Gated with `groups`.                                                                       |
| Impound    | `impoundlot`, `galeriimpound`, `airdepot`, `seadepot`           | `type = GarageType.DEPOT`, `skipGarageCheck = true`, accepts OUT / IMPOUNDED / SEIZED states. |
| Business   | `terminalgarage`, `taxigarage`, `pier76garage`, `altastreet`   | Several use `dropPoint` for a separate storage zone.                                       |

#### Full garage list

| Name                        | Label                       | Type    | Vehicle Type | Groups     | Notes                                  |
| --------------------------- | --------------------------- | ------- | ------------ | ---------- | -------------------------------------- |
| `motelgarage`               | Motel Parking               | normal  | CAR          | —          | Has `dropPoint`. Default for transfers. |
| `sapcounsel`                | San Andreas Parking         | normal  | CAR          | —          |                                        |
| `spanishave`                | Spanish Ave Parking         | normal  | CAR          | —          |                                        |
| `caears24`                  | Caears 24 Parking           | normal  | CAR          | —          |                                        |
| `littleseoul`               | Little Seoul Parking        | normal  | CAR          | —          |                                        |
| `lagunapi`                  | Laguna Parking              | normal  | CAR          | —          |                                        |
| `airportp`                  | Airport Parking             | normal  | CAR          | —          |                                        |
| `beachp`                    | Beach Parking               | normal  | CAR          | —          |                                        |
| `themotorhotel`             | Motor Hotel Parking         | normal  | CAR          | —          |                                        |
| `liqourparking`             | Liquor Store Parking        | normal  | CAR          | —          |                                        |
| `shoreparking`              | Shore Parking               | normal  | CAR          | —          |                                        |
| `haanparking`               | Bell Farms Parking          | normal  | CAR          | —          |                                        |
| `dumbogarage`               | Dumbo Private Parking       | normal  | CAR          | —          |                                        |
| `gallerygarage`             | Gallery Garage              | normal  | CAR          | —          |                                        |
| `pillboxgarage`             | Pillbox Garage Parking      | normal  | CAR          | —          |                                        |
| `iskelegarage`              | Pier Garage                 | normal  | CAR          | —          |                                        |
| `plaj1garage`               | Beach 1 Garage              | normal  | CAR          | —          |                                        |
| `bahamagarage`              | Bahama Garage               | normal  | CAR          | —          |                                        |
| `kuyumcugarage`             | Jeweller Side Garage        | normal  | CAR          | —          |                                        |
| `hayesgarage`               | Hayes Back Garage           | normal  | CAR          | —          |                                        |
| `otelgarage`                | Hotel Garage                | normal  | CAR          | —          |                                        |
| `uwugarage`                 | UWU Garage                  | normal  | CAR          | —          |                                        |
| `iskurgarage`               | Job Center Garage           | normal  | CAR          | —          |                                        |
| `pillboxgarage2`            | Pillbox Garage              | normal  | CAR          | —          |                                        |
| `pdgarage`                  | PD Side Garage              | normal  | CAR          | —          |                                        |
| `tacogarage`                | Taco Garage                 | normal  | CAR          | —          |                                        |
| `jbfclubgarage`             | JBF Club Garage             | normal  | CAR          | —          |                                        |
| `beanmachinegarage`         | Bean Machine Garage         | normal  | CAR          | —          |                                        |
| `megamallgarage`            | Megamall Garage             | normal  | CAR          | —          |                                        |
| `hastahanegarage`           | Hospital Garage             | normal  | CAR          | —          |                                        |
| `lspdgarage`                | LSPD Garage                 | normal  | CAR          | —          |                                        |
| `intairport`                | Airport Hangar              | normal  | AIR          | —          |                                        |
| `higginsheli`               | Higgins Helitours           | normal  | AIR          | —          |                                        |
| `airsshores`                | Sandy Shores Hangar         | normal  | AIR          | —          |                                        |
| `lsymc`                     | LSYMC Boat House            | normal  | SEA          | —          |                                        |
| `paleto`                    | Paleto Boat House           | normal  | SEA          | —          |                                        |
| `millars`                   | Millars Boat House          | normal  | SEA          | —          |                                        |
| `police`                    | Police Garage               | normal  | CAR          | police     | Two access points.                     |
| `policeair`                 | Police Helipad              | normal  | AIR          | police     |                                        |
| `sahp`                      | SAHP Garage                 | normal  | CAR          | sahp       |                                        |
| `sahpair`                   | SAHP Helipad                | normal  | AIR          | sahp       |                                        |
| `ambulance`                 | EMS Garage                  | normal  | CAR          | ambulance  |                                        |
| `ambulanceair`              | EMS Helipad                 | normal  | AIR          | ambulance  | Includes paramedic NPC.                |
| `sheriff`                   | Sheriff Garage              | normal  | CAR          | bcso       |                                        |
| `families`                  | La Familia                  | normal  | CAR          | families   |                                        |
| `lostmc`                    | Lost MC                     | normal  | CAR          | lostmc     |                                        |
| `cartel`                    | Cartel                      | normal  | CAR          | cartel     |                                        |
| `impoundlot`                | Impound Lot                 | DEPOT   | CAR          | —          | Accepts OUT / IMPOUNDED / SEIZED.       |
| `galeriimpound`             | Gallery Impound             | DEPOT   | CAR          | —          |                                        |
| `airdepot`                  | Air Depot                   | DEPOT   | AIR          | —          |                                        |
| `seadepot`                  | LSYMC Depot                 | DEPOT   | SEA          | —          |                                        |
| `terminalgarage`            | Terminal Garage             | normal  | CAR          | —          | `states = { IMPOUNDED, GARAGED }`. Has `dropPoint`. |
| `terminaldepot`             | Terminal Impound            | DEPOT   | CAR          | —          |                                        |
| `kasabadepot`               | Town Impound                | DEPOT   | CAR          | —          |                                        |
| `taxigarage`                | Taxi Stand Garage           | normal  | CAR          | —          |                                        |
| `shopownergarage`           | Business Centre Garage      | normal  | CAR          | —          |                                        |
| `arcadeusbusinesscentre`    | Business Centre Garage      | normal  | CAR          | —          |                                        |
| `pier76garage`              | Pier 76 Garage              | normal  | CAR          | —          | Has `dropPoint`.                        |
| `altastreet`                | Alta Street Parking         | normal  | CAR          | —          | Blip carries explicit `coords`.         |

---

## Commands

The resource registers a single player-facing command via `lib.addCommand`. Admin-grade actions (impound a vehicle, force-park, etc.) are exposed as exports rather than commands so they can be wired into your existing admin menu, MDT, or police job.

### `/transfervehicle [id]`

Transfers ownership of the vehicle the player is currently driving to another player.

| Parameter | Type      | Description                                                                                   |
| --------- | --------- | --------------------------------------------------------------------------------------------- |
| `id`      | playerId  | Optional. Target player server ID. If omitted, the closest player within 10 metres is picked. |

Behaviour:

1. Verifies the player is the driver of an owned, registered vehicle.
2. Resolves the target either from the explicit `id` argument or the nearest player.
3. Calls `Vehicles.SetPlayerVehicleOwner(vehicleId, targetIdentifier)` through the bridge.
4. Wipes any shared keys for the plate (since they were issued by the previous owner).
5. Resets `custom_name` and `favorite`, then routes the vehicle into `Config.defaultGarage` with state `GARAGED`.
6. Hands keys over via `community_bridge.VehicleKey` (no-op if no key resource is running).
7. Notifies both the source and target players.

Localised strings come from `locales/en.json` under the `success.transfer_sent`, `success.transfer_received`, and `error.*` keys.

---

## Customization

### UI Theme

Open `Config.UI` in `config.lua` and override the eleven hex colour values. The change applies on the next garage open — restart the resource and the new theme takes effect.

### Adding a Garage

Add a new entry to `Config.garages`. The minimum schema is a label, a vehicle type, and one access point with coords and spawn:

```lua
Config.garages.mygarage = {
    label = 'My Garage',
    vehicleType = VehicleType.CAR,
    accessPoints = {
        {
            blip   = { name = 'My Garage', sprite = 357, color = 3 },
            coords = vec4(123.4, 567.8, 30.0, 90.0),
            spawn  = vec4(125.6, 567.8, 30.0, 180.0),
        },
    },
}
```

You can also register garages at runtime from another resource via the `RegisterGarage` export — see the **Exports & Events** section.

### Removing a Garage

Delete the entry from `Config.garages` (or comment it out). All zones, blips, and NPCs spawned for it will be skipped on the next start.

### Gating with `groups` and `canAccess`

Restrict a garage to specific jobs or gangs:

```lua
Config.garages.fbi = {
    label = 'FBI Garage',
    vehicleType = VehicleType.CAR,
    groups = { 'fbi', 'doj' },
    accessPoints = { ... },
}
```

For complex predicates (rank check, ACE perms, on-duty status), use `canAccess`:

```lua
Config.garages.fbi_swat = {
    label = 'FBI SWAT Garage',
    vehicleType = VehicleType.CAR,
    groups = 'fbi',
    canAccess = function(source)
        return exports.my_dutytracker:IsOnDuty(source) and exports.qbx_core:HasGroupRank(source, 'fbi', 4)
    end,
    accessPoints = { ... },
}
```

Group membership is evaluated by `Player.HasPrimaryGroup(source, groups)` in `bridge/server/init.lua`, which checks job, gang, and the ACE permission `clads.garage.<group>` so you can grant individual access without adding a player to a job.

`shared = true` lets every member of the group see every member's vehicles parked at the garage — useful for pooled service vehicles.

### Depot / Impound Garages

Set `type = GarageType.DEPOT` to make a garage behave like an impound lot. Depots:

- Cannot accept "park here" actions — they are retrieve-only.
- Charge `Config.calculateImpoundFee(vehicleId, modelName)` if `depot_price > 0`.
- Pull vehicles regardless of which garage they are in (because of `skipGarageCheck = true`).
- Show vehicles in `OUT`, `IMPOUNDED`, and `SEIZED` states (configurable via the `states` field).

```lua
Config.garages.impound = {
    label = 'Impound Lot',
    type = GarageType.DEPOT,
    states = { VehicleState.OUT, VehicleState.IMPOUNDED, VehicleState.SEIZED },
    skipGarageCheck = true,
    vehicleType = VehicleType.CAR,
    accessPoints = {
        { blip = { name = 'Impound Lot', sprite = 225, color = 1 }, coords = ..., spawn = ... },
    },
}
```

### Multiple Spawn Slots / Access Points

A garage can have any number of `accessPoints`, each with its own coords, spawn, drop point, blip, and NPC. The shipping `police` garage uses two access points:

```lua
police = {
    label = 'Police Garage',
    vehicleType = VehicleType.CAR,
    groups = 'police',
    accessPoints = {
        { coords = vec4(454.6, -1017.4, 28.4, 0), spawn = vec4(438.4, -1018.3, 27.7, 90.0) },
        { coords = vec4(844.16, -1340.99, 26.09, 0), spawn = vec4(445.88, -991.21, 25.7, 269.14) },
    },
},
```

Each access point automatically gets its own zone and (if defined) blip and NPC.

### Custom NPC per Access Point

Add an `npc` block to spawn a ped at the access point. The ped is invincible, frozen, and runs the requested scenario:

```lua
{
    coords = vec4(1171.82, -1571.85, 39.4, 0.0),
    spawn  = vec4(1174.64, -1562.33, 39.4, 0.0),
    npc    = { model = 's_m_m_paramedic_01', scenario = 'WORLD_HUMAN_CLIPBOARD' },
}
```

NPCs are tracked and removed automatically on resource stop.

### Drop Points

Some access points use a `dropPoint` so the player can store a vehicle at a separate location from the showroom interaction zone (for example, a kerb-side parking spot near the storefront). When `dropPoint` is set, an additional `_drop_` zone is created with a "Store Vehicle" target option.

```lua
{
    coords    = vec4(971.47, -2548.49, 28.3, 0.0),
    spawn     = vec4(981.44, -2547.69, 27.83, 354.34),
    dropPoint = vec3(976.35, -2547.44, 28.0),
}
```

Drop points are skipped for depot garages.

### Blip Styling

Each access point can carry an optional `blip` block:

```lua
blip = {
    name   = 'Public Parking',
    sprite = 357,
    color  = 3,
    coords = vec3(?, ?, ?), -- optional override; otherwise the access-point coords are used
    scale  = 0.8,           -- optional
}
```

Omit `blip` entirely to skip the map marker. Faction garages typically omit blips so undercover units don't reveal their HQ.

### Mechanic Integration Setup

Wire your mechanic resource into `Config.Mechanic`:

```lua
Config.Mechanic = {
    enabled              = true,
    resource             = 'my_mechanic',
    propsExport          = 'getVehicleProperties',
    statebagExport       = 'getVehicleStatebagProperties',
    statebagSaveCallback = 'my_mechanic:server:saveStatebag',
    statebagApplyEvent   = 'my_mechanic:server:applyStatebag',
    stanceTable          = 'mechanic_vehicledata',
}
```

Then implement the corresponding callback and event in your mechanic resource:

```lua
lib.callback.register('my_mechanic:server:saveStatebag', function(source, plate, data)
    MySQL.update.await('UPDATE mechanic_vehicledata SET data = ? WHERE plate = ?', { json.encode(data), plate })
    return true
end)

RegisterNetEvent('my_mechanic:server:applyStatebag', function(vehicle, plate)
    local row = MySQL.single.await('SELECT data FROM mechanic_vehicledata WHERE plate = ?', { plate })
    if not row then return end
    local data = json.decode(row.data)
    -- Re-apply nitrous, stance, tuning, etc.
end)
```

When `stanceTable` is set, the spawn flow runs the following query before applying props:

```sql
SELECT data FROM <stanceTable> WHERE TRIM(plate) = TRIM(?)
```

If the row's JSON contains `enableStance = true`, the spawner clones the props and clears `modSuspension` so the mechanic resource can reapply the correct stance values from its statebag.

---

## Exports & Events

### Server Exports

| Export                                                           | Returns               | Description                                                                                                       |
| ---------------------------------------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `exports.clads_garages:GetGarages()`                             | `table<string, GarageConfig>` | Returns the live garage registry.                                                                                  |
| `exports.clads_garages:RegisterGarage(name, config)`             | —                     | Register a new garage at runtime. Triggers `clads_garages:server:garageRegistered` and broadcasts to all clients. |
| `exports.clads_garages:SetVehicleGarage(vehicleId, garageName)`  | `boolean, ErrorResult?` | Move a vehicle into a specific garage. Sets state to `IMPOUNDED` for depots, `GARAGED` otherwise.                |
| `exports.clads_garages:SetVehicleDepotPrice(vehicleId, depotPrice)` | `boolean, ErrorResult?` | Override the impound fee for a specific vehicle.                                                                  |

### Server Events

| Event                                          | Payload                       | Direction        | Notes                                                                       |
| ---------------------------------------------- | ----------------------------- | ---------------- | --------------------------------------------------------------------------- |
| `clads_garages:server:garageRegistered`        | `(name, config)`              | Server-internal  | Fired when `RegisterGarage` is called.                                      |
| `clads_garages:server:vehicleSpawned`          | `(vehicle)`                   | Server-internal  | Fired after a successful spawn from a garage.                               |

### Server Callbacks (`lib.callback.register`)

These are consumed by the bundled NUI but can be invoked from any resource that uses `ox_lib`.

| Callback                                              | Args                                              | Returns                                            |
| ----------------------------------------------------- | ------------------------------------------------- | -------------------------------------------------- |
| `clads_garages:server:getGarages`                     | —                                                 | `table<string, GarageConfig>`                      |
| `clads_garages:server:getGarageVehicles`              | `garageName`                                      | `PlayerVehicle[]?`                                 |
| `clads_garages:server:isParkable`                     | `garage, netId`                                   | `boolean`                                          |
| `clads_garages:server:parkVehicle`                    | `netId, props, garage, propsHash`                 | `needsCapture, vehicleId, photoSpec`               |
| `clads_garages:server:spawnVehicle`                   | `vehicleId, garageName, accessPointIndex`         | `netId?`                                           |
| `clads_garages:server:saveVehiclePhoto`               | `vehicleId, imageUrl, visualHash`                 | `boolean`                                          |
| `clads_garages:server:setVehicleCustomName`           | `vehicleId, customName`                           | `boolean`                                          |
| `clads_garages:server:toggleVehicleFavorite`          | `vehicleId`                                       | `boolean, boolean newFavoriteState`                |

### Client Net Events

| Event                                              | Payload                       | Direction          | Notes                                                                |
| -------------------------------------------------- | ----------------------------- | ------------------ | -------------------------------------------------------------------- |
| `clads_garages:client:garageRegistered`            | `(name, config)`              | Server → Client    | Triggered when a new garage is registered at runtime.                 |
| `clads_garages:client:warpIntoVehicle`             | `(netId)`                     | Server → Client    | Warps the player into the seat after a successful spawn.              |
| `clads_garages:client:setVehicleFuel`              | `(netId, fuelLevel)`          | Server → Client    | Restores fuel using the configured fuel resource via `community_bridge`. |
| `clads_garages:client:applyVehicleProperties`      | `(netId, props)`              | Server → Client    | Re-applies wheels, mods, extras, livery, and tint client-side.       |

---

## Troubleshooting

### Vehicles not appearing in the garage menu

- **Wrong garage.** The vehicle is parked at a different garage. With `Config.remoteRetrieveFee > 0`, it should still appear with a fee tag — set the fee to a non-zero value if it does not.
- **Wrong vehicle type.** A boat won't show in a CAR garage. Check `Config.garages.<name>.vehicleType` against the vehicle's class.
- **Vehicle is in the wrong state.** Each garage's `states` field whitelists which `VehicleState` values are returned. Default is `GARAGED`. Depots default to `OUT / IMPOUNDED / SEIZED`.
- **Vehicle is currently spawned.** The query in `getGarageVehicles` skips any vehicle whose plate is currently active on the server (`FindPlateOnServer`). Despawn the entity or wait for the auto-respawn on resource start.
- **Group gating.** If the garage has `groups` set and the player isn't in that job/gang/ACE permission, the menu returns nothing.

### Impound not charging

- **Depot price is zero.** `Config.calculateImpoundFee` is called when a vehicle moves into the `OUT` state. If the vehicle was impounded before the resource was started or before the callback was modified, its `depot_price` may be `0`. Set it explicitly via `exports.clads_garages:SetVehicleDepotPrice(vehicleId, fee)`.
- **Wrong garage type.** Only garages with `type = GarageType.DEPOT` charge the depot fee. Normal garages charge only the cross-garage retrieval fee.
- **Player is broke.** `Player.PayFee` tries cash first, then bank. If neither covers the fee, retrieval is blocked with the `error.not_enough` notification.

### Faction garage accessible to civilians

- **`groups` not set.** Verify the garage entry in `Config.garages` includes `groups = '<jobname>'` (or an array of names). Without it, anyone can open the garage.
- **ACE override.** `Player.HasPrimaryGroup` also checks `clads.garage.<groupname>` ACE perms. Run `add_ace` lines might be granting unintended access.
- **Bridge framework mismatch.** If `Config.Framework` is forced to the wrong value, group lookups can return false positives. Use `'auto'` unless you have a specific reason.

### Drop point not working

- **Drop point set on a depot.** Drop points are skipped for `GarageType.DEPOT` garages — depots are retrieve-only.
- **`dropPoint` not a `vec3` / `vec4`.** The zone creator accepts both, but plain tables don't work. Use `vec3(x, y, z)` or `vec4(...)`.
- **Player not the driver.** The drop-point zone has `onlyInVehicle = true, onlyDriver = true`. Passengers cannot store the vehicle.
- **Plate not registered.** The plate must resolve to a row via `Vehicles.GetVehicleIdByPlate`. Vehicles created with admin commands or vehicle shops without a DB row will fail the parkable check.

### Photos disabled but expected

- **Provider stuck on `'disabled'`.** Double-check `Config.Photo.provider`. The default ships disabled.
- **Empty credentials.** When `provider = 'fivemanage'` with an empty `apiKey`, the photo flow logs a warning and returns `needsCapture = false`. Same for `provider = 'custom'` with no `url`.
- **Local resource not running.** With `provider = 'local'`, the named resource must be `started`; otherwise the bridge logs a warning and skips capture.
- **`visual_hash` already matches.** Capture is skipped when the stored `visual_hash` matches the current props hash. Modify the vehicle's visuals to trigger a re-capture.

### Mechanic integration not loading stance

- **`enabled = false`.** Set `Config.Mechanic.enabled = true` and verify the resource name matches what's running.
- **Resource not started.** All Mechanic helpers no-op if `GetResourceState(cfg.resource) ~= 'started'`.
- **Wrong export name.** `propsExport` and `statebagExport` must match exactly (case-sensitive).
- **`statebagApplyEvent` missing.** Without it, statebags are saved but never re-applied on spawn.
- **Stance table column mismatch.** The query is `SELECT data FROM <stanceTable> WHERE TRIM(plate) = TRIM(?)`. The `data` column must hold a JSON object with `enableStance` for the suspension override to fire.

### Shared keys not surfacing

- **`Config.SharedKeys.enabled = false`.** Toggle it on.
- **Wrong table or column names.** Verify `tableName`, `plateColumn`, and `targetColumn`. The query `SELECT <plateColumn> AS plate FROM <tableName> WHERE <targetColumn> = ?` must run cleanly.
- **Plate whitespace.** The check normalises plates with `REPLACE(<col>, ' ', '')`. If your phone resource stores untrimmed plates, this is handled — but custom column types (e.g. `CHAR(8)` with padding) may need a manual `RPAD` adjustment in `Integrations.HasSharedKey`.
- **Target citizenid mismatch.** Some servers store the target as a player ID rather than a citizenid. Make sure the column you point at really holds the recipient's citizenid (matching `Framework.GetPlayerIdentifier(source)`).

---

## Support

Reach out via the Creative Lads Discord.
