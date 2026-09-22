# Contract: Admin Panel (G_Admin)

- Status: Active
- Updated: 2026-09-20

> The operator/content-admin control plane for the G3 platform. Companion: [content-authoring.md](content-authoring.md) · [moderation.md](moderation.md) · [remote-config.md](remote-config.md) · [telemetry.md](telemetry.md). Design rationale: admin-ops-panel/plan.md, ADR-0001..0010.

## 1. Scope

- **Runtime operations**: accounts, player saves, inventory, mail/grants, moderation, match operations, live-ops, telemetry/logs/audit.
- **Content authoring**: via a separate local service (Content Studio), see [content-authoring.md](content-authoring.md).
- **Home**: new repository `G_Admin`.
- **Non-goals**: game data-plane protocol changes; multi-tenancy; arbitrary command execution; direct database editing of content templates.

## 2. Architecture

```
Web ── gRPC/Protobuf (connect-go) ── BFF (Go)
    BFF ── official Nakama Go client ── Nakama
    BFF ── Nakama RPC (JSON) ── Nakama Go plugin (backend.so)   # runtime admin actions
    BFF ── Postgres (schema: admin)                             # auth / RBAC / audit
```

- Protobuf boundary: **panel ⇄ BFF only**. `G_Admin` owns its `.proto`; no cross-repo proto sharing, no custom codec.
- The **plugin RPC contract** (names, request/response JSON) is defined here as the interface between BFF and the Nakama Go plugin.

## 3. Action catalog

Every action is typed and declared with metadata:

```
{ action, request, response,
  scope: runtime | content,
  onlinePolicy: live | block | kick_then_write | at_rest,
  risk: low | medium | high | critical,
  permission,
  cooldown: <seconds>,          // per (actor, action)
  limit: <numeric cap>,         // condition check; over-cap -> approval
  approval: none | required,    // dual control for high-risk
  idempotency: required | none }
```

| Domain | Actions | Status |
|---|---|---|
| Auth (panel) | login, logout, whoami | ✅ |
| Account | get, ban, unban, kick, update, **recovery/bind/merge**（不做建号） | partial |
| Player | get, patch, reset, rollback | partial（get/patch ✅） |
| Inventory | list, grant, remove | ✅ |
| Mail | send, list, delete | partial（send ✅） |
| Match | list, terminate, broadcast, get, kick | partial |
| Moderation | mute, unmute, report, list, appeal | planned（见 [moderation.md](moderation.md)） |
| LiveOps | announce, motd_set, flag_set, event_start, event_stop | partial（announce ✅） |
| Telemetry | overview, funnel, query | planned（见 [telemetry.md](telemetry.md)） |
| Audit | list | ✅ |
| Command | catalog, execute | planned |
| Release | list, verify | planned |

- **No command strings**: callers send an action id + parameters; the server validates before execution.
- `onlinePolicy` governs targets that are currently online (see §5).
- **Approval**: actions with `approval: required` (or over `limit`) create a pending `approval_request`; execution requires **dual approval** (two admins, or operator+admin). See ADR-0008.
- **Idempotency**: mutating actions with `idempotency: required` carry an `idempotencyKey`; the server rejects replays.

## 4. RBAC and audit

- Roles: `viewer < gm < operator < admin < superadmin`.
- Permissions are dotted `domain.action` (e.g. `player.write`, `account.ban`, `liveops.announce`, `chat.mute`, `moderation.action`, `liveops.motd`, `liveops.flag`, `liveops.event`, `telemetry.read`, `command.execute`, `approval.decide`).
- **Separate mute vs ban**: `chat.mute` (support) is distinct from `account.ban` (elevated + approval). See [moderation.md](moderation.md).
- Roles/permissions are stored in the `admin` schema; enforced in the BFF and re-checked server-side.
- Middleware chain: **Auth → RBAC → ConditionCheck → Approval → Execute → Audit**. Over-limit/high-risk actions produce a pending `approval_request` (dual approval; never self-approve).
- Every write action is audited (**append-only**, failure/denial included):

```
AuditRecord {
  id, ts, actorId, actorRole, actorIp, action, risk,
  targetType, targetId, before, after, result, reason, requestId,
  prevHash, hash          // tamper-evident hash chain
}
```

- Retention: ≥ 180 days for sensitive actions (grant/ban/export). See ADR-0008.

## 5. Online policy

| Value | Behavior |
|---|---|
| `live` | Apply immediately to the connected client (server→client push). |
| `block` | Reject if the target is online. |
| `kick_then_write` | Disconnect the target, then persist (effective on next login). |
| `at_rest` | Persist only; effective on next load/login. |

The BFF resolves online state before dispatch. `live` pushes require authorization.

## 6. Plugin RPC contract (BFF ⇄ Nakama Go plugin)

- Transport: Nakama RPC, JSON payload (mature), documented here.
- **Server-side authorization**: every plugin RPC requires the shared-secret header `X-G3-Admin-Token` matching the server env `G3_ADMIN_RPC_TOKEN` (else `code 7`). Deployments MUST set a non-default `http_key` and `G3_ADMIN_RPC_TOKEN`, and MUST NOT expose the Nakama HTTP port publicly. See ADR-0004.
- Match: `admin_match_list`, `admin_match_get`, `admin_match_kick`, `admin_match_terminate`, `admin_match_broadcast`, `admin_presence_list`
- Player: `admin_player_get`, `admin_player_patch`
- Account: `admin_account_get`, `admin_account_ban`, `admin_account_unban`, `admin_account_list`, `admin_account_update`
- Inventory: `admin_inventory_list` (returns the BFF-compatible `inventory` defId view **and** the v4 `items` instance view: `{instanceId, defId, count, slot, bound, durability}`), `admin_inventory_grant`, `admin_inventory_remove`
- Mail/LiveOps: `admin_mail_send`, `admin_liveops_announce`, `admin_liveops_motd_set`, `admin_liveops_flag_set`, `admin_liveops_event_start/stop`, `admin_config_get`（客户端拉取）
- Moderation: `admin_moderation_mute`, `admin_moderation_unmute`, `admin_moderation_report`, `admin_moderation_report_list`
- Telemetry: `admin_telemetry_overview`, `admin_telemetry_funnel`, `admin_telemetry_query`

> `MatchKick` exists only on `MatchDispatcher` (in-match); external kick therefore routes via `MatchSignal` (see ADR-0003).
> Config/flags/events delivered to clients via RPC `config_get` (see [remote-config.md](remote-config.md)); mute enforced on the message path (see [moderation.md](moderation.md)).

## 7. Boundaries

1. **Never modify `G_World`**; content changes go through Content Studio (local-first) and the content package contract.
2. Generic and reusable across content providers; no single-provider customization.
3. Game protocol and game data plane are unchanged.
4. One change per repository; contract first.
5. **GM domain separation**: ops identities/credentials are separate from player auth; a leaked ops account must not equal game-server root. The **dev-only** in-engine GM (`gm.command`/`gm.panel`, release-stripped) is a *developer/QA* surface and is **not** this ops panel; both must not share the player auth domain.
6. No direct edits to player data tables by the panel: runtime mutations go through the Nakama Go plugin RPCs only (see `G_Server/AGENTS.md`).

## 8. Versioning

- Control-plane services are versioned under `G_Admin/proto/admin/v1/`.
- Fields are add-only; removal requires a major bump.
- Panel↔BFF library choices are mature components and are finalized (and recorded) at P0 kickoff.