# 内容包契约（Manifest 与 Def Schema）

- 状态：生效中
- 更新日期：2026-09-15

> 本文件是引擎组与世界组之间的**接口协议**。契约变更须遵循 overview.md 的兼容策略。

## 1. 内容包目录结构（`G_World/`）

```
G_World/
  base/core/                     # 基础规则与公共 Def
  packs/continent_aurelia/       # 大陆：地形/城市坐标/生物群系
    manifest.json
    defs/{biomes,cities,caves}/*.json
    scenes/ironhold_gate.scene
    scripts/quest_gate.gd
  packs/dlc_northlands/          # 后续新增内容包，不改引擎
```

`G_Engine/` 与 `G_World/` **仓库与 CI 边界分离**；内容以 PCK 交付（`_meta.json` 声明 `content` 版本与 `schema_major`），各自版本、各自权限。内容变更**绝不触发**引擎重新编译。详见 ../architecture/repository-layout.md。

`_meta.json`（由 `content build` 生成于内容根，随 PCK 打包）：
```json
{ "content": "0.9.0", "schema_major": "1" }
```
引擎启动挂载 PCK 后校验 `schema_major` ∈ 引擎支持集合，否则**拒绝启动**（见 [content-tooling.md](content-tooling.md) §5）。

## 2. Manifest 规范

```json
{
  "id": "world.continent.aurelia",
  "version": "1.4.0",
  "schema": "1.0",
  "requires": { "engine": ">=1.2 <2.0", "packs": ["base.core@1.x"] },
  "capabilities": ["scene.switch@>=1.0", "world.stream@>=1.0", "script.gdscript@>=1.0", "render.light2d@>=1.0"],
  "loadAfter": ["base.core"],
  "entry": { "worlds": "worlds/", "defs": "defs/", "scenes": "scenes/", "scripts": "scripts/", "i18n": "i18n/" }
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `id` | 是 | 全局唯一，小写点分命名 |
| `version` | 是 | 本包 SemVer |
| `schema` | 是 | 使用的 Content Schema 版本 |
| `requires.engine` | 是 | 引擎版本区间 |
| `requires.packs` | 否 | 依赖的其他内容包 |
| `capabilities` | 是 | 所需引擎能力，支持 `name@>=X.Y`；缺失/版本不满足则拒绝加载 |
| `loadAfter` / `loadBefore` | 否 | 加载顺序 |
| `entry` | 是 | 各内容类型目录；`entry.i18n` 指向本包翻译 CSV 目录 |

引擎启动时校验：能力缺失 / 版本不满足 / 依赖缺失 → **拒绝加载并给出可读修复建议**。

**i18n（可选）**：声明 `entry.i18n` 后，该包进入**强校验**——Def 的 `labelKey` 与对话 `textKey` 必须在每个 locale 列有非空翻译（`content lint` 报 `LOC_KEY_MISSING`）。CSV 约定与运行时见 ../architecture/localization.md。

## 3. Def 规范（数据驱动，无代码）

```json
{
  "type": "city",
  "id": "ironhold",
  "label": "铁炉堡",
  "worldPos": { "x": 2410, "y": 1180 },
  "gateScene": "scenes/ironhold_gate.scene",
  "landing": { "radius": 64, "requires": ["flight"] },
  "biome": "highland",
  "lighting": { "profile": "lotr_bright", "timeOfDay": "golden" }
}
```

规则：
- 所有内容用 `Def` 表达，`type` 对应引擎注册的 Def 类型。
- 继承与 Patch 的 JSON 写法见 [content-schema.md](content-schema.md) §0。
- 玩法逻辑变化放 `scripts/`，不写进数据。
- 字段级定义以 [content-schema.md](content-schema.md) 为准。

## 4. 版本与兼容

| 对象 | 规则 | 承诺 |
|---|---|---|
| Content Schema | 独立 `MAJOR.MINOR` | 引擎声明支持的 Schema 主版本集合（≥ 最近两个）；机器可读为 `capabilities.json.schemaSupported` |
| 内容包 | 各包独立 SemVer | 声明引擎/依赖区间 |
| Def 字段 | 只增不删（除非 MAJOR） | 删除字段须提前一个 MAJOR 弃用 |

**拒绝策略（引擎/校验器强制）**
1. 包 `schema` 主版本 ∉ `schemaSupported` → **拒绝加载**（`content validate` 报 `SCHEMA_INVALID`）。
2. `requires.engine` 区间不满足当前引擎 → **拒绝**（`DEP_MISSING`）。
3. 声明能力缺失 / 暂缓 / 版本低于下限 → **拒绝**（`CAP_MISSING`）。

**弃用与公告（A1）**
- 字段 / 能力 / 协议消息的弃用：先在 CHANGELOG.md 公告，保留至少一个 **MINOR** 周期（仅告警），移除须升 **MAJOR**。
- 契约冻结：进入里程碑前冻结 `capabilities.json` 与 Content Schema 主版本；冻结期内只增 MINOR。

## 5. 工具

Schema 导出、校验器与能力矩阵见 [content-tooling.md](content-tooling.md)。

## 6. 资产授权登记（`licenses.json`）

每个包可选提供 `<pack>/licenses.json`：资产**文件路径** → 授权条目。

```json
{
  "scenes/dummy.tscn": { "license": "CC0-1.0", "source": "in-house", "author": "G3" }
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `license` | 是 | SPDX 标识或授权名称 |
| `source` | 否 | 来源（自制 / 采购 / 外链） |
| `author` | 否 | 作者 / 署名 |

- `content lint` 校验：`license` 非空；**场景引用的资产文件**（`scene.file` / `scene.terrain`）必须在登记表内，否则报 `LICENSE_MISSING`。
- 未提供 `licenses.json` 且包内有场景资产 → **告警**（渐进接入，不阻断）。
