# Contract: Data Migrations (Migrations)

- Status: Active
- Updated: 2026-09-17

> Applies to **platform-owned data**: server business tables (Postgres, `G_Server`) and client save SQLite multi-tables (see [save-schema.md](save-schema.md) §7).
> **Does not take over Nakama's own schema** (handled upstream by `nakama migrate up`).
> Ownership: platform implementation; this contract lets consumers organize migration files per the directory/ledger conventions.

## 1. Directory conventions

```text
migrations/
├── base/       baseline (idempotent table creation, squashable)
├── released/   published increments (executed in filename order)
├── custom/     local / custom
├── pending/    pending review (must not enter released before merging)
└── archive/    archive of squashed increments (no longer executed)
```

## 2. Ledger

The default JSON file is `<dir>/.ledger.json` (`name → { sha256, state, applied_at, ms }`); field semantics:

| Field | Description |
|---|---|
| `name` | Migration file name (key; unique) |
| `sha256` | File hash (**renaming does not change the hash** → recognized as a rename) |
| `state` | `base` / `released` / `custom` / `pending` |
| `applied_at` | Applied time |
| `ms` | Elapsed time |

## 3. Execution semantics

- Execute in **filename order**; if already applied with a matching hash → skip.
- **Same name, changed hash** → replay as needed (`--rehash`); **same hash, renamed** → recorded under the new name (not executed again).
- **Idempotent**: repeated execution yields 0 changes; can resume after interruption.
- `pending` is by default **not applied automatically** (requires explicit `--pending`).
- Records use `UPSERT` (REPLACE semantics).

## 4. CLI

```text
content migrate [--dir <migrations>] [--ledger <file>] [--pending] [--rehash] [--dry-run] [--apply] [--exec <tpl>] [--json]
```

- **By default it only plans** (does not write the ledger); `--apply` actually executes and writes the ledger.
- Executor template `--exec` (default `psql -v ON_ERROR_STOP=1 -f {file}`).
- **Baseline squash**: `--squash <name>` merges `released/*.sql` into `base/<name>.sql`, moves the original files into `archive/`, and replaces the original released entries in the ledger with base entries (existing databases are **idempotent and not re-executed**); only takes effect with `--apply`.
- Exit codes follow the `content` CLI (see [content-tooling.md](content-tooling.md) §2).
- `--json`: `{ ok, applied[], skipped[], renamed[], errors[] }`.

## 5. Relationship with saves

- The save version migration chain ([save-schema.md](save-schema.md) §6) and SQL migrations follow the same principle: **one-way upward, idempotent**.