# 契约：数据迁移（Migrations）

- 状态：生效中
- 更新日期：2026-09-17

> 适用**平台自有数据**：服务端业务表（Postgres，`G_Server`）与客户端存档 SQLite 多表（见 [save-schema.md](save-schema.md) §7）。
> **不接管 Nakama 自身 schema**（由上游 `nakama migrate up` 负责）。
> 归属：平台实现；本契约供消费方按目录/台账约定组织迁移文件。

## 1. 目录约定

```text
migrations/
├── base/       基线（幂等建表，可 squash）
├── released/   已发布增量（按文件名排序执行）
├── custom/     本地 / 自定义
├── pending/    待审（未合入前不得进入 released）
└── archive/    已 squash 的增量归档（不再执行）
```

## 2. 台账

默认 JSON 文件 `<dir>/.ledger.json`（`name → { sha256, state, applied_at, ms }`）；字段语义：

| 字段 | 说明 |
|---|---|
| `name` | 迁移文件名（键；唯一） |
| `sha256` | 文件哈希（**改名不改哈希** → 识别为重命名） |
| `state` | `base` / `released` / `custom` / `pending` |
| `applied_at` | 应用时间 |
| `ms` | 耗时 |

## 3. 执行语义

- 按**文件名排序**执行；已应用且哈希一致 → 跳过。
- **同名哈希变化** → 按需重放（`--rehash`）；**同哈希改名** → 记为新名（不重复执行）。
- **幂等**：重复执行 0 变更；中断后可续跑。
- `pending` 默认**不自动应用**（需 `--pending` 显式）。
- 记录采用 `UPSERT`（REPLACE 语义）。

## 4. CLI

```text
content migrate [--dir <migrations>] [--ledger <file>] [--pending] [--rehash] [--dry-run] [--apply] [--exec <tpl>] [--json]
```

- **默认只规划**（不写台账）；`--apply` 实际执行并落台账。
- 执行器模板 `--exec`（默认 `psql -v ON_ERROR_STOP=1 -f {file}`）。
- **基线 squash**：`--squash <name>` 把 `released/*.sql` 合并为 `base/<name>.sql`、原文件移入 `archive/`，并把台账中原 released 条目替换为 base 条目（既有库**幂等，不重复执行**）；仅 `--apply` 生效。
- 退出码沿用 `content` CLI（见 [content-tooling.md](content-tooling.md) §2）。
- `--json`：`{ ok, applied[], skipped[], renamed[], errors[] }`。

## 5. 与存档的关系

- 存档版本迁移链（[save-schema.md](save-schema.md) §6）与 SQL 迁移遵循同一原则：**单向往上、幂等**。