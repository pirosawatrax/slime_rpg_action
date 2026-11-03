# Slime RPG

A chunked voxel world (Python + Ursina) with fog, hold-to-break mining, bridges, a 20-slot inventory, save/load,
minerals you dig, carry and eat, **action combat against slimes standing in the world**, levels, death, villages with
commands, a day/night cycle, caves, snow, flowers, composed music, data-only MODs and the Slime King.

The slime's colour is your colour: eat minerals to move it, and fight with colours that beat your enemy's
(a three-way cycle: green beats red, blue beats green, red beats blue).

## Run

    pip install -r requirements.txt
    python main.py          # continues the saved world if there is one (otherwise a new random map)
    python main.py --new    # a new game on a new random map; the old save is moved aside, never deleted
    python main.py --seed N # a new game on map N

On Linux, `main.py` sets `XMODIFIERS=@im=none` itself (before Ursina loads) so fcitx5/ibus cannot swallow
key-release events (symptom: keys keep "holding" and Space does nothing). `SLIME_KEEP_IME=1 python main.py` opts out.

## Controls

| Input | Action |
|---|---|
| WASD / Space | Move / jump |
| Hold left mouse | Dig (hold time depends on the block). With the crosshair on a slime within 2 blocks: tackle it instead |
| Right mouse | Place the selected block (water accepts Wood only: bridges), or plant a seed. Holding a mineral: hit the slime you aim at with it |
| Mouse wheel, 1-0 | Select inventory slot |
| E | Eat the selected mineral (works any time, also while slimes chase you) |
| Q | Discard one item from the selected slot |
| Esc | Open / close the pause menu (Resume, Save, Load last save, New world, Save and quit) |

Village screens (a terminal-style panel) are used with the number keys 1-9 and 0, or by clicking; Esc leaves.

## Action combat

Slimes are real cubes standing in the world. There are no random encounters and no battle screen. The damage, level,
colour and experience rules are the ones the old turn-based battles used (`game/battle.py` holds the formulas,
`game/combat.py` applies them); only timing and aiming are new.

- **Slimes:** one coloured block (0.9 wide) with two black eyes. They are placed by `world/enemy_spawn.py`,
  deterministically per seed and chunk, on the ground above and in caves below. A slime only ever stands on solid ground
  with free air in its cell and the two cells above it, so none is sunk into the ground or a wall. None are placed within
  10 blocks of where you start, near villages or inside the Slime King's lair. Slimes exist while their chunk is loaded;
  a beaten slime comes back when you leave the area and return (nothing about slimes is saved).
- **Behaviour:** a slime stands still. Come within **4 blocks** and it chases you; get more than 4 blocks away and it
  stands still again. It climbs or drops **one block at most**; a bigger step (or water) stops it. When it reaches you it
  bites every `ENEMY_ATTACK_INTERVAL`, using the tackle formula with its colour against yours. **On a Wood block it never
  bites** (Wood floors, bridges and a village's Wood tile are safe).
- **Your attacks:** hold **left click** while the crosshair is on a slime within 2 blocks to tackle it (anywhere else the
  button digs as before). **Right click with a mineral in hand** hits it with the mineral: same reach, a strong colour
  does big damage, a weak colour heals the slime. A mineral is used up only when it connects; a miss keeps it.
  The crosshair label shows "tackle: strong/weak" and "mineral: strong/weak" for the slime you aim at. Enemy HP is
  not shown.
- **Feedback:** a hit that does damage makes the slime flash white and jump back a little; when a slime bites you the
  screen flashes red. The battle tune plays while any slime is chasing you.
- **Night:** about twice as many slimes on the surface. The extra, night-only slimes appear when it gets dark and fade
  away at dawn (those far from you; none pop out in front of you).
- **Enemy level:** mostly depth below the surface (about 0.5 level per block), plus a small saturating bonus for distance
  from the start point (at most +6). Until the Slime King has fallen no ordinary enemy exceeds 90% of his level.
  Colour is random in 60..200 per channel; stats are 85% of the average for the level, +-10%. MOD slimes join the
  weighted draw.
- **Levels:** XP = 6 x enemy level x clamp(enemy level / yours, 0.25..2). Level L to L+1 costs 8 x L^2 XP. Each level
  hands out 4-6 random growth points to HP / ATK / DEF / SPD. There is no level cap.
- **Death:** when HP reaches 0 you lose **everything you carry**, wake at the start point (or the last inn you slept at)
  with full HP, and the game is saved at once. Eating can never kill you.
- **Tuning:** everything is in `config.py` under "Stage 9: action combat" (chase distance, speed, bite interval, reach,
  cooldowns, knock-back, flash times, slimes per chunk) and `ARENA_*` for the lair.

### The Slime King

- A level-30 slime, four blocks big (2 x 2 x 2) with two black eyes, with fixed stats (no variation). Low attack, high
  defence on purpose, so a plain tackle is weak and **well-chosen colours are what win**.
- **His room** is 41 x 41 x 16, wrapped in solid rock, with its floor at a seed-picked height (Y 2..10, so the roof is
  always below the land surface). It lies a few hundred blocks from the origin; you dig down to it (a village guide tells
  you x, z, direction and distance). He waits on the Wood tile in the middle of the floor; four Wood posts stand in the room.
- **There is no door and no prompt.** Come within 4 blocks and he chases you; get more than 4 blocks away and he stops.
  You can always run away. If you leave and come back he is at full health again.
- **His colour changes every time you hit him** (the damage is worked out with the colour he had when you hit), and a
  new colour is always at least 90 away from the old one, so the colour you hit with has to keep changing.
- **No single hit takes more than 20% of his HP** (so over-levelling cannot skip the fight: at least 5 hits).
  Winning gives triple XP and the post-boss world below.

### After the Slime King falls (saved at once)

1. Enemy levels are no longer capped at 90% of the boss's level.
2. **New colours** appear: channels now run 40..220 (10 levels, 1000 colours) instead of 60..200 (512). About a third
   of the ore in newly generated chunks turns into the new colours; chunks you have not changed are regenerated at once.
   Enemies' colours use the wider range too.
3. Villages offer the **Services** command, paid in the new colours.

## Villages

The world is cut into 128-block regions; each region hashes to "no village" (about 40%) or one village on gentle dry
land, so a village is the same in every chunk and every save without storing anything. A village has **no buildings**:
a small levelled patch with ONE Wood tile in the middle and a signpost (post + crossbar + tip) beside it, made of
**Water blocks**: blue, so it cannot be mistaken for a tree, and water cannot be dug out. **Step onto the Wood tile and
the village commands open**; the world stands still while they are open. The HUD shows the nearest village and its
compass direction. Everything a village offers is decided by the seed and the village, so nothing is stored except
what you change.

| Command | What it does |
|---|---|
| Inn | Stay the night for **1 mineral of any colour** (the plainest one you carry is used first; the screen shows which one): full HP, and this inn becomes your **respawn point**. **Staying at night skips to the next morning (07:12)**; by day the clock is left alone. |
| Storage | 20 slots **per village** (not shared). Depositing a stack costs 1 mineral (any colour, plainest first); taking things out is free. Survives death. |
| Trade | The merchant swaps minerals of the colours *she* wants for goods: Wood x24, Rock x32, or one rare mineral. Every village wants different colours, which is what gives colours a value. |
| Guide | Tips about the dangers (how slimes behave), colours, minerals, and **where the Slime King lives**. Telling you puts a line in the HUD. |
| Request | Bring 2 minerals of the villager's rare colour: +5 storage slots in this village (once). |
| Services | **Only after the boss.** Paid in the new colours, once per village: +10 storage slots, or a Vitality Tonic (+10 max HP, permanent). |

## Minerals

- **Colours:** 8 levels per channel (60, 80, ... 200) = 512 colours (1000 after the boss). Your slime starts grey (130,130,130).
- **Where:** ore is a single small dot on a rock face. Underground ore is about 3.2% of rock near the surface and 4.1%
  below 20 blocks (the deeper, the richer); surface boulders hold little. Only the original gray ROCK holds ore.
- **Rarity** follows how vivid the colour is: muted = common (stack 32, ~2.3 s to dig), vivid = rare (stack 16, ~3.8 s).
  Uncommon colours start 8 blocks below the surface, rare ones at 20. Digging ore gives the mineral and the rock;
  if the inventory is short of space the rock is lost first.
- **Looking at ore:** the HUD shows your colour/hue/saturation, and a "Held" swatch shows the true colour of the selected mineral.
- **Eating (E):** within `EAT_SAFE_DISTANCE` (RGB distance) of your colour it heals up to 25% of max HP; beyond it you
  lose up to 15% (never below 1 HP). Either way your colour moves 20% toward the mineral.
- **Matchup** (`game/colors.py`): `1 + 0.5 * sin(hue difference) * saturation factor`. A one-way three-cycle, grey is neutral,
  and it is mirrored when attacker and defender swap. It applies to tackles, mineral hits and enemy bites alike.

## World

- **Terrain and water:** terrain features are small (`TERRAIN_SCALES`), so lakes and rivers are narrow: 90% of water
  crossings are under about 40 blocks, 99% under about 70 (2-3 stacks of Wood for a bridge). Wide seas exist but are rare.
- **Caves** are Minecraft-like: winding tunnels, a finer second network and big caverns, about 13% of the underground,
  almost all of it one connected network. A scatter of **entrances** (about 3% of land columns) lets caves break through
  to the surface; trees and boulders keep clear of them. Lake beds and the bottom three layers stay sealed (`CAVE_*`).
- **Day and night:** one day lasts 10 minutes (`DAY_LENGTH`); a new game starts in the morning. **Only the sky changes**
  (light blue by day, orange at sunrise/sunset, dark blue at night); the fog shares the sky colour so the horizon blends,
  and everything near you stays as bright as by day (`NIGHT_WORLD_TINT` can darken the whole world; off by default).
  Underground nothing changes. The time of day is saved.
- **Maps:** a new game gets a **random map** (`world/seeds.py`); a saved game keeps its own map (its villages, caves and
  your edits belong to it). The pause menu has **New world** (asks first; the old game is kept in a backup folder next to
  the save) and shows the current seed. The start point is the nearest open patch of grass to the origin, never inside a
  tree or a hole (`world/spawn.py`).
- **Music** is composed in code (`audio/music.py`, `audio/director.py`) and written to `assets/music/` (about a second at
  start-up): a slow (72 BPM) C-major-pentatonic bell melody over pads while you explore, and a fast (156 BPM)
  A-minor-pentatonic riff with drums while a slime chases you. Every launch composes new tunes, and a new tune of each
  kind is composed in the background every time one is used. Panda3D streams sound files over 1 MB, which can stutter,
  so `main.py` sets `audio-preload-threshold`, and a watchdog restarts a tune that has stopped. `SLIME_NO_MUSIC=1` turns the
  music off; volume is `MUSIC_VOLUME`.

## Snowy regions

Cold-climate areas appear naturally within the ordinary world; no separate map or new game mode is needed. The temperature field is shared with the tree-species climate: colder patches become snow-covered, exposed lake water becomes walkable ice while the water remains underneath, and the existing pine-tree preference makes snowy forests feel distinct. Snow replaces only the surface grass block, so terrain height, caves, underground minerals and the surrounding map layout stay unchanged. Village plots are kept clear.

## Flower cultivation

Digging Soil or Grass has a 2.5% chance to uncover a colour-coded flower seed. Select the seed and right-click a Grass/Soil block beside water to plant it in the empty cell above. The seedling appears on a thin brown tile with a small green sprout; after one full in-game day (600 seconds), the tile shows a coloured flower. The tile is decorative and does not behave like a full-height solid block. Breaking an immature seedling returns its seed; breaking a mature flower yields the mineral with the same colour code. Plant colour and planting time are saved with the chunk, so growth continues correctly after reloading.

## Flower cultivation

Digging Soil or Grass has a 2.5% chance to uncover a colour-coded flower seed. Select the seed and right-click a Grass/Soil block beside water to plant it in the empty cell above. The seedling appears on a thin brown tile with a small green sprout; after one full in-game day (600 seconds), the tile shows a coloured flower. The tile is decorative and does not behave like a full-height solid block. Breaking an immature seedling returns its seed; breaking a mature flower yields the mineral with the same colour code. Plant colour and planting time are saved with the chunk, so growth continues correctly after reloading.

## MODs

The game supports **data-only MODs**: folders of JSON files in `mods/` (or in the folder named by `SLIME_MODS_DIR`). The game never
runs code from a MOD. A MOD can add **new enemy slime types** and **custom cube blocks** with individual colours, hardness, sounds,
and inventory drops.

Included examples: `mods/crystal_slime` adds a Crystal Slime; `mods/sandstone_colors` adds four sandstone colours;
`mods/tree_species` adds birch, pine and darkwood logs and leaves; `mods/common_rocks` adds granite, basalt, limestone and red rock. Dry regions naturally generate sandstone layers, small, scattered geological patches naturally generate the four ordinary rock varieties underground, with rare matching outcrops also appearing on the surface, and climate-based forests naturally grow the tree species. The four common rock varieties do not contain minerals; mineral deposits remain exclusive to the original gray ROCK blocks. Custom wood logs can also be placed on water as bridges.

**To add a MOD:** copy `mods/_template`, rename the copy, and edit `mod.json` plus `slimes.json` and/or `blocks.json`. Restart the game.
The full format, limits and troubleshooting guide are in **[MODDING.md](MODDING.md)**.

**Feedback while you work:** a MOD that cannot load is skipped (the others still load) and the game tells you at the start, in the
terminal and on screen; the pause menu (Esc) lists the loaded MODs and counts problems. A misspelt key is ignored and reported as a
warning, so it never fails silently. A MOD has hard limits (file size, slimes per MOD, number of MODs) so that a mistake cannot break
the game. Saves do not depend on MODs, so removing a MOD never damages a save.

Not supported yet: custom minerals, village trades, music, textures, translations, custom shapes and scripts. Python-script MODs are
intentionally not enabled, because they could run arbitrary code on the player's computer.

## Saving

- Location: `saves/default/` next to `main.py` (override with the `SLIME_SAVE_DIR` environment variable).
- Saved: world seed, player position/view, inventory, selected slot, spawn point, every block you changed, level, XP, the
  four stats, HP, colour, time of day, village storage and progress, the inn respawn point, whether the boss has fallen
  and whether the guide has told you where he lives. Slimes are not saved (they come from the seed).
- Save from the Esc menu. Autosave every `AUTOSAVE_SECONDS` (120 s), on "Save and quit", when the window is closed, and
  right after a death or the boss's fall.
- "Load last save" rolls the world back to the last save; everything since then is discarded.
- No function keys are used, so nothing can clash with Ursina's own hotkeys.
- Format: `meta.json` plus one `chunks/c_<cx>_<cz>.json` per edited chunk (a diff over generated terrain, so only changed
  chunks are written). Writes go to a temp file and are renamed into place.
- `config.SAVE_VERSION` is the file format; a save from a newer build is refused untouched. A corrupt save is moved to
  `default.corrupt-<time>` and a new world starts. `config.WORLDGEN_VERSION` (now 12) flags terrain changes: bump it
  whenever you change terrain generation, since old diffs may no longer fit. **Saves from before the action-combat
  build show a warning** (the Slime King's lair moved and grew); use `--new` for a clean start.

## Tests (no Ursina needed)

    python -m unittest discover -s tests -t .     # 261 engine-free tests
    python tests/smoke_stub.py                    # runs main.py's loop against a fake ursina module

`tests/test_action.py` covers the combat rules, aiming, chasing, steps, placement and the manager;
`tests/smoke_stub.py` plays a whole session (digging, eating, chasing, bites, kills, Wood safety, steps, night, death,
saving, the Slime King's lair and fight).

## Layout

    main.py            Ursina glue only: window, camera, HUD, chunk entities, slime drawing, input
    config.py          every tunable number
    strings.py         every player-facing string (English)
    blocks.py          built-in block types; MOD blocks add their own hardness, drops and colours
    world/             noise, worldgen, villages (tile + signpost), boss_arena (the lair), enemy_spawn (where slimes
                       stand), seeds, spawn, world, mesher, manager
    save/              store (files, atomic writes, validation), game_save (live objects <-> files)
    items.py           item ids; minerals = 1000 + colour code (rarity tiers, stack limits, dig times)
    game/              raycast, physics, inventory, mining, building, fog, colors, eating, stats,
                       progression (levels, xp, enemy level), combat (hits, aiming, bites), enemies (slimes: chase,
                       steps, knock-back, manager), battle (the damage formulas and enemy makers), death, daytime,
                       village (command screens as a state machine), village_data (what each village offers)
    audio/sfx.py       procedural sound effects -> assets/sfx/*.wav (written on first run)
    audio/music.py     composes and renders the background tunes; audio/director.py keeps them playing and makes new ones

Old code kept: the `Battle` class in `game/battle.py`, `game/presenter.py`, `game/sprites.py`, `game/encounter.py` and
`BossGate` in `game/village.py` belong to the removed turn-based screen. The game no longer uses them (their tests still
pass); delete them when you no longer need them.

## Tuning

- Dig times: `blocks.py` (`hardness`). Hit-sound rate: `config.HIT_INTERVAL`.
- Combat: `config.py`, "Stage 9: action combat" (`ENEMY_*`, `KNOCKBACK_*`, `PLAYER_ATTACK_*`, `MINERAL_ATTACK_COOLDOWN`,
  flash times) and the Slime King's lair (`ARENA_*`, `BOSS_*`).
- Terrain / lakes / caves / trees: `config.py` (`TERRAIN_*`, `CAVE_*`, `TREE_CHANCE`). View distance: `RENDER_RADIUS` (chunks).

## Known limits of this build

- Written and tested without a real Ursina install for most of its life. Engine-free logic is covered by tests; the Ursina
  calls in `main.py` are only exercised against a fake module. If something looks wrong on first run, check these first:
  1. Slimes: they are drawn as `Entity(model="cube")` with two child cubes as eyes, and the red hit flash is a full-screen
     `quad` in `camera.ui` (`make_enemy_entity()`, `set_flash()`). If a slime looks too dark or its eyes are on the wrong
     side, look at `make_enemy_entity()` (the eyes sit on the local +Z face; `rotation_y` = `ActionEnemy.facing`).
  2. Fog not fading distant terrain: `on_mesh()` sets `e.shader = None` so scene fog applies.
  3. Terrain colours missing: vertex colours on `Mesh(colors=...)`.
  4. No sounds: `Audio(name)` looks the wav up under the project folder.
  5. Crack overlay invisible: `make_crack_textures()` (needs Pillow; falls back to a dark overlay).
  6. Night darkening uses `scene.setColorScale`; if the world does not darken at night, look at `apply_fog()`.
- Block meshes are opaque, including water and leaves.
- A slime fits in a one-block-wide corridor; the Slime King (2 x 2) needs open space, which his big room provides.
- Slimes are simulated only within `ENEMY_ACTIVE_DISTANCE` (20 blocks) and drawn within `ENEMY_RENDER_DISTANCE` (40).
