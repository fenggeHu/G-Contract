# G3 契约仓（G-Contract）

面向**内容制作方**的公开契约与对接仓。**只读镜像**：由平台从 `G_Shared` 发布，请勿在本仓直接修改契约（改动一律走 Issue → 平台契约 PR）。

## 内容
- `shared/contract/`：`capabilities.json`、`content-<ver>.json`（内容 Schema）、入口点 `def/manifest/defs-file.json`
- `shared/protocol/`、`shared/model/`：协议 IDL 与共享模型
- `docs/contracts/`：字段级契约与工具链（Schema/示例/包/CLI/SDK/事件/钩子/迁移/协议/存档）

## 使用（不依赖平台源码）
```bash
# 1) 取契约（pin 版本见 Release）
# 2) 校验：能力/schema 兼容 + 引用/依赖/命名/本地化/授权
content verify-release --release release.json --caps shared/contract/capabilities.json
content validate --sdk <dir> --content <你的内容根>   # <dir> 含 capabilities.json 与 schema/content-<ver>.json
content lint --sdk <dir> --content <你的内容根>
```
Schema 版本 `MAJOR.MINOR` 独立于引擎；引擎声明支持的**主版本**集合见 `capabilities.json.schemaSupported`。

## 产物命名（发布 Release）
`content`（CLI）、`contract-<schema>.zip`、`release.json`、`engine-devkit-lite-<ver>.zip`。

## 反馈与能力请求
- 缺陷 / 契约歧义 / 新能力：使用本仓 Issue 模板（`bug` / `capability-request` / `content-handoff`）。
- 变更流程：Issue → 平台评估 → 契约 PR → 发版公告（CHANGELOG）→ 消费方自行 pin。

## 许可与安全
- 契约与示例文本：**Apache-2.0**（见 `LICENSE`）。
- 安全问题请按 `SECURITY.md` 私下报告。
