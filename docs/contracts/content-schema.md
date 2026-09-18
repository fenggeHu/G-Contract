# 契约：Content Schema（字段级）

- 状态：生效中
- 更新日期：2026-09-17

> 内容数据的**字段与语义权威**。机器可读 Schema 构建时导出为 `schema/content-<ver>.json`，由校验器加载。

## 0. 通用约定

- 格式：JSON（UTF-8）。命名：`id` 全局唯一，见 world-and-scenes.md §7。
- 通用字段：`type`（必填）、`id`（具体 Def 必填）、`labelKey`（本地化 Key，文本一律用它）、`tags[]`。
- 所有坐标单位：**米**；高度见 §8。

### 0.1 继承（JSON 表示）

```json
{ "type": "city", "name": "base_city", "abstract": true,
  "landing": { "radius": 64, "requires": ["flight"] } }

{ "type": "city", "id": "city_ironhold", "parent": "base_city",
  "labelKey": "CITY_IRONHOLD", "worldPos": { "x": 2410, "y": 1180 } }
```

- `abstract: true` 不入库，仅作父；`parent` 按 `name` 引用。
- 子覆盖父同名字段；数组默认**替换**（`mergeMode: "append"` 改追加）。

### 0.2 Patch（差异覆盖）

```json
{ "type": "patch", "target": "city_ironhold",
  "ops": [ { "op": "set", "path": "landing.radius", "value": 80 },
           { "op": "remove", "path": "legacyFlag" } ] }
```

- 支持 `set` / `remove` / `append`，路径点分；在构建 DB 前应用。

### 0.3 多 Def 文件封装

```json
{ "defs": [ { "type": "city", "id": "city_ironhold" },
            { "type": "biome", "id": "biome_highland" } ] }
```

- 单对象文件与 `{"defs":[...]}` 等价；校验器与构建器都须支持两种。

## 1. `manifest`
见 [content-package.md](content-package.md)。

## 2. `world`（世界图）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `seed` | int | 是 | 确定性种子 |
| `sizeM` | {x,y} | 是 | 默认 8192×8192 |
| `chunkSizeM` | int | 是 | 默认 256 |
| `rasterCellM` | int | 是 | 世界图粗粒度，默认 8 |
| `imageDir` | string | 否 | 烘焙俯瞰贴图子目录（`world/<x>_<y>.jpg`） |
| `zones[]` | {id,rect} | 否 | 分区（宏观），预研 4–9 个 |
| `route[]` | {x,y} | 否 | 飞行航线（有界/环形） |
| `biomeMap` | 引用 | 是 | 生物群系分布 |
| `overrides[]` | 引用 | 否 | 手工覆盖 |
| `defaultLighting` | 引用 | 否 | 光照预设 |

## 3. `biome`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `terrainSet` | 引用 | 是 | 图块集 |
| `lighting` | 引用 | 是 | 光照预设 |
| `ambience` | 引用 | 否 | 环境音事件 |
| `spawnTable` | 引用 | 否 | 刷怪/资源表 |
| `weather[]` | string | 否 | 天气 |

## 4. `terrain`（地形数据格式，**非 Def 类型**）

> 注意：`terrain` 不是可独立声明的 Def（机器 Schema 中无此类型）；`scene.terrain` 为 Tiled 文件路径字符串。

**复用成熟格式**：场景地形以 **Tiled 地图（`.tmj`，JSON）** 为权威，引擎经 Godot TileSet 导入。**不自研格式。**

| Tiled 图层 | 用途 |
|---|---|
| `ground` / `detail` | 视觉图层 |
| `height` | 高度层级（高台/悬崖） |
| `pathing` | 通行位图（0/1，不渲染） |

- 图层用途以 Tiled **自定义属性**（`class`）标注。
- `tileSet` 指向 Godot `TileSet` 资源（atlas + 图块 + 自动拼接）。
- 世界图**不用**此格式，用粗栅格（见 §2 与 world-and-scenes.md §2.1）。

## 5. `scene`（局部场景）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `kind` | enum | 是 | `city/village/cave/interior` |
| `file` | string | 否 | 场景文件（`.tscn`）路径 |
| `sizeM` | {x,y} | 是 | 见比例尺规范 |
| `terrain` | 引用 | 是 | 地形数据（§4） |
| `regions[]` | 引用 | 否 | 区域 |
| `objects[]` | {def,pos,altitude?,rot} | 否 | 初始物件/NPC |
| `lighting` | 引用 | 否 | 光照预设 |
| `music` | 引用 | 否 | 音乐事件 |
| `entryPoints[]` | {id,pos,altitude?} | 否 | 入口（城门/洞口） |

## 6. `region`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `shape` | rect/poly | 是 | 形状 |
| `purpose` | enum | 是 | `trigger/music/spawn/patrol/landing` |
| `onEnter` / `onExit` | 脚本引用 | 否 | 回调 |
| `params` | object | 否 | 用途参数 |

## 7. `poi`（世界图兴趣点）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `kind` | enum | 是 | `city/cave/landmark/portal` |
| `worldPos` | {x,y} | 是 | 米坐标（±4096） |
| `altitude` | number | 否 | 绝对高度（米） |
| `scene` | 引用 | 否 | 承载场景 |
| `icon` | 引用 | 否 | 地图图标 |
| `landing` | object | 否 | `{radius, requires}` |
| `reveal` | enum | 否 | `visible/undiscovered` |

## 8. 高度（altitude）

- 一律**绝对米高**（0 = 海平面），存于 `poi.altitude`、`scene.objects[].altitude`、`entryPoints[].altitude`。
- 飞行时玩家有独立 altitude；着陆时与 POI/入口高度对齐。
- 仅影响飞行规则与表现，**不进入 2D 碰撞求解**（见 physics.md）。
- **例外**：地面通行高程（`navgrid.altitude`，§24）参与寻路通行性/代价判定，但仍不进入碰撞求解。

## 9. `npc`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `species` | 引用 | 是 | 外观/种族 |
| `faction` | 引用 | 否 | 阵营 |
| `stats` | 引用 | 否 | 属性（§13） |
| `abilities[]` | 引用 | 否 | 技能（§14） |
| `behavior` | 引用 | 否 | 行为脚本 |
| `dialog` | 引用 | 否 | 对话树（§18） |
| `spawn` | 引用 | 否 | 生成规则（§12） |

## 10. `item`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `category` | enum | 是 | `weapon/armor/artifact/consumable/quest/resource` |
| `slot` | string | 否 | 装备槽位 id（引用 `equipment.slots[]`，§25） |
| `stats` | object | 否 | 数值（§13） |
| `stack` | int | 否 | 堆叠上限 |
| `icon` | 引用 | 是 | 图标 |
| `dropWeight` | number | 否 | 掉落权重 |

## 11. `lighting_profile`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `preset` | enum | 是 | `lotr_bright/iceland_cold/highland_gloom` |
| `timeOfDay` | enum | 否 | `dawn/noon/golden/night` |
| `ambient` | color | 是 | 环境色 |
| `fogLayers` | int | 否 | 雾层数 |
| `post` | object | 否 | Bloom/LUT/暗角 |

## 12. `spawn`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `targets[]` | 引用 | 是 | 生成的 Def |
| `count` | {min,max} | 是 | 数量 |
| `area` | 引用 | 是 | 区域 |
| `conditions` | object | 否 | 时间/天气/任务 |
| `respawnMs` | int | 否 | 重生 |

## 13. `stats`（属性）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `base` | map<string,number> | 是 | 基础属性（hp/attack/defense…） |
| `modifiers[]` | {stat,op,value,source?,durationMs?} | 否 | `op: flat/percent` |
| `resources[]` | {id,max,regenPerSec?} | 否 | 资源池（如 mana/energy）；`ability.cost.resource` 引用其 `id` |
| `xp` | int | 否 | 击杀奖励经验 |
| `loot[]` | {item,chance} | 否 | 掉落表 |

## 14. `ability`（技能）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cost` | {resource,amount} | 否 | 消耗；`resource` 引用 `stats.resources[].id` |
| `cooldownMs` | int | 否 | 冷却 |
| `cooldownGroup` | string | 否 | 冷却组 id；同组技能共享冷却计时（通用原语） |
| `castMs` | int | 否 | 施法时间 |
| `range` | number | 否 | 射程（米） |
| `target` | enum | 是 | `self/enemy/ally/point/area` |
| `effects[]` | 引用 | 是 | 效果（§15） |
| `script` | 脚本引用 | 否 | 特殊逻辑 |

## 15. `effect`（效果）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `kind` | enum | 是 | `damage/heal/buff/debuff/summon` |
| `amount` | number/公式 | 否 | 数值 |
| `statModifiers[]` | 引用 | 否 | 属性修正 |
| `durationMs` | int | 否 | 持续 |
| `stacks` / `maxStacks` | int | 否 | 初始层数 / 层数上限 |
| `stackMode` | enum | 否 | `refresh`（重apply刷新持续时间） / `stack`（叠加层数） |
| `tags[]` | string | 否 | 分类（内容层据此表达抗性/免疫等规则） |

## 16. `interaction`

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `promptKey` | Key | 是 | 提示文案 |
| `range` | number | 否 | 交互距离 |
| `conditions` | object | 否 | 前置条件 |
| `actions[]` | 脚本/效果/quest/dialog | 是 | 触发动作 |

## 17. `quest`（任务）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `objectives[]` | {kind,target,count} | 是 | `kill/collect/reach/talk` |
| `states[]` | enum | 是 | 状态机（inactive/active/done） |
| `rewards` | {items[],xp} | 否 | 奖励 |
| `prerequisites[]` | 引用 | 否 | 前置 |
| `onComplete` | 脚本引用 | 否 | 完成回调 |

## 18. `dialog`（对话）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `nodes[]` | 节点 | 是 | 对话节点 |
| 节点字段 | `id` / `speaker` / `textKey` / `conditions` / `actions[]` / `choices[]` | | `choices: {textKey,next,conditions}` |

## 19. `sound_event`（音频事件）

> 能力 `audio.play` / `audio.music` / `audio.mixer` 的事件源；`biome.ambience` / `scene.music` / `region.music` 均引用本类型。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `assets[]` | 资源路径 | 是 | 候选音频（随机取一） |
| `bus` | enum | 否 | `master/sfx/music/ui`，默认 `sfx` |
| `volume` | number | 否 | 线性音量（默认 1.0） |
| `pitchRange` | [min,max] | 否 | 随机音高区间 |
| `loop` | bool | 否 | 循环（音乐） |
| `cooldownMs` | int | 否 | 同事件最小间隔 |
| `maxInstances` | int | 否 | 同事件最大并发 |
| `priority` | int | 否 | 抢占优先级（高优先） |
| `attenuation` | number | 否 | 距离衰减（预留） |

## 20. 版本与兼容

| 对象 | 规则 |
|---|---|
| Schema 版本 | `MAJOR.MINOR`，独立于引擎 |
| 引擎支持 | 声明支持的 Schema 主版本集合（≥ 最近两个）；**机器可读**为 `capabilities.json.schemaSupported` 与 `release.json.schemaSupported`（[content-tooling.md](content-tooling.md) §5） |
| 字段 | 只增不删；删除须 MAJOR + 弃用期 |
| 枚举 | 新增为 MINOR；移除须 MAJOR |

## 21. 工具

机器可读 Schema、校验器 CLI、能力矩阵见 [content-tooling.md](content-tooling.md)。

## 22. `character`（角色控制器档案）

> 引擎能力 `movement.character` 的 Def 参数。单位：**米、秒**；运行时按 PPU 换算像素。具体玩法（是否启用冲刺/攀爬/坐骑）由内容决定。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `controller` | enum | 否 | `ground/flight`，默认 `ground` |
| `collision` | object | 否 | `{shape: circle/aabb, radius, size{x,y}, layer, mask}` |
| `moveSpeed` | number | 否 | 步行速度（默认 6） |
| `accel` / `friction` | number | 否 | 加速 / 摩擦 |
| `gravity` / `maxFallSpeed` | number | 否 | 重力 / 终端下落速度 |
| `jump` | object | 否 | `{height, maxCount, cooldownMs}` |
| `sprint` | object | 否 | `{speed, staminaPerSec}` |
| `climb` | object | 否 | `{speed, mask}` |
| `fly` | object | 否 | `{speed, accel, altitudeMin, altitudeMax}`（altitude 独立通道） |
| `mount` | object | 否 | `{speed, accel, turnSpeed}` |
| `camera` | 引用 | 否 | 相机档案（§23） |
| `equipment` | 引用 | 否 | 装备槽位集合（§25） |

## 23. `camera`（相机档案）

> 引擎能力 `camera.control` 的 Def 参数；表现层，不参与逻辑权威。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `follow` | object | 否 | `{smooth, deadzone}` |
| `rotation` | object | 否 | `{mode: fixed/follow/free, smooth}` |
| `distance` | object | 否 | `{min, max, default, step}`（缩放档位） |
| `collision` | object | 否 | `{enabled, mask, padding}` |
| `shake` | object | 否 | `{decay}` |

## 24. `navgrid`（寻路网格）

> 引擎能力 `path.find` 的数据源：可通行位图 + 地面高程。用于 `PathFinder`（高度感知网格 A*），单位：`cellM` 米/格、`altitude` 米。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `cols` / `rows` | int | 是 | 网格列/行数 |
| `cellM` | number | 否 | 每格边长（米），默认 1 |
| `walkable` | array | 是 | 可通行位图（0/1）；一维 `cols*rows` 或二维 `rows × cols` |
| `altitude` | array | 否 | 每格地面高程（米），同 `walkable` 形状；缺省视为全 0 |

> 陡壁判定与加价规则（`maxClimb` / `stepPenalty` / `eightWay`）由调用方查询时传入，不在 Def 内。

## 25. `equipment`（装备槽位集合）

> 通用原语：槽位集合由**内容声明**；`item.slot` 与存档 `player.equipped` 的键均引用此集合。引擎不内置固定槽位（不写死 `weapon/armor/artifact`）。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `slots[]` | {id,labelKey?} | 是 | 槽位列表；`id` 唯一，`labelKey` 供 UI |

## 26. `fx`（特效）

> 能力 `fx.play` 的表现源（纯表现，不影响逻辑权威）。

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `scenePath` | string | 否 | 表现场景（`.tscn`）路径 |
| `attach` | enum | 否 | `self/target/point`（默认 `point`） |
| `durationMs` | int | 否 | 自动销毁时长（`loop` 时忽略） |
| `loop` | bool | 否 | 循环表现 |

## 27. `conditions`（条件表达式）

> 能力 `conditions.core` 的表达式格式；用于 `interaction.conditions` / `region.params` / `spawn.conditions` / dialog 节点等的 `conditions` 字段。
> 求值：`Conditions.eval(spec, ctx)`；**空/缺省 = 真**。`ctx` 可覆盖数据源（`flags/quests/items/stats/level/kv`）以便测试或服务端权威。

| 形式 | 说明 |
|---|---|
| `{ "all": [expr…] }` | 全部为真（数组形式等价 all） |
| `{ "any": [expr…] }` | 任一为真 |
| `{ "not": expr }` | 取反 |
| `{ "flag": "key", "eq": bool? }` | 特性开关（`config.flag`） |
| `{ "quest": id, "state": "active/ready/done/inactive" }` | 任务状态（默认 `active`） |
| `{ "item": id, "count": n? }` | 背包持有（默认 1） |
| `{ "stat": name, "op": ">=/<=/>/</==/!=", "value": n }` | 派生属性（默认 `>=`） |
| `{ "level": n, "op": ">=…" }` | 等级 |
| `{ "kv": key, "value": any }` | 存档 KV 相等 |