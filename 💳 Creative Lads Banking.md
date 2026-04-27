# 💳 Creative Lads Banking

A complete banking experience for your server: a glass-morphism bank UI, world-walkable ATMs with PIN security, society / faction accounts, IBAN transfers, transaction history, and a runtime-themable color palette.

Framework-agnostic. Drops into ESX, QBCore, QBox, or standalone setups through [`community_bridge`](https://github.com/The-Order-Of-The-Sacred-Framework/community_bridge). Existing scripts that talk to `Renewed-Banking` exports keep working without changes.

## Overview

Built for owners that want one bank for everybody — citizens, police, ambulance, gangs — without bolting on a second resource for each role.

- **Bank pins** at every major bank in Los Santos (8 default locations).
- **ATM model targeting** for the four GTA ATM hashes — animated card-insert, PIN pad, and a fast jump from PIN-correct → full UI.
- **IBAN system** with auto-generated codes, custom prefix and configurable length.
- **PIN code** with optional "first-time-free" policy.
- **Society accounts** for any job/gang in your framework, with optional rank gating. The Renewed-Banking exports surface is preserved for backward compatibility.
- **Transactions ledger** — searchable table, 7-day rolling chart, income / outcome / net stats.
- **Themable palette** — every color is a CSS variable; flip the whole bank from cyber-cyan to dark-red without touching the bundle.
- **Brandable** — set a bank name, optional logo PNG, currency symbol, and locale-aware amount formatting.
- **Fees and limits** — flat + percent fees per action, min/max amount clamps, per-source rate-limit cooldown.

## Requirements

| Dependency | Why |
| --- | --- |
| `community_bridge` | Framework, money accounts, notifications, targeting. |
| `ox_lib` | Callbacks and notification fallback. |
| `oxmysql` | Persists IBAN, PIN, society accounts, transactions. |

QB-target / ox-target / sleepless_interact are auto-picked through community_bridge — no manual flag.

## Installation

1. Drop `clads_banking` into your `resources/` folder.
2. Make sure `community_bridge`, `ox_lib`, and `oxmysql` start before it.
3. `ensure clads_banking`. The first start auto-runs the schema:
   - `players.iban` (VARCHAR)
   - `players.pincode` (INT)
   - `clads_banking_societies`
   - `clads_banking_transactions`
4. If your setup forbids `ALTER` on the players table, set `Config.Database.external = true` — the resource then maintains its own `clads_banking_player` table for IBAN/PIN.
5. Walk up to a bank pin or any ATM model. The first time a player opens the bank, an IBAN is generated automatically.

## Configuration

All options live in `config.lua` at the resource root. The file is left editable after escrow (`escrow_ignore`).

### Branding

```lua
Config.Branding = {
    bankName     = 'Creative Pay',
    bankTagline  = 'Your money, your rules.',
    logo         = '',          -- 'logo.png' to load /web/build/logo.png
    currency     = '$',
    currencyAfter = false,      -- 'USD 1,000' instead of '$1,000'
    locale       = 'en-US',     -- toLocaleString locale
}
```

Drop `web/build/logo.png` into the resource folder (256x256 PNG, transparent background) and set `logo = 'logo.png'`. The icon is replaced everywhere a brand row is rendered.

### Theme

`Config.UI` exposes the full color palette. Each value is forwarded to the bundle on every open and applied to a CSS variable on `:root`. Two preset palettes are commented inside `config.lua`:

| Token | Default | Purpose |
| --- | --- | --- |
| `primary` | `#9D4EDD` | Brand accent (sidebar, buttons, dots). |
| `primaryHover` | `#B47AE8` | Lighter brand on hover. |
| `primaryDim` | `rgba(157, 78, 221, 0.15)` | Subtle tint behind cards. |
| `primaryGlow` | `rgba(157, 78, 221, 0.4)` | Outer glow on focus rings. |
| `action` | `#FFEA00` | Wallet badge, transfer button. |
| `secondary` | `#00E5FF` | Withdraw accent. |
| `danger` | `#FF2A6D` | Logout, error states. |
| `success` | `#00E676` | Deposits, positive amounts. |
| `bgCore` / `bgSurface` / `bgElevated` / `bgGlass` | Dark grays | Layered surfaces. |
| `textPrimary` / `textSecondary` / `textMuted` | White tones | Type ramp. |
| `borderSubtle` / `borderGlow` | Brand-tinted | Dividers. |

Set any token to `nil` (or remove the line) to keep the default. The change applies on the next bank open — no resource restart needed.

### IBAN

```lua
Config.IBAN = {
    prefix       = 'BANK',
    numbers      = 6,
    allowLetters = true,
    maxChars     = 10,
    changeCost   = 6250,
}
```

`numbers` controls auto-generated IBANs (`BANK482910`). `maxChars` limits the editable suffix when a player buys a custom IBAN from the Settings tab. Setting `allowLetters = false` forces digits-only customs.

### PIN

```lua
Config.PIN = {
    length     = 4,
    changeCost = 1250,
    firstFree  = true,   -- first PIN set is free, subsequent edits cost changeCost
}
```

### Fees and limits

```lua
Config.Fees = {
    deposit         = { flat = 0, percent = 0.0 },
    withdraw        = { flat = 0, percent = 0.0 },
    transfer        = { flat = 0, percent = 0.0 },
    societyTransfer = { flat = 0, percent = 0.0 },
}

Config.Limits = {
    minDeposit       = 1, maxDeposit       = 0,
    minWithdraw      = 1, maxWithdraw      = 0,
    minTransfer      = 1, maxTransfer      = 0,
    actionCooldownMs = 350,
}
```

`actionCooldownMs` is the per-source debounce on duplicate clicks. `0` disables it.

### Society accounts

```lua
Config.Societies = {
    list        = { 'police', 'ambulance' },
    accessRanks = {},        -- empty = every member of the job
    autoCreate  = true,
}
```

When `autoCreate = true`, the first time a member of a listed job opens the bank, the resource creates `society_<job>` with the framework-supplied job label and an IBAN of `<prefix>..<JOB_NAME_UPPER>`.

### Bank locations and ATMs

```lua
Config.BankLocations = {
    { x = 150.266, y = -1040.203, z = 29.374, interactDistance = 3.0, blip = 108, blipColor = 2, blipScale = 0.9 },
    -- ...
}

Config.ATMs = {
    interactDistance = 1.5,
    animMs           = 2000,
    models = {
        { hash = -870868698 },
        { hash = -1126237515 },
        { hash = -1364697528 },
        { hash = 506770882 },
    },
}
```

Set `Config.ShowBlips = true` to render bank blips on the map. Most servers leave this off and let a radial-menu / minimap script handle it.

### Database

```lua
Config.Database = {
    autoMigrate         = true,
    societiesTable      = 'clads_banking_societies',
    transactionsTable   = 'clads_banking_transactions',
    playerIdColumn      = 'citizenid',     -- 'identifier' on ESX
    external            = false,
    externalPlayerTable = 'clads_banking_player',
}
```

Disable `autoMigrate` if you prefer to run `sql/install.sql` by hand. Setting `external = true` skips the `ALTER TABLE players` and uses a dedicated table — useful on tightly-locked ESX setups.

## Commands

| Command | Permission | Description |
| --- | --- | --- |
| `/paytransfer <id> <amount>` | All players | Hand cash to a player within `Config.Commands.payTransfer.maxDistance`. |
| `/clads_banking_setpin <serverId> <pin>` | `group.admin` | Force a PIN reset for a player. |
| `/clads_banking_resettx <serverId>` | `group.admin` | Wipe a player's transaction history. |

Any command can be disabled via `Config.Commands.<name>.enabled = false`.

## Customization

- **Theme** — override `Config.UI` (above). Two preset palettes are commented inside `config.lua`. Override applies on the next bank open.
- **Locales** — drop a new file at `locales/<code>.json`, mirror the keys in `en.json`, and set `clads_locale "<code>"` (or `setr ox:locale "<code>"`). Five locales ship by default: `en`, `tr`, `de`, `fr`, `es`.
- **Logo** — drop `web/build/logo.png` (256x256 PNG, transparent background) and set `Config.Branding.logo = 'logo.png'`. The path is in `escrow_ignore`, so swapping the file does not require an unpack.

## Exports & Events

### Player accounts

```lua
exports.clads_banking:GetPlayerBank(src)         -- number
exports.clads_banking:GetPlayerCash(src)         -- number
exports.clads_banking:AddPlayerBank(src, amount) -- boolean
exports.clads_banking:RemovePlayerBank(src, amount) -- boolean
exports.clads_banking:TransferPlayerToPlayer(srcSrc, targetCid, amount, txType)
```

### IBAN / PIN

```lua
exports.clads_banking:GetPlayerIban(src)
exports.clads_banking:SetPlayerIban(src, iban)   -- false if iban already in use
exports.clads_banking:GetPlayerPin(src)
exports.clads_banking:SetPlayerPin(src, pin)
```

### Society accounts

```lua
exports.clads_banking:GetSocietyMoney(name)
exports.clads_banking:AddSocietyMoney(name, amount)
exports.clads_banking:RemoveSocietyMoney(name, amount)
```

### Renewed-Banking compatibility

Resources that hard-code `Renewed-Banking` exports work as-is:

```lua
exports['Renewed-Banking']:getAccountMoney(name)
exports['Renewed-Banking']:addAccountMoney(name, amount)
exports['Renewed-Banking']:removeAccountMoney(name, amount)
```

### Server net events

Triggered from the NUI; not intended to be raised by other resources. They re-validate everything server-side regardless of caller.

| Event | Payload |
| --- | --- |
| `clads_banking:server:deposit` | `(amount)` |
| `clads_banking:server:withdraw` | `(amount)` |
| `clads_banking:server:transfer` | `(amount, iban)` |
| `clads_banking:server:depositSociety` | `(amount, jobName)` |
| `clads_banking:server:withdrawSociety` | `(amount, jobName)` |
| `clads_banking:server:updateIban` | `(iban)` |
| `clads_banking:server:updatePin` | `(pin)` |

### Client net events (server → client)

| Event | Effect |
| --- | --- |
| `clads_banking:client:syncBalances` | Updates wallet + bank balance in the open UI. |
| `clads_banking:client:syncIban` | Updates the IBAN displayed in the UI. |
| `clads_banking:client:refreshOverview` | Reloads the overview tab. |
| `clads_banking:client:refreshSociety` | Reloads the society tab. |
| `clads_banking:client:payAnimation` | Plays the give-cash animation after `/paytransfer`. |

## Troubleshooting

- **The bank UI does not render.** Re-download the resource from your Tebex account — the bundle ships pre-built. If the issue persists, open a ticket.
- **`No society found` when a faction member opens the bank.** Check `Config.Societies.list` includes the job name (not the label). Set `autoCreate = true` to spawn the row on first access.
- **Players see another player's IBAN listed as taken when they pick a custom IBAN.** Expected — IBANs are unique across the whole server. They need to choose a different suffix.
- **My ESX server can't `ALTER` the players table.** Set `Config.Database.external = true`. The resource then keeps IBAN/PIN in `clads_banking_player`, keyed on the framework identifier.
- **PIN resets are not free for first-time users.** Set `Config.PIN.firstFree = true`.
- **The Society "Withdraw" button refuses every attempt.** Another member is mid-withdraw and the row is `is_withdrawing = 1`. The flag clears once the previous withdraw completes; on a server crash, the flag is force-cleared on next startup.

## Support

- **Documentation:** <https://creative-lads.gitbook.io/creative-lads-docs/>
- **Tickets:** open a ticket via the panel in `#create-ticket` in the Creative Lads Discord.
