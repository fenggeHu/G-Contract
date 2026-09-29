# Contract: Server Trigger Model

- Status: Active
- Updated: 2026-09-29

> Server triggers are a **separate, closed projection** for Nakama Lua authority. They are not client `trigger` Defs with a `server_authoritative` switch and must not reuse a general client-side interpreter. The runtime module loads in the Nakama image and the projection path carries `serverTriggers`; production migration/e2e evidence is required before release. See ADR-0013 and [trigger-model.md](trigger-model.md).

## 1. Purpose

Server triggers allow published content to compose a small set of server-owned events, predicates and actions without allowing arbitrary expression evaluation or content code on the server.

The server remains authoritative for combat, rewards, inventory, economy, progression, save state and match outcomes.

## 2. Projection shape

```json
{
  "pack": "pack.id",
  "packVersion": "1.0.0",
  "rulesetVersion": "ruleset-12",
  "rulesetHash": "sha256:...",
  "serverTriggers": [
    {
      "id": "encounter.wave_2",
      "event": "authoritative_entity_died",
      "condition": {
        "all": [
          { "quest_state": "encounter.active" },
          { "entity_alive": "boss_1", "eq": false }
        ]
      },
      "actions": [
        { "id": "spawn_declared_entity", "params": { "entity": "wave_2", "count": 3 } }
      ],
      "once": true,
      "cooldownMs": 0
    }
  ]
}
```

The example is illustrative. The exact event, predicate and action ids are owned by the server capability registry and are not content-defined.

## 3. Closed capability registry

The server implementation generates or validates one registry containing:

- event id, payload fields and emission point;
- predicate id, accepted parameter types and state source;
- action id, accepted parameters, authority boundary and idempotency behavior.

The registry is represented by the implementation-level `SUPPORTED_SERVER_TRIGGERS` set and a generated release artifact. It must not be hand-maintained in multiple repositories.

Initial proposed set:

| Class | IDs | Boundary |
|---|---|---|
| Events | `match_started`, `match_ended`, `instance_enter`, `instance_exit`, `authoritative_ability_resolved`, `authoritative_entity_died` | Emitted by Nakama authority only |
| Predicates | `flag`, `quest_state`, `entity_alive`, `cooldown_ready` | Read-only projected match state |
| Actions | `damage`, `heal`, `spawn_declared_entity`, `grant_declared_reward`, `end_encounter` | Server-implemented, typed params only |

`region_enter`, `on_tick`, render events and client-only interaction events are excluded until the server has an authoritative equivalent.

## 4. Loading and rejection

The server MUST reject the ruleset or match before activation when:

- the projection is missing, malformed, unsigned or hash-mismatched;
- `packVersion` or `rulesetVersion` is unsupported;
- an event, predicate or action id is absent from the server registry;
- parameters have the wrong type, exceed limits or reference undeclared content;
- two triggers create an invalid duplicate/ordering conflict;
- an action is not idempotent where replay is possible.

Rejections use stable codes:

| Code | Meaning |
|---|---|
| `server_trigger_projection_not_found` | Published projection is absent |
| `server_trigger_capability_unsupported` | Event/predicate/action is not enabled |
| `server_trigger_param_invalid` | Parameter type/range/reference invalid |
| `server_trigger_version_mismatch` | Pack/ruleset/engine version incompatible |
| `server_trigger_hash_mismatch` | Projection hash or signature failed |
| `server_trigger_conflict` | Duplicate or conflicting trigger definition |

## 5. Execution semantics

- Events are ordered by the authoritative match clock, not client arrival time.
- A trigger evaluates against a snapshot of the current authoritative state.
- Actions run in declaration order, inside the server's existing match command boundary.
- `once` and `cooldownMs` are server state, not client state.
- Replayed input or duplicated delivery must not duplicate an idempotent action.
- A failed action returns a structured result; the server does not silently fall back to client execution.

## 6. Compatibility and testing

- Server trigger capabilities use `domain.action@MAJOR.MINOR` style versions.
- Adding an action or predicate is a capability change and requires contract review, Lua tests, negative tests and client observation handling.
- The server must have golden cases shared with the client only for observable outcomes; it does not share or copy the client `Conditions.eval` or `Actions.execute_def` implementation.
- The projection generator must emit `serverTriggers` and fail closed when any referenced capability is unsupported.

## 7. Non-goals

- No Lua or GDScript supplied by content.
- No formulas, callbacks, arbitrary expressions or executable DSL.
- No client request can select an unsupported server trigger.
- No server trigger may be introduced for only one content pack; use the capability review process.
