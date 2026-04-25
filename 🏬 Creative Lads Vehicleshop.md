# Creative Lads — Vehicle Shop

`clads_vehicleshop` is a framework-agnostic dealership system for FiveM. It ships with four prebuilt shops out of the box (cars, motorcycles, aircraft, and boats), a cinematic showroom NUI, an optional client-side green-screen vehicle photo capture pipeline with four upload providers, and a community_bridge layer that auto-detects ESX, QB-Core, qbx_core, or runs standalone.

---

## 1. Overview

`clads_vehicleshop` is a multi-shop vehicle dealership designed to slot into any RP server without forcing you to commit to a single framework or target system.

### Feature highlights

- **Multi-shop, multi-type architecture.** Define as many dealerships as you want. Each entry in `Config.VehicleShops` declares its own `type` (`car`, `motorcycle`, `air`, `sea`), location, vehicle inventory, NPC, blip, and target zone.
- **Cinematic showroom.** Selecting a shop teleports the player into a private showroom view: the chosen vehicle spawns, a script camera frames it, the player ped is hidden, and the NUI panel opens. The player can rotate the vehicle, change its colour, and either purchase or take a test drive.
- **Per-shop vehicle pools.** `config.vehicles.lua` defines a flat pool of every available vehicle, then auto-splits it into four pools (`Vehicles[1]`–`Vehicles[4]`) — one per default shop type. Bikes, quads, and bicycles are moved into the bike pool automatically; aircraft and boat defaults are layered on top of pools 3 and 4.
- **Per-shop blips.** Every shop registers its own short-range blip with configurable sprite, scale, colour, display mode, and locale-resolved name.
- **Test drive.** Optional per-shop, with its own price and per-shop coordinate / time / max-distance override. A timer text UI counts down. If the player drifts beyond the test-drive radius they are teleported back automatically.
- **Optional vehicle photo capture.** A four-mode upload pipeline (`disabled`, `fivemanage`, `custom`, `local`) captures green-screen thumbnails for every model, caches the URL keyed by model in MySQL, and serves them to the showroom NUI. Custom mode supports imgbb, imgur, Discord webhooks, S3 presigned URLs, and any generic JSON / multipart / binary upload endpoint.
- **Routing-bucket isolation.** Both the showroom view and (optionally) the test drive can isolate the player into their own routing bucket so other players cannot see or collide with the showroom vehicle.
- **Framework-agnostic.** All money, persistence, job, and key-grant calls go through `community_bridge`. Vehicle persistence is delegated to one of four adapters loaded at runtime: `qbx_core`, `qb-core`, `es_extended`, or a standalone fallback.
- **Personal-vehicle hook.** On QB-Core and qbx_core servers, newly purchased vehicles are written to a configurable metadata key. If the player's previous personal vehicle is still on the map, the new one is parked into the default garage instead of being delivered.
- **Localisation.** Built-in English, German, Spanish, French, and Turkish. Locale auto-detects from `clads_locale`, `ox:locale`, `qb_locale`, or `lang` convars.
- **Themable UI.** A single `Config.UI` table re-skins the panel by injecting hex colours into CSS variables at runtime — no rebuild required.

---

## 2. Requirements

### Hard dependencies

| Resource           | Purpose                                                              |
| ------------------ | -------------------------------------------------------------------- |
| `community_bridge` | Framework, target, notify, and vehicle-key abstraction.              |
| `ox_lib`           | Callbacks, notifications, vehicle properties, model streaming, NUI helpers. |

Both are declared in `fxmanifest.lua` under `dependencies` and must be started before `clads_vehicleshop`.

### Soft / runtime requirements

| Resource             | Required when                                                                |
| -------------------- | ---------------------------------------------------------------------------- |
| `oxmysql`            | Always — provides the global `MySQL` table used by the photo cache and persistence adapters. |
| Framework            | One of `es_extended`, `qb-core`, or `qbx_core`. Auto-detected via `community_bridge`. Standalone is supported with no money checks. |
| Target backend       | One of `ox_target`, `qb-target`, or `sleepless_interact`. Auto-detected. Optional — set `Config.UseTarget = false` to fall back to a marker + E key prompt. |
| Vehicle keys backend | One of `qbx_vehiclekeys`, `qb-vehiclekeys`, `qs-vehiclekeys`, `mrnewbvehiclekeys`, or `wasabi_carlock`. Optional. |
| MySQL                | Required for the stored photo cache (`clads_vehicleshop_photos`) and for the standalone vehicle table when no framework is present. |

---

## 3. Installation

1. **Drop the resource into your `resources/` folder** (or any nested category folder).

2. **Ensure load order.** In `server.cfg`, start dependencies first:

   ```cfg
   ensure oxmysql
   ensure ox_lib
   ensure community_bridge
   ensure clads_vehicleshop
   ```

3. **Run the SQL migration.** Import `sql/install.sql` into your database. This creates the `clads_vehicleshop_photos` table that caches the captured thumbnail URL per model:

   ```sql
   CREATE TABLE IF NOT EXISTS `clads_vehicleshop_photos` (
       `model`       VARCHAR(50)  NOT NULL,
       `image_url`   VARCHAR(500) NOT NULL,
       `captured_by` VARCHAR(64)  DEFAULT NULL,
       `captured_at` TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
       PRIMARY KEY (`model`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
   ```

   The resource also calls `CREATE TABLE IF NOT EXISTS` on startup (server/main.lua), so the SQL import is technically optional. Running it once cleanly via your DB tool is recommended for predictable migrations.

   On **standalone** (no framework), the standalone adapter additionally creates `clads_vehicleshop_vehicles` on first start:

   ```sql
   CREATE TABLE IF NOT EXISTS `clads_vehicleshop_vehicles` (
       `id`         INT UNSIGNED NOT NULL AUTO_INCREMENT,
       `owner`      VARCHAR(64) NOT NULL,
       `model`      VARCHAR(50) NOT NULL,
       `plate`      VARCHAR(16) NOT NULL,
       `props`      LONGTEXT NULL,
       `garage`     VARCHAR(64) DEFAULT NULL,
       `state`      TINYINT NOT NULL DEFAULT 0,
       `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       PRIMARY KEY (`id`),
       UNIQUE KEY `uniq_plate` (`plate`),
       KEY `idx_owner` (`owner`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
   ```

   On QB-Core / qbx_core / ESX, the adapter writes into the framework's existing `player_vehicles` (QB / QBox) or `owned_vehicles` (ESX) table.

4. **Configure photo capture (optional).** If you want vehicle thumbnails in the showroom, pick one of the four providers in `Config.Photo.provider`:

   - **`disabled`** — default. Showroom uses placeholders. No setup needed.
   - **`fivemanage`** — paste your fivemanage API key into `Config.Photo.fiveManage.apiKey`.
   - **`custom`** — fill in `Config.Photo.custom.url` plus optional headers, body format, and response URL dot-path. Works with imgbb, imgur, Discord webhooks, S3 presigned URLs, and any generic upload endpoint. Examples are inlined in `config.lua`.
   - **`local`** — provide the resource name of a buyer-side resource that exposes a `:GenerateUploadToken(citizenid, key, ttl)` export that returns `(token, base_url)`.

   Once a provider is configured, an admin runs `/clads_vehicleshop_capture` once on a connected character to walk every shop's vehicle pool and capture each missing thumbnail. The URL is cached per model in `clads_vehicleshop_photos`; subsequent server restarts reuse the cached entries.

5. **Grant the capture command to your admins.** Add the ACE permission to your `permissions.cfg`:

   ```cfg
   add_ace group.admin command.clads_vehicleshop_capture allow
   add_ace group.admin clads.vehicleshop.capture allow
   ```

   The command itself is registered with `'group.admin'` as the permission group. The runtime `captureGroupGate` check (defense-in-depth on the photo callbacks) accepts either a `community_bridge` group / job / gang match, or the `clads.vehicleshop.<group>` ACE permission.

6. **Restart the server** and confirm the resource starts cleanly. With `Config.Debug = true` you should see lines such as `[clads_vehicleshop] server bridge ready (framework: qbx)`.

---

## 4. Configuration

All configuration lives in `config.lua` (general settings) and `config.vehicles.lua` (the flat vehicle pool plus auto-split per shop). Both are escrow-ignored and remain editable after Tebex / CFX encryption.

### Framework

```lua
Config.Framework = 'auto'
```

| Value         | Behaviour                                                                 |
| ------------- | ------------------------------------------------------------------------- |
| `'auto'`      | Let `community_bridge` detect the running core (recommended).             |
| `'esx'`       | Force `es_extended`.                                                      |
| `'qb'`        | Force `qb-core`.                                                          |
| `'qbox'`      | Force `qbx_core`.                                                         |
| `'standalone'`| No framework. Every purchase is free, no money checks. Persistence uses the standalone adapter and the `clads_vehicleshop_vehicles` table. |

### Locale

```lua
Config.Locale = 'auto'
```

Built-in locales: `en`, `tr`, `de`, `fr`, `es`. When set to `'auto'`, the loader walks the following convars and uses the first non-empty value: `clads_locale`, `ox:locale`, `qb_locale`, `lang`. Falls back to `en`.

To add a new language, drop a JSON file in `locales/<code>.json` mirroring the keys in `locales/en.json`.

### Debug

```lua
Config.Debug = false
```

Enables a handful of diagnostic prints from the bridge layers, the locale loader, and the capture pipeline.

### UI Theme

```lua
Config.UI = {
    primary       = '#FF003C',
    primaryLight  = '#FF3D63',
    primaryDark   = '#CC0030',
    action        = '#FF003C',
    actionDark    = '#CC0030',
    secondary     = '#888888',
    success       = '#22C55E',
    warning       = '#F59E0B',
    danger        = '#FF003C',
    bgCore        = '#0D0D0D',
    bgSurface     = '#1A1A1A',
    bgElevated    = '#2E2E2E',
}
```

The defaults shipped with the resource are the **Dark Red** palette. Every value is a 6-digit hex colour (with leading `#`). The web app injects these into `:root` at runtime, so accents, glows, and gradients all follow without rebuilding the NUI.

| Key            | Used for                                                                |
| -------------- | ----------------------------------------------------------------------- |
| `primary`      | Main brand colour: buttons, active states, glow.                        |
| `primaryLight` | Lighter shade for hover and gradient end.                               |
| `primaryDark`  | Darker shade for gradient start and depth.                              |
| `action`       | "Buy" CTA and price colour.                                             |
| `actionDark`   | Hover state of the action button.                                       |
| `secondary`    | Test-drive button and info accent.                                      |
| `success`      | Success notifications.                                                  |
| `warning`      | Warning notifications.                                                  |
| `danger`       | Close button and error states.                                          |
| `bgCore`       | Darkest background colour.                                              |
| `bgSurface`    | Panel background under the glass blur.                                  |
| `bgElevated`   | Cards and elevated surfaces.                                            |

Tip: keep the ratio between `primary`, `primaryLight`, and `primaryDark` consistent (all blues, all reds, etc.) so highlights stay readable.

### Target

```lua
Config.TargetBackend = 'auto'
Config.UseTarget     = true
```

`Config.TargetBackend` lets `community_bridge` pick between `ox_target`, `qb-target`, or `sleepless_interact`. Force a specific backend by setting it explicitly.

`Config.UseTarget = false` disables target entirely. Players then approach the shop on the world marker and press E (configurable via `Config.KeyOpen`).

### Payment accounts

```lua
Config.PaymentAccounts = { 'cash', 'bank' }
```

Order in which payment accounts are tried. The first account with enough balance is charged. On standalone all purchases are free regardless of this setting.

### Vehicle keys

```lua
Config.GiveKeys = true
```

When `true`, attempts to grant vehicle keys via `community_bridge` after a purchase or test drive starts. Auto-detects `qbx_vehiclekeys`, `qb-vehiclekeys`, `qs-vehiclekeys`, `mrnewbvehiclekeys`, or `wasabi_carlock`.

### Plate format

```lua
Config.PlateLetters      = 3
Config.PlateNumbers      = 4
Config.PlateCustomPrefix = nil
```

Generates plates of the form `LLL NNNN`. Set `PlateCustomPrefix = "ABC"` to force a fixed letter prefix. The server-side `clads_vehicleshop:server:isPlateTaken` callback validates uniqueness against the active framework's vehicle table.

### Showroom behaviour (global defaults)

Every flag below is overridable per shop where it makes sense.

```lua
Config.UseFadeWithSpawn             = true
Config.SoundsEffects                = true
Config.UseVehicleColorsRGB          = true
Config.UseRoutingBucketsInShowRoom  = true
Config.UseRoutingBucketsOnTestDrive = true
```

| Flag                                | Effect                                                                                       |
| ----------------------------------- | -------------------------------------------------------------------------------------------- |
| `UseFadeWithSpawn`                  | Fades the screen out and back in around the buy delivery for a clean transition.             |
| `SoundsEffects`                     | Plays GTA HUD click / select / confirm sounds for navigation feedback.                       |
| `UseVehicleColorsRGB`               | When `true`, uses arbitrary RGB primary / secondary colours; when `false`, uses GTA palette indices. |
| `UseRoutingBucketsInShowRoom`       | Isolates the player into routing bucket equal to their src ID while inside the showroom.     |
| `UseRoutingBucketsOnTestDrive`      | When `false`, restores the player to bucket 0 for the test drive (so traffic and other players are visible). |

### Marker fallback

Used only when `Config.UseTarget = false`.

```lua
Config.AccessOnMarker     = false
Config.DistanceViewMarker = 20.0
Config.DistanceAccess     = 2.0
Config.KeyOpen            = 38   -- 38 = E
```

| Field                | Effect                                                                       |
| -------------------- | ---------------------------------------------------------------------------- |
| `AccessOnMarker`     | Master toggle for the marker thread. Must be `true` for the marker to draw.  |
| `DistanceViewMarker` | Distance from the shop coords at which the marker is drawn.                  |
| `DistanceAccess`     | Distance within which the E prompt opens the shop.                           |
| `KeyOpen`            | Control ID for the open key (38 = E).                                        |

### Test drive defaults

```lua
Config.TestDrive = {
    displayTimer = true,
    time         = 60,                                       -- seconds
    coords       = vector4(-1267.47, -3374.01, 12.94, 327.4),
    maxDistance  = 500.0,
}
```

| Field          | Type    | Description                                                                 |
| -------------- | ------- | --------------------------------------------------------------------------- |
| `displayTimer` | boolean | Show the countdown text UI in the top-centre during the test drive.         |
| `time`         | number  | Test-drive duration in seconds.                                             |
| `coords`       | vector4 | Spawn location and heading for the test drive vehicle.                      |
| `maxDistance`  | number  | If the player drifts more than this many units from the spawn, they are teleported back. |

Each shop can override the defaults via `shop.testDriveOverride = { time = ..., coords = ..., maxDistance = ... }`. The override falls back to `Config.TestDrive` field-by-field for any value it does not specify.

### Photo capture

The vehicle thumbnail flow is captured client-side via a green-screen scene (`stream/jim_g_green_screen_v1.*`) and uploaded to the configured backend. Captures happen once per model and the URL is cached in `clads_vehicleshop_photos`.

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

    autoCapture  = true,
    captureGroup = 'admin',
}
```

#### Provider modes

| Provider       | Description                                                                 |
| -------------- | --------------------------------------------------------------------------- |
| `'disabled'`   | No photos. Showroom uses placeholders. Default.                             |
| `'fivemanage'` | fivemanage.com convenience preset. Set `fiveManage.apiKey`.                 |
| `'custom'`     | Generic HTTP endpoint. Works with imgbb / imgur / Discord webhooks / S3 presigned / self-hosted CDNs. |
| `'local'`      | Buyer-side FiveM resource exposing `:GenerateUploadToken(citizenid, key, ttl)` returning `token, base_url`. Use this when uploads must be authenticated through your own server (signed tokens, audit log, etc.). |

#### `fivemanage`

The simplest paid SaaS option. The bridge expands the API key into a generic custom spec internally:

```lua
Config.Photo = {
    provider   = 'fivemanage',
    fiveManage = { apiKey = 'fmt_XXXXXXXXXXXXX' },
}
```

Internally this resolves to:

- `uploadUrl = 'https://api.fivemanage.com/api/image'`
- `headers   = { Authorization = '<your apiKey>' }`
- `bodyFormat = 'multipart'`
- `fieldName  = 'image'`
- `responseUrlPath = 'url'`

#### `custom` — generic HTTP endpoint

```lua
Config.Photo = {
    provider = 'custom',
    custom   = {
        url             = '',         -- full upload endpoint URL
        headers         = {},         -- arbitrary request headers
        bodyFormat      = 'multipart', -- 'multipart' | 'json-base64' | 'binary'
        fieldName       = 'image',    -- multipart field name
        responseUrlPath = 'url',      -- dot-path into the JSON response
    },
}
```

| Field             | Purpose                                                                  |
| ----------------- | ------------------------------------------------------------------------ |
| `url`             | Full upload endpoint URL. Required.                                       |
| `headers`         | Arbitrary request headers (e.g. `Authorization`, `X-API-Key`).           |
| `bodyFormat`      | `multipart`, `json-base64`, or `binary`.                                 |
| `fieldName`       | Multipart field name (only used when `bodyFormat = 'multipart'`).        |
| `responseUrlPath` | Dot-path into the JSON response that contains the uploaded image URL. Use indices for arrays, e.g. `attachments.0.url`. |

##### imgbb.com

```lua
custom = {
    url             = 'https://api.imgbb.com/1/upload?key=YOUR_API_KEY',
    bodyFormat      = 'multipart',
    fieldName       = 'image',
    responseUrlPath = 'data.url',
}
```

##### imgur (anonymous)

```lua
custom = {
    url             = 'https://api.imgur.com/3/image',
    headers         = { ['Authorization'] = 'Client-ID YOUR_CLIENT_ID' },
    bodyFormat      = 'multipart',
    fieldName       = 'image',
    responseUrlPath = 'data.link',
}
```

##### Discord webhook

```lua
custom = {
    url             = 'https://discord.com/api/webhooks/.../',
    bodyFormat      = 'multipart',
    fieldName       = 'file',
    responseUrlPath = 'attachments.0.url',
}
```

##### Self-hosted CDN (e.g. S3 presigned URL)

```lua
custom = {
    url             = 'https://cdn.myserver.com/upload',
    headers         = { ['X-API-Key'] = 'secret' },
    bodyFormat      = 'binary',
    responseUrlPath = 'url',
}
```

#### `local` — buyer-side resource

Use when uploads must go through your own infrastructure with signed, short-lived tokens.

```lua
Config.Photo = {
    provider = 'local',
    ['local'] = { resource = 'my_upload_resource' },
}
```

Your resource must export `GenerateUploadToken(citizenid, key, ttlSeconds)` returning two values: `token, base_url`. The NUI sends the captured payload to `base_url` with the token attached.

#### Common toggles

| Field          | Description                                                                |
| -------------- | -------------------------------------------------------------------------- |
| `autoCapture`  | Reserved for future use. Capture is currently admin-driven via the bulk-capture command. |
| `captureGroup` | Group / job / ACE name required to drive the capture command and the upload callbacks. Default `'admin'`. |

### Personal vehicle (QB / QBox metadata)

```lua
Config.PersonalVehicle = {
    enabled       = true,
    metadataKey   = 'personal_vehicle',
    defaultGarage = 'pillboxgarage',
}
```

| Field           | Effect                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------- |
| `enabled`       | Toggle the entire personal-vehicle bookkeeping. ESX and standalone ignore this regardless.        |
| `metadataKey`   | Player metadata key written with the new vehicle's plate after each purchase.                     |
| `defaultGarage` | Garage name used in the `player_vehicles.garage` column. Also used when a previous personal vehicle is still in the world — the new vehicle is parked here. |

When the player buys a vehicle and a previous personal vehicle is still present in the world, the new vehicle is deleted client-side and routed into the garage instead. The new plate is written to the metadata regardless.

### Vehicle in showroom hook

```lua
Config.VehicleInShowRoom = function(vehicle)
    SetVehicleDirtLevel(vehicle, 0.0)
    SetVehicleNumberPlateText(vehicle, 'CREATIVE')
    SetVehicleNeedsToBeHotwired(vehicle, false)
end
```

Runs against every vehicle the moment it spawns in the showroom. Use this to set demo plates, lock doors, force fuel level, set extras, apply mods, or run any other native against the showroom display copy.

### `Config.VehicleShops` table

Each entry is one dealership. Mix car, motorcycle, aircraft, and boat shops freely.

#### Required fields

| Field        | Type           | Description                                                                                  |
| ------------ | -------------- | -------------------------------------------------------------------------------------------- |
| `type`       | string         | `'car'`, `'motorcycle'`, `'air'`, or `'sea'`. Used by helpers and test-drive defaults.       |
| `nameKey`    | string         | Locale key for the shop title. Resolved via `Locale.resolve` so plain literals also work.    |
| `icon`       | string         | Font Awesome icon class for the target / NUI title.                                          |
| `categories` | string[]       | Vehicle categories shown in the NUI category bar.                                            |
| `coords`     | vector3        | Where the player approaches the shop (target zone / marker / blip).                          |
| `carCoords`  | vector4        | Where the showroom display vehicle spawns.                                                   |
| `camCoord`   | vector3        | Where the cinematic camera sits.                                                             |
| `buyCoords`  | vector4        | Where the purchased vehicle is delivered.                                                    |
| `blip`       | table          | `{ sprite, scale, color, display, nameKey? }`. `nameKey` is a locale key; falls back to the shop name. |
| `vehicles`   | table          | Map of `model -> entry`. Reference a pool from `config.vehicles.lua` (`Vehicles[N]`) or define inline. |

#### Optional fields

| Field               | Type               | Description                                                                              |
| ------------------- | ------------------ | ---------------------------------------------------------------------------------------- |
| `testDrive`         | boolean            | Show the test-drive button.                                                              |
| `testDrivePrice`    | number             | `0` = free.                                                                              |
| `testDriveOverride` | table              | Per-shop test-drive override: `{ time, coords, maxDistance, displayTimer? }`.            |
| `targetRotation`    | number             | `ox_target` box rotation around Z.                                                       |
| `drawable`          | table              | `{ marker = bool, ['3dtext'] = bool }`. Used by the marker fallback.                     |
| `npc`               | table              | `{ enabled, model, coords, heading, scenario }`. Spawns a frozen, invincible dealer ped. |
| `requiredJob`       | string \| string[] | Job name(s) gating access.                                                               |
| `requiredJobGrade`  | table              | Minimum grade name.                                                                      |

#### Example shop entry

```lua
[1] = {
    type           = 'car',
    nameKey        = 'shop.pdm',
    icon           = 'fa-solid fa-car',
    categories     = { 'compact', 'classic', 'muscle', 'coupe', 'sedan', 'sport', 'supercar', 'offroad', 'suv', 'pickup' },
    testDrive      = true,
    testDrivePrice = 0,
    coords         = vector3(-33.37, -1103.75, 25.35),
    carCoords      = vector4(-69.79, -824.51, 221.0, 61.72),
    camCoord       = vector3(-74.72, -824.49, 223.15),
    buyCoords      = vector4(-45.31, -1081.42, 26.69, 67.34),
    targetRotation = -20.0,
    blip           = { sprite = 225, scale = 0.65, color = 27, display = 4, nameKey = 'blip.pdm' },
    drawable       = { marker = true, ['3dtext'] = true },
    npc = {
        enabled  = true,
        model    = 'a_m_y_business_03',
        coords   = vector3(-33.52, -1103.74, 26.42),
        heading  = 66.62,
        scenario = 'WORLD_HUMAN_STAND_IMPATIENT',
    },
    vehicles = Vehicles and Vehicles[1] or nil,
}
```

#### Default shipping shops

| ID  | Name                          | Type         | Test drive  | Notes                                              |
| --- | ----------------------------- | ------------ | ----------- | -------------------------------------------------- |
| 1   | Premium Deluxe Motorsport     | `car`        | Free        | Main car dealership. Showroom is on the studio rooftop. |
| 2   | Bike Shop                     | `motorcycle` | Free        | Motorcycles, quads, and bicycles auto-merged from the main pool. |
| 3   | Higgins Helitours             | `air`        | $5,000      | Helicopters and planes. Test-drive override: 90 s, 4000 unit radius, spawn at the airfield. |
| 4   | LSYMC Boats                   | `sea`        | $1,500      | Boats and jet skis. Test-drive override: 90 s, 2500 unit radius, spawn off the coast. |

---

## 5. Commands

### `/clads_vehicleshop_capture`

Bulk-captures vehicle photo thumbnails for every model that does not yet have a cached entry in `clads_vehicleshop_photos`.

| Property      | Value                                                                              |
| ------------- | ---------------------------------------------------------------------------------- |
| Type          | Server command, dispatched to the calling client.                                  |
| Permission    | `'group.admin'` (registered with the third arg of `RegisterCommand`).             |
| Runtime gate  | Defense-in-depth: also requires `Config.Photo.captureGroup` group / job match, or `clads.vehicleshop.capture` ACE. |
| Behaviour     | Walks every shop's vehicle pool, identifies models missing from the cache, then triggers a client-side bulk capture. Each capture takes ~1.5 s; the player is locked into the green-screen scene for the duration. |
| Notifications | `admin.capture_started`, `admin.capture_done`, `admin.no_models_to_capture`.       |

The command must be run from a connected character. Console invocation is rejected with `admin.capture_only_in_game`.

The bulk capture is implemented client-side via `clads_vehicleshop:client:bulkCapture`, which iterates the model list, calling `Photo.Capture(model, doneCb)` for each. The success / failure tally is reported when finished.

---

## 6. Customization

### UI theme

Edit `Config.UI` in `config.lua`. Every value is a 6-digit hex colour. The web app injects them into `:root` at runtime via `web/src/utils/theme.ts` — no rebuild needed. Drop in a different palette and the entire panel re-skins itself.

### Adding a new shop

Add an entry to `Config.VehicleShops`:

```lua
Config.VehicleShops[5] = {
    type           = 'car',
    nameKey        = 'My Custom Garage',
    icon           = 'fa-solid fa-car-side',
    categories     = { 'sport', 'supercar' },
    testDrive      = true,
    testDrivePrice = 0,
    coords         = vector3(-1300.0, 200.0, 65.0),
    carCoords      = vector4(-1305.0, 195.0, 65.0, 90.0),
    camCoord       = vector3(-1310.0, 195.0, 67.0),
    buyCoords      = vector4(-1295.0, 200.0, 65.0, 270.0),
    blip           = { sprite = 326, scale = 0.7, color = 1, display = 4 },
    drawable       = { marker = true, ['3dtext'] = true },
    npc = {
        enabled  = true,
        model    = 'a_m_m_business_01',
        coords   = vector3(-1300.0, 200.0, 65.0),
        heading  = 0.0,
        scenario = 'WORLD_HUMAN_STAND_IMPATIENT',
    },
    vehicles = {
        ['adder'] = { brand = 'Truffade', name = 'Adder', model = 'adder', price = 1000000, category = 'supercar' },
    },
}
```

### Defining vehicles in `config.vehicles.lua`

The flat pool is `Vehicles[1]`. Each entry is a model-keyed table:

```lua
Vehicles[1] = {
    ["panto"] = {
        brand    = "Benefactor",
        name     = "Panto",
        model    = "panto",
        price    = 20500,
        category = "compact",
    },
    -- ... more vehicles
}
```

| Field      | Description                                                                |
| ---------- | -------------------------------------------------------------------------- |
| `brand`    | Manufacturer label shown in the NUI vehicle card.                          |
| `name`     | Display name of the vehicle.                                               |
| `model`    | Spawn model name. Must be a valid model the player can stream.             |
| `price`    | Price in the active framework's currency.                                  |
| `category` | One of the categories listed in any shop's `categories` field.             |

After the flat list, an auto-split block at the bottom of `config.vehicles.lua` redistributes models into per-shop pools:

- **`Vehicles[1]`** — cars (everything left after the auto-move).
- **`Vehicles[2]`** — anything with `category` of `motorcycle`, `quad`, or `bicycle` is moved here.
- **`Vehicles[3]`** — aircraft. A default helicopter / plane set is layered in.
- **`Vehicles[4]`** — boats. A default boat set is layered in.

Edit `Vehicles[N]` directly, or move models between pools, to customize what each shop sells.

### Custom blip per shop

Each shop's `blip` table accepts:

```lua
blip = {
    sprite  = 225,            -- GTA blip sprite ID
    scale   = 0.65,
    color   = 27,             -- GTA blip colour ID
    display = 4,              -- 4 = always shown on the main map and minimap
    nameKey = 'blip.pdm',     -- Locale key, falls back to shop.nameKey
}
```

### Per-shop test-drive override

```lua
testDriveOverride = {
    time        = 90,
    coords      = vector4(-1023.0, -2880.0, 13.95, 330.0),
    maxDistance = 4000.0,
}
```

The override is applied field-by-field on top of `Config.TestDrive`. Any field you omit keeps the global default.

### `requiredJob` and `requiredJobGrade`

`requiredJob` accepts a single string or an array of strings, gating shop access to those jobs. `requiredJobGrade` accepts a table specifying the minimum grade name. These are advisory fields on the shop entry — wire them into your own access policy via the `clads_vehicleshop:client:open` event handler or by enforcing on the server-side `clads_vehicleshop:server:openShop` callback.

---

## 7. Exports & Events

### Client exports

| Export             | Signature                                  | Description                                                                 |
| ------------------ | ------------------------------------------ | --------------------------------------------------------------------------- |
| `GeneratePlate()`  | `() -> string`                             | Generates a unique plate matching `Config.PlateLetters` / `Config.PlateNumbers`. Validates via the server callback `clads_vehicleshop:server:isPlateTaken`. |
| `GetVehiclePrice(model)` | `(model: string) -> number?`         | Looks up the price for a given model across every shop's vehicle pool. Returns `nil` if not found. |

### Server exports

| Export                   | Signature                                  | Description                                          |
| ------------------------ | ------------------------------------------ | ---------------------------------------------------- |
| `GetVehiclePrice(model)` | `(model: string) -> number?`               | Server-side counterpart to the client export.        |

### Server callbacks (`lib.callback.register`)

| Event                                          | Args                                         | Returns                              | Purpose                                                        |
| ---------------------------------------------- | -------------------------------------------- | ------------------------------------ | -------------------------------------------------------------- |
| `clads_vehicleshop:server:isPlateTaken`        | `(plate: string)`                            | `boolean`                            | Plate uniqueness check against the active framework's vehicle table. |
| `clads_vehicleshop:server:openShop`            | `(shopId: number)`                           | `{ vehicles = table }`               | Returns the shop's vehicles merged with cached photo URLs.     |
| `clads_vehicleshop:server:buyTestDrive`        | `(price: number)`                            | `boolean`                            | Charges the test-drive fee.                                    |
| `clads_vehicleshop:server:buyVehicle`          | `(entry, plate, props)`                      | `boolean`                            | Validates funds, persists ownership, charges, returns success. |
| `clads_vehicleshop:server:requestPhotoUpload`  | `(model: string)`                            | `photoSpec`                          | Returns an upload spec for the configured provider, or `{ needsCapture = false }` if disabled, cached, or the requester lacks permission. |
| `clads_vehicleshop:server:saveModelPhoto`      | `(model: string, imageUrl: string)`          | `boolean`                            | Caches an uploaded URL keyed by model.                         |

### Net events (server -> client)

| Event                                                       | Payload                                       | Purpose                                                       |
| ----------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------- |
| `clads_vehicleshop:client:open`                             | `(shopId: number)`                            | Force-open a shop from the server.                            |
| `clads_vehicleshop:client:routingBucketSet`                 | `(toggle: boolean)`                           | Notifies the client that the routing bucket has been updated. |
| `clads_vehicleshop:client:notify`                           | `(message, ntype, duration)`                  | Routes a notification through the bridge notify wrapper.      |
| `clads_vehicleshop:client:bulkCapture`                      | `(models: string[])`                          | Triggers a client-side bulk capture pass.                     |
| `clads_vehicleshop:client:personalVehicleResult`            | `(existingOnWorld: boolean, cleanPlate: string)` | Result of the personal-vehicle conflict check.            |

### Net events (client -> server)

| Event                                              | Payload                          | Purpose                                                          |
| -------------------------------------------------- | -------------------------------- | ---------------------------------------------------------------- |
| `clads_vehicleshop:server:setRoutingBucket`        | `(restore: boolean?)`            | Sets the player's routing bucket to their src ID, or restores 0. |
| `clads_vehicleshop:server:registerSpawned`         | `(plate: string, netId: number)` | Notifies the server which networked entity belongs to a purchase. |
| `clads_vehicleshop:server:garageNewVehicle`        | `(plate: string, netId: number?)` | Marks the new vehicle as garaged and deletes the spawn entity. |

### Cooperative menu lock-out

The client emits `clads:ui:menuOpened` with the resource name when the showroom opens, and listens for the same event from any other resource — when triggered by another resource it auto-closes the panel. Wire your other full-screen menus into this event to get free interop:

```lua
TriggerEvent('clads:ui:menuOpened', GetCurrentResourceName())
```

### Optional add-on hooks (no-op if missing)

The client calls into these resources opportunistically and silently no-ops when they are not started:

| Resource         | Export / event                                | Purpose                                                    |
| ---------------- | --------------------------------------------- | ---------------------------------------------------------- |
| `bv_bridge`      | `RequestTeleportBypass(ms)`                   | Anti-cheat compatibility shim around teleports.            |
| `bv_nameplate`   | `SetNameplatesEnabled(state)`                 | Toggles nameplates while in the showroom.                  |
| `bv_driveby`     | `SetDrivebyEnabled(state)`                    | Toggles drive-by while in the showroom.                    |

Weather sync is automatically suspended for the duration of a photo capture, supporting `vSync`, `Renewed-Weathersync`, `qb-weathersync`, and `qbx_weathersync`.

---

## 8. Troubleshooting

### Showroom vehicle does not spawn

- Confirm the model is valid on your server (streamed in `stream/` or shipped with the base game). Invalid models are silently skipped by `lib.requestModel`.
- Check `Config.VehicleShops[<id>].carCoords` — the spawn coordinate must be reachable. The default PDM showroom is on the rooftop at z = 221.0 and may collide with map mods that reshape that building.
- The collision around `carCoords` must load. The spawner waits up to 2000 ticks for `HasCollisionLoadedAroundEntity`. If the area is in an interior MLO, ensure the IPL is loaded.
- If you opened the shop from a routing bucket, vehicles spawned in bucket 0 will not be visible. The showroom intentionally moves the player into bucket = src; ensure no other resource pulls the player back to bucket 0 mid-flow.

### Photos are blank / placeholders only

- Verify `Config.Photo.provider` is not `'disabled'`.
- Run `/clads_vehicleshop_capture` as an admin on a connected character. Console invocation is rejected.
- For `fivemanage`: confirm `Config.Photo.fiveManage.apiKey` is non-empty. The bridge prints `[clads_vehicleshop] Photo.provider=fivemanage but Config.Photo.fiveManage.apiKey is empty; skipping capture.` when missing.
- For `custom`: confirm `Config.Photo.custom.url` is non-empty. The bridge prints a similar warning when missing.
- For `local`: confirm the resource exists, is started, and exports `GenerateUploadToken(citizenid, key, ttlSeconds)` returning two values.
- Check the database. `SELECT * FROM clads_vehicleshop_photos;` should list one row per model captured. If rows exist but the showroom still shows placeholders, the cached `image_url` is unreachable from the player's browser context — open it in a normal browser to confirm.

### Test drive teleports to the wrong place

- The default test-drive coords (`Config.TestDrive.coords`) sit at the airfield-adjacent runway. Override per shop via `testDriveOverride`.
- The maximum distance check teleports the player back to the spawn coords if they wander beyond `maxDistance`. Increase `maxDistance` (or the per-shop override) for long-range vehicles like aircraft.
- Confirm `Config.UseRoutingBucketsOnTestDrive` is set as expected. With `false`, the player is restored to bucket 0 for the test drive (so traffic is visible). With `true`, the player remains isolated.

### `requiredJob` is not enforced

- `requiredJob` and `requiredJobGrade` are descriptive fields on the shop entry. The shipping resource does not enforce them automatically — they are exposed for buyer customization. Add an enforcement check in your wrapper or, server-side, register a custom `clads_vehicleshop:server:openShop` middleware that validates `Player.Get(src).PlayerData.job.name` against the shop's `requiredJob` before returning the vehicle list.

### Payment account is not charged

- Confirm `Config.Framework` is `'auto'` or matches your live core. On `'standalone'`, all purchases are free.
- Check `Config.PaymentAccounts`. The first account with sufficient balance is charged. Both must exist in your framework's account list.
- Confirm `community_bridge` exposes `Framework.RemoveAccountBalance`. If you see `Player.ChargeFee` always returning `false, nil` in the logs, your bridge layer may not be wired to the framework correctly. Restart `community_bridge` before `clads_vehicleshop`.

### `fivemanage` upload fails

- Verify the API key is correct and has not been revoked. fivemanage requires a paid plan for production-volume uploads.
- The upload request is `multipart/form-data` to `https://api.fivemanage.com/api/image` with the `Authorization` header set to your API key (no `Bearer ` prefix). Test the endpoint manually with `curl` if uploads silently fail.
- The response must contain a `url` field at the root of the JSON. If fivemanage's API contract changes, switch to `provider = 'custom'` and adjust `responseUrlPath` accordingly.
- Inspect the server console with `Config.Debug = true` to see the bridge's upload-spec resolution. The actual HTTP call happens client-side inside the NUI; check the F8 client console and the in-game NUI dev tools for the response.

### Vehicle does not appear in player inventory after purchase

- Confirm the matching adapter loaded for your framework: with `Config.Debug = true`, the server prints `[clads_vehicleshop] server bridge ready (framework: <slug>)`.
- For QB / QBox: confirm the player has a citizenid (i.e. is fully loaded) at the moment of purchase. The adapter writes to `player_vehicles` keyed by citizenid.
- For ESX: vehicles are written to `owned_vehicles` with `owner = identifier`. Confirm the identifier resolves correctly via `Framework.GetPlayerIdentifier`.
- For standalone: confirm `oxmysql` is started. The standalone adapter creates `clads_vehicleshop_vehicles` on first run and writes purchase records there.

---

## 9. Support

Reach out via the Creative Lads Discord.
