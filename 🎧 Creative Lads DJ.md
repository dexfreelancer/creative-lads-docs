# clads_dj

Creative Lads — DJ. A framework-agnostic, modular DJ console for FiveM with placeable decks, surround speaker grouping, queue management, plug-and-play USB items, and a megaphone-style microphone broadcast.

---

## 1. Overview

`clads_dj` is an admin-placeable DJ system built around two cooperating prop archetypes — a **deck** (the control surface) and one or more **speakers** (audio outputs). Decks are persistent across server restarts via MySQL, and the entire stack runs through `community_bridge` so the same build works on Standalone, ESX, QBCore, and QBox without changing exports.

Highlights:

- **Admin-placeable props** — placement is gated by an ACE permission. Admins raycast a position with the gameplay camera and confirm with `E`. Both the deck and the speaker share the same placement flow.
- **Modular audio mixing per deck** — each deck owns its own volume, falloff distance, mute, and pause state. Tracks are streamed via `xsound` URL playback and routed to every speaker within `Config.DeskSpeakerRadius` of the deck.
- **Queue system** — capped at `Config.MaxQueueSize` tracks per deck. Tracks support reorder, remove, jump-to-play, "Play Now", and auto-advance when the active track ends. YouTube URLs auto-resolve their title via the oEmbed endpoint when the operator does not supply one.
- **USB plug-and-play** — when the player is carrying `Config.UsbItem` (default `music_usb`), the NUI exposes a USB shortcut on the composer. Inventory presence is checked through `community_bridge.Inventory`, so it works against any supported inventory backend without extra wiring.
- **Microphone broadcast** — a megaphone-style proximity boost combined with an audio submix (radio FX) attached to the broadcaster's right hand bone. When `pma-voice` is started the proximity range is overridden for the duration of the broadcast; without it the prop attachment + mumble submix still apply.
- **DJ idle animation** — a looping idle clip (`anim@amb@nightclub@djs@solomun@ / sol_idle_01` by default) plays whenever the panel is open, and is cleared on close or resource stop.
- **Persistent across restart** — every placed prop is written to `clads_dj_props` (auto-created on first boot). On startup the server respawns each prop, restores state, and resyncs to clients.
- **Framework-agnostic** — `community_bridge` handles framework, inventory, target, and notify abstraction. The `fxmanifest.lua` declares no hard framework dependency.
- **Themable NUI** — every visible color, gradient, border, and font on the console is wired to a `Config.Theme` token that pipes straight into a CSS variable at panel open time, so buyers can repaint without unpacking the bundle.

---

## 2. Requirements

| Requirement       | Type     | Purpose                                                                                  |
| ----------------- | -------- | ---------------------------------------------------------------------------------------- |
| `community_bridge` | Required | Framework / inventory / target / notify abstraction.                                     |
| `ox_lib`          | Required | Callbacks, context menus, notifications.                                                 |
| `oxmysql`         | Required | Persistence layer for placed DJ props.                                                   |
| `xsound`          | Runtime  | URL audio streaming. Resource warns and refuses to operate if `xsound` is not started.   |
| `ox_target` (or compatible) | Required (via bridge) | Target zones around each placed prop. `community_bridge.Target` picks the active backend (`ox_target`, `qb-target`, `sleepless_interact`). |
| Inventory backend | Required for USB | Any inventory supported by `community_bridge.Inventory` (`ox_inventory`, `qb-inventory`, `qs-inventory`, `origen_inventory`, `codem-inventory`, `tgiann-inventory`). The USB item must exist in your items DB. |
| `pma-voice`       | Optional | Megaphone proximity range override. When absent, the mic boost step is silently skipped and only the prop attachment + submix run. |
| Framework         | Optional | `community_bridge` auto-detects ESX / QBCore / QBox; standalone is fully supported.      |

---

## 3. Installation

1. **Place the resource** in your `resources/` directory, for example `resources/[creativelads]/clads_dj/`.
2. **Install dependencies** (`community_bridge`, `ox_lib`, `oxmysql`, your target backend, your inventory backend, `xsound`, optionally `pma-voice`).
3. **Add to `server.cfg`**, ensuring start order:

   ```cfg
   ensure oxmysql
   ensure ox_lib
   ensure community_bridge
   ensure xsound
   ensure pma-voice    # optional but recommended
   ensure clads_dj
   ```

4. **Add the USB item** to your inventory's items database. The default name is `music_usb`. Match the name to `Config.UsbItem`.
5. **Grant admin ACE.** Placement and removal are gated by `Config.AdminAce` (default `group.admin`). Either map a Discord role / identifier to that group, or change the ACE to your custom one.

   ```cfg
   add_ace group.admin command allow
   add_principal identifier.fivem:1234567 group.admin
   ```

6. **Start the server.** On first boot the resource creates the `clads_dj_props` table automatically — no SQL file to run.
7. **Confirm `xsound` is started** — without it audio playback is disabled and a debug log warning is printed when `Config.Debug = true`.

---

## 4. Configuration

All configuration lives in `config.lua`. The file is left out of escrow encryption so it can be edited freely after install.

### Framework

```lua
Config.Framework = 'auto'
```

| Value          | Behavior                                                              |
| -------------- | --------------------------------------------------------------------- |
| `'auto'`       | Let `community_bridge` pick. Recommended.                             |
| `'esx'`        | Force `es_extended`.                                                  |
| `'qb'`         | Force `qb-core`.                                                      |
| `'qbox'`       | Force `qbx_core`.                                                     |
| `'standalone'` | No framework. Anyone can place props (still gated by ACE), no charinfo lookup. |

### Locale

```lua
Config.Locale = 'auto'
```

Built-in: `en`, `tr`, `de`, `fr`, `es`. Set to `'auto'` to follow server convars in this order: `clads_locale` → `ox:locale` → `qb_locale` → `lang` → fallback `'en'`. Add a new file in `locales/<code>.json` to ship more languages.

### Debug

```lua
Config.Debug = false
```

Enables console logging for prop spawn failures, persisted prop count on boot, missing `xsound`, and legacy table migration messages.

### Commands

```lua
Config.EnableCommands = true
Config.Commands = {
    place  = 'djplace',
    manage = 'djmanage',
}
Config.AdminAce = 'group.admin'
```

| Key                    | Purpose                                                                                            |
| ---------------------- | -------------------------------------------------------------------------------------------------- |
| `Config.EnableCommands` | When `false`, both slash commands are unregistered. Players still receive the target prompt on placed props. |
| `Config.Commands.place` | Slash command that opens the placement menu (Desk / Speaker).                                     |
| `Config.Commands.manage` | Slash command that opens the panel for the closest placed prop.                                  |
| `Config.AdminAce`      | ACE permission required to place or remove props. Defaults to `group.admin`.                       |

### Placement & Interaction

```lua
Config.MaxPlaceDistance   = 25.0
Config.InteractionDistance = 3.0
Config.TargetRadius        = 1.75
Config.DeskSpeakerRadius   = 28.0
```

| Key                       | Default | Purpose                                                                                          |
| ------------------------- | ------- | ------------------------------------------------------------------------------------------------ |
| `Config.MaxPlaceDistance` | `25.0`  | Maximum raycast distance (meters) when placing a prop. Admins bypass the distance check.         |
| `Config.InteractionDistance` | `3.0` | Range (meters) at which the panel target option appears.                                         |
| `Config.TargetRadius`     | `1.75`  | Sphere radius (meters) of the `ox_target` zone wrapping each prop.                               |
| `Config.DeskSpeakerRadius` | `28.0` | Maximum distance (meters) from the deck a speaker can sit and still receive that deck's audio.   |

### Audio Mixing

```lua
Config.DefaultVolume   = 0.45
Config.DefaultDistance = 36.0
Config.MinVolume   = 0.0
Config.MaxVolume   = 1.0
Config.MinDistance = 8.0
Config.MaxDistance = 80.0
```

| Key                      | Default | Purpose                                                          |
| ------------------------ | ------- | ---------------------------------------------------------------- |
| `Config.DefaultVolume`   | `0.45`  | Volume (0.0 – 1.0) for newly placed decks.                       |
| `Config.DefaultDistance` | `36.0`  | Falloff distance (meters) for newly placed decks.                |
| `Config.MinVolume`       | `0.0`   | Lower bound exposed to the volume slider.                        |
| `Config.MaxVolume`       | `1.0`   | Upper bound exposed to the volume slider.                        |
| `Config.MinDistance`     | `8.0`   | Lower bound exposed to the distance slider.                      |
| `Config.MaxDistance`     | `80.0`  | Upper bound exposed to the distance slider.                      |

### Queue Limits

```lua
Config.MaxQueueSize        = 25
Config.MaxTrackTitleLength = 64
Config.MaxTrackUrlLength   = 512
```

| Key                            | Default | Purpose                                                  |
| ------------------------------ | ------- | -------------------------------------------------------- |
| `Config.MaxQueueSize`          | `25`    | Maximum tracks per deck queue (server-enforced).         |
| `Config.MaxTrackTitleLength`   | `64`    | Track title is truncated server-side.                    |
| `Config.MaxTrackUrlLength`     | `512`   | Track URL is rejected if longer.                         |

### Inventory

```lua
Config.UsbItem = 'music_usb'
```

| Key              | Default       | Purpose                                                                                                |
| ---------------- | ------------- | ------------------------------------------------------------------------------------------------------ |
| `Config.UsbItem` | `'music_usb'` | Item name used by the USB plug-and-play shortcut. The deck checks this through `community_bridge.Inventory`. Set to `''` (or any name not in your items DB) to disable the USB UI. |

### Props

```lua
Config.Props = {
    desk = {
        label     = 'prop_desk_label',
        model     = 'prop_dj_deck_02',
        invisible = true,
    },
    speaker = {
        label     = 'prop_speaker_label',
        model     = 'prop_speaker_05',
        invisible = false,
    },
}
```

| Key                            | Default              | Purpose                                                                                              |
| ------------------------------ | -------------------- | ---------------------------------------------------------------------------------------------------- |
| `Props.desk.label`             | `'prop_desk_label'`  | Locale key (or any literal string) used as the deck label in target prompts and the panel header.    |
| `Props.desk.model`             | `'prop_dj_deck_02'`  | Spawned model for the deck.                                                                          |
| `Props.desk.invisible`         | `true`               | When `true`, deck entity alpha is forced to `0` after spawn — the typical workflow is to skin the deck via clothing or an MLO and use the prop only as a hitbox/anchor. Set to `false` to render the model normally. |
| `Props.speaker.label`          | `'prop_speaker_label'` | Locale key used as the speaker label.                                                              |
| `Props.speaker.model`          | `'prop_speaker_05'`  | Spawned model for the speaker.                                                                       |
| `Props.speaker.invisible`      | `false`              | When `true`, hides the speaker model the same way as the deck.                                       |

### DJ Animation

```lua
Config.DjAnimation = {
    enabled = true,
    dict    = 'anim@amb@nightclub@djs@solomun@',
    clip    = 'sol_idle_01',
    flag    = 1,
}
```

| Key       | Default                                | Purpose                                                                                |
| --------- | -------------------------------------- | -------------------------------------------------------------------------------------- |
| `enabled` | `true`                                 | When `false`, no idle animation is played while the panel is open.                     |
| `dict`    | `'anim@amb@nightclub@djs@solomun@'`    | Animation dictionary requested at panel open.                                          |
| `clip`    | `'sol_idle_01'`                        | Clip played from the dictionary on the local ped.                                      |
| `flag`    | `1`                                    | `TaskPlayAnim` flag — `1` = repeat (default loop). See native docs for other flags.    |

### Microphone

```lua
Config.Microphone = {
    enabled       = true,
    cooldownMs    = 15000,
    durationMs    = 8000,
    voiceRange    = 120.0,
    volumeMult    = 2.0,
    hearRange     = 140.0,
    submixFreqLow = 600.0,
    submixFreqHi  = 6000.0,
    submixFudge   = 2.0,
    submixOFreqLo = 300.0,
    submixOFreqHi = 8000.0,
    propModel     = 'prop_microphone_02',
    attachBone    = 57005,
    attachOffset  = { x = 0.13, y = 0.02, z = -0.02 },
    attachRotation = { x = -80.0, y = 20.0, z = 10.0 },
}
```

| Key             | Default                | Purpose                                                                                              |
| --------------- | ---------------------- | ---------------------------------------------------------------------------------------------------- |
| `enabled`       | `true`                 | When `false`, the panel mic button reports `err_mic_disabled` and no broadcast event is fired.       |
| `cooldownMs`    | `15000`                | Minimum gap between consecutive broadcasts (UI-side guard).                                          |
| `durationMs`    | `8000`                 | How long the boost stays active.                                                                     |
| `voiceRange`    | `120.0`                | `pma-voice` proximity range (meters) used during the boost.                                          |
| `volumeMult`    | `2.0`                  | Mumble volume override applied to listeners within `hearRange`.                                      |
| `hearRange`     | `140.0`                | Distance (meters) other players still receive the boost.                                             |
| `submixFreqLow` | `600.0`                | Radio FX submix `freq_low`. Lower = darker / more muffled.                                           |
| `submixFreqHi`  | `6000.0`               | Radio FX submix `freq_hi`. Higher = brighter / clearer.                                              |
| `submixFudge`   | `2.0`                  | Radio FX submix `fudge`.                                                                             |
| `submixOFreqLo` | `300.0`                | Radio FX submix `o_freq_lo` (out band).                                                              |
| `submixOFreqHi` | `8000.0`               | Radio FX submix `o_freq_hi` (out band).                                                              |
| `propModel`     | `'prop_microphone_02'` | Mic prop attached to the broadcaster.                                                                |
| `attachBone`    | `57005`                | Ped bone index for attachment (default: right hand).                                                 |
| `attachOffset`  | `{0.13, 0.02, -0.02}`  | XYZ attach offset.                                                                                   |
| `attachRotation` | `{-80.0, 20.0, 10.0}` | XYZ attach rotation.                                                                                 |

### UI Theme

The NUI panel reads `Config.Theme` at open time and pipes every value straight into a matching CSS variable. Every value is forwarded verbatim — accepts hex, `rgb()`, `rgba()`, `hsl()`, gradients (`linear-gradient(...)`), or any valid CSS expression. Leaving a key out (or setting it to `''`) keeps the built-in default. The default palette is **dark red**.

#### Branding

| Variable      | Default                | Controls                                |
| ------------- | ---------------------- | --------------------------------------- |
| `brandTitle`  | `'CLADS DJ CONSOLE'`   | Header headline text.                   |
| `fontDisplay` | `"'Rajdhani', sans-serif"` | Display font (titles, pills, headings). |
| `fontBody`    | `"'Manrope', sans-serif"`  | Body font (labels, queue items, hints). |

#### Layout

| Variable       | Default                                                                                | Controls                              |
| -------------- | -------------------------------------------------------------------------------------- | ------------------------------------- |
| `radius`       | `'22px'`                                                                               | Outer shell border radius.            |
| `radiusCard`   | `'16px'`                                                                               | Inner panel radius.                   |
| `radiusBtn`    | `'10px'`                                                                               | Button radius.                        |
| `shellWidth`   | `'min(1220px, 96vw)'`                                                                  | Panel width.                          |
| `shellHeight`  | `'min(92vh, 880px)'`                                                                   | Panel height.                         |
| `shellMargin`  | `'4vh auto'`                                                                           | Outer shell margin.                   |
| `shellShadow`  | `'0 0 80px rgba(0,0,0,0.6), inset 0 1px 0 rgba(255,255,255,0.03)'`                     | Drop shadow + inner gloss.            |

#### Core Surfaces

| Variable           | Default                                                                          | Controls                                  |
| ------------------ | -------------------------------------------------------------------------------- | ----------------------------------------- |
| `bgCore`           | `'#0D0D0D'`                                                                      | Root canvas behind shell.                 |
| `bgShell`          | `'#1A1A1A'`                                                                      | Main shell background.                    |
| `bgShellTopGloss`  | `'linear-gradient(180deg, rgba(255,255,255,0.02), transparent 20%)'`             | Top gloss overlay on the shell.           |
| `bgShellSideGloss` | `'linear-gradient(290deg, rgba(255,255,255,0.01), transparent 30%)'`             | Diagonal side gloss overlay.              |
| `shellBorder`      | `'rgba(180, 180, 180, 0.08)'`                                                    | Outer shell border.                       |

#### Header / Footer

| Variable       | Default                          | Controls                                       |
| -------------- | -------------------------------- | ---------------------------------------------- |
| `headerBg`     | `'rgba(0, 0, 0, 0.5)'`           | Header background.                             |
| `headerStroke` | `'rgba(180, 180, 180, 0.05)'`    | Header bottom stroke.                          |
| `footerBg`     | `'rgba(0, 0, 0, 0.5)'`           | Footer background.                             |
| `footerStroke` | `'rgba(180, 180, 180, 0.05)'`    | Footer top stroke.                             |

#### Brand Icon

| Variable     | Default                                                                          | Controls                          |
| ------------ | -------------------------------------------------------------------------------- | --------------------------------- |
| `iconBg`     | `'linear-gradient(145deg, rgba(180,180,180,0.08), rgba(100,100,100,0.04))'`      | Logo wrapper background.          |
| `iconBorder` | `'rgba(180, 180, 180, 0.20)'`                                                    | Logo wrapper border.              |

#### Meta Pills

| Variable     | Default                          | Controls                                          |
| ------------ | -------------------------------- | ------------------------------------------------- |
| `pillBg`     | `'rgba(255, 255, 255, 0.02)'`    | Background of meta pills (`DESK`, `SPEAKER`, `#id`). |
| `pillBorder` | `'rgba(180, 180, 180, 0.12)'`    | Border of meta pills.                             |

#### Inner Panels

| Variable      | Default                          | Controls                                  |
| ------------- | -------------------------------- | ----------------------------------------- |
| `panelBg`     | `'rgba(8, 8, 8, 0.8)'`           | Background of "Now Playing" / "Queue".    |
| `panelBorder` | `'rgba(180, 180, 180, 0.05)'`    | Border of inner panels.                   |

#### Track Hero Card

| Variable          | Default                                                                                                                         | Controls                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `trackHeroBg`     | radial + linear gradient (`rgba(180,180,180,0.06)` over `rgba(20,20,20,0.95)`)                                                  | Hero card background.             |
| `trackHeroBorder` | `'rgba(180, 180, 180, 0.05)'`                                                                                                   | Hero card border.                 |

#### Cover Disc

| Variable      | Default                                                                                                                                  | Controls                          |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| `coverBg`     | radial + conic gradient simulating a vinyl disc                                                                                          | Cover disc fill.                  |
| `coverBorder` | `'rgba(180, 180, 180, 0.20)'`                                                                                                            | Cover disc rim.                   |

#### Sliders / Fields

| Variable           | Default                          | Controls                                   |
| ------------------ | -------------------------------- | ------------------------------------------ |
| `fieldBg`          | `'rgba(0, 0, 0, 0.4)'`           | Text field background.                     |
| `fieldBorder`      | `'rgba(180, 180, 180, 0.10)'`    | Text field border.                         |
| `fieldBorderFocus` | `'rgba(180, 180, 180, 0.35)'`    | Text field border on focus.                |
| `fieldFocusGlow`   | `'rgba(180, 180, 180, 0.08)'`    | Soft glow ring on focus.                   |
| `sliderWrapBg`     | `'rgba(255, 255, 255, 0.015)'`   | Slider wrapper background.                 |
| `sliderWrapBorder` | `'rgba(180, 180, 180, 0.05)'`    | Slider wrapper border.                     |
| `composerBg`       | `'rgba(255, 255, 255, 0.015)'`   | Track composer (title + URL row) background. |
| `composerBorder`   | `'rgba(180, 180, 180, 0.05)'`    | Composer border.                           |

#### Queue List

| Variable          | Default                          | Controls                       |
| ----------------- | -------------------------------- | ------------------------------ |
| `queueListBg`     | `'rgba(0, 0, 0, 0.30)'`          | Queue scroll container bg.     |
| `queueListBorder` | `'rgba(180, 180, 180, 0.05)'`    | Queue container border.        |
| `queueItemBg`     | `'rgba(255, 255, 255, 0.015)'`   | Individual queue row bg.       |
| `queueItemBorder` | `'rgba(180, 180, 180, 0.10)'`    | Individual queue row border.   |

#### Accent (success / primary buttons / live state / sliders)

| Variable       | Default                          | Controls                                          |
| -------------- | -------------------------------- | ------------------------------------------------- |
| `accent`       | `'#FF003C'`                      | Primary accent (buttons, slider thumb, LIVE pill). |
| `accentSoft`   | `'rgba(255, 0, 60, 0.18)'`       | Soft accent fill (button bg).                     |
| `accentBorder` | `'rgba(255, 0, 60, 0.40)'`       | Accent border.                                    |
| `accentText`   | `'#FFFFFF'`                      | Text on accent buttons.                           |

#### Danger (stop / delete / mic active)

| Variable       | Default                          | Controls                                  |
| -------------- | -------------------------------- | ----------------------------------------- |
| `danger`       | `'#FF003C'`                      | Danger color (Stop, Delete, mic active).  |
| `dangerSoft`   | `'rgba(255, 0, 60, 0.10)'`       | Soft danger fill.                         |
| `dangerBorder` | `'rgba(255, 0, 60, 0.40)'`       | Danger border.                            |
| `dangerText`   | `'#FF8AA0'`                      | Danger text color.                        |

#### Paused State

| Variable       | Default                          | Controls                          |
| -------------- | -------------------------------- | --------------------------------- |
| `paused`       | `'#ffb300'`                      | Paused pill foreground (yellow).  |
| `pausedSoft`   | `'rgba(255, 180, 0, 0.10)'`      | Paused pill background.           |
| `pausedBorder` | `'rgba(255, 180, 0, 0.35)'`      | Paused pill border.               |

#### Generic Buttons

| Variable          | Default                          | Controls                          |
| ----------------- | -------------------------------- | --------------------------------- |
| `btnBg`           | `'rgba(255, 255, 255, 0.03)'`    | Generic button background.        |
| `btnBorder`       | `'rgba(180, 180, 180, 0.10)'`    | Generic button border.            |
| `btnBorderHover`  | `'rgba(180, 180, 180, 0.25)'`    | Generic button hover border.      |

#### Secondary Button (Pause, Add Queue)

| Variable             | Default                                                                          | Controls                          |
| -------------------- | -------------------------------------------------------------------------------- | --------------------------------- |
| `btnSecondaryBg`     | `'linear-gradient(140deg, rgba(180,180,180,0.08), rgba(120,120,120,0.04))'`      | Secondary button background.      |
| `btnSecondaryBorder` | `'rgba(180, 180, 180, 0.20)'`                                                    | Secondary button border.          |
| `btnSecondaryText`   | `'#d0d0d0'`                                                                      | Secondary button text.            |

#### Alt Button (Mute)

| Variable       | Default                          | Controls                          |
| -------------- | -------------------------------- | --------------------------------- |
| `btnAltBg`     | `'rgba(180, 180, 180, 0.05)'`    | Mute button background.           |
| `btnAltBorder` | `'rgba(180, 180, 180, 0.18)'`    | Mute button border.               |
| `btnAltText`   | `'#b0b0b0'`                      | Mute button text.                 |

#### Neutrals

| Variable       | Default        | Controls                                         |
| -------------- | -------------- | ------------------------------------------------ |
| `neutral`      | `'#b0b0b0'`    | Mid neutral (text, icons).                       |
| `neutralLight` | `'#d0d0d0'`    | Light neutral (secondary button text).           |
| `neutralDim`   | `'#707070'`    | Dim neutral (progress bar trail, muted glyphs).  |

#### Text

| Variable        | Default                          | Controls                          |
| --------------- | -------------------------------- | --------------------------------- |
| `textPrimary`   | `'rgba(255, 255, 255, 0.92)'`    | Primary readable text.            |
| `textSecondary` | `'rgba(190, 190, 190, 0.72)'`    | Secondary labels.                 |
| `textMuted`     | `'rgba(150, 150, 150, 0.55)'`    | Muted hints / placeholder.        |

#### Strokes

| Variable    | Default                          | Controls                          |
| ----------- | -------------------------------- | --------------------------------- |
| `line`      | `'rgba(180, 180, 180, 0.10)'`    | Standard divider stroke.          |
| `lineSoft`  | `'rgba(180, 180, 180, 0.05)'`    | Soft divider stroke.              |

#### Misc

| Variable          | Default                                                                  | Controls                                  |
| ----------------- | ------------------------------------------------------------------------ | ----------------------------------------- |
| `progressBarGrad` | `'linear-gradient(90deg, var(--accent), var(--neutral-dim))'`            | Progress bar fill gradient.               |
| `keyHintBorder`   | `'rgba(180, 180, 180, 0.10)'`                                            | Footer key-hint border.                   |
| `keyHintText`     | `'rgba(190, 190, 190, 0.72)'`                                            | Footer key-hint text color.               |
| `dotColor`        | `'rgba(180, 180, 180, 0.40)'`                                            | Background dot pattern color.             |

---

## 5. Commands

| Command     | Source           | Purpose                                                                              |
| ----------- | ---------------- | ------------------------------------------------------------------------------------ |
| `/djplace`  | `Config.Commands.place`  | Opens the placement context menu (Desk / Speaker). Placement itself is gated by `Config.AdminAce`. The menu opens for anyone, but `/clads_dj:server:createProp` enforces the ACE distance bypass and admins are the only role permitted to delete props. |
| `/djmanage` | `Config.Commands.manage` | Opens the panel of the closest placed prop within `Config.InteractionDistance + 1.0` meters. Reports `err_no_nearby_prop` otherwise. |

Both commands are unregistered if `Config.EnableCommands = false`. Players still get the `ox_target` "DJ Panel" prompt on each placed prop regardless of the command setting.

ACE permissions required:

```cfg
add_ace group.admin command.djplace allow
add_principal identifier.fivem:1234567 group.admin
```

The deletion path (`clads_dj:server:deleteProp`) requires the ACE on the server side regardless of how the event is triggered.

---

## 6. Items

### `music_usb` (Config.UsbItem)

| Property       | Value                                                                                          |
| -------------- | ---------------------------------------------------------------------------------------------- |
| Default name   | `music_usb`                                                                                    |
| Configured by  | `Config.UsbItem`                                                                               |
| Detected via   | `community_bridge.Inventory.GetItemCount`                                                      |
| Purpose        | Unlocks the **USB Plug & Play** shortcut on the deck composer.                                 |

Usage flow:

1. The player carries at least one `music_usb` item.
2. They open a deck panel (`/djmanage` or via target).
3. The panel polls inventory every 2 seconds while open and surfaces a `USB detected` chip and the **USB Plug & Play** action when present.
4. Adding a track flagged `fromUsb = true` is re-validated server-side; if the item count is `0` at submit time the action returns `err_no_usb`.

To disable the USB UI entirely set `Config.UsbItem = ''` (or any name not in your items DB). The panel will silently hide the USB chip.

---

## 7. Customization

### UI Theme

Repaint the entire console by editing `Config.Theme` in `config.lua`. Three preset blocks ship at the bottom of the same file — paste any of them over the active table:

- **Neon Purple** (the original BetterV palette) — purple `#9D4EDD` accent, pink `#FF2A6D` danger.
- **Sunset** — warm `#ff9966` accent on a rust shell.
- **Ice Blue** — cyan `#4dd0e1` accent on a navy shell.
- **Pure Mono / Greyscale** — pure-black shell with a white accent.

Every value is forwarded verbatim to CSS, so you can mix gradients, layered backgrounds, and any valid CSS color expression without code changes.

### Changing prop models

Replace `Config.Props.desk.model` and `Config.Props.speaker.model` with any streamed prop hash. The visibility is controlled by `Config.Props.<type>.invisible` — leave the deck invisible if you want to overlay your own MLO scene or use the prop strictly as a hitbox/anchor.

### Adjusting ranges

- **Placement reach** — `Config.MaxPlaceDistance`.
- **Target prompt range** — `Config.InteractionDistance`, `Config.TargetRadius`.
- **Speaker grouping** — `Config.DeskSpeakerRadius`. A deck routes its audio to every speaker prop within this radius. Increase for stadium-scale grouping; decrease to isolate adjacent venues.
- **Audio falloff caps** — `Config.MinDistance` / `Config.MaxDistance` clamp the per-deck distance slider.

### Mic submix tuning

The submix is constructed once at script load using `Config.Microphone.submixFreqLow / submixFreqHi / submixFudge / submixOFreqLo / submixOFreqHi`. Lower `freq_low` / higher `freq_hi` give a brighter, less filtered megaphone. Raise both `submixOFreqLo` and `submixOFreqHi` toward the same value to over-band the radio FX. Edit and restart the resource for the new values to take effect.

### Adding language files

1. Copy `locales/en.json` to `locales/<code>.json`.
2. Translate every key. Keys missing from the new file fall back to the literal key string (handled by `_t` and `Locale.resolve`).
3. Set `Config.Locale = '<code>'` or set the convar `setr clads_locale <code>` and leave `Config.Locale = 'auto'`.

---

## 8. Exports & Events

### Server net events (incoming, from client)

| Event                              | Payload                                       | Purpose                                                                       |
| ---------------------------------- | --------------------------------------------- | ----------------------------------------------------------------------------- |
| `clads_dj:server:requestSync`      | none                                          | Player requests a full prop sync (auto-fired on first load and on `community_bridge:Client:OnPlayerLoaded`). |
| `clads_dj:server:createProp`       | `propType, coords, heading`                   | Creates a deck or speaker, persists to DB, spawns the entity, broadcasts the new prop. |
| `clads_dj:server:deleteProp`       | `propId`                                      | Stops playback, deletes the entity + DB row, broadcasts removal. ACE-gated.   |
| `clads_dj:server:control`          | `propId, action, payload`                     | All panel actions (see action list below).                                    |
| `clads_dj:server:trackEnded`       | `propId, token`                               | Fired by xSound's `songStopPlaying` event; advances the queue if the token matches. |

#### `clads_dj:server:control` actions

| Action         | Payload                              | Effect                                                              |
| -------------- | ------------------------------------ | ------------------------------------------------------------------- |
| `addTrack`     | `{ title, url, fromUsb?, playNow? }` | Validates URL, optionally fetches YouTube title, adds to queue or plays now. |
| `removeTrack`  | `{ index }`                          | Removes a queue entry by index.                                     |
| `moveTrack`    | `{ from, to }`                       | Reorders a queue entry.                                             |
| `playTrack`    | `{ index }`                          | Pulls the entry from the queue and plays it immediately.            |
| `nextTrack`    | none                                 | Advances to the next queued track (or stops if the queue is empty). |
| `stop`         | none                                 | Stops the current track and clears `current`.                       |
| `toggleMute`   | none                                 | Toggles the muted flag (volume forced to `0.0` on clients).         |
| `togglePause`  | none                                 | Pauses/resumes the active xSound stream.                            |
| `setVolume`    | `{ value }`                          | Clamps to `[MinVolume, MaxVolume]`, broadcasts a mix update.        |
| `setDistance`  | `{ value }`                          | Clamps to `[MinDistance, MaxDistance]`, broadcasts a mix update.    |
| `mic`          | none                                 | Triggers the megaphone broadcast for the requesting player.         |
| `clearQueue`   | none                                 | Empties the queue (does not stop the current track).                |

### Server callbacks

| Callback                    | Args      | Returns                                                                  |
| --------------------------- | --------- | ------------------------------------------------------------------------ |
| `clads_dj:server:getStation` | `propId` | `{ prop, station, hasUsb }` if the requester is in range / admin, `nil` otherwise. |

### Client net events (incoming, from server)

| Event                              | Payload                                       | Purpose                                                            |
| ---------------------------------- | --------------------------------------------- | ------------------------------------------------------------------ |
| `clads_dj:client:syncProps`        | `propList`                                    | Full sync (overwrites local prop cache, rebuilds zones).           |
| `clads_dj:client:updateProp`       | `prop`                                        | Single prop add/update.                                            |
| `clads_dj:client:removeProp`       | `propId`                                      | Removes a prop client-side, destroys its sounds.                   |
| `clads_dj:client:updateStation`    | `propId, station`                             | Pushes new station state into the open panel (if matching).        |
| `clads_dj:client:playStation`      | `{ propId, token, url, volume, distance, muted, outputs }` | Starts xSound playback at every speaker output.        |
| `clads_dj:client:updateMix`        | `{ propId, token, volume, distance, muted, outputs }` | Mid-track volume/distance/mute change.                  |
| `clads_dj:client:stopStation`      | `{ propId, token }`                           | Destroys all sounds for a deck.                                    |
| `clads_dj:client:pauseStation`     | `{ propId }`                                  | Pauses every active xSound for the deck.                           |
| `clads_dj:client:resumeStation`    | `{ propId }`                                  | Resumes paused sounds.                                             |
| `clads_dj:client:micBroadcast`     | `{ propId, durationMs, voiceRange }`          | Activates the broadcaster's local mic boost + prop attachment.     |

### State bag

| Bag                          | Scope        | Purpose                                                                        |
| ---------------------------- | ------------ | ------------------------------------------------------------------------------ |
| `Player(serverId).state.djMicActive` | Player state | Replicated boolean. Other clients use this to apply the mumble submix + volume override during the broadcast window. |

### NUI callbacks

| Callback   | Payload                                 | Purpose                                                                |
| ---------- | --------------------------------------- | ---------------------------------------------------------------------- |
| `close`    | none                                    | Releases NUI focus, clears the open station, stops the idle anim.      |
| `control`  | `{ action, payload? }`                  | Forwards every panel action to `clads_dj:server:control`. The special action `refreshUsb` short-circuits and re-publishes inventory state to the panel. |

### NUI messages

| Action      | Data                                                       | Purpose                                              |
| ----------- | ---------------------------------------------------------- | ---------------------------------------------------- |
| `open`      | `{ prop, station, hasUsb, context }`                       | Open the panel and inject locale + theme + limits.   |
| `close`     | none                                                       | Visually close the panel.                            |
| `station`   | `station` payload                                          | Push fresh station state to the panel.               |
| `usbState`  | `{ hasUsb }`                                               | Update the USB chip in real time.                    |
| `micState`  | `{ active, durationMs? }`                                  | Sync the mic countdown indicator.                    |

### External exports consumed

| Resource           | Export                                                  | Purpose                                                |
| ------------------ | ------------------------------------------------------- | ------------------------------------------------------ |
| `community_bridge` | `Bridge()` → `Framework`, `Inventory`, `Target`, `Notify` | All cross-framework abstraction.                       |
| `xsound`           | `PlayUrlPos`, `Distance`, `setVolume`, `Pause`, `Resume`, `Destroy`, `soundExists` | Audio streaming. Listens to `xSound:songStopPlaying`. |
| `pma-voice`        | `overrideProximityRange`, `clearProximityOverride`      | Optional megaphone proximity boost.                    |

---

## 9. Troubleshooting

### `/djplace` does nothing

- Confirm `Config.EnableCommands = true`.
- Confirm your identifier is mapped to `Config.AdminAce` (default `group.admin`). Without the ACE the command opens the menu but the server rejects `clads_dj:server:createProp` if you're farther than `Config.MaxPlaceDistance`, and deletion is permanently blocked.
- Confirm `ox_lib` is started before `clads_dj` — the placement menu uses `lib.registerContext` / `lib.showContext`.

### Speakers not receiving audio

- Speakers must be within `Config.DeskSpeakerRadius` of the deck. Decks compute the output list per playback event using the deck's own coords.
- Confirm `xsound` is started. The client logs a debug warning at boot when `Config.Debug = true` if it's missing, and silently skips all playback events otherwise.
- Confirm the URL is reachable from the player's machine. The server enforces only protocol + length validation (`http://` or `https://`, `<= MaxTrackUrlLength`).
- If you placed a speaker after starting playback, stop and restart the track — the output list is captured at play time and the mix update only resizes existing sounds.

### Mic boost not working

- Confirm `pma-voice` is started. Without it the proximity override step is silently skipped, and the notification falls back to the `inform` style. The mic prop attachment + mumble submix still apply, but range/volume will not change.
- Confirm `Config.Microphone.enabled = true`. When `false`, the server returns `err_mic_disabled` and no broadcast event is fired.
- Confirm listeners are within `Config.Microphone.hearRange`. Listeners outside the range have their volume override and submix cleared automatically.

### USB not detected

- Confirm the item exists in your inventory's items database under exactly the name in `Config.UsbItem`.
- Confirm your inventory is a backend supported by `community_bridge.Inventory` (`ox_inventory`, `qb-inventory`, `qs-inventory`, `origen_inventory`, `codem-inventory`, `tgiann-inventory`).
- The panel polls inventory every 2 seconds while open. If nothing changes, click **Close** and reopen the deck to force a refresh.
- Server-side validation (`getUsbCount`) re-checks at submit time, so a recently-removed USB will still hit `err_no_usb` even if the panel chip is stale.

### Prop not invisible

- The deck is hidden client-side by forcing `SetEntityAlpha(entity, 0, false)`. This requires the network ID to exist on the requesting client. If a client streams in late, the prop becomes invisible on the next sweep — there is a 5-second background thread that re-applies alpha to every known prop.
- Confirm `Config.Props.<type>.invisible = true` for the prop type you placed.
- Confirm the spawned model matches `Config.Props.<type>.model`. If a buyer changes the model in `config.lua` after props are already in the DB, `applyPropVisibility` will refuse to hide the wrong-model entity to avoid hiding bystanders.

---

## 10. Support

Reach out via the Creative Lads Discord.
