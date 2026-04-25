# Creative Lads Documentation

Welcome to the official documentation for the Creative Lads FiveM resource suite. Each guide here is written for server owners installing, configuring, and maintaining the resources on their own server.

## Resource Catalog

| Resource | Purpose |
| --- | --- |
| [🔧 Creative Lads Mechanic](<🔧 Creative Lads Mechanic.md>) | Mechanic tablet — vehicle servicing, tuning, nitrous, stance, speed limiter |
| [🅿️ Creative Lads Garages](<🅿️ Creative Lads Garages.md>) | 47-garage system with public, faction, gang, depot, and business types |
| [🏬 Creative Lads Vehicleshop](<🏬 Creative Lads Vehicleshop.md>) | Multi-shop dealership — cars, bikes, aircraft, boats, photo capture |
| [🚙 Creative Lads Rentals](<🚙 Creative Lads Rentals.md>) | Vehicle rentals for new players, multi-location |
| [👥 Creative Lads Multichar](<👥 Creative Lads Multichar.md>) | Character selection screen with multi-slot support |
| [📻 Creative Lads Radio](<📻 Creative Lads Radio.md>) | Production radio with jammer + GPS modules |
| [🎧 Creative Lads DJ](<🎧 Creative Lads DJ.md>) | Modular DJ console with placeable decks and speakers |
| [🎯 Creative Lads Crosshair](<🎯 Creative Lads Crosshair.md>) | Parametric crosshair overlay with in-game editor |
| [📸 Creative Lads Freecam](<📸 Creative Lads Freecam.md>) | Cinematic freecam with anti-exploit wall detection |
| [🥊 Creative Lads Boxing](<🥊 Creative Lads Boxing.md>) | 1v1 boxing arena with rounds, gloves, HUD timer, and parimutuel betting |

## Common Stack

Every resource is framework-agnostic and built on top of [community_bridge](https://github.com/The-Order-Of-The-Sacred-Framework/community_bridge), which auto-detects:

- **Frameworks** — ESX, QBCore, QBox, Standalone
- **Inventories** — ox_inventory, qb-inventory, qs-inventory, codem-inventory, origen_inventory, and more
- **Targets** — ox_target, qb-target, sleepless_interact
- **Locales** — `clads_locale` → `ox:locale` → `qb_locale` → `lang` → `en`

## Default Theme

All resources ship with the **Dark Red** palette by default:

- Primary accent: `#FF003C`
- Pure black background: `#0D0D0D`
- Dark gray surfaces: `#1A1A1A`
- Mid gray cards: `#2E2E2E`
- Light gray secondary text: `#888888`

Each resource exposes its own theme override block — see the **Customization** section of any guide for the exact keys.

## Support

Reach out via the Creative Lads Discord.
