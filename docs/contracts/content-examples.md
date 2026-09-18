# Content Examples (Minimal Working / Common Errors)

- Status: Active
- Updated: 2026-09-17

> For AI / content authors: the **minimal working examples** and **common errors** for each type of Def. Field semantics are governed solely by [content-schema.md](content-schema.md) as the single source of truth; use `content validate` / `content lint` for machine validation.
> Each Def file takes the form `{"defs": [ … ]}` (see [content-schema.md](content-schema.md) §0.3).

## 0. General Conventions (Remember First)

- `type` is required and must be a registered type; `id` naming follows `^[a-z][a-z0-9_]*$` (convention: `type_lowercase_underscores`).
- **User-visible types must have a `labelKey`**: `biome` / `lighting_profile` / `scene` / `poi` / `item` / `npc`; packs that declare `entry.i18n` must also have corresponding translations.
- References (`lighting` / `scene` / `item` / `effect` / `dialog` / `stats` …) must point to existing Defs, otherwise `REF_NOT_FOUND`.
- **Common errors**: `"id": "IronHold"` (uppercase), missing `type`, duplicate `id`, missing `labelKey`, dangling references.

## 1. manifest
See [content-package.md](content-package.md) §2.
```json
{ "id": "world.pack.aurelia", "version": "1.0.0", "schema": "1.0",
  "requires": { "engine": ">=0.1.0 <2.0", "packs": ["base.core@1.x"] },
  "capabilities": ["scene.switch@>=1.0"], "loadAfter": ["base.core"],
  "entry": { "defs": "defs/", "scenes": "scenes/", "i18n": "i18n/" } }
```
❌ The `schema` major version is not in the engine's `schemaSupported`; declaring an unknown/deferred capability (`CAP_MISSING`) or an engine range that is not satisfied (`DEP_MISSING`).

## 2. world
```json
{ "type": "world", "id": "world_aurelia", "seed": 1337,
  "sizeM": { "x": 2048, "y": 2048 }, "chunkSizeM": 256, "rasterCellM": 8,
  "imageDir": "world", "biomeMap": "biome_highland" }
```
❌ Missing `biomeMap`; `imageDir` does not match the actual bake directory.

## 3. biome
```json
{ "type": "biome", "id": "biome_highland", "labelKey": "BIOME_HIGHLAND",
  "terrainSet": "tileset_highland", "lighting": "lighting_lotr", "weather": ["clear", "fog"] }
```
❌ Missing `labelKey`; `lighting` is dangling.

## 4. lighting_profile
```json
{ "type": "lighting_profile", "id": "lighting_lotr", "labelKey": "LIGHTING_LOTR",
  "preset": "bright_epic", "timeOfDay": "golden" }
```

## 5. scene
```json
{ "type": "scene", "id": "scene_ironhold_gate", "labelKey": "SCENE_IRONHOLD_GATE",
  "kind": "city", "file": "scenes/ironhold_gate.tscn", "sizeM": { "x": 256, "y": 256 },
  "terrain": "terrain/ironhold_gate.tmj", "lighting": "lighting_lotr",
  "entryPoints": [ { "id": "gate", "pos": { "x": 128, "y": 240 }, "altitude": 0 } ] }
```
❌ `file` uses `.scene` (should be `.tscn`); the `terrain` file does not exist (`ASSET_MISSING`); the asset is not registered in `licenses.json` (`LICENSE_MISSING`, see content-package §6).

## 6. region
```json
{ "type": "region", "id": "ironhold_region_landing",
  "shape": { "x": 0, "y": 0, "w": 64, "h": 64 }, "purpose": "landing", "params": {} }
```
❌ `purpose` is not in the enum `trigger / music / spawn / patrol / landing`.

## 7. poi
```json
{ "type": "poi", "id": "poi_ironhold", "labelKey": "POI_IRONHOLD", "kind": "city",
  "worldPos": { "x": 1200, "y": -340 }, "altitude": 120, "scene": "scene_ironhold_gate",
  "landing": { "radius": 64, "requires": ["flight"] }, "reveal": "visible" }
```
❌ `kind` is not in the enum `city / cave / landmark / portal`; `scene` is dangling; `worldPos` exceeds `±sizeM/2`.

## 8. npc
```json
{ "type": "npc", "id": "npc_guard", "labelKey": "NPC_GUARD", "species": "human",
  "stats": "stats_guard", "dialog": "dialog_guard", "behavior": "idle", "faction": "ironhold" }
```

## 9. item
```json
{ "type": "item", "id": "item_steel_sword", "labelKey": "ITEM_STEEL_SWORD",
  "category": "weapon", "slot": "weapon", "icon": "icon_sword", "stack": 1, "stats": { "attack": 10 } }
```
❌ `category` is not one of the agreed values (`weapon / armor / artifact / consumable / quest / resource`); missing `labelKey`; `slot` is not declared in `equipment.slots[]`.

## 10. interaction
```json
{ "type": "interaction", "id": "inter_gate_open", "promptKey": "UI_OPEN",
  "range": 48, "actions": [ { "op": "teleport", "scene": "scene_ironhold" } ] }
```

## 11. spawn
```json
{ "type": "spawn", "id": "spawn_grunts", "targets": ["stats_grunt"], "count": 2,
  "area": "ironhold_region_landing", "respawnMs": 30000 }
```

## 12. stats
```json
{ "type": "stats", "id": "stats_grunt",
  "base": { "hp": 60, "attack": 8, "defense": 1, "moveSpeed": 70 },
  "resources": [ { "id": "mana", "max": 50, "regenPerSec": 5 } ],
  "xp": 40, "loot": [ { "item": "item_rusty_sword", "chance": 1.0 } ] }
```
❌ `loot[].item` is dangling; `base` is missing key attributes; `ability.cost.resource` references an `id` that is not declared in `resources[]`.

## 13. ability
```json
{ "type": "ability", "id": "ability_strike", "labelKey": "ABILITY_STRIKE",
  "cost": { "resource": "mana", "amount": 5 }, "cooldownMs": 600, "cooldownGroup": "gcd",
  "range": 60, "target": "enemy", "effects": ["effect_strike_dmg"] }
```
❌ `effects[]` references a non-existent effect; `cost.resource` is not declared in `stats.resources[]`.

## 14. effect
```json
{ "type": "effect", "id": "effect_strike_dmg", "kind": "damage", "amount": 1.0 }
```
❌ `kind` is not one of the agreed values (`damage` is implemented; `heal / buff / debuff / summon` are reserved in the design, see content-schema §15).

## 15. quest
```json
{ "type": "quest", "id": "quest_grunt", "labelKey": "QUEST_GRUNT",
  "objectives": [ { "kind": "kill", "target": "grunt", "count": 1 } ],
  "states": ["inactive", "active", "ready", "done"],
  "rewards": { "items": ["item_talisman_fire"], "xp": 50 } }
```
❌ The objective type is not recognized by the engine; the reward item is dangling.

## 16. dialog
```json
{ "type": "dialog", "id": "dialog_guard", "labelKey": "DIALOG_GUARD",
  "nodes": [
    { "id": "start", "speaker": "guard", "textKey": "D_GUARD_HELLO",
      "choices": [ { "textKey": "D_ACCEPT", "next": "end",
        "actions": [ { "op": "start_quest", "quest": "quest_grunt" } ] } ] },
    { "id": "end", "speaker": "guard", "textKey": "D_GUARD_BYE", "choices": [] } ] }
```
❌ `next` points to a non-existent node; `op` is not an implemented action (currently supported: `start_quest` / `turn_in_quest`).

## 17. patch
```json
{ "type": "patch", "target": "poi_ironhold",
  "ops": [ { "op": "set", "path": "landing.radius", "value": 80 } ] }
```
❌ `target` does not exist (`REF_NOT_FOUND`); `ops` is missing `op/path`. Paths are **dot-separated** (`landing.radius`), not JSON Pointer.

## 18. character (character controller profile)
```json
{ "type": "character", "id": "char_player", "controller": "ground",
  "collision": { "shape": "circle", "radius": 0.5, "layer": 1, "mask": 3 },
  "moveSpeed": 6.0, "accel": 40.0, "friction": 60.0, "gravity": 22.0, "maxFallSpeed": 40.0,
  "jump": { "height": 2.4, "maxCount": 1 }, "sprint": { "speed": 9.0 },
  "climb": { "speed": 3.0 }, "fly": { "speed": 40.0, "altitudeMin": 0, "altitudeMax": 300 },
  "mount": { "speed": 14.0 }, "camera": "cam_player", "equipment": "equip_player" }
```
❌ `controller` is not `ground/flight`; `camera` / `equipment` are dangling (`REF_NOT_FOUND`); values use pixels instead of meters/seconds.

## 19. camera (camera profile)
```json
{ "type": "camera", "id": "cam_player",
  "follow": { "smooth": 8.0 }, "rotation": { "mode": "follow", "smooth": 5.0 },
  "distance": { "min": 0.5, "max": 3.0, "default": 1.0, "step": 0.1 },
  "collision": { "enabled": true, "mask": 1, "padding": 8.0 } }
```
❌ `mode` is not `fixed/follow/free`; `collision.mask` is omitted yet obstacle avoidance is expected.

## 20. navgrid (pathfinding grid)
```json
{ "type": "navgrid", "id": "nav_cave", "cols": 4, "rows": 3, "cellM": 1.0,
  "walkable": [ [1,1,1,1], [1,0,0,1], [1,1,1,1] ],
  "altitude": [ [0,0,0,0], [0,5,5,0], [0,0,0,0] ] }
```
❌ `walkable` length ≠ `cols*rows`; altitude units use pixels (should be meters); expecting steep walls to block but passing too large a `maxClimb` (a query parameter, not in the Def).

## 21. equipment (equipment slot set)
```json
{ "type": "equipment", "id": "equip_player",
  "slots": [ { "id": "weapon", "labelKey": "SLOT_WEAPON" },
             { "id": "artifact", "labelKey": "SLOT_ARTIFACT" } ] }
```
❌ `item.slot` references a slot id not declared here; the engine does not have built-in fixed slots (do not rely on `weapon/armor/artifact` always existing).

## 22. sound_event (audio event)
```json
{ "type": "sound_event", "id": "sfx_sword_hit", "assets": ["audio/hit.ogg"],
  "bus": "sfx", "volume": 0.8, "pitchRange": [0.9, 1.1], "cooldownMs": 60, "maxInstances": 4 }
```
❌ `assets` is empty (playback will silently degrade); `bus` is not `master/sfx/music/ui`.

## 23. fx (special effects)
```json
{ "type": "fx", "id": "fx_slash", "scenePath": "fx/slash.tscn", "attach": "target", "durationMs": 300 }
```
❌ `scenePath` points to a non-existent `.tscn` (`fx.play` will return null); `attach` is not `self/target/point`.

## 24. `item.grantsAbility` / `ability.source` (granted abilities)

```json
{ "type": "item", "id": "item_artifact_caps", "labelKey": "ITEM_ARTIFACT",
  "category": "artifact", "icon": "icon", "grantsAbility": ["ability_starfall"] }
{ "type": "ability", "id": "ability_starfall", "source": "artifact", "target": "enemy",
  "range": 8.0, "cooldownMs": 4000, "effects": ["effect_caps"] }
```
❌ `grantsAbility` points to a missing `ability` (`REF_NOT_FOUND`); `source` not in `player/weapon/artifact`.
