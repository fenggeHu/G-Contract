# Contract: Protocol and Shared Models

- Status: Active
- Updated: 2026-09-20

> `shared/` is the **single shared source for Client and Server**; the protocol and shared models exist **only here**, read-only for both ends.

## 1. Shared layer responsibilities

```
shared/
├── protocol/   message definitions (versioned) + encoding/decoding
├── model/      shared types for entities / components / Def
└── contract/   generated artifacts: content schema, capabilities, protocol IDL
```

> **Current status**: a **versioned IDL** already exists — `shared/protocol/protocol-1.0.json`, `shared/model/model-1.0.json` (source: `App/engine/sdk/{protocol,model}/`, synced by `content sync`). `content gen` generates constants and encode/decode helpers for both ends (`App/engine/sdk/generated/gdscript/g3_protocol.gd`, etc., drift gate `arch_check` R8). During pre-research the transport is still inline JSON (Nakama Realtime match state, see §8); binary (Protobuf) is a separate follow-up item.

## 2. Message categories

| Category | Direction | Transport | Example |
|---|---|---|---|
| `world` | Bidirectional | Reliable + sparse broadcast | Position, AOI enter/leave, entity state |
| `instance` | Bidirectional | Real-time (prediction) | Movement, abilities, hits, state sync |
| `social` | Bidirectional | Reliable | Chat, party, friends, mail |
| `system` | Bidirectional | Reliable | Login, version negotiation, errors, heartbeat |

## 3. Transport

| Channel | Purpose | Characteristics |
|---|---|---|
| World channel | Presence/AOI | Reliable, low frequency |
| Instance channel | Real-time action | Low latency, droppable snapshots + prediction |
| Social channel | Chat/party | Reliable, cross-region |

- World and instances **use different connections/protocols**, so they do not hinder each other.

## 4. Serialization and versions

- Prefer **binary** (compact, fast); JSON mirrors may be used for debugging.
- Each message carries a **type ID + version**; follow the compatibility policy (fields may only be added; removal requires a MAJOR bump).
- The handshake performs **version negotiation**: if incompatible, reject and notify.

## 5. Shared models

`shared/model` defines the **entity/component/Def ID** and enums agreed upon by both ends:

| Model | Description |
|---|---|
| `Entity` | `{ id, archetype(defRef), components[] }` |
| Components | `Transform/Movement/Appearance/Presence/Social/Replicable` (the instance side additionally has combat components) |
| Def reference | content pack id + Def id (cross-pack references must be declared) |

> **Def = static prototype**; **Entity = runtime state**. Prototypes come from content, runtime state is authoritative on the server.

## 6. Generated artifacts (`shared/contract`)

- `content-<ver>.json`: content schema.
- `capabilities.json`: engine capability matrix.
- Protocol IDL → code for both ends (type-safe, drift-proof): `content gen` produces `g3_protocol.gd` / `g3_model.gd` / `g3_protocol.gen.go` / `g3_protocol_gen.lua` (including `T_*` constants, `MESSAGES`, `encode/decode`). For the drift gate see [content-tooling.md](content-tooling.md) §6, `arch_check` R8.
- **Single source of generation**; generated artifacts must not be edited by hand.

## 7. Contract tests

- Protocol compatibility tests (old and new versions interoperate).
- Consistency validation between shared models and both ends' implementations.
- Wired into the CI engine matrix (see internal platform doc).

## 8. Pre-research messages (P6, current implementation)

Transport: **Nakama built-in Realtime match state** (JSON objects), with `t` as the type. Joining a match uses Nakama **presence join** (no `t="join"` data message).

### RPC

| RPC | Input (JSON) | Output (JSON) | Description |
|---|---|---|---|
| `telemetry_report` | `{}` | `{ok,partial,users,events,steps}` | 遥测聚合（`loop_progress` 步骤计数；跨用户尽力，失败回退当前用户） |
| `telemetry_sink` | `{events:[…]}` | `{ok,stored}` | Telemetry ingestion (requires user consent); stored at `g3/telemetry_<user>`, rate-limited |
| `create_match` | `{mode:"pve"\|"pvp", ...}` | `{match_id, mode}` | Create a generic match; default `mode="pve"` |
| `save_read` | `{}` | `{ok, exists, revision, snapshot}` | Read the caller's cloud save (server-authoritative; `revision` is the CAS token) |
| `save_write` | `{snapshot, expected_revision}` | `{ok, revision, snapshot}` or `{ok:false, err:"conflict", revision, snapshot}` | Server-authoritative cloud write (CAS); on mismatch returns the server snapshot (**server wins**). See [save-schema.md](save-schema.md) §9 |
| `enter_world` | `{prefer_poi?, presence_meta?}` | `{ok, world_id, match_id, proto, spawn:{poi_id,x,y,altitude}, revision}` | Enter the **persistent world** room (creates/ensures the singleton world match); the server resolves the authoritative spawn from the cloud save `player.location`, else the content projection `presence_meta` (`default_entry` / `pois`) |
| `leave_world` | `{}` | `{ok}` | Mark the caller as leaving the world (cross-authority switch / logout) |
| `instance_enter` | `{mode, params?}` | `{ok, match_id, mode}` | Cross-authority switch world → instance: creates the instance match; the client leaves the world presence and joins it |
| `instance_exit` | `{}` | `{ok}` | Cross-authority switch instance → world: the client leaves the instance and re-enters the world (`enter_world`) |
| `appearance_get` | `{user_id, species}` | `{ok, species, params}` or `{ok:false, err:"not_found"}` | Fetch a player's full appearance params for replication (by entity `id` + `species`; bounded cache; see §8 appearance) |

- `pvp` optional parameters: `max_players` (default 4) · `respawn_seconds` (default 3) · `score_target` (first to reach wins, 0 = unlimited) · `time_limit` (tick limit, 0 = unlimited) · `arena{w,h}` (coordinate bounds).
- `aoi` (optional, **AOI targeted broadcast**): `{ enabled?, cell, radius, hysteresis?, maxRadius? }` (units match `arena` = pixels; `radius` is clamped by `maxRadius`, and the **lower bound = engagement distance** (the maximum ability `range` in the `RANGE`/`combat` projection, preventing "invisible attackers"); providing it enables it, `enabled:false` explicitly disables it). **Disabled by default**; recommended to enable at the world layer.
- `combat` (optional, generic combat projection): `{ abilities: { <abilityId>: { cooldownMs, range, multiplier, cooldownGroup?, cost? } }, attack, defense, resources? }`. `cost = {resource, amount}` and `resources = { <id>: {max, regenPerSec?} }` make the server authoritative for resource spend/regen (CR-02): insufficient cost rejects the cast; resources regen per tick; current values are in `snapshot.players[].res`. **Damage semantics (CR-08):** `damage = max(1, floor(attack * multiplier - defense))`, identical to the client offline formula (`Combat.damage`). Legacy `abilities[id].damage` (absolute) is still accepted when `multiplier` is absent. Produced by content (rules/values are in the content layer); the server uses it to **authoritatively resolve** `cast`, contains no gameplay values and no per-content code. Clients can use `NetClient.build_combat_data()` to generate it from Defs.
- Teams: the client's join metadata carries `team`; without `team` it is FFA (all hostile).

### Match state

| Direction | Message | Field | Implementation |
|---|---|---|---|
| C→S | `input` | `dx`, `dy` (-1..1, server clamps) | ✅ |
| C→S | `attack` | — (fixed-constant extra mechanic; compatibility path when there is no `combat` projection) | ✅ |
| C→S | `cast` | `ability` (content ability id), `target` (optional entity id) | ✅ |
| S→C | `snapshot` | `proto`, `mode`, `tick`, `players{}`, (`pve` includes `enemy{}`); when AOI is enabled it **only includes entities within that player's area of interest radius**, along with `enter[]`/`leave[]` deltas | ✅ |
| S→C | `welcome` | `id`, `mode`, `tick`, `proto`, `arena` | ✅ |

**AOI (area-of-interest culling)**: when `aoi.enabled`, the server broadcasts to each presence only the entities within its radius (grid `aoi.lua`) and provides `enter`/`leave` deltas; **hysteresis**: entering requires `d ≤ radius`, while **retention** allows `d ≤ radius×(1+hysteresis)` (eliminating boundary jitter); **only broadcast is culled, not simulation**, and interacting parties (attacker/target/victim) always reach each other.

**Join rejection**: match full → `match_full`; version mismatch → `proto_mismatch` (Nakama join returns failure; the client emits the `match_join_failed(reason)` signal).

**Version negotiation (handshake)**: `proto` is the match protocol version (currently `1`). The client's `join metadata` carries `{"proto": <n>}`; the server rejects on mismatch in `match_join_attempt` (`proto_mismatch`) and allows it when not provided (compatible with old clients). Both `welcome` / `snapshot` return `proto`; the client errors if validation does not match.

Entity fields: `id` · `x` · `y` · `hp` · `max_hp` · `dead`; `pvp` additionally includes `team` · `kills` · `deaths` · `respawn_at`.
**Server authority**: HP/damage/respawn/win-loss are resolved by the server (online-and-instances.md):
- `pve`: `attack` hits enemies within range.
- `pvp`: `attack` hits the nearest hostile player within range (same `team` takes no damage); death respawns after `respawn_seconds`.

### World state

The persistent world is a **singleton authoritative match** (label `g3/world`, low tick rate, **no combat simulation**), served by `G_Server/nakama/modules/world.lua`; presence + AOI broadcast reuse `aoi.lua` (AOI **enabled by default** at the world layer).

| Direction | Message | Field | Implementation |
|---|---|---|---|
| S→C | `world_welcome` | `id`, `world_id`, `mode="world"`, `tick`, `tick_rate`, `proto`, `spawn{poi_id,x,y,altitude}`, `revision` | ✅ sent per presence on join |
| S→C | `world_snapshot` | `proto`, `world_id`, `tick`, `players{}`, `enter[]`, `leave[]` | ✅ AOI-filtered presence broadcast |

- Joining uses the same `create/join` mechanics as matches (`enter_world` returns `match_id`; the client joins with `{proto, species, akey}`).
- **Spawn is server-authoritative**: derived from the persisted `player.location` (cloud save) when present, otherwise the content `world.defaultEntry` / `poi`; the client must adopt `world_welcome.spawn` rather than computing it locally.
- **Reconnect**: socket `closed` → backoff → re-auth → rejoin the world match (same rule as matches).

**Appearance replication**: each entity in `world_snapshot.players{}` (and in match `snapshot.players{}`) carries `species` + `akey` (`<species>@<hash>`, a client-computed cache key over `player.appearance.params`, [content-schema.md](content-schema.md) §28). Clients cache params by `akey` and, on a cache miss, fetch via `appearance_get` keyed by the entity `id` (user id) + `species`; the server never sends full params per tick. On `save_write` the server normalizes params against the optional `appearance_schema` projection ([save-schema.md](save-schema.md) §10).