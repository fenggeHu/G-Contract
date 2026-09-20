# Contract: Admin Panel (G_Admin)

- Status: Proposed (execution gated by ADR-0006 owner approval)
- Updated: 2026-09-20

> The operator/content-admin control plane for the G3 platform. Companion: [content-authoring.md](content-authoring.md). Design rationale: admin-ops-panel-plan.md, ADR-0001..0006.

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
  permission, idempotency: required | none }
```

| Domain | Actions (initial) |
|---|---|
| Auth (panel) | login, logout, whoami |
| Account | list, get, create, update, ban, unban, kick, set_role, reset_password |
| Player | get, patch, reset, rollback |
| Inventory | list, grant, remove |
| Mail | list, send, delete |
| Match | list, get, kick, terminate, broadcast |
| Moderation | report, list, ban, mute, unmute |
| LiveOps | announce, motd_set, flag_set, event_start, event_stop |
| Telemetry | overview, funnel, query |
| Audit | list |
| Command | catalog, execute |
| Release | list, verify |

- **No command strings**: callers send an action id + parameters; the server validates before execution.
- `onlinePolicy` governs targets that are currently online (see §5).

## 4. RBAC and audit

- Roles: `viewer < gm < operator < admin < superadmin`.
- Permissions are dotted `domain.action` (e.g. `player.write`, `account.ban`, `liveops.announce`).
- Roles/permissions are stored in the `admin` schema; enforced in the BFF and re-checked server-side.
- Every write action is audited (append-only):

```
AuditRecord {
  id, ts, actorId, actorRole, actorIp, action, risk,
  targetType, targetId, before, after, result, reason, requestId
}
```

- Optional hash chaining for tamper evidence.

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
- `admin_match_list` → `{ matches: [{ id, size, label }] }`
- `admin_match_kick` → in: `{ match_id, user_id }`; the plugin calls `MatchSignal`; `match.lua`'s `match_signal` calls `dispatcher.match_kick`.
- `admin_match_terminate` → in: `{ match_id }`
- `admin_match_broadcast` → in: `{ match_id, message }`
- `admin_presence_list` → `{ presences: [...] }`

> `MatchKick` exists only on `MatchDispatcher` (in-match); external kick therefore routes via `MatchSignal` (see ADR-0003).

## 7. Boundaries

1. **Never modify `G_World`**; content changes go through Content Studio (local-first) and the content package contract.
2. Generic and reusable across content providers; no single-provider customization.
3. Game protocol and game data plane are unchanged.
4. One change per repository; contract first.

## 8. Versioning

- Control-plane services are versioned under `G_Admin/proto/admin/v1/`.
- Fields are add-only; removal requires a major bump.
- Panel↔BFF library choices are mature components and are finalized (and recorded) at P0 kickoff.