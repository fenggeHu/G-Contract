# Contract: Moderation (mute / report / appeal)

- Status: Proposed
- Updated: 2026-09-20

> Operator moderation for the G3 platform. Companion: [admin-panel.md](admin-panel.md). Rationale: ADR-0008.

## 1. Principles

- **Mute and ban are separate permissions**: `chat.mute` (support/GM) vs `account.ban` (elevated + approval). Acknowledge UNIX/GM practice (GameDevMind).
- Enforcement is **server-side**: client filtering is UX only and never trusted.
- Every moderation action is audited (see [admin-panel.md](admin-panel.md) §4).

## 2. Mute (timed server-side state)

- Record: Nakama storage collection `g3`, key `mute_<user_id>`:
  ```
  { scope: "chat"|"match", reason, issued_by, issued_at, expires_at }
  ```
- Enforcement: the server runtime intercepts message sends (`ChannelMessageSend`) and rejects while a non-expired mute exists; the sender is notified on apply and on expiry.
- Unmute removes the record. Duration/scope set by the operator action `moderation.mute`.

## 3. Content safety (server-side)

- Sensitive-word/pattern filter + **rate limit** (per-user send frequency) + **auto-mute** on repeated violations; every filter hit is logged (`g3/filter_log_<user>` or audit).
- PII masking (ID/phone/email) in operator UI by default; unmasking is a permissioned, audited action.

## 4. Reports and appeals

- Report record (`g3/reports`): `{ id, reporter_id, target_id, category, evidence, ts, status }`; `category` enum (abuse/spam/cheat/other).
- Lifecycle: `open → reviewing → actioned | dismissed → appealed → reviewed`.
- Evidence is stored **before** disposition (immutable); bans driven by reports carry the report id.
- Operator actions: `moderation.report` (list/dispose), `moderation.appeal` (list/decide).

## 5. Permissions

| Permission | Meaning |
|---|---|
| `moderation.read` | list reports/appeals |
| `chat.mute` | apply/remove mute |
| `moderation.action` | dispose report / decide appeal |
| `account.ban` | ban/unban (requires approval for duration > 0) |

## 6. Non-goals

- No in-engine GM command surface here (that is the dev-only `gm.command`/`gm.panel`, release-stripped; see [admin-panel.md](admin-panel.md) §7).
- No client-side-only enforcement.