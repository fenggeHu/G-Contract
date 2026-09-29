# Contract: Content Trigger Model

- Status: Active
- Updated: 2026-09-29

> A generic, declarative binding of **event → condition → action** over primitives the platform already implements. It is a *binding*, not a logic language: no formulas, callbacks or script strings. See ADR-0013 §D6.
> Reuses: [events.md](events.md) (whitelisted events), [content-schema.md](content-schema.md) §27 (condition expressions), `action` Defs (capability `action.core`), `region.trigger`, `hooks.md`.
> **Status note**: Active for client/offline execution after `make test-trigger`, `make test-sceneplayer`, `make test`, `make validate` and `make gen-check`. The separate server-trigger contract remains Proposed until Lua runtime and projection acceptance are complete.

## 1. Model

```
Trigger {
  id,                      # Def id (globally unique)
  type: "trigger",
  on: <eventId> | { event: <eventId>, match: { <payloadKey>: <value> } },
  filter: <condition spec>,          # content-schema §27; absent = true
  run: [ { action: <actionDefId>, when?: <condition spec> } ],
  authority: "client" | "offline"       # 普通 Trigger 只产生非权威结果
  flags: { once?: bool, cooldownMs?: int }
}
```

- `on.match` is **exact equality** on whitelisted payload keys only (no expressions).
- `run` is evaluated **in order**; each entry fires its `action` Def when its `when` (if any) is true.
- `once`: fire at most once per session/save. `cooldownMs`: minimum interval between fires.
- `authority` is limited to `client`/`offline`; it cannot grant authority over server state.
- A trigger is **data only**; it declares binding and parameters, never executable code.

## 2. Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | Yes | Def id |
| `type` | literal `trigger` | Yes | New Def type (schema MINOR) |
| `on` | string \| object | Yes | Whitelisted event id (see §3); optional payload `match` |
| `filter` | condition spec | No | content-schema §27; absent = true |
| `run[]` | array | Yes | `{ action, when? }`; `action` references an `action` Def |
| `authority` | enum | No | `client` or `offline`; default `client` |
| `flags.once` | bool | No | Fire at most once |
| `flags.cooldownMs` | int | No | Minimum interval |

Scenes reference triggers additively: `scene.triggers[]` (references `trigger` Defs). No inlining of trigger bodies.

## 3. Events

- `on.event` MUST be in the **whitelist** of [events.md](events.md). New events require a platform contract change (never content-defined).
- Only events the platform actually emits are eligible.
- `tick`, physics-frame and presentation-only signals are not eligible for gameplay Trigger bindings.
- Each event must have one platform-owned mapping: event id, Godot signal, ordered payload fields, delivery class (`logical`/`presentation`), and whether it can feed a future server trigger.

## 4. Conditions

- `filter` and `run[].when` use the existing [content-schema §27](content-schema.md) forms: `all/any/not/flag/quest/item/stat/level/kv`, evaluated by `Conditions.eval(spec, ctx)` (capability `conditions.core`).
- No new condition vocabulary is introduced by this contract. Adding a predicate is a platform capability change.

## 5. Actions

- `run[].action` references an `action` Def (capability `action.core`; kinds `learn_ability/teleport/consume/give/grant_items/destroy/pick_up` — add-only).
- New behaviour is expressed by **adding an action primitive on the platform**, not by extending the trigger language.

## 6. Authority

- Ordinary `trigger` is evaluated by Godot for local presentation/offline behavior only. It MUST NOT compute authoritative HP, loot, progression, inventory, economy, save or match results.
- Server authority uses a separate `server_trigger` projection and capability directory. It is not a client `trigger` with a boolean switch.
- A `server_trigger` must declare `event`, `condition` and `action` ids from the server registry. The server rejects unknown ids before a match or ruleset becomes active.
- The server remains the sole authority for gameplay results (see online-and-instances.md).

### 6.1 Server trigger boundary

The first server-trigger release is limited to:

- server-observed lifecycle events (`instance_enter`, `instance_exit`, match start/end, authoritative ability resolution, authoritative entity death);
- closed predicates over projected state (`flag`, `quest_state`, `entity_alive`, `cooldown_ready`);
- closed actions already implemented by the server (`damage`, `heal`, `spawn_declared_entity`, `grant_declared_reward`, `end_encounter`).

Client-only region and presentation events remain ordinary Triggers until the server has an authoritative equivalent.

## 7. Validation

- `on.event` is whitelisted; `on.match` keys are valid payload keys.
- Every `run[].action` resolves to an existing `action` Def (`REF_NOT_FOUND` otherwise).
- Every condition key/op is a §27 form.
- No cycles among triggers; no executable content (reject formula/callback/script strings).
- `authority` is `client` or `offline`; server authority cannot be requested by a content field.
- Server trigger ids, when present in a published projection, resolve against the generated server capability registry.

## 8. Versioning

- Schema: **add-only**, MINOR. New Def type `trigger` and `scene.triggers[]` are additive.
- New events / actions / condition predicates are platform capability increments (own version).
- Capability `trigger.core` and its generated declaration are emitted by the platform implementation (single source); this document does not hand-author capability data.

## 9. Boundaries

- This contract introduces **no arbitrary logic**: it composes existing declarative primitives (no DSL, no script field).
- No content-defined events, actions or predicates; new ones go through capability-review.md.
- The engine opens no API for a single content pack.

## 10. Deferred (reserved, not in this contract)

- **Content scripting** (e.g. Lua sandbox / UGC) attached to triggers — separate ADR + security review + VM selection.
- Server triggers and their closed capability registry are a separate contract extension, not an implementation detail of this client contract.
- Complex state machines / persistent trigger variables.
- Per-trigger custom formulas (intentionally excluded, not merely deferred).
