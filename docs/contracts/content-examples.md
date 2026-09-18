# 内容示例（最小可用 / 常见错误）

- 状态：生效中
- 更新日期：2026-09-17

> 面向 AI / 内容作者：每类 Def 的**最小可用示例**与**常见错误**。字段语义以 [content-schema.md](content-schema.md) 为唯一权威；机器校验用 `content validate` / `content lint`。
> 每个 Def 文件形如 `{"defs": [ … ]}`（见 [content-schema.md](content-schema.md) §0.3）。

## 0. 通用约定（先记）

- `type` 必填且为注册类型；`id` 命名 `^[a-z][a-z0-9_]*$`（约定 `类型_小写下划线`）。
- **用户可见类型必须 `labelKey`**：`biome` / `lighting_profile` / `scene` / `poi` / `item` / `npc`；声明 `entry.i18n` 的包还须有对应翻译。
- 引用（`lighting` / `scene` / `item` / `effect` / `dialog` / `stats` …）必须指向存在的 Def，否则 `REF_NOT_FOUND`。
- **常见错误**：`"id": "IronHold"`（大写）、缺 `type`、`id` 重复、缺 `labelKey`、悬空引用。

## 1. manifest
见 [content-package.md](content-package.md) §2。
```json
{ "id": "world.pack.aurelia", "version": "1.0.0", "schema": "1.0",
  "requires": { "engine": ">=0.1.0 <2.0", "packs": ["base.core@1.x"] },
  "capabilities": ["scene.switch@>=1.0"], "loadAfter": ["base.core"],
  "entry": { "defs": "defs/", "scenes": "scenes/", "i18n": "i18n/" } }
```
❌ `schema` 主版本不在引擎 `schemaSupported`；声明未知/暂缓能力（`CAP_MISSING`）或引擎区间不满足（`DEP_MISSING`）。

## 2. world
```json
{ "type": "world", "id": "world_aurelia", "seed": 1337,
  "sizeM": { "x": 2048, "y": 2048 }, "chunkSizeM": 256, "rasterCellM": 8,
  "imageDir": "world", "biomeMap": "biome_highland" }
```
❌ 缺 `biomeMap`；`imageDir` 与实际烘焙目录不符。

## 3. biome
```json
{ "type": "biome", "id": "biome_highland", "labelKey": "BIOME_HIGHLAND",
  "terrainSet": "tileset_highland", "lighting": "lighting_lotr", "weather": ["clear", "fog"] }
```
❌ 缺 `labelKey`；`lighting` 悬空。

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
❌ `file` 用 `.scene`（应为 `.tscn`）；`terrain` 文件不存在（`ASSET_MISSING`）；资产未登记 `licenses.json`（`LICENSE_MISSING`，见 content-package §6）。

## 6. region
```json
{ "type": "region", "id": "ironhold_region_landing",
  "shape": { "x": 0, "y": 0, "w": 64, "h": 64 }, "purpose": "landing", "params": {} }
```
❌ `purpose` 不在枚举 `trigger / music / spawn / patrol / landing`。

## 7. poi
```json
{ "type": "poi", "id": "poi_ironhold", "labelKey": "POI_IRONHOLD", "kind": "city",
  "worldPos": { "x": 1200, "y": -340 }, "altitude": 120, "scene": "scene_ironhold_gate",
  "landing": { "radius": 64, "requires": ["flight"] }, "reveal": "visible" }
```
❌ `kind` 不在枚举 `city / cave / landmark / portal`；`scene` 悬空；`worldPos` 超出 `±sizeM/2`。

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
❌ `category` 非约定值（`weapon / armor / artifact / consumable / quest / resource`）；缺 `labelKey`；`slot` 未在 `equipment.slots[]` 声明。

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
❌ `loot[].item` 悬空；`base` 缺关键属性；`ability.cost.resource` 引用了未在 `resources[]` 声明的 `id`。

## 13. ability
```json
{ "type": "ability", "id": "ability_strike", "labelKey": "ABILITY_STRIKE",
  "cost": { "resource": "mana", "amount": 5 }, "cooldownMs": 600, "cooldownGroup": "gcd",
  "range": 60, "target": "enemy", "effects": ["effect_strike_dmg"] }
```
❌ `effects[]` 引用不存在的 effect；`cost.resource` 未在 `stats.resources[]` 声明。

## 14. effect
```json
{ "type": "effect", "id": "effect_strike_dmg", "kind": "damage", "amount": 1.0 }
```
❌ `kind` 非约定值（`damage` 已实现；`heal / buff / debuff / summon` 设计保留，见 content-schema §15）。

## 15. quest
```json
{ "type": "quest", "id": "quest_grunt", "labelKey": "QUEST_GRUNT",
  "objectives": [ { "kind": "kill", "target": "grunt", "count": 1 } ],
  "states": ["inactive", "active", "ready", "done"],
  "rewards": { "items": ["item_talisman_fire"], "xp": 50 } }
```
❌ 目标类型引擎不认识；奖励 item 悬空。

## 16. dialog
```json
{ "type": "dialog", "id": "dialog_guard", "labelKey": "DIALOG_GUARD",
  "nodes": [
    { "id": "start", "speaker": "guard", "textKey": "D_GUARD_HELLO",
      "choices": [ { "textKey": "D_ACCEPT", "next": "end",
        "actions": [ { "op": "start_quest", "quest": "quest_grunt" } ] } ] },
    { "id": "end", "speaker": "guard", "textKey": "D_GUARD_BYE", "choices": [] } ] }
```
❌ `next` 指向不存在的节点；`op` 非实现动作（现支持 `start_quest` / `turn_in_quest`）。

## 17. patch
```json
{ "type": "patch", "target": "poi_ironhold",
  "ops": [ { "op": "set", "path": "landing.radius", "value": 80 } ] }
```
❌ `target` 不存在（`REF_NOT_FOUND`）；`ops` 缺 `op/path`。路径为**点分**（`landing.radius`），非 JSON Pointer。

## 18. character（角色控制器档案）
```json
{ "type": "character", "id": "char_player", "controller": "ground",
  "collision": { "shape": "circle", "radius": 0.5, "layer": 1, "mask": 3 },
  "moveSpeed": 6.0, "accel": 40.0, "friction": 60.0, "gravity": 22.0, "maxFallSpeed": 40.0,
  "jump": { "height": 2.4, "maxCount": 1 }, "sprint": { "speed": 9.0 },
  "climb": { "speed": 3.0 }, "fly": { "speed": 40.0, "altitudeMin": 0, "altitudeMax": 300 },
  "mount": { "speed": 14.0 }, "camera": "cam_player", "equipment": "equip_player" }
```
❌ `controller` 非 `ground/flight`；`camera` / `equipment` 悬空（`REF_NOT_FOUND`）；数值用像素而非米/秒。

## 19. camera（相机档案）
```json
{ "type": "camera", "id": "cam_player",
  "follow": { "smooth": 8.0 }, "rotation": { "mode": "follow", "smooth": 5.0 },
  "distance": { "min": 0.5, "max": 3.0, "default": 1.0, "step": 0.1 },
  "collision": { "enabled": true, "mask": 1, "padding": 8.0 } }
```
❌ `mode` 非 `fixed/follow/free`；`collision.mask` 缺省却期望避障。

## 20. navgrid（寻路网格）
```json
{ "type": "navgrid", "id": "nav_cave", "cols": 4, "rows": 3, "cellM": 1.0,
  "walkable": [ [1,1,1,1], [1,0,0,1], [1,1,1,1] ],
  "altitude": [ [0,0,0,0], [0,5,5,0], [0,0,0,0] ] }
```
❌ `walkable` 长度 ≠ `cols*rows`；高程单位用像素（应为米）；期望陡壁阻挡却把 `maxClimb` 传得过大（查询参数，不在 Def）。

## 21. equipment（装备槽位集合）
```json
{ "type": "equipment", "id": "equip_player",
  "slots": [ { "id": "weapon", "labelKey": "SLOT_WEAPON" },
             { "id": "artifact", "labelKey": "SLOT_ARTIFACT" } ] }
```
❌ `item.slot` 引用了此处未声明的槽位 id；引擎不内置固定槽位（不要依赖 `weapon/armor/artifact` 一定存在）。

## 22. sound_event（音频事件）
```json
{ "type": "sound_event", "id": "sfx_sword_hit", "assets": ["audio/hit.ogg"],
  "bus": "sfx", "volume": 0.8, "pitchRange": [0.9, 1.1], "cooldownMs": 60, "maxInstances": 4 }
```
❌ `assets` 为空（播放将静默降级）；`bus` 非 `master/sfx/music/ui`。

## 23. fx（特效）
```json
{ "type": "fx", "id": "fx_slash", "scenePath": "fx/slash.tscn", "attach": "target", "durationMs": 300 }
```
❌ `scenePath` 指向不存在的 `.tscn`（`fx.play` 将返回 null）；`attach` 非 `self/target/point`。
