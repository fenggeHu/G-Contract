# 契约：内容工具链（Schema / 校验器）

- 状态：生效中
- 更新日期：2026-09-17

> 内容契约的**工具实现规范**：机器可读 Schema、校验器 I/O、能力矩阵。字段语义见 [content-schema.md](content-schema.md)。

## 1. 机器可读 Schema

| 项 | 定义 |
|---|---|
| 生成物 | `shared/contract/content-<ver>.json`（JSON Schema 2020-12） |
| 入口点 | `shared/contract/{def,manifest,defs-file}.json`：按文件类型指向 `$defs/def` / `$defs/manifest`，供 IDE / 工具校验 |
| 唯一来源 | `App/engine/sdk/schema/`（**禁止手改生成物**）；协议/模型 IDL 源在 `App/engine/sdk/{protocol,model}/` |
| 结构 | 每个 Def 类型一个 `$defs` 条目；公共字段（`type/id/labelKey/tags`）；引用用 `$ref` |
| 版本 | `MAJOR.MINOR`，随 `content-schema.md` |

> **IDE 集成**：将 `**/defs/*.json` 映射到 `defs-file.json`、`**/manifest.json` 映射到 `manifest.json`（如 VS Code `json.schemas`），即可得字段级补全与校验；入口点相对引用同目录的 `content-<ver>.json`。示例见 [content-examples.md](content-examples.md)。

## 2. 校验器 CLI `content`

| 命令 | 作用 |
|---|---|
| `content validate <pack>` | Schema + 引用 + 依赖 + 能力 |
| `content lint <pack>` | 命名 + 本地化 Key + 授权（`licenses.json`） |
| `content build <pack>` | 校验门（validate+lint）+ 生成 `_meta.json` +（`--godot`）导出 PCK |
| `content preview <scene>` | 解析 scene Def → `.tscn` 路径；`--godot` 时启动引擎预览 |
| `content diff <a> <b>` | 内容差异（`type:id` 增删改；信息性，退出码 0）；参数可为目录或快照文件 |
| `content snapshot <root>` | 导出 Def 快照（`--out`，默认 `dist/defs.snapshot.json`），供回归比对 |
| `content i18n-template <pack>` | 生成/刷新翻译 CSV 模板（保留已有译文，补全 key） |
| `content perf` | 跑世界基准或 `--scene <id>`；可选 p95/max 预算门（`--budget-p95` / `--budget-max`，0=不设门） |
| `content playtest <scene>` | 无头自动运行内容场景并**断言无错误**（H1 量化门） |
| `content release` | 产出 `dist/release.json`（版本矩阵 + 制品清单） |
| `content verify-release` | 校验 `release.json` 与契约一致（消费侧 pin 前置） |
| `content bake-world` | 烘焙世界图俯瞰贴图 → `<pack>/world/<x>_<y>.jpg`（1024px/chunk） |

**I/O**
- 参数：包路径 / 场景。
- `--json`：单个结果对象 `{ ok, errors[], warnings[] }`。
- 退出码：`0` 成功 · `2` usage · `3` schema · `4` dependency · `5` lint · `6` internal。
- 文本模式对每个错误/告警打印 **`hint:` 修复提示**（按错误码）；错误码→提示见 CLI 实现与本文 §错误码。

**`content build` 用法**
```
content build <content-root> [--godot <bin>] [--project <dir>] [--out <file.pck>] [--content-version <v>]
```
1. 先过 `validate` + `lint` 门（任一失败即停）。
2. 在内容根写 `_meta.json`：`{ "content": <版本>, "schema_major": "<主版本>" }`（引擎启动据此拒绝不兼容内容）。
3. 提供 `--godot` 时执行 `godot --headless --path <project> --export-pack content_pck <out>`；`--project` 默认取内容根父目录，`--out` 默认 `<project>/../content-<版本>.pck`。未提供 `--godot` 只生成 `_meta.json`（告警提示）。

**`content diff` 用法**：`content diff <A> <B>` 比较两处 Def 集合（规范化 JSON），输出 `+ / - / ~` 与计数；参数可为**内容根目录**或 **`content snapshot` 快照文件**。`--json` 输出 `{added,removed,changed}`。用于内容回归与评审（信息性，不失败）。

**`content snapshot` 用法**：`content snapshot <root> [--out <file>]` 导出 `type:id → 原始 Def` 快照；配合 `content diff <快照> <内容>` 做回归（golden 基线）。

**`content i18n-template` 用法**：`content i18n-template <pack> [--out <file>]` 扫描包内 `labelKey`/`textKey`，读取既有翻译 CSV（保留译文），补全缺失 key 并输出模板；列取已有 locale（缺省 `en,zh_CN`）。Manifest 未声明 `entry.i18n` 时告警（声明后才做强校验）。

**错误码**

| 码 | 含义 |
|---|---|
| `SCHEMA_INVALID` | 字段/类型/枚举不合法 |
| `REF_NOT_FOUND` | 悬空引用 |
| `DEP_MISSING` | 依赖缺失 |
| `CAP_MISSING` | 能力缺失/版本不满足 |
| `CYCLE_DETECTED` | 循环依赖 |
| `NAME_DUP` | id 重复 |
| `LOC_KEY_MISSING` | 本地化 Key 缺失（字段缺失，或声明 `entry.i18n` 后某 locale 列无翻译） |
| `ASSET_MISSING` | 资源缺失 |
| `LICENSE_MISSING` | 资产未登记授权 / `license` 字段缺失（`licenses.json`，见 [content-package.md](content-package.md) §6） |

## 3. 能力矩阵 `capabilities.json`

- 路径：`shared/contract/capabilities.json`（引擎构建产出）。

```json
{ "engine": "1.2.0", "schema": "1.2", "schemaSupported": ["1"],
  "capabilities": [ { "name": "world.stream", "version": "1.0", "status": "enabled" } ] }
```

- `status`：`enabled` | `deferred`。校验器据此校验 Manifest 的 `capabilities`。

## 4. CI

- **内容流水线**：`validate` + `lint`（每次内容提交）。
- **引擎流水线**：编译 + 单元 + 导出 `capabilities.json`。
- **契约测试**：引擎版本矩阵上加载冒烟（见 平台内部文档）。
- **内容仓 CI 模板**：随 devkit-lite 发布（`ci/content-ci.yml`）；世界组复制到 `.github/workflows/` 即得「pin 校验 + validate + lint」流水线。
- 兼容判定：`content validate` 按 `schemaSupported` 接受**主版本受支持**的包（非当前版本仅告警）；校验 `requires.engine` 区间与能力版本下限，不满足即拒绝。

## 5. 发布制品与版本矩阵 `release.json`

引擎发布时产出 `release.json`（与 `content` CLI、`devkit-lite` 同批），供内容/服务端消费方 **pin 版本并校验兼容**。生成：`make release`（内部调 `content release`）。

```json
{
  "engine": "0.1.0",
  "godot": "4.7.2",
  "schema": "1.2",
  "schemaSupported": ["1"],
  "capabilities": { "total": 36, "enabled": 36, "deferred": 0, "digest": "sha256:…" },
  "artifacts": {
    "contentCli": "content",
    "devkitLite": "engine-devkit-lite-0.1.0.zip",
    "contract": "shared/contract",
    "capabilities": "App/engine/sdk/capabilities.json",
    "contentSchema": "App/engine/sdk/schema/content-1.2.json"
  }
}
```

| 字段 | 说明 |
|---|---|
| `engine` | 引擎 SemVer（源：`App/engine/sdk/capabilities.json`） |
| `godot` | 配套 Godot 版本（源：`deps.json`，可空） |
| `schema` | 当前 Content Schema 版本 |
| `schemaSupported` | **兼容窗口**：引擎支持的 Schema 主版本集合（源：capabilities.json） |
| `capabilities.total/enabled/deferred` | 能力计数 |
| `capabilities.digest` | 能力集合确定性指纹（`sha256`，按 name 排序的 `name@version:status`） |
| `artifacts.*` | 同批制品名/路径（相对发布包） |

**消费方规则**
- 内容/服务端 `deps.json` 记录引擎版本并 pin 上述制品；构建前校验：
  1. 内容包 `schema` 主版本 ∈ `schemaSupported`，否则**拒绝加载**；
  2. 契约 `capabilities.json` 的 `digest` 与 `release.json` 一致，防漂移。
- 校验命令：`content verify-release --release <release.json> --caps <capabilities.json>`（退出码同 §2）。
- `release.json` **不入库**（`dist/` 为构建产物），随发布制品分发。
- **内容组交付包**（`make release` 产出于 `dist/`）：`content`（CLI）、`contract/`（契约：`capabilities.json` + `content-<ver>.json`）、`release.json`（版本矩阵）、`engine-devkit-lite-<ver>.zip`（打包设置 + CI 模板）。
- 内容组本地布局：`devkit/bin/content` + `devkit/sdk/{capabilities.json,schema/content-<ver>.json}` + `devkit/release.json`；`content validate/lint/... --sdk devkit/sdk`。

## 6. 代码生成

> 目的：由**唯一源**生成**类型化产物**，编译期暴露字段/枚举错误，并统一类型/枚举清单。

- **唯一源**：`App/engine/sdk/schema/content-<ver>.json`（+ `capabilities.json`）。
- **命令**：`content gen [--sdk <dir>] [--out <dir>] [--check]`
  - 产物（确定性、排序稳定）落 `App/engine/sdk/generated/`：
    `types.json`（中性：engine / schema / defTypes / fields / enums）
    `gdscript/g3_defs.gd`、`go/g3_defs.gen.go`、`lua/g3_defs_gen.lua`
  - 枚举来源：Schema `properties.<field>.enum`（顶层字段）。
- **漂移门**：`content gen --check`（生成物 ≠ 源即失败）→ `make gen-check` + `arch_check` **R8**。
- **边界**：内容组只**读**生成物；是否采用、何时采用由内容组自定（平台不改内容层）。