# Creative Lads — Boxing (`clads_boxing`)

A self-contained boxing arena resource for FiveM. Players walk up to a marker,
open a NUI menu, join one of two corners, optionally enable gloves and rounds,
and step into a 1v1 melee match with damage scaling, a HUD timer, point-based
round scoring, and an in-built betting pool that anyone in the lobby can pay
into during a configurable countdown.

The resource is framework-agnostic and is wired through `community_bridge`,
allowing it to run on QBCore, QBox, ESX, or in standalone mode without forks.
On standalone the betting layer disables itself silently — matches still work,
but no money changes hands.

---

## 1. Overview

`clads_boxing` is a lightweight activities resource. It does not try to be a
full fight-club ecosystem; it ships two arenas (Tequila La-La and Vespucci
Beach), a marker-based interaction, a React NUI lobby, glove props that
attach to the player's hands, configurable damage multipliers, and a parimutuel
betting pool that splits the losing side's pot among the winning side
proportionally.

**Who it's for**

- Roleplay servers that want a clean boxing minigame players can run between
  jobs without spinning up a dedicated fight resource.
- Drift / lifestyle servers running events, where the betting pool acts as
  side-action income for the host.
- Standalone or hybrid setups that need a melee arena that does not require
  inventory, target, or banking dependencies beyond `community_bridge`.

**Feature highlights**

- Two shipping arenas at Tequila La-La and Vespucci Beach with full corner
  coordinates; add more entries to `Config.Arenas` to define your own.
- 1-to-5 round matches with point-based scoring, HUD round/timer overlay, and
  an automatic decision when either fighter drops below 25 HP.
- Optional glove mode that attaches `prop_boxing_glove_01` to both hands and
  applies a softer damage multiplier so matches last longer.
- Parimutuel betting pool with a configurable countdown, multi-account
  payment fallback (cash/bank), refund-on-tie, and refund-on-walkout logic.
- Theme block that pushes every accent colour into the NUI's CSS variables at
  runtime — no rebuild required.
- JSON locales (en/tr/de/fr/es) with `clads_locale` → `ox:locale` → `qb_locale`
  → `lang` resolution and per-arena label overrides.
- Single export (`isBoxing`) so other resources can gate behaviour while a
  player is mid-match.
- Admin reset command for stuck arenas, plus automatic cleanup on
  `playerDropped` and `community_bridge:Server:OnPlayerUnload`.

---

## 2. Requirements

### Hard dependencies

Pulled directly from `fxmanifest.lua:43-46`:

```lua
dependencies {
    'community_bridge',
    'ox_lib',
}
```

| Dependency        | Purpose                                                       |
| ----------------- | ------------------------------------------------------------- |
| `community_bridge` | Framework, notify, and account-balance abstraction.          |
| `ox_lib`          | Callbacks, notifications, `lib.addCommand`, text UI.          |

There is no SQL requirement. Match state lives in memory only — winners and
totals are not persisted.

### Frameworks (auto-detected)

`Config.Framework = 'auto'` defers to `community_bridge`. The bridge probes
for `qbx_core`, `qb-core`, then `es_extended`; otherwise it returns standalone.
Override with `'qb'`, `'qbox'`, `'esx'`, or `'standalone'` if you need to pin
the choice (`config.lua:22`).

| Framework    | Money APIs used                                                 | Notes                                                  |
| ------------ | --------------------------------------------------------------- | ------------------------------------------------------ |
| `qbox` / `qb` | `Framework.GetAccountBalance` / `RemoveAccountBalance` / `AddAccountBalance` | `'money'` is mapped to `'cash'` automatically by the bridge. |
| `esx`        | Same as above via `community_bridge`                            | First account in `Config.PaymentAccounts` with funds is charged. |
| `standalone` | None                                                            | Betting toggle is hidden in the UI; `notify_betting_disabled` if forced. |

### Optional integrations (auto-routed by community_bridge)

- **Notifications:** `ox_lib`, `okok`, `ps-ui`, `lation_ui`, `qb`, `esx` — the
  resource calls `Bridge.Notify.SendNotification` first and falls back to
  `ox_lib:notify`.
- **Locales:** ships with `en`, `tr`, `de`, `fr`, `es` JSON files; English is
  the source of truth and the fallback for missing keys.

---

## 3. Installation

1. **Download** the resource and copy the `clads_boxing/` folder into your
   server's `resources/` directory (commonly `resources/[scripts]/clads_boxing`).
2. **Install dependencies** so they start before this resource:
   - `community_bridge`
   - `ox_lib`
3. **Start the resource.** Add to `server.cfg` after dependencies:
   ```cfg
   ensure community_bridge
   ensure ox_lib
   ensure clads_boxing
   ```
4. **Test.**
   - Drive or walk to one of the configured arenas — by default
     `vec3(-561.27, 283.99, 77.67)` (Tequila La-La) or
     `vec3(-1270.30, -1531.93, 4.31)` (Vespucci Beach).
   - The floor marker should appear with a `[E] — Boxing Menu` text-UI hint.
   - Press **E** to open the lobby. Have a second test client join slot 2,
     hit **Start**, and verify the countdown plays before the round begins.

> No SQL setup is needed. Match state is rebuilt from `Config.Arenas` on every
> server restart (`server/main.lua:41-45`).

---

## 4. Configuration

`config.lua` is split into eight commented sections. Headings below mirror the
order in the file.

### 4.1 Framework

Defined in `config.lua:22`.

```lua
Config.Framework = 'auto'   -- 'auto' | 'esx' | 'qb' | 'qbox' | 'standalone'
```

`auto` is recommended; pin it only if you run multiple cores side by side or
want to force standalone for testing. With `'standalone'` the betting toggle
is hidden client-side because `bettingAvailable()` returns false
(`server/main.lua:108-112`).

### 4.2 Locale

Defined in `config.lua:31`.

```lua
Config.Locale = 'auto'   -- 'auto' | 'en' | 'tr' | 'de' | 'fr' | 'es'
```

`auto` resolves through convars in this order: `clads_locale` → `ox:locale` →
`qb_locale` → `lang` → `en` (`shared/locale.lua:20-30`). Drop additional files
into `locales/<code>.json` to add new languages — English remains the
hard fallback for any missing key.

### 4.3 Debug

Defined in `config.lua:36`.

```lua
Config.Debug = false
```

Reserved hook — currently only used as a future toggle for verbose logs.

### 4.4 UI Theme

Defined in `config.lua:46-62`. Every value is a 6-digit hex colour with a
leading `#`. The web bundle injects the values into `:root` at runtime, so
restyles take effect on resource restart — no rebuild required.

```lua
Config.UI = {
    primary       = '#9D4EDD',
    primaryLight  = '#C77DFF',
    primaryDark   = '#7B2CBF',
    action        = '#FFEA00',
    actionDark    = '#CCBB00',
    secondary     = '#00E5FF',
    secondaryDark = '#00B8D4',
    danger        = '#FF2A6D',
    dangerDark    = '#C2185B',
    success       = '#00E676',
    bgCore        = '#0D0B14',
    bgSurface     = '#1F1B29',
    bgElevated    = '#2A2438',
    textPrimary   = '#FFFFFF',
    textMuted     = '#B5B5B5',
}
```

| Key             | Used for                                                         |
| --------------- | ---------------------------------------------------------------- |
| `primary*`      | Title glow, sliders, primary accents                             |
| `action*`       | CTA buttons (JOIN, START, BET)                                   |
| `secondary*`    | Player 2 accents (blue corner)                                   |
| `danger*`       | Player 1 accents (red corner) and the Leave button               |
| `success`       | Healthy timer, success states                                    |
| `bg*`           | Layered backgrounds (core / surface / elevated)                  |
| `textPrimary`   | Default text colour                                              |
| `textMuted`     | Subtitle / muted labels                                          |

> The default marker colour `r=157, g=78, b=221` matches `Config.UI.primary`.
> If you change the primary, you may want to update `Config.Marker.color` to
> match (`config.lua:118`).

### 4.5 Display

Defined in `config.lua:69-72`.

```lua
Config.NameSource = 'character'   -- 'character' | 'steam'
Config.MoneyForm  = '$'
```

`character` reads the framework's first/last name via `Framework.GetPlayerName`.
On standalone or when no character record exists, the resource falls back to
`GetPlayerName(src)` (Steam handle) automatically (`server/main.lua:72-81`).
`steam` forces the Steam handle even when a framework record is present.

`MoneyForm` is the symbol shown next to bet amounts inside notifications.

### 4.6 Betting

Defined in `config.lua:84-90`.

```lua
Config.PaymentAccounts = { 'cash', 'bank' }
Config.TimeToBet       = 30   -- seconds
Config.MinBet          = 1
```

| Key                 | Type    | Default          | Description                                                               |
| ------------------- | ------- | ---------------- | ------------------------------------------------------------------------- |
| `PaymentAccounts`   | table   | `{ 'cash', 'bank' }` | Order in which accounts are checked. The first account with enough balance is charged. |
| `TimeToBet`         | number  | `30`             | Countdown seconds between match start and round 1, while bets are open.    |
| `MinBet`            | number  | `1`              | Minimum accepted amount; values below this trigger `notify_invalid_bet`.   |

Server-side guardrails (`server/main.lua:281-288`) also reject any bet above
`1e9` (a sanity cap), any non-numeric amount, or a duplicate bet from the same
player on the same arena. The pool is parimutuel — winners split the losing
side's pot proportionally to their stake, and on a tie or walkout every bet is
refunded to the originating account (`server/main.lua:246-271`).

### 4.7 Combat tuning

Defined in `config.lua:97-105`.

```lua
Config.DamageModifier = {
    enabled = true,
    basic   = 0.25,   -- bare-knuckle damage multiplier
    glove   = 0.15,   -- gloved damage multiplier (lower = longer matches)
}

Config.DisableControls = { 22 }   -- 22 = INPUT_JUMP
```

The multiplier is applied via `N_0x4757f00bc6323cfe(joaat('WEAPON_UNARMED'), mult)`
on round start and reset to `basic` on match end (`client/main.lua:264-267`,
`:324-326`). Disable the whole block with `enabled = false` if your server
balances melee damage globally.

`DisableControls` lists the input IDs blocked while the player is mid-round.
Jump (22) is on the default list to prevent escaping the ring. Add more IDs if
you want to disable e.g. melee block (140), aim (24) or weapon wheel (12).

### 4.8 Interaction marker

Defined in `config.lua:112-124`.

```lua
Config.Marker = {
    enabled       = true,
    type          = 20,
    bobUpAndDown  = true,
    rotate        = false,
    size          = vec3(0.3, 0.2, 0.2),
    color         = { r = 157, g = 78, b = 221, a = 200 },
    drawDistance  = 10.0,
    interactRange = 2.0,
}

Config.OpenMenuKey = 38   -- 38 = E
```

| Key             | Type    | Default                    | Description                                                                  |
| --------------- | ------- | -------------------------- | ---------------------------------------------------------------------------- |
| `enabled`       | bool    | `true`                     | Set to `false` if you wire your own target / blip system.                    |
| `type`          | number  | `20`                       | Marker type passed to `DrawMarker` — see FiveM marker reference.             |
| `bobUpAndDown`  | bool    | `true`                     | Vertical bob animation.                                                      |
| `rotate`        | bool    | `false`                    | Continuous Y-axis rotation.                                                  |
| `size`          | vec3    | `vec3(0.3, 0.2, 0.2)`      | Marker scale (X, Y, Z).                                                      |
| `color`         | table   | `{r=157,g=78,b=221,a=200}` | RGBA tint — defaults to `Config.UI.primary`.                                 |
| `drawDistance`  | number  | `10.0`                     | Maximum distance at which the marker is rendered.                            |
| `interactRange` | number  | `2.0`                      | Distance inside which the [E] hint shows and the menu can be opened.         |
| `OpenMenuKey`   | number  | `38`                       | Control ID for opening the menu — passed straight to `IsControlJustReleased`. |

### 4.9 Arenas

Defined in `config.lua:136-153` (`Config.Arenas`). Each entry is a flat table:

```lua
{
    id      = 'tequila',
    label   = 'arena_tequila',
    time    = 60,
    start   = vec3(-561.2779, 283.9954, 77.6763),
    player1 = vec4(-554.7122, 281.7168, 78.5265, 38.8104),
    player2 = vec4(-558.6957, 286.1812, 78.5265, 221.8874),
}
```

| Field      | Type   | Description                                                                                                       |
| ---------- | ------ | ----------------------------------------------------------------------------------------------------------------- |
| `id`       | string | Stable identifier used by callbacks, the `/resetarena` command, and internal arena state.                          |
| `label`    | string | Either a translation key (e.g. `arena_tequila`) or a hardcoded literal string. Resolved through `Locale.resolve`. |
| `time`     | number | Seconds per round before the match auto-judges on remaining HP.                                                    |
| `start`    | vec3   | Floor coordinates for the marker / interaction point.                                                              |
| `player1`  | vec4   | Spawn coordinates for the player joining slot 1 (red corner) — XYZ + heading.                                       |
| `player2`  | vec4   | Spawn coordinates for the player joining slot 2 (blue corner) — XYZ + heading.                                      |

Two arenas ship by default:

| ID        | Location          | Coords                                  |
| --------- | ----------------- | --------------------------------------- |
| `tequila` | Tequila La-La (interior boxing ring) | `vec3(-561.27, 283.99, 77.67)` |
| `beach`   | Vespucci Beach   | `vec3(-1270.30, -1531.93, 4.31)`        |

Each arena runs its own match state independently — players in `tequila` do
not block players in `beach`. Add as many entries to `Config.Arenas` as you
like; the server initialises a blank state for each one on startup
(`server/main.lua:41-45`).

---

## 5. Commands

| Command           | Source file              | Permission                | Description                                                                |
| ----------------- | ------------------------ | ------------------------- | -------------------------------------------------------------------------- |
| `/resetarena <id>` | `server/main.lua:367`   | `group.admin` (Ace)       | Force-resets the named arena's state. Use when a player crashes mid-match and the slot is stuck. |

The command is registered through `lib.addCommand` so it inherits ox_lib's
chat suggestions and parameter help (`cmd_resetarena_help`,
`cmd_resetarena_arg_help`). Pass the arena `id` from `Config.Arenas`, e.g.
`/resetarena tequila` or `/resetarena beach`.

There are no client-side chat commands — all player-facing interactions go
through the marker prompt and NUI menu.

---

## 6. Customization

### Theme

Edit `Config.UI` in `config.lua:46-62`. Values are pushed into the NUI's
`:root` CSS variables on menu open via `pushTheme()` in
`client/main.lua:59-61`, so changing a colour just needs a resource restart —
no rebuild required. To match the marker tint to your primary, also update
`Config.Marker.color` (`config.lua:118`).

### Locales

Five locales ship in `locales/`: `en.json`, `tr.json`, `de.json`, `fr.json`,
`es.json`. To add a new language, copy `en.json`, translate the values, save
as `locales/<code>.json`, and either set `Config.Locale = '<code>'` or rely on
the convar chain (`clads_locale`, `ox:locale`, `qb_locale`, `lang`).

Arena labels can be either a locale key (e.g. `arena_tequila`) or a literal
string — `Locale.resolve` (`shared/locale.lua:76-80`) returns the value if it
exists, otherwise the key itself. So you can quickly override one label with
`label = 'My Custom Ring'` without touching JSON.

### Adding a new arena

Append a new table to `Config.Arenas`:

```lua
Config.Arenas[#Config.Arenas + 1] = {
    id      = 'warehouse',
    label   = 'Warehouse Boxing',
    time    = 90,
    start   = vec3(123.4, 567.8, 30.0),
    player1 = vec4(120.0, 565.0, 30.5, 90.0),
    player2 = vec4(127.0, 570.0, 30.5, 270.0),
}
```

The server creates a fresh match state for the new key on next start
(`server/main.lua:41-45`). Use a unique `id` — duplicates will silently
overwrite each other in `gArenas`.

### Disabling betting

Either set `Config.Framework = 'standalone'` (hides the toggle for everyone),
or run on standalone naturally — `bettingAvailable()` checks for
`Framework.GetAccountBalance` and returns false if the bridge has no money
APIs hooked up (`server/main.lua:108-112`). The UI then hides the bet toggle
via the `canBet` flag passed in the `open` NUI message (`client/main.lua:83`).

### Asset paths

The web bundle lives in `web/dist/`. The logo at `web/dist/logo.png` is one of
three files in `escrow_ignore` (`fxmanifest.lua:52-56`), so you can swap it
for your own logo without touching the encrypted JS.

---

## 7. Exports & Events

### Client export

Defined in `client/main.lua:347-348` and declared in `fxmanifest.lua:48`.

| Export                          | Purpose                                              |
| ------------------------------- | ---------------------------------------------------- |
| `exports.clads_boxing:isBoxing()` | Returns `true` while the local player is mid-round.  |

Use this from another resource to gate behaviour (e.g. block radio use,
inventory access, or vehicle entry) while the player is fighting.

### Notable client net events

All under the `clads_boxing:client:` namespace.

| Event                                  | Purpose                                                                                  |
| -------------------------------------- | ---------------------------------------------------------------------------------------- |
| `clads_boxing:client:menuRefresh`      | Pushes new arena state to anyone with the menu open.                                     |
| `clads_boxing:client:joinAck`          | Confirms a join request to the calling player.                                            |
| `clads_boxing:client:betTimer`         | Broadcasts the bet-window countdown to everyone watching the lobby.                       |
| `clads_boxing:client:startRound`       | Teleports the player to their corner, attaches gloves if enabled, starts the round timer. |
| `clads_boxing:client:matchEnd`         | Closes the HUD, returns the player to the start marker, plays the end notification.       |

### Notable server net events / callbacks

| Name                                          | Type     | Purpose                                                                |
| --------------------------------------------- | -------- | ---------------------------------------------------------------------- |
| `clads_boxing:server:getArena`                | Callback | Returns the current arena state for the given `id`.                     |
| `clads_boxing:server:bettingAvailable`        | Callback | Returns `true` when framework money APIs are hooked up.                 |
| `clads_boxing:server:join`                    | Event    | Adds the caller to a free corner of the named arena.                    |
| `clads_boxing:server:leave`                   | Event    | Removes the caller from their slot if the match has not started.        |
| `clads_boxing:server:start`                   | Event    | Locks the lobby, optionally opens the bet window, then starts round 1.  |
| `clads_boxing:server:bet`                     | Event    | Validates funds, charges the player, and credits the chosen pool.       |
| `clads_boxing:server:roundOver`               | Event    | Awards a round point and either starts the next round or ends the match. |
| `clads_boxing:server:removeJoin`              | Event    | Voluntary lobby leave fired when the menu is closed before joining.     |

---

## 8. Troubleshooting

### Marker won't appear

- Confirm `Config.Marker.enabled = true` (`config.lua:113`).
- Check that the player is within `Config.Marker.drawDistance` (default
  `10.0`) of `arena.start`.
- Verify `community_bridge`, `ox_lib`, and `clads_boxing` all started without
  errors in the server console.
- If you replaced the marker with a target / zone system, the hint text key is
  `ui_press_e` and the keybind is `Config.OpenMenuKey` (default 38 / E).

### Menu opens but Bet toggle is missing

The toggle is hidden when `bettingAvailable()` returns false — the bridge has
no `GetAccountBalance` / `RemoveAccountBalance` / `AddAccountBalance` APIs
(`server/main.lua:108-112`). Confirm:

- `Config.Framework` is not `'standalone'`.
- The detected framework (qb / qbox / esx) is loaded and exposing money APIs
  through `community_bridge`.
- The menu picks up the flag from the `canBet` field of the `open` NUI
  message (`client/main.lua:83`). Reload the menu after starting the bridge.

### "Betting is disabled on this server" notification

Same root cause as above — the server received a `bet` event but
`bettingAvailable()` returned false. Either fix the framework hook or hide the
button by setting `Config.Framework = 'standalone'`.

### Player gets stuck in the arena

The `/resetarena <id>` admin command (`server/main.lua:367`) clears that
arena's state and broadcasts a refresh. The client teleports back to the
arena's `start` coords on `matchEnd`; if a player crashed mid-round and the
HUD never closed, they will be stuck. Use `/resetarena` to wipe the slot.

### Glove props left behind

`onResourceStop` in `client/events.lua:8-13` deletes any tracked glove props on
restart. If a player rejoins and finds a stuck glove near their ped, the
`community_bridge:Client:OnPlayerLoaded` handler at `client/events.lua:15-28`
sweeps a 3-metre radius for `prop_boxing_glove_01` and deletes anything it
finds.

### Round never ends

Round resolution is reported by player 1 only (`client/main.lua:297-311`). If
player 1 disconnects mid-round, `removePlayer` ends the match and notifies
player 2 (`server/main.lua:321-349`). If player 2 disconnects, the match
likewise ends with no winner. If neither disconnects but the round just hangs,
the most likely cause is the player 1 client failing to reach
`elapsed >= arenaCfg.time` — restart the resource and use `/resetarena` to
wipe the lobby.

### Damage feels too high / too low

Adjust `Config.DamageModifier.basic` and `Config.DamageModifier.glove`
(`config.lua:97-101`). The values are direct multipliers on the unarmed
weapon's damage stat, applied per-round and reset on match end. Disable the
whole modifier with `enabled = false` if your server already balances melee
globally.

### Names show as "Player 12"

`resolveName` first calls `Framework.GetPlayerName` and falls back to
`GetPlayerName(src)` (`server/main.lua:72-81`). On standalone with no Steam
handle the final fallback is `Player <serverId>`. Set `Config.NameSource =
'steam'` to force the Steam handle.

---

## 9. Support

Reach out via the Creative Lads Discord.
