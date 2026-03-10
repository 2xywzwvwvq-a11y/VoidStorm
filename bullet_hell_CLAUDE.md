# VOID STORM — Bullet Hell

Single-file browser bullet hell (`Voidstorm.html`). ~4,750 lines / ~163 KB. Open directly in browser.

## Canvas & Loop
- `ZOOM = 0.9` — canvas is 1/ZOOM larger than window, scaled down via CSS
- `W = ceil(innerWidth / ZOOM)`, `H = ceil(innerHeight / ZOOM)`; mouse: `e.clientX / ZOOM`
- Loop: `requestAnimationFrame`, `dt = min((timestamp - lastTime) / 16.67, 3)`
- Nebula pre-rendered to offscreen `nebulaCanvas` on stage init — not per frame

## Game States
`'title'` → `'playing'` ↔ `'paused'` / `'upgrading'` → `'dead'`
Controls: **WASD** move · **Mouse** aim (auto-fire) · **Space** bomb · **Escape** pause

## Player
`{ x, y, hp: 6, maxHp: 6, invuln: 0, fireTimer: 0, speed: 3.5 }`
- Hitbox radius: **8px** (hardcoded: `dist < 8 + b.radius`)
- Hit: 0.9 HP damage, 60 invuln frames. Death: `hp <= 0.01`
- Movement: normalized dir × `player.speed * moveSpeed`, clamped 15px from edges
- **`slowMo` does NOT affect player movement** — only bullets, enemies, particles

## Auto-Fire
- `fireTimer -= fireRate` each tick; fires at ≤0, resets to 8
- Fires `bulletCount` bullets spread over `spreadAngle` rad centered on `shootDir`
- Bullet: speed `8 * bulletSpeed`, damage `bulletDamage`, radius 3; despawn 20px off-screen

## Upgrades
| Variable | Default | Upgrade | Effect |
|---|---|---|---|
| `fireRate` | 1 | RAPID FIRE ×1.25 | Higher = faster fire |
| `bulletDamage` | 1 | POWER SHOT ×1.3 | Damage per bullet |
| `bulletSpeed` | 1 | VELOCITY ×1.2 | Bullet velocity multiplier |
| `bulletCount` | 1 | MULTI SHOT +1 | Also: `spreadAngle = min(+0.12, 0.8)` |
| `magnetRange` | 60 | MAGNET ×1.4 | XP gem pull radius (px) |
| `moveSpeed` | 1 | THRUSTERS ×1.15 | Player speed multiplier |
| `maxHP` | 6 | ARMOR +2hp, heal 1 / PLATING +1hp, full heal | — |
| `bombs` | 1 | BOMB+ +1 | — |
| — | — | REPAIR | Heal 2 HP |

3 random upgrades shown per level-up. Bomb (Space): clears all enemy bullets, 10 dmg to all enemies, `slowMo=0.2/20f`, shake=20, flash=0.7, CA=8.

## Enemies
| Type | Radius | Base HP | Speed | Target Dist | Fire Interval | Score |
|---|---|---|---|---|---|---|
| `basic` | 12 | 2 | 1–1.5 | 150 | 70 | 10 |
| `spinner` | 14 | 3 | 1–1.5 | 150 | 55 | 25 |
| `burst` | 14 | 3 | 1–1.5 | 150 | 120 | 25 |
| `sniper` | 14 | 3 | 0.8 | 300 | 100 | 25 |
| `boss` | 28 | 50 | 0.6 | 200 | 35 | 500 |

HP scale: `× (1 + (wave-1) * 0.15)`. Movement: accel `0.08 * speed`, friction `0.96`.

## Enemy Bullet Patterns
`spdScale = 1 + wave * 0.025`
- **basic:** 1 aimed, spd `2.5×`, `#ff4488`
- **spinner:** 6-way radial + `spinAngle += 0.3`, spd `2×`, `#ffaa00`
- **burst:** 12-way circle, spd `1.8×`, `#ff00ff`
- **sniper:** 1 aimed fast, spd `5×`, `#00ffff`, r=4
- **boss:** cycles every 120 age ticks — P0: 16-way spin, P1: 5-way aimed spread, P2: 24-way circle

Player bullets destroy enemy bullets (collision `pb.r + eb.r + 2`), +1 score.

## XP & Levels
- Tier 1 gem (green): value 1 — basic drops 2, others drop 3
- Tier 2 gem (gold): value 5 — boss drops 6, big enemies (wave≥5) drop 1
- Pull within `magnetRange`px; pickup at 18px (T1) / 22px (T2)
- XP threshold: starts 10, each level `min(floor(prev × 1.4) + 5, 200)`

## Wave & Stage System
- `waveEnemiesTotal = wave % 5 === 0 ? 1 : min(3 + wave*1.5, 30)`
- Boss every 5 waves. Spawn gap: boss=80t, others=`max(25, 50-wave*2)`. 150t between waves.
- Bonus bomb every 3 waves. Wave ends when all spawned + `enemies.length === 0`.

| Stage | Name | Hazards (visual only) |
|---|---|---|
| 0 | DEEP VOID | none |
| 1 | ASTEROID BELT | 8 asteroids |
| 2 | CRIMSON NEBULA | none |
| 3 | FROZEN EXPANSE | 10 ice crystals |
| 4 | THE CORE | 6 energy fields |

Stage changes every 5 waves → `initNebula()`, `initStarfield()`, `initHazards()`.

## Screen Effects
| Effect | Decay | Notable triggers |
|---|---|---|
| `screenShake` | ×0.88/f | Explosions, hits, bomb (20), boss kill (20) |
| `screenFlash` | ×0.9/f | Player hit (0.5), bomb (0.7) |
| `chromaticAberration` | ×0.85/f | Hits (5), bomb (8), boss kill (6) |
| `slowMo` | resets after timer | Bomb: 0.2/20f · Boss kill: 0.15/40f |
| `killCombo` | resets at 90t | 5+ kills: extra shake+CA; displayed at 3+ |

## Store System
`storeData` persisted to `localStorage` (`voidstorm_store`): `{ credits, ownedShips[], ownedEnemies[], activeShip, activeEnemy }`.
- `activeShipSkin` / `activeEnemySkin` — global strings read by draw code
- Credits earned in-game (score → credits conversion somewhere in end-of-run flow)

### Ship Skins (`SHIP_SKINS`)
| id | Name | Rarity | Cost | Shape notes |
|---|---|---|---|---|
| `default` | VOIDHAWK | common | 0 | Swept wings, triple cannon, red visor |
| `crimson` | CRIMSON GHOST | rare | 500 | Narrow body, **canard fins**, single railgun |
| `void_wraith` | VOID WRAITH | epic | 1000 | **Bat wings** (bezier), eye cockpit, needle ×3 |
| `solar_fury` | SOLAR FURY | legendary | 2500 | **Square wings** + solar grid, triple pods, dual cannons |

Drawing: inside `if (!blink)` block, branched on `activeShipSkin`. Each skin has its own exhaust, wing, fuselage, cockpit, and cannon geometry. `drawShipSkinPreview(canvas, skinId)` renders store thumbnails.

### Enemy Skins (`ENEMY_SKINS`)
Palette-swapped via `_SKIN_PALETTES[id]` / `getEnemySkinPalette()`. IDs: `default`, `neon_corps`, `frost_proto`, `inferno`.

## Protected (do NOT change)
- Player hitbox 8px (hardcoded in bullet collision check)
- Enemy fire patterns — bullet counts, speeds, `spdScale` formula
- Wave formula: `3 + wave * 1.5`, boss `% 5`
- XP scaling: `floor(xpToNext * 1.4) + 5`, cap 200
- `ZOOM = 0.9` and canvas resize logic
- Default ship draw coordinates (VOIDHAWK geometry — the `else` branch)
- Stage `STAGES` array (colors, cloud configs)
- `slowMo` must not affect player movement
