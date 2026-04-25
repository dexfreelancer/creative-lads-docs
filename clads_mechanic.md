# Creative Lads — Mechanic (`clads_mechanic`)

A full-feature mechanic shop resource for FiveM. It bundles a tablet-style NUI,
vehicle servicing with mileage-based wear, multi-stage tuning (engine swaps,
turbos, drivetrains, brakes, tyres, drift, gearboxes), nitrous, stance, lighting
and neon controls, an orbital preview camera, lifts, dynos, shops, stashes,
society banking, employee management, and a speed limiter that introduces
realistic engine damage above a configurable threshold.

The resource is framework-agnostic and is wired through `community_bridge`,
allowing it to run on QBCore, QBox, ESX, or in standalone mode without forks.

---

## 1. Overview

`clads_mechanic` is the workshop side of a roleplay vehicle ecosystem. Players
park their car at a configured location, mechanics open the tablet UI, connect
to the vehicle, and then perform anything from a quick respray to swapping a V8
into a sedan. Customers can be invoiced; shops can be self-service or owned and
managed by an in-game job; balances flow either through society funds or
personal accounts.

**Who it's for**

- Server owners running an RP server who want a single, modern mechanic
  resource that covers cosmetics, tuning, servicing, repair, and consumables.
- Mechanic-job operators who need lifts, stashes, employee hiring/firing,
  invoices, and society banking out of the box.
- Drift / car-meet servers that want stance, drift tuning, manual gearbox
  toggle, nitrous bottles, and turbo profile sound packs.

**Feature highlights**

- Tablet UI with a connect-to-vehicle workflow, orbital preview camera, and
  React-driven menus for every modification class.
- Vehicle servicing system with mileage-based wear on suspension, tyres,
  brakes, oil, clutch, air filter, spark plugs, and EV components.
- Multi-stage performance tuning — engine swaps (I4/V6/V8/V12), turbos with
  custom audio profiles, drivetrains, brakes, drift kit, and manual gearbox.
- Stance editor (suspension height, camber, track width) with live preview and
  per-frame reapplication so wheel offsets survive collisions.
- Nitrous system with bottle slots, fuel drain, purge, screen FX, rear-light
  trails, and a configurable activation key.
- Speed limiter that progressively damages the engine above a configurable
  km/h threshold, with a vehicle blacklist for emergency services.
- Society fund integration with deposit / withdraw and per-mechanic balances,
  plus job-exempt invoice support for police / EMS.
- Employee management, hire flow with target-player confirmation, fire,
  role updates, and per-employee personal stash.

---

## 2. Requirements

### Hard dependencies

Pulled directly from `fxmanifest.lua:53-57`:

```lua
dependencies {
    'community_bridge',
    'ox_lib',
    'oxmysql',
}
```

| Dependency        | Purpose                                                       |
| ----------------- | ------------------------------------------------------------- |
| `community_bridge` | Framework, inventory, target, notify, banking, text-UI shim. |
| `ox_lib`          | Skill checks, progress bars, contexts, zones, callbacks.      |
| `oxmysql`         | Persistent data (vehicles, employees, invoices, orders).      |

### Frameworks (auto-detected)

`framework/main.lua` resolves the active framework from `Config.Framework`. With
the default `auto` value the loader probes for `qbx_core`, `qb-core`, then
`es_extended`, otherwise falls back to standalone.

| `Config.FrameworkKind` | Vehicles table     | Player table | Identifier  |
| ---------------------- | ------------------ | ------------ | ----------- |
| `qbox` / `qb`          | `player_vehicles`  | `players`    | `citizenid` |
| `esx`                  | `owned_vehicles`   | `users`      | `identifier`|
| `standalone`           | `player_vehicles`  | `players`    | `citizenid` |

### Optional integrations (auto-routed by community_bridge)

The values below default to `auto`; only override them if community_bridge
auto-detection misfires on your stack.

- **Inventory:** `ox_inventory`, `qb-inventory`, `qs-inventory`, `codem-inventory`.
- **Target:** `ox_target` or `qb-target` (used for shops, stashes, lifts).
- **Notifications:** `ox_lib`, `okok`, `ps-ui`, `lation_ui`, `qb`, `esx`.
- **DrawText / TextUI:** `ox_lib`, `okok`, `ps-ui`, `lation_ui`, `qb-core`.
- **Society banking:** `Renewed-Banking`, `qb-banking`, `qs-banking`,
  `okokBanking`, `tgg-banking`.
- **Skill check:** `ox` (default) or `qb-skillbar`.

### Optional resources detected at runtime

- `qbx_core` + `qbx_vehicles` — required by the persistent job-vehicle (tow
  truck) feature in `server/sv-joyvehicle.lua`.
- `betterv_vehicleshop` — used to read the real vehicle price for percentage
  pricing (`server/sv-mods.lua` and `framework/cl-functions.lua`).
- `brazzers-fakeplates` — original-plate lookups so persisted vehicle data
  matches the real plate (`framework/sv-functions.lua:34-52`).

### Voice & audio

Audio packs ship in `audiodirectory/jg_mechanic.awc` and
`data/jg_mechanic_sounds.dat54.rel`. The audio bank name (`jg_mechanic`) is
the original; renaming it requires regenerating the binaries.

### MySQL

`oxmysql` is required. Tables auto-create on first start when
`Config.AutoRunSQL = true` (default). Manual schema and `INSERT` statements are
provided under `install/database/run.sql` for buyers who prefer to import them
explicitly.

---

## 3. Installation

1. **Download** the resource and copy the `clads_mechanic/` folder into your
   server's `resources/` directory (commonly `resources/[scripts]/clads_mechanic`).
2. **Install dependencies** so they start before this resource:
   - `community_bridge`
   - `ox_lib`
   - `oxmysql`
3. **(Optional) Run the SQL manually.** Auto-creation is enabled by default, but
   if you want explicit control, import `install/database/run.sql`:
   ```sh
   mysql -u user -p database < clads_mechanic/install/database/run.sql
   ```
   Tables created (also documented in `server/sv-initsql.lua`):
   - `clads_mechanic_data`
   - `clads_mechanic_employees`
   - `clads_mechanic_servicing_history`
   - `clads_mechanic_orders`
   - `clads_mechanic_invoices`
   - `clads_mechanic_settings`
   - `clads_mechanic_vehicledata`
   - `clads_mechanic_connected_vehicles`
   - `clads_mechanic_lifts`
4. **Add inventory items.** Pick the file matching your inventory backend from
   `install/inventory/` and merge the entries into your inventory's items file:
   - `ox_inventory-items.lua`
   - `qb-core-items.lua`
   - `quasar-inventory-items.lua`
   - `esx-items.sql`
   Item images live under `install/inventory/images/`.
5. **Start the resource.** Add to `server.cfg` after dependencies:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure oxmysql
   ensure clads_mechanic
   ```
6. **Test.**
   - Spawn a `mechanic_tablet` item (or stand at an `AdminMechanic`-eligible
     spot and run `/tamirciadmin`).
   - Drive into a configured mechanic zone (e.g. Bennys at
     `vector3(-211.14, -1325.13, 30.65)`), exit the vehicle, and run `/tablet`.
   - Confirm the orbital camera engages and the tablet UI loads.

---

## 4. Configuration

`config.lua` is split into seven labelled sections. Headings below mirror the
order in the file.

### 4.1 Locale

Defined in `config.lua:45-47`.

```lua
Config.Locale              = "auto"   -- 'auto' or 'en', 'tr', 'de', etc.
Config.NumberAndDateFormat = "en-US"
Config.Currency            = "USD"
```

`auto` resolves through convars in this order: `clads_locale` →
`ox:locale` → `qb_locale` → `lang` → `en`. Available locale files live in
`locales/` (en, ar, cn, de, es, fr, hu, it, ja, pt, sv, tr, zh-tw). English is
the source of truth and the fallback for missing keys.

### 4.2 UI Theme

Defined in `config.lua:67-226`. The theme block exposes every CSS custom
property the tablet UI uses, so colour tweaks apply at runtime without
rebuilding the React bundle.

```lua
Config.Theme = {
    preset = "dark_red",  -- 'dark_red' | 'purple_haze' | 'crimson_steel' | 'cyan_frost' | 'custom'
    presets = { ... },     -- Built-in palettes, do not edit unless making your own preset
    custom  = { ... },     -- Overrides applied when preset = "custom"
}
```

**Default preset:** `dark_red`. The four shipped presets are:

| Preset          | Primary  | Action   | Mood                    |
| --------------- | -------- | -------- | ----------------------- |
| `dark_red`      | `#FF003C` | `#FF003C` | High contrast, default. |
| `purple_haze`   | `#9D4EDD` | `#FFEA00` | Vapour-wave purple/yellow. |
| `crimson_steel` | `#E63946` | `#FFB800` | Warm steel.             |
| `cyan_frost`    | `#00B4D8` | `#FFD60A` | Cool cyan with gold CTAs. |

To roll your own scheme, set `preset = "custom"` and uncomment overrides under
`Config.Theme.custom`. Any key omitted there falls back to `purple_haze`.

### 4.3 Job system

Defined in `config.lua:239`.

```lua
Config.UseFrameworkJobs = true  -- false uses internal employee table only
```

When `true`, role detection uses the framework's job + grade. Each mechanic
location declares a `job` and `jobManagementRanks` (see §4.10 Locations). When
`false`, the resource falls back to its internal `clads_mechanic_employees`
table and the `owner_id` column on `clads_mechanic_data`.

### 4.4 Tablet

Defined in `config.lua:242-246`.

```lua
Config.UseTabletCommand            = "tablet"             -- false to disable command
Config.TabletConnectionMaxDistance = 4.0
Config.RequireMechanicJob          = true
Config.RequireTabletItem           = true
Config.TabletItemName              = "mechanic_tablet"
```

`Config.RequireMechanicJob` accepts the player if their job name contains the
substring `"mechanic"` *or* exactly matches the zone's `job` field. Per-location
overrides for the tablet item live on the location entry as `requireTabletItem`.

### 4.5 Shops & target

Defined in `config.lua:249-251`.

```lua
Config.Target         = "ox_target"   -- "qb-target" or "ox_target" — shops/stashes only
Config.UseSocietyFund = true          -- false charges player balance instead
Config.PlayerBalance  = "bank"        -- "bank" or "cash"
```

Note this `Config.Target` is the *shop/stash* target backend — the tablet UI's
target requirement is handled by `community_bridge`. The two settings are kept
separate so you can run `ox_target` for tablet workflows and `qb-target` for
shop interactions, or vice versa.

### 4.6 Skill bars

Defined in `config.lua:254-258`.

```lua
Config.UseSkillbars             = true
Config.ProgressBarDuration      = 10000  -- ms (when skillbars are off)
Config.MaximumSkillCheckAttempts = 3
Config.SkillCheckDifficulty     = { "easy", "easy", "easy", "easy", "easy" }  -- ox_lib only
Config.SkillCheckInputs         = { "w", "a", "s", "d" }                       -- ox_lib only
```

The skill check is wrapped by `Framework.Client.SkillCheck` in
`framework/cl-functions.lua:66-83`. With skillbars off the call falls back
through to a plain progress circle (or bar, controlled by
`Config.ProgressBar = "ox-circle" | "ox-bar" | "qb-progressbar"`).

### 4.7 Vehicle servicing

Wear toggles and thresholds (`config.lua:261-266`):

```lua
Config.EnableVehicleServicing = true
Config.ServiceRequiredThreshold = 20    -- below = "service required"
Config.ServicingRepairThreshold = 100   -- below = show repair button + Repair All
Config.ServicingBlacklist = { "police", "police2" }
```

Per-part data lives in `Config.Servicing` at the bottom of `config.lua`
(lines 3049-3128). Each part defines `enableDamage`, `lifespanInKm`, `itemName`,
`itemQuantity`, and an optional `restricted = "combustion"|"electric"`:

| Part         | Lifespan (km) | Required item             | Qty | Restriction |
| ------------ | ------------- | ------------------------- | --- | ----------- |
| `suspension` | 6,400         | `suspension_parts`        | 1   | —           |
| `tyres`      | 4,000         | `tyre_replacement`        | 4   | —           |
| `brakePads`  | 2,600         | `brakepad_replacement`    | 4   | —           |
| `engineOil`  | 1,500         | `engine_oil`              | 1   | combustion  |
| `clutch`     | 5,400         | `clutch_replacement`      | 1   | combustion  |
| `airFilter`  | 3,600         | `air_filter`              | 1   | combustion  |
| `sparkPlugs` | 4,400         | `spark_plug`              | 4   | combustion  |
| `evMotor`    | 13,000        | `ev_motor`                | 1   | electric    |
| `evBattery`  | 15,000        | `ev_battery`              | 1   | electric    |
| `evCoolant`  | 5,400         | `ev_coolant`              | 1   | electric    |

> Don't remove rows from `Config.Servicing` — disable the whole subsystem with
> `Config.EnableVehicleServicing = false` instead.

### 4.8 Speed limiter

Defined in `config.lua:269-279`. The thread in `client/cl-speedlimiter.lua`
ticks every `SpeedLimiterCheckInterval` ms, and applies linear-interpolated
engine damage between min and max thresholds.

```lua
Config.EnableSpeedLimiter        = true
Config.SpeedLimiterMinSpeed      = 330.0   -- km/h, damage starts here
Config.SpeedLimiterMaxSpeed      = 390.0   -- km/h, damage caps here
Config.SpeedLimiterMinDamage     = 5.0     -- %/sec at min speed
Config.SpeedLimiterMaxDamage     = 50.0    -- %/sec at max speed
Config.SpeedLimiterCheckInterval = 500     -- ms
Config.SpeedLimiterBlacklist = {
    "police", "police2", "police3", "police4", "policeb",
    "sheriff", "sheriff2",
    "ambulance", "firetruk",
}
```

Damage scales with the interval — at 500 ms, 50 %/sec applies 25 % per tick.
The engine cuts out when health drops to `<= 100`.

### 4.9 Nitrous

Defined in `config.lua:282-289`.

```lua
Config.NitrousScreenEffects       = true
Config.NitrousRearLightTrails     = true   -- visible at night only
Config.NitrousPowerIncreaseMult   = 2.0
Config.NitrousDefaultKeyMapping   = "LSHIFT"
Config.NitrousMaxBottlesPerVehicle = 3
Config.NitrousBottleDuration      = 15     -- seconds per bottle
Config.NitrousBottleCooldown      = 5      -- seconds between bottles
Config.NitrousPurgeDrainRate      = 1
```

Activation is bound through `RegisterKeyMapping("+nitrous", …)` so end-users
can rebind it under FiveM's *Key Bindings* menu. Electric vehicles are skipped
inside `client/cl-nitrous.lua`.

### 4.10 Stance

Defined in `config.lua:292-298`. Min/max clamp the slider in the stance editor.

```lua
Config.StanceMinSuspensionHeight = -0.3
Config.StanceMaxSuspensionHeight = 0.3
Config.StanceMinCamber           = 0.0
Config.StanceMaxCamber           = 0.5
Config.StanceMinTrackWidth       = 0.5
Config.StanceMaxTrackWidth       = 1.25
Config.StanceNearbyVehiclesFreqMs = 500   -- per-frame reapply tick for nearby cars
```

Stance only applies to `IsThisModelACar` / `IsThisModelAQuadbike` results.
Wheel camber and offset are reapplied on a tick because the natives lose state
through collisions.

### 4.11 Repair

Defined in `config.lua:301-303`.

```lua
Config.AllowFixingAtOwnedMechanicsIfNoOneOnDuty = false
Config.DuctTapeMinimumEngineHealth = 100.0    -- vehicle must be < this to use duct tape
Config.DuctTapeEngineHealthIncrease = 150.0   -- HP added by duct tape
```

Behaviour lives in `client/cl-fixing.lua`. Duct tape will refuse if the engine
is healthier than `DuctTapeMinimumEngineHealth` (i.e. the player's car still
runs fine).

### 4.12 Tuning

Defined in `config.lua:306` and the tuning data table at `config.lua:2723-3017`.

```lua
Config.TuningGiveInstalledItemBackOnRemoval = false
```

Tuning categories:

| Category        | Lookup key in `Config.Tuning` | Notes                                             |
| --------------- | ----------------------------- | ------------------------------------------------- |
| Engine swaps    | `engineSwaps`                 | I4 / V6 / V8 / V12, `restricted = "combustion"`.  |
| Tyres           | `tyres`                       | Slick, semi-slick, off-road.                      |
| Brakes          | `brakes`                      | Ceramic.                                          |
| Drivetrains     | `drivetrains`                 | AWD / RWD / FWD via `fDriveBiasFront`.            |
| Turbocharging   | `turbocharging`               | Custom `turboProfile` with sound packs.           |
| Drift tuning    | `driftTuning`                 | Single drift tuning kit.                          |
| Gearboxes       | `gearboxes`                   | Manual transmission (`minGameBuild = 3095`).      |

Each tune entry supports `name`, `info`, `itemName`, `price`, `handling`,
`handlingApplyOrder`, `handlingOverwritesValues`, `restricted`, `blacklist`,
`conflictsWith`, and `minGameBuild`. See the comment block at
`config.lua:2685-2722` for full semantics. Conflicts are checked
server-side in `server/sv-tuning.lua:60-71`.

### 4.13 Locations

Defined in `config.lua:354-2143` (`Config.MechanicLocations`). Every shop entry
follows the same shape:

```lua
SomeMechanic = {
  type = "owned",         -- or "self-service"
  job  = "mechanic1",     -- framework job name (omit on self-service)
  jobManagementRanks = {1},
  logo = "Mechanic.png",  -- file inside /logos
  vehicleSpawn   = vector4(...),  -- optional, job-vehicle spawn
  vehiclePickup  = vector4(...),  -- optional
  vehicleDropoff = vector4(...),  -- optional
  locations = { { coords = ..., size = ..., showBlip = false, employeeOnly = false } },
  blip = { id = 402, color = 5, scale = 0.8 },
  mods = { repair = {...}, performance = {...}, ... },
  tuning = { engineSwaps = { enabled = true, requiresItem = true }, ... },
  carLifts = { vector4(...), ... },
  tabletPoints = { { coords = ..., size = 1.5 }, ... },
  shops = { { name, coords, items = { ... } } },
  stashes = { { name, coords, slots, weight } },
  personalStash = { name, coords, slots, weight },
  allowedVehicleModels = { ... },     -- optional whitelist
  blockedVehicleModels = { ... },     -- optional blacklist
  allowedVehicleClasses = { ... },    -- optional whitelist
  blockedVehicleClasses = { ... },    -- optional blacklist
  requireTabletItem = false,          -- per-location override
}
```

Shipped locations cover law enforcement (`MissionRow`, `VespucciPD`,
`VespucciPD2`, `SandySO`, `PaletoSO`, `DavisSO`, `Prison`, `StatePolice`),
medical and fire (`pillbox`, `firestation7`), commercial mechanics
(`Atomic`, `LSCgrove`, `LSCgreenwich`, `CypressFlats`, `Bennys`,
`LSClamesa`, `LaMesa`, `AutoExotic`, `LSCcentral`, `LSC68`, `Flywheels`,
`Beekers`, `AfterLife`, `Hayes`, `Ottos`, `Tuners`, `Pier76Mechanic`),
aircraft mechanics (`PoliceAirMechanic`, `SAHPAirMechanic`,
`AmbulanceAirMechanic`), and the synthetic admin location (`AdminMechanic`)
used by `/tamirciadmin`. Each entry can be deleted, duplicated, or rewritten
without restarting the resource source — just add or remove the key.

The mod block per location accepts these subkeys:

| Key                  | Effect                                                                                           |
| -------------------- | ------------------------------------------------------------------------------------------------ |
| `repair`             | Whether the repair button is shown and what it costs.                                            |
| `performance`        | EMS, brakes, transmission, suspension, armour, turbo toggle.                                     |
| `cosmetics`          | Bumpers, hoods, wheels, license plate styling, etc.                                              |
| `stance`             | Stance editor visibility per shop.                                                               |
| `respray`            | Colour, finish, pearlescent, dashboard, interior, wheel.                                         |
| `wheels`             | Wheel-type swaps.                                                                                |
| `neonLights`         | Underglow + RGB.                                                                                 |
| `headlights`         | Xenon controller + colour wheel.                                                                 |
| `tyreSmoke`          | Tyre smoke colour swap.                                                                          |
| `bulletproofTyres`   | Bulletproof tyres install.                                                                       |
| `extras`             | Vehicle extras.                                                                                  |
| `livery`             | Livery slot (cars + air vehicles).                                                               |

Each block accepts `enabled`, `price`, `percentVehVal`, and (for some)
`priceMult`.

### 4.14 Misc

Defined in `config.lua:309-338`.

```lua
Config.UseCarLiftPrompt          = "[E] Use Lift"
Config.UseCarLiftKey             = 38       -- INPUT_PICKUP
Config.CustomiseVehiclePrompt    = "[E] Customise Vehicle"
Config.CustomiseVehicleKey       = 38

Config.UpdatePropsOnChange       = true     -- broadcast vehicle props on any change
Config.SmoothFirstGear           = false    -- prevent first-gear redline
Config.ManualHighRPMNotifications = true

Config.UniqueBlips                       = true   -- include shop name in blip text
Config.ModsPricesAsPercentageOfVehicleValue = true
Config.AdminsHaveEmployeePermissions     = false
Config.MechanicEmployeesCanSelfServiceMods = false
Config.FullRepairAdminCommand            = "vfix"
Config.MechanicAdminCommand              = "tamirciadmin"
Config.ChangePlateDuringPreview          = "ONIZLEME"
Config.RequireManagementForOrderDeletion = false
Config.UseCustomNamesInTuningMenu        = false
Config.DisableNoPaymentOptionForEmployees = false

Config.InvoiceExemptJobs = { "police", "bcso", "lspd", "sahp" }

Config.JobVehicles = {
    enabled = true,
    garageName = 'mechanic',
    vehicles = {
        { model = 'flatbed', label = 'Flatbed Tow Truck' },
        { model = 'towtruck4', label = 'Towtruck' },
    },
}

Config.DisableSound  = false
Config.AutoRunSQL    = true
Config.Debug         = false
```

Other knobs scattered across `config.lua`:

| Setting                 | Default | Purpose                                                       |
| ----------------------- | ------- | ------------------------------------------------------------- |
| `Config.AdminDutyStateBag` | `"admin_staff_duty"` | Statebag the admin tablet command checks before opening (`client/cl-tablet.lua:400`). |
| `Config.ElectricVehicles`  | List   | Models forced to `electric` mode when running on game builds before *Bottom Dollar Bounties*. |
| `Config.Mods.ItemsRequired` | Map  | Item required (and whether to consume) per mod category.      |

---

## 5. Commands & Keybinds

### Commands

| Command                | Source file                          | Permission                                       | Description                                                            |
| ---------------------- | ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------- |
| `/tablet`              | `client/cl-tablet.lua:367`           | Mechanic job + tablet item (configurable)         | Opens the mechanic tablet at the player's current zone. Command name configurable via `Config.UseTabletCommand`. |
| `/tamirciadmin`        | `client/cl-tablet.lua:402` and `server/sv-admin.lua:5` | Admin (Ace `command` permission) on duty | Opens the tablet against the player's current vehicle in admin mode (free, all options enabled). The server-side variant opens the admin panel. |
| `/vfix`                | `server/sv-fixing.lua:5`             | Admin (Ace `command`)                            | Fully repairs the player's current vehicle. Renamable via `Config.FullRepairAdminCommand`. |
| `/driftmode`           | `client/cl-handling.lua:777`         | Driver of a vehicle that has a drift kit installed | Toggles drift mode handling on the current vehicle.                    |
| `/neonac`              | `client/cl-lightcontroller.lua:191`  | Driver / passenger of vehicle                    | Forces all four neons on for the current vehicle.                      |
| `/neonkapat`           | `client/cl-lightcontroller.lua:214`  | Driver / passenger of vehicle                    | Turns all neons off.                                                   |
| `/+nitrous` / `/-nitrous` | `client/cl-nitrous.lua:459` / `:597` | Driver with nitrous installed and bottles + fuel | Activates / deactivates nitrous boost. Bound via `RegisterKeyMapping`. |

### Keybinds

- **Nitrous activation:** registered as `+nitrous` via
  `RegisterKeyMapping("+nitrous", "Activate nitrous", "keyboard", Config.NitrousDefaultKeyMapping or "RMENU")`.
  Default key is `LSHIFT` (`Config.NitrousDefaultKeyMapping`). Players can
  rebind it under FiveM *Settings → Key Bindings → FiveM*.
- **Use Lift prompt:** `Config.UseCarLiftKey = 38` (E by default).
- **Customise Vehicle prompt:** `Config.CustomiseVehicleKey = 38` (E by default).

---

## 6. Items

All items use the inventory backend selected by `community_bridge`. Item images
live in `install/inventory/images/`. Reference list of items consumed by the
resource (sourced from `Config.Mods.ItemsRequired` at `config.lua:2179-2191`,
`Config.Tuning` at `config.lua:2723-2992`, `Config.Servicing` at
`config.lua:3049-3128`, and `install/inventory/ox_inventory-items.lua`).

### Tablet & utility

| Item                | Used by                                                                  |
| ------------------- | ------------------------------------------------------------------------ |
| `mechanic_tablet`   | Required to open the tablet when `Config.RequireTabletItem = true`. Triggers `clads_mechanic:client:use-tablet`. |
| `cleaning_kit`      | Cleans the nearest vehicle. Triggers `clads_mechanic:client:clean-vehicle`. |
| `repair_kit`        | Full repair on the nearest vehicle (with skill / progress check). Triggers `clads_mechanic:client:repair-vehicle`. |
| `repairkit`         | Engine-only repair (alternate item kept for compatibility). Triggers `clads_mechanic:client:repair-engine`. |
| `duct_tape`         | Temporary engine fix below `Config.DuctTapeMinimumEngineHealth`. Triggers `clads_mechanic:client:use-duct-tape`. |
| `lighting_controller` | Opens the lighting/neon controller UI. Triggers `clads_mechanic:client:show-lighting-controller`. |
| `stancing_kit`      | Opens the stance editor on the player's current vehicle. Triggers `clads_mechanic:client:show-stancer-kit`. |

### Servicing parts

| Item                   | Replaces      |
| ---------------------- | ------------- |
| `engine_oil`           | Engine oil    |
| `tyre_replacement`     | Tyres (×4)    |
| `clutch_replacement`   | Clutch        |
| `air_filter`           | Air filter    |
| `spark_plug`           | Spark plugs (×4) |
| `brakepad_replacement` | Brake pads (×4) |
| `suspension_parts`     | Suspension    |
| `ev_motor`             | EV motor      |
| `ev_battery`           | EV battery    |
| `ev_coolant`           | EV coolant    |

### Modification kits (consumed when applying mods)

| Item                | Mod category       |
| ------------------- | ------------------ |
| `repair_kit`        | `repair`           |
| `performance_part`  | `performance`      |
| `cosmetic_part`     | `cosmetics`        |
| `stancing_kit`      | `stance` (not consumed; `removeItem = false`) |
| `respray_kit`       | `respray`          |
| `vehicle_wheels`    | `wheels`           |
| `lighting_controller` | `neonLights` and `headlights` |
| `tyre_smoke_kit`    | `tyreSmoke`        |
| `bulletproof_tyres` | `bulletproofTyres` |
| `extras_kit`        | `extras`           |

### Tuning items

| Item                 | Tune entry                                  |
| -------------------- | ------------------------------------------- |
| `i4_engine`, `v6_engine`, `v8_engine`, `v12_engine` | Engine swaps. |
| `turboracing`, `turboultimate` (and `turbocharger`) | Turbo profiles. |
| `awd_drivetrain`, `rwd_drivetrain`, `fwd_drivetrain` | Drivetrains. |
| `slick_tyres`, `semi_slick_tyres`, `offroad_tyres` | Tyre tuning. |
| `ceramic_brakes`     | Brakes.                                    |
| `drift_tuning_kit`   | Drift tuning.                              |
| `manual_gearbox`     | Gearbox swap.                              |

### Nitrous

| Item                     | Purpose                                                                |
| ------------------------ | ---------------------------------------------------------------------- |
| `nitrous_install_kit`    | Required at the tablet to install a nitrous system on a vehicle.       |
| `nitrous_bottle`         | Filled bottle. Triggers `clads_mechanic:client:use-nitrous-bottle` to load into the connected vehicle. |
| `empty_nitrous_bottle`   | Returned when a bottle is removed from a vehicle.                      |

---

## 7. Customization

### Theme

Set `Config.Theme.preset` to `"dark_red"` (default), `"purple_haze"`,
`"crimson_steel"`, `"cyan_frost"`, or `"custom"`. With `"custom"`, the
overrides under `Config.Theme.custom` win over the fallback (`purple_haze`).
Add or rename presets directly inside `Config.Theme.presets` if you want a
per-server palette under a friendlier name.

```lua
Config.Theme.preset = "custom"
Config.Theme.custom = {
    colorPrimary       = "#9D4EDD",
    colorAction        = "#FFEA00",
    bgCore             = "#0D0B14",
    -- any other CSS variable from the preset block
}
```

### Adding a new shop / location

Append a new entry to `Config.MechanicLocations` (`config.lua:354+`). Minimum
required shape:

```lua
Config.MechanicLocations.MyShop = {
  type = "owned",                 -- or "self-service"
  job  = "myshop_mechanic",       -- omit on self-service
  jobManagementRanks = {1},
  logo = "Mechanic.png",          -- placed in logos/
  locations = {
    {
      coords = vector3(123.0, 456.0, 30.0),
      size   = 30.0,
      showBlip = true,
      employeeOnly = false,
    },
  },
  blip = { id = 446, color = 47, scale = 0.7 },
  mods = {
    repair      = { enabled = true, price = 500, percentVehVal = 0.01 },
    performance = { enabled = true, price = 500, percentVehVal = 0.01, priceMult = 0.1 },
    -- (full list in §4.13)
  },
  tuning = {
    engineSwaps = { enabled = true, requiresItem = true },
    -- (full list in §4.12)
  },
  carLifts = { vector4(123.0, 460.0, 30.0, 90.0) },
}
```

The resource creates a row in `clads_mechanic_data` for each new key on the
next start (`server/sv-main.lua:30-33`).

### Modifying skill checks

`Config.SkillCheckDifficulty` and `Config.SkillCheckInputs` are passed straight
through to `lib.skillCheck` from `framework/cl-functions.lua:73`. To replace
with a flat progress bar entirely, set `Config.UseSkillbars = false` —
installations then use `Config.ProgressBarDuration` (default 10 s) and
`Config.ProgressBar = "ox-circle" | "ox-bar" | "qb-progressbar"`.

### Vehicle servicing thresholds

```lua
Config.ServiceRequiredThreshold = 20    -- below %, "service required" badge
Config.ServicingRepairThreshold = 100   -- below %, repair button + Repair All
```

Per-part lifespan lives in `Config.Servicing[partName].lifespanInKm`. To make
oil burn faster, e.g.:

```lua
Config.Servicing.engineOil.lifespanInKm = 800
```

To exempt a vehicle entirely, add it to `Config.ServicingBlacklist`.

### Blacklists

| List                          | Effect                                                       |
| ----------------------------- | ------------------------------------------------------------ |
| `Config.ServicingBlacklist`   | Models in this table do not accumulate servicing wear.       |
| `Config.SpeedLimiterBlacklist`| Models exempt from speed-limiter engine damage.              |
| `MechanicLocations.*.allowedVehicleModels` / `blockedVehicleModels` | Per-location whitelist / blacklist, checked in `client/cl-tablet.lua:104-129`. |
| `MechanicLocations.*.allowedVehicleClasses` / `blockedVehicleClasses` | Per-location class-level filter using `GetVehicleClass`. |
| `Config.InvoiceExemptJobs`    | Players in these jobs are not charged for invoices or applied mods. |

---

## 8. Exports & Events

### Client exports

Defined in `client/cl-handling.lua:811-822`, `client/cl-vehicleprops.lua:293-813`,
and `shared/main.lua:12`.

| Export                            | Purpose                                                        |
| --------------------------------- | -------------------------------------------------------------- |
| `exports.clads_mechanic:config()` | Returns the live `Config` table.                               |
| `exports.clads_mechanic:calculateTuningHandling(...)` | Computes the resulting handling delta for an installed tuning set. |
| `exports.clads_mechanic:calculateServicingHandling(...)` | Computes handling penalties from worn parts. |
| `exports.clads_mechanic:applyVehicleTuningHandling(vehicle, tuningConfig)` | Applies a `tuningConfig` table to the vehicle's handling at runtime. |
| `exports.clads_mechanic:isDriftModeDisabled(vehicle)` | True if `/driftmode` was used to disable the drift handling on the given vehicle. |
| `exports.clads_mechanic:getVehicleProperties(vehicle, includeStateBag?)` | Reads the resource's flat property table from a vehicle. |
| `exports.clads_mechanic:setVehicleProperties(vehicle, props)` | Writes a property table back onto a vehicle. |
| `exports.clads_mechanic:getVehicleStatebagProperties(vehicle)` | Reads the resource's tuning / servicing / stance / nitrous statebag layer. |
| `exports.clads_mechanic:setVehicleStatebagProperties(vehicle, props)` | Writes the statebag layer in one call. |

### Server exports

Defined in `server/sv-vehicleprops.lua:163` and `server/sv-webhooks.lua:28`.

| Export                                                       | Purpose                                                                                |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| `exports.clads_mechanic:applySavedVehicleStatebags(vehicle, plate)` | Re-applies persisted servicing / tuning / stance / nitrous statebags to a freshly spawned vehicle. |
| `exports.clads_mechanic:sendWebhook(url, title, description, color)` | Helper used by internal logging hooks; safe to call from other resources for branded notifications. |

### Notable client net events

Useful targets when extending the resource (all under the
`clads_mechanic:client:` namespace):

| Event                                              | Purpose                                                |
| -------------------------------------------------- | ------------------------------------------------------ |
| `clads_mechanic:client:use-tablet`                 | Opens the tablet (fired by the inventory item).         |
| `clads_mechanic:client:enter-mechanic-zone` *(args: mechanicId)* | Sets the player's `mechanicId` statebag. |
| `clads_mechanic:client:exit-mechanic-zone`         | Tears down the tablet, disconnects vehicle, clears state. |
| `clads_mechanic:client:fix-vehicle-admin`          | Fully repairs the player's current vehicle.            |
| `clads_mechanic:client:repair-vehicle`             | Public repair flow with skill check.                   |
| `clads_mechanic:client:clean-vehicle`              | Cleaning kit flow.                                     |
| `clads_mechanic:client:use-duct-tape`              | Duct-tape engine repair flow.                          |
| `clads_mechanic:client:show-lighting-controller`   | Opens the neon / xenon controller UI.                  |
| `clads_mechanic:client:show-confirm-employment`    | Hire-request confirmation prompt on the target player. |
| `clads_mechanic:client:show-invoice-to-player`     | Pushes an invoice to the customer's tablet.            |
| `clads_mechanic:client:dyno-show-results-sheet`    | Renders the dyno results modal.                        |
| `clads_mechanic:client:open-admin`                 | Opens the admin panel UI.                              |
| `clads_mechanic:client:refresh-mechanic-zones-and-blips` | Rebuilds zones and blips after a job change.    |
| `clads_mechanic:client:notify`                     | Re-broadcast notification helper.                      |
| `clads_mechanic:client:tablet-hidden-for-interaction` / `:tablet-shown-after-interaction` | UI lifecycle hooks for chained interactions. |

### Notable server net events / callbacks

| Name                                                | Type     | Purpose                                                                                       |
| --------------------------------------------------- | -------- | --------------------------------------------------------------------------------------------- |
| `clads_mechanic:server:remove-item`                 | Event    | Removes a quantity of an item from the calling player.                                        |
| `clads_mechanic:server:request-hire-employee`       | Event    | Sends a hire request to a target player.                                                      |
| `clads_mechanic:server:hire-employee`               | Event    | Confirms hire after the target accepts.                                                       |
| `clads_mechanic:server:fire-employee`               | Event    | Removes an employee row.                                                                      |
| `clads_mechanic:server:update-employee-role`        | Event    | Updates an employee's role.                                                                   |
| `clads_mechanic:server:save-veh-statebag-data-to-db`| Callback | Forces persistence of a vehicle's statebag layer.                                             |
| `clads_mechanic:server:get-mechanic-data`           | Callback | Returns mechanic config + DB row + employee status for the caller.                            |
| `clads_mechanic:server:get-mechanic-balance` / `mechanic-deposit` / `mechanic-withdraw` | Callbacks | Society fund operations gated by management role. |
| `clads_mechanic:server:apply-mods`                  | Callback | Charges and persists a cart of mods against a vehicle plate.                                  |
| `clads_mechanic:server:repair-vehicle`              | Callback | Charges the configured price (society or personal) and confirms a repair.                     |
| `clads_mechanic:server:install-tune` / `uninstall-tune` / `has-tuning-item` | Callbacks | Tuning install pipeline including server-side conflict resolution. |
| `clads_mechanic:server:install-nitrous` / `add-nitrous-bottle` / `refill-nitrous` / `remove-nitrous` / `get-nitrous-data` | Callbacks | Nitrous lifecycle. |
| `clads_mechanic:server:save-invoice` / `send-invoice` / `pay-invoice` / `delete-invoice` / `resend-invoice` / `get-unpaid-invoices` | Callbacks | Invoice lifecycle. |
| `clads_mechanic:server:claimVehicle` / `spawnDutyVehicle` | Callbacks | QBox-only persistent job-vehicle (tow truck) flow. |

---

## 9. Troubleshooting

### Tablet won't open

- **`notAtMechanic` notification:** the player's `LocalPlayer.state.mechanicId`
  is `nil`. They need to be inside one of the `Config.MechanicLocations` zones
  defined in §4.13. Set `Config.Debug = true` to draw zones with `lib.zones.box`.
- **`permissionRequiredJob`:** `Config.RequireMechanicJob = true` and the
  player's job either doesn't match the zone's `job` field or doesn't contain
  the substring `"mechanic"`. Double-check `Config.UseFrameworkJobs` is set
  correctly for your stack.
- **`permissionRequiredItem`:** the player doesn't have a `mechanic_tablet`
  item. Either give them one, set `Config.RequireTabletItem = false`, or set
  `requireTabletItem = false` on that specific location entry.
- **NUI never appears:** make sure `community_bridge`, `ox_lib`, and `oxmysql`
  start before `clads_mechanic`. Check the F8 console for errors loading
  `web/dist/index.html`.

### Target options missing

- Verify `Config.Target` matches your installed target backend (`ox_target` or
  `qb-target`). It only governs shops and stashes.
- For tablet target prompts (e.g. open lift, customise vehicle), the resource
  uses `community_bridge`'s target abstraction. Make sure the bridge's target
  module recognises your installation.

### Framework detection wrong

`framework/main.lua` runs `HasResource(...)` against `qbx_core`, `qb-core`, and
`es_extended` in order. If it picks the wrong one (for example, if you're
running both `qb-core` and `es_extended` for migration purposes), set
`Config.Framework` explicitly to `"qb"`, `"qbox"`, or `"esx"`. The detected
value is exposed as `Config.FrameworkKind` for debugging.

### Speed limiter affecting wrong vehicles

- Add the model to `Config.SpeedLimiterBlacklist`. The check uses
  `GetEntityArchetypeName(vehicle)`, so use the spawn model name (e.g.
  `"police"`).
- Disable globally with `Config.EnableSpeedLimiter = false`.
- Lower the damage values (`SpeedLimiterMinDamage`, `SpeedLimiterMaxDamage`) if
  you only want a soft penalty.
- Only cars, bikes, and quadbikes are affected (other vehicle types are
  filtered out in `client/cl-speedlimiter.lua:50`).

### Society fund payments fail

- Set `Config.SocietyBanking` explicitly if `community_bridge` cannot
  auto-detect your banking resource (`Renewed-Banking`, `qb-banking`,
  `qs-banking`, `okokBanking`, `tgg-banking`).
- For self-service / unowned shops, set `Config.UseSocietyFund = false` so the
  resource charges the player's `Config.PlayerBalance` (`bank` or `cash`)
  instead. `MechanicMarket` purchases also follow this flag (see
  `server/sv-shops-stashes.lua:31-81`).

### Nitrous won't activate

- The vehicle must have a nitrous system installed (consumes
  `nitrous_install_kit`).
- The vehicle must have at least one bottle and `nitrousFuel > 0`.
- Electric vehicles are skipped (`isVehicleElectric` returns true → early-out
  in `cl-nitrous.lua`).
- The activation key must be bound. Default is `LSHIFT`; players can rebind
  via *Settings → Key Bindings*.
- Cooldown applies between bottles (`Config.NitrousBottleCooldown`).

### Stance changes don't persist visually

Stance values are reapplied per frame for nearby vehicles every
`Config.StanceNearbyVehiclesFreqMs`. If they appear to revert after a
collision, increase the frequency (lower number) — wheel offsets and camber
have to be re-pushed because the GTA natives lose them.

### Job-vehicle (tow truck) feature does nothing

The persistent job-vehicle code only runs when both `qbx_core` and
`qbx_vehicles` are present (`server/sv-joyvehicle.lua:13`). On QBCore or ESX
the feature silently no-ops. Set `Config.JobVehicles.enabled = false` to hide
the menu entry entirely.

### `tamirciadmin` won't open

Two things have to be true:

1. `Framework.Server.IsAdmin(src)` returns true — i.e. the player has the Ace
   `command` permission.
2. The client-side admin-duty statebag is set. Default key:
   `admin_staff_duty`. Override with `Config.AdminDutyStateBag`. If your admin
   resource sets a different statebag, configure that name here.

### Plate is "ONIZLEME" while previewing

That's the configurable preview plate (`Config.ChangePlateDuringPreview =
"ONIZLEME"`). Change it to a different placeholder or set it to your own
character string.

---

## 10. Support

Reach out via the Creative Lads Discord.
