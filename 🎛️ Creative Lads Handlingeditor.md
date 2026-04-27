# Creative Lads — Handling Editor

A live in-game `CHandlingData` editor for FiveM. Players (or, more typically, staff) jump into a vehicle, run `/handling`, and tune mass, drive force, gear count, traction, suspension, anti-roll, centre-of-mass offset and 30+ other handling fields with sliders that apply to the active vehicle in real time. A separate timing overlay measures 0–60 mph, 0–100 km/h and quarter / half-mile splits with baseline-vs-latest deltas, so any tuning change is verifiable on track.

---

## 1. Overview

`clads_handlingeditor` exposes the entire `CHandlingData` block of any vehicle the player is currently driving, behind a single chat command and an ACE permission gate. Where most "handling editor" scripts ship as a fixed dropdown of pre-baked presets, this resource is built around live sliders + telemetry comparison so you can iterate on a tune in-session.

- **Live editor.** Open `/handling` while sitting in the driver seat. The NUI panel reads every supported field through `GetVehicleHandling*` natives, paints sliders + numeric inputs, and writes back through `SetVehicleHandling*` on every change. Speed-sensitive fields (drive force, drag coefficient, gear count, drive inertia) trigger a one-shot speed-system refresh so the new top speed actually takes hold without restarting the engine.
- **Baseline detection.** The first time you open the editor on a vehicle, the resource spawns an invisible probe of the same model, samples its untouched handling values, then deletes the probe. Those readings become the **Original** column the editor compares against. The probe is frozen, invincible, alpha-zeroed, and collision-disabled while it lives.
- **Telemetry overlay.** Hit **Start Preview** (or `/handlingtiming`) and a corner overlay shows live MPH/KMH, top MPH, current gear, and elapsed run time. Telemetry markers (0–60 mph, 0–100 km/h, ¼ mile, ½ mile by default) latch the first time the run crosses each threshold. Save the run as a baseline and the next run shows the percentage delta per marker — green for faster, red for slower.
- **Presets.** Save the current set of changes as a named preset on the server. Presets are stored in `profiles.json` next to the resource, never on the client. Any player with the editor permission can load, apply, or delete one.
- **XML export.** The Export tab emits a `CHandlingData` XML fragment containing only changed fields, ready to paste into a vehicle's `handling.meta`.
- **Simplified presets.** Six grouped one-click tuning families — acceleration tier, top-speed tier, transmission profile, drivetrain (RWD/AWD/FWD), traction, suspension — plus a body-roll slider that biases anti-roll and roll-centre relative to the captured baseline.
- **Per-vehicle sessions.** Each unique plate + model + network id keeps its own dirty-state and telemetry history, so flipping between two vehicles preserves the work in progress on each one.
- **Framework-agnostic.** Every notification call goes through `community_bridge`; permissions go through standard FiveM `IsPlayerAceAllowed`. Standalone, ESX, QBCore and QBox are supported through one code path.

---

## 2. Requirements

| Dependency | Purpose | Required |
|------------|---------|----------|
| `community_bridge` | Notification routing + framework / identifier resolution | Yes |
| `ox_lib` | Notification fallback when no bridge backend is registered | Yes |
| Framework | Auto-detected via `community_bridge`. Standalone, ESX, QBCore, QBox all supported. | Auto |

No SQL, no targets, no inventories. The resource declares both dependencies in `fxmanifest.lua` and refuses to start without them.

---

## 3. Installation

1. Drop the `clads_handlingeditor` folder into your server's `resources/` directory.
2. Make sure `community_bridge` and `ox_lib` are present and start **before** `clads_handlingeditor`.
3. Add the resource to your `server.cfg`:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure clads_handlingeditor
   ```
4. (Optional) gate the editor + timing tool behind ACE permissions:
   ```cfg
   add_ace group.admin clads_handlingeditor.use    allow
   add_ace group.admin clads_handlingeditor.timing allow
   ```
   Then flip `Config.RequireAce = true` in `config.lua`.
5. Open `config.lua` and tune the values described in section 4.
6. Start (or restart) the server and run `/handling` from the driver seat.

There is no database table, no SQL migration, and no NUI build step on the buyer side — the resource ships with the obfuscated bundle pre-built in `web/build/`.

---

## 4. Configuration

All settings live in `config.lua` at the resource root, which is left out of escrow encryption (`escrow_ignore`) so it stays editable after purchase.

### Framework

```lua
-- 'auto' | 'esx' | 'qb' | 'qbox' | 'standalone'
Config.Framework = 'auto'
```

`community_bridge` auto-detects the running core. Pin a specific value here only if you run multiple cores side by side or want to force standalone mode while testing. The resource only uses the bridge for notifications and the player identifier attached to saved presets, so standalone is a real path — every framework call degrades silently when no core is present.

### Locale

```lua
Config.Locale = 'auto'
```

Built-in locales: `en`, `tr`, `de`, `fr`, `es`. When set to `'auto'`, the resource resolves the language from these convars in order:

1. `clads_locale`
2. `ox:locale`
3. `qb_locale`
4. `lang`
5. fallback `en`

The full locale dict is shipped to the NUI on every editor / preview open, so language changes apply on the next open without a resource restart.

### Debug

```lua
Config.Debug = false
```

Enables verbose console output (currently used by the server when a profile is saved). Leave off in production.

### Permissions

```lua
Config.RequireAce                  = false
Config.EditorAce                   = 'clads_handlingeditor.use'
Config.TimingAce                   = 'clads_handlingeditor.timing'
Config.AllowTimingWithoutEditorAce = true
```

| Key | Effect |
|---|---|
| `RequireAce` | When `false`, every player can open the editor and timing tool. When `true`, the server checks `IsPlayerAceAllowed` before granting access. |
| `EditorAce` | ACE id checked for the full editor. |
| `TimingAce` | ACE id checked for the timing-only mode. |
| `AllowTimingWithoutEditorAce` | When `true`, anyone who has the editor ace also passes the timing check, so you only need to grant one ace per role. |

If a request fails the permission check, the server fires a `clads_handlingeditor:client:openDenied` event and the client surfaces a translated `notify_no_perm_editor` / `notify_no_perm_timing` toast.

### Commands

```lua
Config.Commands = {
    editor = 'handling',
    timing = 'handlingtiming',
}
```

Renaming `editor` updates both the slash command and the chat suggestion entry. `timing` registers a dedicated alias that opens the telemetry overlay directly without the editor step.

### Preview Key Mappings

```lua
Config.PreviewResetKey          = 'LCONTROL'
Config.PreviewCaptureBaselineKey = 'H'
Config.PreviewClearBaselineKey  = 'G'
Config.PreviewToggleMouseKey    = 'LMENU'
Config.PreviewStopKey           = 'BACK'
```

Defaults registered through `RegisterKeyMapping`. Players can rebind every entry from **Settings → Key Bindings → FiveM** under "Creative Lads — Handling Editor". The values are FiveM key codes (`LCONTROL`, `LMENU`, `BACK`, single uppercase letters, etc.).

### Telemetry Timing

```lua
Config.PreviewTickMs           = 100
Config.SpeedStartThresholdMps  = 0.3
```

| Key | Description |
|---|---|
| `PreviewTickMs` | How often (ms) the telemetry thread samples speed, distance, and gear while preview mode is open. The thread idles cheaply when preview is closed. |
| `SpeedStartThresholdMps` | Vehicle speed in m/s at which the run timer starts. Default 0.3 m/s ≈ 1 km/h, so the timer triggers as soon as the car is rolling. Bump it higher if your players want a clean launch reading. |

### Telemetry Markers

```lua
Config.TelemetryMarkers = {
    { id = '0_60_mph',     label = 'marker_0_60_mph',     type = 'speed',    value = 26.8224 },
    { id = '0_100_kmh',    label = 'marker_0_100_kmh',    type = 'speed',    value = 27.7778 },
    { id = 'quarter_mile', label = 'marker_quarter_mile', type = 'distance', value = 402.336 },
    { id = 'half_mile',    label = 'marker_half_mile',    type = 'distance', value = 804.672 },
}
```

Each entry latches a timestamp the first time the run crosses its threshold. Two trigger types:

| `type` | Triggers when… | Units of `value` |
|---|---|---|
| `speed` | speed in m/s ≥ `value` | m/s |
| `distance` | distance from start in m ≥ `value` | m |

`label` is a translation key (`marker_*` in the locale JSON). Add a marker with a literal label by passing a non-key string and it will render verbatim.

### Field Categories

```lua
Config.Categories = {
    { id = 'engine',     label = 'cat_engine'     },
    { id = 'drivetrain', label = 'cat_drivetrain' },
    { id = 'brakes',     label = 'cat_brakes'     },
    { id = 'steering',   label = 'cat_steering'   },
    { id = 'traction',   label = 'cat_traction'   },
    { id = 'suspension', label = 'cat_suspension' },
    { id = 'damage',     label = 'cat_damage'     },
    { id = 'flags',      label = 'cat_flags'      },
    { id = 'vectors',    label = 'cat_vectors'    },
}
```

Sidebar groups the field list by `id`. The "All Fields" pseudo-category is rendered automatically as the first sidebar entry.

### Handling Fields

`Config.Fields` holds every CHandlingData field exposed to the editor. Each entry follows this shape:

```lua
{
    key      = 'fInitialDriveForce',
    label    = 'field_fInitialDriveForce_label',
    desc     = 'field_fInitialDriveForce_desc',
    type     = 'float',          -- 'float' | 'int' | 'flag' | 'vector'
    category = 'drivetrain',
    min      = 0.01, max = 2.0,
    step     = 0.001, decimals = 3,
    useful   = true,             -- highlights as commonly tuned
}
```

| `type` | Native used | Editor control |
|---|---|---|
| `float` | `Get/SetVehicleHandlingFloat` | range slider + numeric input |
| `int` | `Get/SetVehicleHandlingInt` | range slider + numeric input (integer step) |
| `flag` | `Get/SetVehicleHandlingInt` | numeric input + live HEX preview |
| `vector` | `Get/SetVehicleHandlingVector` | three numeric inputs (x / y / z) |

`min` and `max` are clamped both client- and server-side. `useful = true` paints a green "Important" badge in the editor card; `useful = false` paints a neutral "Advanced" badge. The shipped catalog covers 43 fields across the nine categories.

### Simplified Presets

```lua
Config.SimplifiedPresets = {
    acceleration = {
        { id = 'city',  driveForce = 0.22, driveInertia = 1.0 },
        { id = 'sport', driveForce = 0.34, driveInertia = 1.2 },
        { id = 'super', driveForce = 0.46, driveInertia = 1.4 },
    },
    -- topSpeed / transmission / drivetrain / traction / suspension follow the same shape
}
```

Each group's presets pair with translation keys (`preset_<group>_<id>`). Picking one from the simplified panel applies the listed handling fields directly through the same edit pipeline as a slider drag — they show up in the Changes tab and the XML export. The body-roll slider biases `fAntiRollBarForce`, `fRollCentreHeightFront`, and `fRollCentreHeightRear` relative to the captured baseline (clamped to native ranges), so it never stacks past safe limits.

### Meddling Resources

```lua
Config.MeddlingResources = {
    'betterac',
    'qbx_cruise',
    'vehiclehandler',
}
```

A handful of community resources re-write `CHandlingData` after this editor sets it. On startup the server prints a one-shot warning for every name that's currently `started` or `starting`, so you know where to look if values keep flipping back during testing. Add or remove names to match your stack.

---

## 5. Commands

### Chat Commands

```
/handling           Open the handling editor for the current vehicle
/handling timing    Open just the telemetry overlay
/handling editor    Open the editor (default if no arg)
/handlingtiming     Same as /handling timing
```

Both commands require the player to be in the **driver seat** of a vehicle. If `Config.RequireAce` is on, the server checks the configured ace before forwarding the open event.

The exact command names follow `Config.Commands` — change them once and both the chat suggestion entries and the registered commands update on next start.

### Preview Keybinds

While the telemetry overlay is open the following keys are active. Defaults can be remapped by the player in **Settings → Key Bindings → FiveM**.

| Default | Action | Locale label |
|---|---|---|
| `Ctrl` | Reset the active run (with optional capture into `lastRun`) | `keymap_reset` |
| `H` | Save the current run as the baseline | `keymap_capture` |
| `G` | Clear the saved baseline | `keymap_clear` |
| `Left Alt` | Toggle the mouse cursor (so you can click the overlay buttons) | `keymap_mouse` |
| `Backspace` | Stop preview and teleport the vehicle back to its start position | `keymap_stop` |

The hint line under the preview header reads these labels back from the locale dict, so a non-English server shows the localized cue without any extra wiring.

---

## 6. Customization

### Locales

Every player-visible string lives in `locales/<code>.json`. The file ships with five languages:

| Code | Language |
|---|---|
| `en` | English (source of truth, fallback) |
| `tr` | Turkish |
| `de` | German |
| `fr` | French |
| `es` | Spanish |

To add a new language, drop another file in `locales/` (e.g. `pt.json`), copy every key from `en.json`, and translate the values. Every locale ships with the same key set — the loader logs a warning and falls back to English if a key is missing.

### Field labels and descriptions

`Config.Fields` references locale keys (`field_<key>_label`, `field_<key>_desc`) instead of raw strings. To rename a field for your server, override the key in the locale JSON; no Lua edit needed. The same applies to category labels (`cat_*`), simplified preset labels (`preset_*`), and telemetry markers (`marker_*`).

### Profile storage

Saved presets are written to `profiles.json` in the resource folder via `SaveResourceFile`. The format is a flat JSON array of:

```json
{
  "id":        "1714214012-3-7421",
  "name":      "Track Day Comp2",
  "author":    "Kiyam",
  "ownerId":   "license:abcd…",
  "createdAt": "2026-04-27T13:14:55Z",
  "modelHash": -2095434141,
  "values":    { "fInitialDriveForce": 0.42, "fSuspensionForce": 3.1, … }
}
```

Only changed fields are saved. The file is rewritten on every save, every delete, and on `onResourceStop`, so editing it by hand while the resource is running will get clobbered — stop the resource first.

`profiles.json` is excluded from the Tebex zip via `.tebexignore`, so a fresh purchase always starts with an empty list.

---

## 7. Exports & Events

The resource is intentionally self-contained and does not expose public exports.

### Server Net Events

| Event | Direction | Payload | Purpose |
|---|---|---|---|
| `clads_handlingeditor:server:requestOpen` | Client → Server | `mode: 'editor' \| 'timing'` | Permission check + open dispatch. |
| `clads_handlingeditor:server:saveProfile` | Client → Server | `{ name, modelHash, values }` | Persist a new preset to `profiles.json`. |
| `clads_handlingeditor:server:deleteProfile` | Client → Server | `id: string` | Remove a preset by id. |
| `clads_handlingeditor:client:openAllowed` | Server → Client | `mode, profiles[]` | Open the UI in the requested mode and ship the current preset list. |
| `clads_handlingeditor:client:openDenied` | Server → Client | `messageKey: string` | Permission denied — the client toasts the localized key. |
| `clads_handlingeditor:client:profilesUpdated` | Server → Client | `profiles[]` | Push an updated preset list to every connected client (after save / delete). |

### NUI Callbacks (internal)

These callbacks are registered on the client and used by the NUI panel. They are listed for transparency, not as a public API.

| Callback | Payload | Purpose |
|---|---|---|
| `close` | — | Closes the editor and releases NUI focus. |
| `requestRefresh` | — | Re-sends the current state to the NUI. |
| `updateField` | `{ key, value }` | Apply a scalar / flag change. |
| `updateVector` | `{ key, x, y, z }` | Apply a vector change. |
| `resetField` | `{ key }` | Reset one field to its captured baseline. |
| `resetAll` | — | Reset every field to baseline. |
| `setPreview` | `{ enabled }` | Open / close the telemetry overlay. |
| `resetPreviewRun` | — | Reset the active timing run, optionally promoting it to `lastRun`. |
| `captureBaseline` | — | Save the current run as the baseline. |
| `clearBaseline` | — | Drop the saved baseline. |
| `togglePreviewMouse` | `{ enabled }` | Toggle the NUI cursor over the overlay. |
| `applyChanges` | — | Confirm / log the dirty set (no server-side side effects beyond a toast). |
| `saveProfile` | `{ name }` | Forward the current dirty values to the server. |
| `loadProfile` | `{ id }` | Apply a saved preset to the live vehicle. |
| `deleteProfile` | `{ id }` | Forward a deletion to the server. |
| `applySimplified` | `{ acceleration, topSpeed, transmission, drivetrain, traction, suspension, bodyRoll }` | Apply one or more simplified-preset selections. |

### Lifecycle

| Hook | Behaviour |
|---|---|
| `onResourceStart` (self) | Logs a warning for every resource in `Config.MeddlingResources` that's currently `started` or `starting`. |
| `onResourceStop` (self) | Flushes `profiles.json` to disk. |

---

## 8. Troubleshooting

### "Failed to read the original values" toast

The probe-vehicle spawn timed out (5 s default) — usually because the model isn't streamed in (e.g. an add-on car the local client hasn't downloaded yet) or the server is under heavy load. Drive a streamed-in vehicle once to confirm; if it persists, raise the timeout in `client/main.lua → ensureModelLoaded`.

### "Values may be getting overwritten by another script"

After every write, the editor reads the field back. If the read-back doesn't match what was just written, something else is touching the field on a tick. The most common offenders are listed in `Config.MeddlingResources`. Stop the named resource and retry — if values stay put, you found the culprit.

### Speed changes don't stick

Top speed is gated by FiveM's internal max-speed cache, not just `fInitialDriveMaxFlatVel`. The editor automatically calls a refresh routine when one of the speed-sensitive fields changes (drive force, drag, drive inertia, max flat velocity, gear count, clutch shift rates). If your tune touches none of those but speed still feels capped, run a stunt mod (`SetVehicleCheatPowerIncrease`) elsewhere and reload the vehicle, or trigger any of those fields with a 0.001 nudge to force the refresh.

### "Preset not found" after saving

Profiles are pushed to every connected client via `clads_handlingeditor:client:profilesUpdated`, so the dropdown should refresh in under a second. If a player saves a preset and another player can't see it, check that the dispatch is hitting `-1` (broadcast) — the shipped server does this, but a custom fork might have narrowed it.

### Editor opens but sliders are blank

The NUI never received the field array. Open the F8 console and look for fetch errors against `https://cfx-nui-clads_handlingeditor/...`. Most often this means the `web/build/` folder is missing or the resource was forked without rebuilding it. Run `cd web && npm install && npm run build` from the resource root and restart.

### Permission ace passes but the toast still says "no permission"

The client checks happen on `Config.RequireAce` only. If you flipped the flag without restarting the resource, the new value isn't in the bridge yet — `restart clads_handlingeditor` after toggling.

---

## 9. Support

Reach out via the Creative Lads Discord — open a ticket from the panel in `#create-ticket`.
