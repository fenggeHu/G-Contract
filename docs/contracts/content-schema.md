# Contract: Content Schema (field-level)

- Status: Active
- Updated: 2026-09-20

> The **authority on fields and semantics** for content data. The machine-readable Schema is exported at build time as `schema/content-<ver>.json` and loaded by the validator.

## 0. General Conventions

- Format: JSON (UTF-8). Naming: `id` is globally unique, see world-and-scenes.md §7.
- Common fields: `type` (required), `id` (required for concrete Defs), `labelKey` (localization Key; always use it for text), `tags[]`.
- All coordinate units: **meters**; for altitude see §8.

### 0.1 Inheritance (JSON representation)

```json
{ "type": "city", "name": "base_city", "abstract": true,
  "landing": { "radius": 64, "requires": ["flight"] } }

{ "type": "city", "id": "city_ironhold", "parent": "base_city",
  "labelKey": "CITY_IRONHOLD", "worldPos": { "x": 2410, "y": 1180 } }
```

- `abstract: true` is not stored, it only serves as a parent; `parent` references by `name`.
- Children override parent fields of the same name; arrays are **replaced** by default (`mergeMode: "append"` switches to append).

### 0.2 Patch (differential override)

```json
{ "type": "patch", "target": "city_ironhold",
  "ops": [ { "op": "set", "path": "landing.radius", "value": 80 },
           { "op": "remove", "path": "legacyFlag" } ] }
```

- Supports `set` / `remove` / `append`, paths are dot-separated; applied before building the DB.

### 0.2b Relations registry (reference integrity)

Def references are validated against `shared/model/relations-<ver>.json`: each entry declares `{ type, field, target, cardinality, list, crossPack }`. The registry is authored once in `G_Engine/App/engine/sdk/model/` and emitted by `content sync`; `content validate` reports `REF_NOT_FOUND` from it, and Content Studio uses it for graph/navigation. See [content-authoring.md](content-authoring.md) §3.

### 0.3 Multi-Def file packaging

```json
{ "defs": [ { "type": "city", "id": "city_ironhold" },
            { "type": "biome", "id": "biome_highland" } ] }
```

- A single-object file is equivalent to `{"defs":[...]}`; both the validator and the builder must support both forms.

## 1. `manifest`
See [content-package.md](content-package.md).

## 2. `world` (world map)

| Field | Type | Required | Description |
|---|---|---|---|
| `seed` | int | Yes | Deterministic seed |
| `sizeM` | {x,y} | Yes | Default 8192×8192 |
| `chunkSizeM` | int | Yes | Default 256 |
| `rasterCellM` | int | Yes | Coarse granularity of the world map, default 8 |
| `imageDir` | string | No | Baked top-down texture subdirectory (`world/<x>_<y>.jpg`) |
| `zones[]` | {id,rect} | No | Zones (macro), 4–9 planned in pre-research |
| `route[]` | {x,y} | No | Flight routes (bounded/looping) |
| `biomeMap` | reference | Yes | Biome distribution |
| `overrides[]` | reference | No | Manual overrides |
| `defaultLighting` | reference | No | Lighting preset |
| `defaultEntry` | reference | No | Default spawn (a `poi` id); absent → world center |
| `procgen` | object | No | Optional generation inputs for `content bake-world`: `height`/`temperature`/`moisture` (`{scale,octaves,seed}`; temperature also `latitudeBias`), `hydrology` (`{rivers,threshold,waterLevel}`), `erosion` (`{strength}`), `plates` (reserved). **Absent = current deterministic fBm** (backward compatible) |

## 3. `biome`

| Field | Type | Required | Description |
|---|---|---|---|
| `terrainSet` | reference | Yes | Tile set |
| `lighting` | reference | Yes | Lighting preset |
| `ambience` | reference | No | Ambience sound events |
| `spawnTable` | reference | No | Spawn/resource table |
| `weather[]` | string | No | Weather |

## 4. `terrain` (terrain data format, **not a Def type**)

> Note: `terrain` is not a Def that can be declared independently (there is no such type in the machine Schema); `scene.terrain` is a Tiled file path string.

**Reuse a mature format**: scene terrain is authoritatively stored as a **Tiled map (`.tmj`, JSON)**, loaded at runtime by the engine via the `terrain.tiled` capability (Godot `TileSet` + `TileMapLayer`). **Do not build a custom format.**

| Tiled layer | Purpose |
|---|---|
| `ground` / `detail` | Visual layers |
| `height` | Height levels (platforms/cliffs) |
| `pathing` | Walkability bitmap (0/1, not rendered) |

- Layer purpose is annotated with a Tiled **custom property** (`class`).
- `tileSet` points to a Godot `TileSet` resource (atlas + tiles + auto-tiling).
- The world map does **not** use this format; it uses a coarse raster (see §2 and world-and-scenes.md §2.1).

## 5. `scene` (local scene)

| Field | Type | Required | Description |
|---|---|---|---|
| `kind` | enum | Yes | `city/village/cave/interior` |
| `file` | string | No | Scene file (`.tscn`) path |
| `sizeM` | {x,y} | Yes | See the scale specification |
| `terrain` | reference | Yes | Terrain data (§4) |
| `regions[]` | reference | No | Regions |
| `objects[]` | {def,pos,altitude?,rot} | No | Initial objects/NPCs |
| `lighting` | reference | No | Lighting preset |
| `music` | reference | No | Music event |
| `entryPoints[]` | {id,pos,altitude?} | No | Entrances (city gates/cave mouths) |

## 6. `region`

| Field | Type | Required | Description |
|---|---|---|---|
| `shape` | rect/poly | Yes | Shape |
| `purpose` | enum | Yes | `trigger/music/spawn/patrol/landing` |
| `onEnter` / `onExit` | script reference | No | Callbacks |
| `params` | object | No | Purpose parameters |

## 7. `poi` (world map point of interest)

| Field | Type | Required | Description |
|---|---|---|---|
| `kind` | enum | Yes | `city/cave/landmark/portal` |
| `worldPos` | {x,y} | Yes | Meter coordinates (±4096) |
| `altitude` | number | No | Absolute altitude (meters) |
| `scene` | reference | No | Hosting scene |
| `icon` | reference | No | Map icon |
| `landing` | object | No | `{radius, requires}` |
| `reveal` | enum | No | `visible/undiscovered` |

## 8. Altitude

- Always **absolute meters** (0 = sea level), stored in `poi.altitude`, `scene.objects[].altitude`, `entryPoints[].altitude`.
- While flying, the player has an independent altitude; on landing it aligns with the POI/entrance altitude.
- It only affects flight rules and presentation, and does **not enter 2D collision solving** (see physics.md).
- **Exception**: ground traversal elevation (`navgrid.altitude`, §24) participates in pathfinding walkability/cost determination, but still does not enter collision solving.

## 9. `npc`

| Field | Type | Required | Description |
|---|---|---|---|
| `species` | reference | Yes | Appearance/species |
| `faction` | reference | No | Faction |
| `stats` | reference | No | Stats (§13) |
| `abilities[]` | reference | No | Abilities (§14) |
| `behavior` | reference | No | Behavior script |
| `dialog` | reference | No | Dialog tree (§18) |
| `spawn` | reference | No | Spawn rules (§12) |

## 10. `item`

| Field | Type | Required | Description |
|---|---|---|---|
| `category` | enum | Yes | `weapon/armor/artifact/consumable/quest/resource` |
| `slot` | string | No | Equipment slot id (references `equipment.slots[]`, §25) |
| `grantsAbility[]` | reference | No | Abilities granted while equipped (capability `ability.grant`); references `ability` Defs |
| `stats` | object | No | Values (§13) |
| `stack` | int | No | Stack limit |
| `icon` | reference | Yes | Icon |
| `dropWeight` | number | No | Drop weight |

## 11. `lighting_profile`

| Field | Type | Required | Description |
|---|---|---|---|
| `preset` | enum | Yes | `lotr_bright/iceland_cold/highland_gloom` |
| `timeOfDay` | enum | No | `dawn/noon/golden/night` |
| `ambient` | color | Yes | Ambient color |
| `fogLayers` | int | No | Number of fog layers |
| `post` | object | No | Bloom/LUT/vignette |

## 12. `spawn`

| Field | Type | Required | Description |
|---|---|---|---|
| `targets[]` | reference | Yes | Defs to spawn |
| `count` | {min,max} | Yes | Count |
| `area` | reference | Yes | Area |
| `conditions` | object | No | Time/weather/quest |
| `respawnMs` | int | No | Respawn |

## 13. `stats` (attributes)

| Field | Type | Required | Description |
|---|---|---|---|
| `base` | map<string,number> | Yes | Base attributes (hp/attack/defense…) |
| `modifiers[]` | {stat,op,value,source?,durationMs?} | No | `op: flat/percent` |
| `resources[]` | {id,max,regenPerSec?} | No | Resource pools (e.g. mana/energy); `ability.cost.resource` references their `id` |
| `xp` | int | No | Kill reward experience |
| `loot[]` | {item,chance} | No | Loot table |

## 14. `ability` (abilities)

| Field | Type | Required | Description |
|---|---|---|---|
| `cost` | {resource,amount} | No | Cost; `resource` references `stats.resources[].id` |
| `cooldownMs` | int | No | Cooldown |
| `cooldownGroup` | string | No | Cooldown group id; abilities in the same group share cooldown timing (generic primitive) |
| `castMs` | int | No | Cast time |
| `range` | number | No | Range (meters) |
| `target` | enum | Yes | `self/enemy/ally/point/area` |
| `source` | enum | No | `player/weapon/artifact`; advisory source for granted abilities (capability `ability.grant`) |
| `effects[]` | reference | Yes | Effects (§15) |
| `script` | script reference | No | Special logic |

## 15. `effect` (effects)

| Field | Type | Required | Description |
|---|---|---|---|
| `kind` | enum | Yes | `damage/heal/buff/debuff/summon` |
| `amount` | number/formula | No | Value |
| `statModifiers[]` | reference | No | Attribute modifiers (`{stat,op:flat/percent,value}`); applied by capability `effect.aura` |
| `durationMs` | int | No | Duration |
| `stacks` / `maxStacks` | int | No | Initial stacks / max stacks |
| `stackMode` | enum | No | `refresh` (re-apply refreshes duration) / `stack` (accumulate stacks) |
| `tags[]` | string | No | Classification (the content layer uses this to express rules such as resistance/immunity) |

## 16. `interaction`

| Field | Type | Required | Description |
|---|---|---|---|
| `promptKey` | Key | Yes | Prompt text |
| `range` | number | No | Interaction distance |
| `conditions` | object | No | Preconditions |
| `actions[]` | script/effect/quest/dialog | Yes | Triggered actions |

## 17. `quest` (quests)

| Field | Type | Required | Description |
|---|---|---|---|
| `objectives[]` | {kind,target,count} | Yes | `kill/collect/reach/talk` |
| `states[]` | enum | Yes | State machine (inactive/active/done) |
| `rewards` | {items[],xp} | No | Rewards |
| `prerequisites[]` | reference | No | Prerequisites |
| `onComplete` | script reference | No | Completion callback |

## 18. `dialog` (dialogue)

| Field | Type | Required | Description |
|---|---|---|---|
| `nodes[]` | node | Yes | Dialog nodes |
| Node fields | `id` / `speaker` / `textKey` / `conditions` / `actions[]` / `choices[]` | | `choices: {textKey,next,conditions}` |

## 19. `sound_event` (audio events)

> Event source for the capabilities `audio.play` / `audio.music` / `audio.mixer`; `biome.ambience` / `scene.music` / `region.music` all reference this type.

| Field | Type | Required | Description |
|---|---|---|---|
| `assets[]` | resource path | Yes | Candidate audio (one is picked at random) |
| `bus` | enum | No | `master/sfx/music/ui`, default `sfx` |
| `volume` | number | No | Linear volume (default 1.0) |
| `pitchRange` | [min,max] | No | Random pitch range |
| `loop` | bool | No | Loop (music) |
| `cooldownMs` | int | No | Minimum interval for the same event |
| `maxInstances` | int | No | Maximum concurrency for the same event |
| `priority` | int | No | Preemption priority (higher wins) |
| `attenuation` | number | No | Distance attenuation (reserved) |

## 20. Versioning and Compatibility

| Object | Rule |
|---|---|
| Schema version | `MAJOR.MINOR`, independent of the engine |
| Engine support | Declares the set of supported Schema major versions (≥ the most recent two); **machine-readable** as `capabilities.json.schemaSupported` and `release.json.schemaSupported` ([content-tooling.md](content-tooling.md) §5) |
| Fields | Add-only, never delete; deletion requires MAJOR + deprecation period |
| Enums | Additions are MINOR; removal requires MAJOR |

## 21. Tooling

For the machine-readable Schema, validator CLI, and capability matrix see [content-tooling.md](content-tooling.md).

## 22. `character` (character controller profile)

> Def parameters for the engine capability `movement.character`. Units: **meters, seconds**; at runtime converted to pixels via PPU. The specific gameplay (whether sprint/climb/mount are enabled) is decided by the content.

| Field | Type | Required | Description |
|---|---|---|---|
| `controller` | enum | No | `ground/flight`, default `ground` |
| `collision` | object | No | `{shape: circle/aabb, radius, size{x,y}, layer, mask}` |
| `moveSpeed` | number | No | Walk speed (default 6) |
| `accel` / `friction` | number | No | Acceleration / friction |
| `gravity` / `maxFallSpeed` | number | No | Gravity / terminal fall speed |
| `jump` | object | No | `{height, maxCount, cooldownMs}` |
| `sprint` | object | No | `{speed, staminaPerSec}` |
| `climb` | object | No | `{speed, mask}` |
| `fly` | object | No | `{speed, accel, altitudeMin, altitudeMax}` (altitude is a separate channel) |
| `mount` | object | No | `{speed, accel, turnSpeed}` |
| `camera` | reference | No | Camera profile (§23) |
| `equipment` | reference | No | Equipment slot set (§25) |
| `species` | reference | No | Appearance/species and customization parameters (§28) |

## 23. `camera` (camera profile)

> Def parameters for the engine capability `camera.control`; a presentation layer that does not participate in logical authority.

| Field | Type | Required | Description |
|---|---|---|---|
| `follow` | object | No | `{smooth, deadzone}` |
| `rotation` | object | No | `{mode: fixed/follow/free, smooth}` |
| `distance` | object | No | `{min, max, default, step}` (zoom levels) |
| `collision` | object | No | `{enabled, mask, padding}` |
| `shake` | object | No | `{decay}` |

## 24. `navgrid` (pathfinding grid)

> Data source for the engine capability `path.find`: walkability bitmap + ground elevation. Used by `PathFinder` (altitude-aware grid A*), units: `cellM` meters/cell, `altitude` meters.

| Field | Type | Required | Description |
|---|---|---|---|
| `cols` / `rows` | int | Yes | Grid column/row counts |
| `cellM` | number | No | Cell edge length (meters), default 1 |
| `walkable` | array | Yes | Walkability bitmap (0/1); 1D `cols*rows` or 2D `rows × cols` |
| `altitude` | array | No | Ground elevation per cell (meters), same shape as `walkable`; defaults to all 0 |

> Steep-wall determination and cost rules (`maxClimb` / `stepPenalty` / `eightWay`) are passed in by the caller at query time and are not part of the Def.

## 25. `equipment` (equipment slot set)

> Generic primitive: the slot set is **declared by content**; the keys of `item.slot` and the save-game `player.equipped` both reference this set. The engine does not have built-in fixed slots (it does not hardcode `weapon/armor/artifact`).

| Field | Type | Required | Description |
|---|---|---|---|
| `slots[]` | {id,labelKey?} | Yes | Slot list; `id` is unique, `labelKey` is used by the UI |

## 26. `fx` (visual effects)

> Presentation source for the capability `fx.play` (pure presentation, does not affect logical authority).

| Field | Type | Required | Description |
|---|---|---|---|
| `scenePath` | string | No | Presentation scene (`.tscn`) path |
| `attach` | enum | No | `self/target/point` (default `point`) |
| `durationMs` | int | No | Auto-destroy duration (ignored when `loop`) |
| `loop` | bool | No | Looping presentation |

## 27. `conditions` (condition expressions)

> Expression format for the capability `conditions.core`; used for the `conditions` field of `interaction.conditions` / `region.params` / `spawn.conditions` / dialog nodes, etc.
> Evaluation: `Conditions.eval(spec, ctx)`; **empty/absent = true**. `ctx` can override data sources (`flags/quests/items/stats/level/kv`) for testing or server authority.

| Form | Description |
|---|---|
| `{ "all": [expr…] }` | All true (the array form is equivalent to all) |
| `{ "any": [expr…] }` | Any true |
| `{ "not": expr }` | Negation |
| `{ "flag": "key", "eq": bool? }` | Feature flag (`config.flag`) |
| `{ "quest": id, "state": "active/ready/done/inactive" }` | Quest state (default `active`) |
| `{ "item": id, "count": n? }` | Inventory possession (default 1) |
| `{ "stat": name, "op": ">=/<=/>/</==/!=", "value": n }` | Derived attribute (default `>=`) |
| `{ "level": n, "op": ">=…" }` | Level |
| `{ "kv": key, "value": any }` | Save-game KV equality |

## 28. `species` (appearance / parameterized customization)

> Generic primitive for **player appearance**. Content declares the base presentation and the **customization parameter table**; the player picks values, the platform validates/normalizes them and replicates a compact key. The platform defines only the **parameter format** (no art/values). Referenced by `character.species` and `npc.species` (`npc` uses the defaults).

| Field | Type | Required | Description |
|---|---|---|---|
| `presentation` | string | No | Presentation scene (`.tscn`) path; the engine instantiates it and calls `apply_params(params)` on its root |
| `params[]` | param | No | Customization parameters; order is stable and defines the canonical/compact encoding |

**`params[]` entry**

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | id | Yes | Parameter id (unique within the species; key in save `player.appearance.params`) |
| `kind` | enum | Yes | `color` (`#rrggbb` string) / `enum` (one of `options`) / `float` / `int` |
| `default` | any | No | Default value when absent (also used by save migration) |
| `min` / `max` | number | No | Range for `float` / `int` (server clamps on save) |
| `step` | number | No | Quantization step for `float` / `int` |
| `options` | array | No | Allowed values for `enum` |
| `part` | string | No | Presentation hint (e.g. a slot/attachment name); opaque to the platform |
| `labelKey` | Key | No | UI label |

- **Canonical form**: `Player.appearance = { "species": <id>, "params": { <paramId>: <value> } }`; values are normalized to the param's kind/range before persistence (`Appearance.normalize`).
- **Compact replication**: `akey = "<species>@<hash>"` (hash of the canonical params). Snapshots carry `akey` only; the full params are fetched on demand via RPC `appearance_get` ([protocol.md](protocol.md) §8).
- **Server authority**: on cloud write the platform validates/clamps params against the content-projected schema ([save-schema.md](save-schema.md) §10); the save is the source of truth for a player's appearance.

## 29. `sprite_atlas` (sprite atlas + region table)

> Generic primitive for the capability `sprite.atlas`. Content ships **one atlas PNG + a region table** instead of N individual frame PNGs; the engine loads it and exposes regions/frames as `Texture2D` / `SpriteFrames`. The platform defines only the format (no art).

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | string | Yes | Atlas texture path (pack-relative) |
| `regions[]` | region | Yes | `{ name, x, y, w, h }` — pixel region in the atlas; `name` is the lookup key |
| `frames` | map<string,array> | No | Animation name → ordered region names (e.g. `{ "walk": ["walk_0","walk_1"] }`) |

- API: `SpriteAtlas.from_id(id)`, `region(name) -> Texture2D`, `frame_textures(anim)`, `sprite_frames()` (see [engine-sdk.md](engine-sdk.md) §3).
- Packing the PNG itself is content-side (atlas + region table); the engine only loads.