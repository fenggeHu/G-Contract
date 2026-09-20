# Contract: Content Authoring (Content Studio)

- Status: Proposed (execution gated by ADR-0006 owner approval)
- Updated: 2026-09-20

> The interface for authoring content packages (Defs / scenes / maps) used by content providers. Companion: [admin-panel.md](admin-panel.md). Field-level rules: [content-schema.md](content-schema.md); package layout: [content-package.md](content-package.md). Rationale: ADR-0005.

## 1. Model

- **local-first**: Content Studio runs as a local service on the content provider's machine and operates on **their own working copy** of a content package. The platform never commits to or depends on a provider's repository.
- The content package layout is the contracted structure (`manifest.json`, `defs/**`, `scenes/**`, `world/**`, `i18n/**`); field semantics come from the machine content schema (`content-<ver>.json`).
- On-disk content remains **JSON Defs**; Protobuf is used only on the control-plane wire (see [admin-panel.md](admin-panel.md)).

## 2. Operations (local HTTP/JSON)

```
GET    /api/content/packs
GET    /api/content/defs?type=&q=&tag=
GET    /api/content/defs/{type}/{id}
POST   /api/content/defs
PUT    /api/content/defs/{type}/{id}          # If-Match: <fileHash>
DELETE /api/content/defs/{type}/{id}
POST   /api/content/defs/{type}/{id}/patch     # reuse content-schema §0.2 patch ops
GET    /api/content/graph                      # relations registry (below)
POST   /api/content/validate
POST   /api/content/lint
POST   /api/content/preview
POST   /api/content/build                      # _meta.json + PCK via content CLI
POST   /api/content/i18n/template
GET    /api/content/assets                     # licenses.json registry
GET    /api/content/world/{id}
PUT    /api/content/world/{id}
POST   /api/content/world/{id}/bake            # content bake-world
GET    /api/content/scene/{id}
PUT    /api/content/scene/{id}
```

- Forms are generated from the machine content schema (mature JSON-Schema form library).
- Maps/scenes reuse **Tiled (`.tmj`)** and the **Godot editor**; Studio does not implement an editor kernel.

## 3. Relations registry

- `shared/model/relations-1.0.json` declares, per Def field, the target type, cardinality, and cross-pack rules.
- **Single source**: authored under `G_Engine/App/engine/sdk/`, emitted to `G_Shared` by `content sync`.
- Consumed by both Content Studio (graph/navigation) and the validator (referential integrity).
- Must be defined **before** Studio implementation (P0 ordering).

## 4. Integrity and concurrency

- Atomic writes (temp → rename).
- Optimistic concurrency via file hash (`If-Match: <fileHash>`); reject on mismatch to avoid clobbering external edits.
- Git-friendly: Studio edits only the contract structure; no private provider internals.

## 5. Publish pipeline

- Content remains provider-owned; publishing uses the existing CLI: `content validate` / `lint` / `build` → `_meta.json` + PCK, then `content verify-release`.
- The platform does not deploy provider content.

## 6. Boundaries

1. **Never modify `G_World`** from the platform; Studio is run by the content provider on their own copy.
2. Only the contracted package structure and content schema are read/written; no dependency on provider-private structure.
3. Generic across content providers.

## 7. Impact on content providers

| ID | Action | Blocking platform work? |
|---|---|---|
| I1 | Raise requirements via `capability-request` / contract PR | No |
| I2 | Adopt contract/tooling (bump `deps.json`) on their own schedule | No |
| I3 | Run Content Studio on their own package | No |
| I4 | Adjust Defs where the relations/namespace rules require (self-scheduled) | No |