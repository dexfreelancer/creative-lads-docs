# Creative Lads Documentation

Welcome to the official documentation for the Creative Lads FiveM resource suite. Each guide here is written for server owners installing, configuring, and maintaining the resources on their own server.

## Resource Catalog

| Resource | Purpose |
| --- | --- |
| [clads_mechanic](clads_mechanic.md) | Mechanic tablet — vehicle servicing, tuning, nitrous, stance, speed limiter |
| [clads_garages](clads_garages.md) | 47-garage system with public, faction, gang, depot, and business types |
| [clads_vehicleshop](clads_vehicleshop.md) | Multi-shop dealership — cars, bikes, aircraft, boats, photo capture |
| [clads_rentals](clads_rentals.md) | Vehicle rentals for new players, multi-location |
| [clads_multichar](clads_multichar.md) | Character selection screen with multi-slot support |
| [clads_radio](clads_radio.md) | Production radio with jammer + GPS modules |
| [clads_dj](clads_dj.md) | Modular DJ console with placeable decks and speakers |
| [clads_crosshair](clads_crosshair.md) | Parametric crosshair overlay with in-game editor |
| [clads_freecam](clads_freecam.md) | Cinematic freecam with anti-exploit wall detection |

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
