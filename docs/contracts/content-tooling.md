# Contract: Content Toolchain (Schema / Validator)

- Status: Active
- Updated: 2026-09-17

> The **tooling implementation specification** for the content contract: machine-readable Schema, validator I/O, capability matrix. For field semantics, see [content-schema.md](content-schema.md).

## 1. Machine-Readable Schema

| Item | Definition |
|---|---|
| Generated artifact | `shared/contract/content-<ver>.json` (JSON Schema 2020-12) |
| Entry points | `shared/contract/{def,manifest,defs-file}.json`: point to `$defs/def` / `$defs/manifest` by file type, for IDE / tool validation |
| Single source of truth | `App/engine/sdk/schema/` (**editing generated artifacts by hand is forbidden**); protocol/model IDL sources are in `App/engine/sdk/{protocol,model}/` |
| Structure | One `$defs` entry per Def type; common fields (`type/id/labelKey/tags`); references use `$ref` |
| Version | `MAJOR.MINOR`, following `content-schema.md` |

> **IDE integration**: map `**/defs/*.json` to `defs-file.json` and `**/manifest.json` to `manifest.json` (e.g. VS Code `json.schemas`) to get field-level completion and validation; the entry point relatively references `content-<ver>.json` in the same directory. For examples, see [content-examples.md](content-examples.md).

## 2. Validator CLI `content`

| Command | Purpose |
|---|---|
| `content validate <pack>` | Schema + references + dependencies + capabilities |
| `content lint <pack>` | Naming + localization keys + licensing (`licenses.json`) |
| `content build <pack>` | Validation gate (validate+lint) + generate `_meta.json` + (`--godot`) export PCK |
| `content preview <scene>` | Resolve the scene Def → `.tscn` path; with `--godot`, launch the engine preview |
| `content diff <a> <b>` | Content differences (`type:id` additions/removals/changes; informational, exit code 0); arguments may be directories or snapshot files |
| `content snapshot <root>` | Export a Def snapshot (`--out`, default `dist/defs.snapshot.json`) for regression comparison |
| `content i18n-template <pack>` | Generate/refresh the translation CSV template (preserve existing translations, fill in missing keys) |
| `content perf` | Run the world benchmark or `--scene <id>`; optional p95/max budget gates (`--budget-p95` / `--budget-max`, 0 = no gate) |
| `content playtest <scene>` | Headlessly auto-run a content scene and **assert no errors** (H1 quantitative gate) |
| `content release` | Produce `dist/release.json` (version matrix + artifact manifest) |
| `content verify-release` | Validate that `release.json` is consistent with the contract (prerequisite for consumer-side pinning) |
| `content bake-world` | Bake the world map overhead texture → `<pack>/world/<x>_<y>.jpg` (1024px/chunk) |

**I/O**
- Arguments: pack path / scene.
- `--json`: a single result object `{ ok, errors[], warnings[] }`.
- Exit codes: `0` success · `2` usage · `3` schema · `4` dependency · `5` lint · `6` internal.
- In text mode, print a **`hint:` remediation tip** for each error/warning (by error code); for error code → tip, see the CLI implementation and §Error Codes below.

**`content build` usage**
```
content build <content-root> [--godot <bin>] [--project <dir>] [--out <file.pck>] [--content-version <v>]
```
1. First pass the `validate` + `lint` gate (stop on any failure).
2. Write `_meta.json` at the content root: `{ "content": <version>, "schema_major": "<major>" }` (the engine refuses incompatible content based on this at startup).
3. When `--godot` is provided, run `godot --headless --path <project> --export-pack content_pck <out>`; `--project` defaults to the parent directory of the content root, and `--out` defaults to `<project>/../content-<version>.pck`. Without `--godot`, only `_meta.json` is generated (with a warning).

**`content diff` usage**: `content diff <A> <B>` compares two sets of Defs (normalized JSON), outputting `+ / - / ~` and counts; arguments may be a **content root directory** or a **`content snapshot` snapshot file**. `--json` outputs `{added,removed,changed}`. Used for content regression and review (informational, does not fail).

**`content snapshot` usage**: `content snapshot <root> [--out <file>]` exports a `type:id → raw Def` snapshot; combine with `content diff <snapshot> <content>` for regression (golden baseline).

**`content i18n-template` usage**: `content i18n-template <pack> [--out <file>]` scans the pack's `labelKey`/`textKey`, reads the existing translation CSV (preserving translations), fills in missing keys, and outputs a template; columns are taken from existing locales (default `en,zh_CN`). It warns when the Manifest does not declare `entry.i18n` (strong validation only begins after declaration).

**Error Codes**

| Code | Meaning |
|---|---|
| `SCHEMA_INVALID` | Invalid field/type/enum |
| `REF_NOT_FOUND` | Dangling reference |
| `DEP_MISSING` | Missing dependency |
| `CAP_MISSING` | Missing capability/unsatisfied version |
| `CYCLE_DETECTED` | Circular dependency |
| `NAME_DUP` | Duplicate id |
| `LOC_KEY_MISSING` | Missing localization key (field missing, or after declaring `entry.i18n`, a locale column has no translation) |
| `ASSET_MISSING` | Missing asset |
| `LICENSE_MISSING` | Asset licensing not registered / `license` field missing (`licenses.json`, see [content-package.md](content-package.md) §6) |

## 3. Capability Matrix `capabilities.json`

- Path: `shared/contract/capabilities.json` (produced by the engine build).

```json
{ "engine": "1.2.0", "schema": "1.3", "schemaSupported": ["1"],
  "capabilities": [ { "name": "world.stream", "version": "1.0", "status": "enabled" } ] }
```

- `status`: `enabled` | `deferred`. The validator uses it to validate the Manifest's `capabilities`.

## 4. CI

- **Content pipeline**: `validate` + `lint` (on every content commit).
- **Engine pipeline**: compile + unit + export `capabilities.json`.
- **Contract tests**: load smoke tests across the engine version matrix (see internal platform doc).
- **Content repo CI template**: shipped with devkit-lite (`ci/content-ci.yml`); the World team copies it to `.github/workflows/` to get a "pin validation + validate + lint" pipeline.
- Compatibility determination: `content validate` accepts packs with a **supported major version** according to `schemaSupported` (non-current versions produce a warning only); it validates the `requires.engine` range and capability version minimums, and refuses if unsatisfied.

## 5. Release Artifacts and Version Matrix `release.json`

At release time the engine produces `release.json` (in the same batch as the `content` CLI and `devkit-lite`), for content/server consumers to **pin versions and validate compatibility**. Generation: `make release` (which internally invokes `content release`).

```json
{
  "engine": "0.1.0",
  "godot": "4.7.2",
  "schema": "1.3",
  "schemaSupported": ["1"],
  "capabilities": { "total": 38, "enabled": 38, "deferred": 0, "digest": "sha256:…" },
  "artifacts": {
    "contentCli": "content",
    "devkitLite": "engine-devkit-lite-0.1.0.zip",
    "contract": "shared/contract",
    "capabilities": "App/engine/sdk/capabilities.json",
    "contentSchema": "App/engine/sdk/schema/content-1.3.json"
  }
}
```

| Field | Description |
|---|---|
| `engine` | Engine SemVer (source: `App/engine/sdk/capabilities.json`) |
| `godot` | Matching Godot version (source: `deps.json`, may be empty) |
| `schema` | Current Content Schema version |
| `schemaSupported` | **Compatibility window**: the set of Schema major versions supported by the engine (source: capabilities.json) |
| `capabilities.total/enabled/deferred` | Capability counts |
| `capabilities.digest` | Deterministic fingerprint of the capability set (`sha256`, `name@version:status` sorted by name) |
| `artifacts.*` | Same-batch artifact names/paths (relative to the release package) |

**Consumer rules**
- Content/server `deps.json` records the engine version and pins the above artifacts; before building, validate:
  1. Content pack `schema` major version ∈ `schemaSupported`, otherwise **refuse to load**;
  2. The contract `capabilities.json` `digest` matches `release.json`, to prevent drift.
- Validation command: `content verify-release --release <release.json> --caps <capabilities.json>` (exit codes as in §2).
- `release.json` is **not committed** (`dist/` is a build artifact); it is distributed with the release artifacts.
- **Mirroring and provenance**: `G-Contract` is **generated** from this repo by `tools/publish_contract.sh` (read-only, allowlisted), and writes `.g3-mirror.json` (`source_commit`) for per-commit verification; `--check` detects drift; G-Contract CI has a **read-only mirror guard** (PRs are forbidden from directly modifying `shared/`, `docs/contracts/`, `public/`). Source-side CI (`publish.yml`) automatically publishes when `shared/**`/`docs/contracts/**`/`public/**` change or on a `contract-v*` tag.
- **External publishing**: the contract and artifacts are published to the **Releases** of the public repo `G-Contract` (`contract.zip`, `release.json`, `content`, `engine-devkit-lite-<ver>.zip`), for content parties to pin and download.
- **Content team delivery package** (produced by `make release` in `dist/`): `content` (CLI), `contract/` (contract: `capabilities.json` + `content-<ver>.json`), `release.json` (version matrix), `engine-devkit-lite-<ver>.zip` (package settings + CI template).
- Content team local layout: `devkit/bin/content` + `devkit/sdk/{capabilities.json,schema/content-<ver>.json}` + `devkit/release.json`; `content validate/lint/... --sdk devkit/sdk`.

## 6. Code Generation

> Purpose: generate **typed artifacts** from the **single source of truth**, surface field/enum errors at compile time, and unify the type/enum list.

- **Single source of truth**: `App/engine/sdk/schema/content-<ver>.json` (+ `capabilities.json`).
- **Command**: `content gen [--sdk <dir>] [--out <dir>] [--check]`
  - Artifacts (deterministic, stably sorted) are placed in `App/engine/sdk/generated/`:
    `types.json` (neutral: engine / schema / defTypes / fields / enums)
    `gdscript/g3_defs.gd`, `go/g3_defs.gen.go`, `lua/g3_defs_gen.lua`
  - Enum source: Schema `properties.<field>.enum` (top-level fields).
- **Drift gate**: `content gen --check` (fails if generated artifacts ≠ source) → `make gen-check` + `arch_check` **R8**.
- **Boundary**: the content team only **reads** the generated artifacts; whether and when to adopt them is decided by the content team (the platform does not change the content layer).