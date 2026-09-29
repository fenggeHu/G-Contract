# Contract: Remote Config & LiveOps delivery

- Status: Proposed
- Updated: 2026-09-29

> Delivery of feature flags, MOTD, announcements and activities from the platform to clients and the runtime. Companion: [admin-panel.md](admin-panel.md). Rationale: ADR-0009.

## 1. Model

- **Config document** (Nakama storage `g3`, key `config`), version-managed (storage `version` = CAS):
  ```
  { flags: { <key>: <bool|value> }, motd: { text, level }, version }
  ```
- **Events/activities** (Nakama storage `g3`, key `events`): array of `{ id, kind, starts_at, ends_at, payload, enabled }`.
- **Announcements** are delivered as persistent Nakama notifications (already available) and recorded with a send log.
- **Mail/grants** go through the player save (inventory) + notification (see [admin-panel.md](admin-panel.md) §6).

## 2. Delivery (client)

- Client fetches on login/reconnect via RPC `config_get` → `{ revision, config, events }`; caches locally; falls back to cached/last-known values offline.
- Flags are applied via the engine flag API (`Config.set_flag`) — the **remote value overrides** env/ProjectSettings defaults; local dev overrides still win in dev builds.
- MOTD/announcements are surfaced as notifications (existing channel); the client renders them.
- Gray/bucketing: deterministic `hash(user_id) % n` (no server session state), used for flag/experiment rollout.

## 3. Versioning, gray, rollback

- Every config mutation bumps the storage `version` (CAS). Publishing is `draft → validate → apply`; rollback = write back the previous document (kept in `g3/config_history`).
- Gray rollout: a flag value may be an object `{ rollout: n, value }`; client evaluates by bucketing. Server-side evaluation (authoritative) uses the same rule.

## 4. Scheduled / bulk

- Announcements and mail support **batch and scheduled** sending; scheduled jobs are **idempotent by periodic key** (e.g. `event:<id>:<occurrence>`), executed by a scheduler (BFF/worker), not by ad-hoc manual DB edits.

## 5. Operator actions

| Action | Permission | Notes |
|---|---|---|
| `liveops.motd_set` | `liveops.motd` | versioned, audited |
| `liveops.flag_set` | `liveops.flag` | versioned, audited |
| `liveops.event_start/stop` | `liveops.event` | storage + broadcast |
| `liveops.announce` | `liveops.announce` | batch/scheduled + send log |

## 6. Client impact

- Phase A (server-only) delivers storage + admin + notifications.
- Phase B (included) adds the **G_Engine** fetch/apply (`config_get` + `Config.set_flag` + cache + offline fallback). Cross-repo: contract first, then `G_Server`, then `G_Engine`, then `G_Admin`.

## 7. Ruleset delivery (content rules / config)

> Companion to ADR-0014. Distinct from LiveOps flags above: **ruleset** is **content-authored, authoritative** data published through the `G_Admin` release gate; flags are operator-facing rollout controls.

- **Ruleset bundle** = content-authored, versioned artifact (`{ pack, version, hash, data }`) delivered like a content package: `draft → validate → sign → publish → gray → rollback`.
- **Authority**: the **server consumes the published ruleset projection**; gameplay results are server-authoritative (see online-and-instances.md). Clients read ruleset values via RPC `config_get` for **presentation/gating only** and MUST NOT decide authoritative outcomes.
- **Versioning / rollback**: carried by the `G_Admin` control plane (version + hash); rollback = republish previous bundle. Signing/hash verified on load.
- **Operators**: actions `content.ruleset.publish` / `content.ruleset.rollback` follow the [admin-panel.md](admin-panel.md) action catalog (RBAC, `approval`, `idempotency`, audit).
- **Non-goals**: expressing gameplay logic in a ruleset (logic belongs to the [trigger model](trigger-model.md) primitives); replacing the content projection as the authority source.

## 8. Status note

- This document remains **Proposed**; §7 requires implementation + acceptance evidence before promotion to **Active** (see [docs/README.md](../README.md) rule 4).