# Contract: Data and Save Schema

- Status: Active
- Updated: 2026-09-20

> Covers two modes: **offline (local authority)** and **logged-in (cloud save)**. See internal platform doc for details.

## 1. Storage

- **Local**: `user://save.json` (offline authority, continue playing when disconnected). The file is an **envelope**: `{ "snapshot": <snapshot>, "hash": "sha256:…" }`, where `hash` is the SHA-256 of the snapshot's canonical JSON.
- **Backup slot**: `user://save.bak` (the previous usable main save, rotated before writing); fall back to it when the main save is corrupted.
- **Cloud**: Nakama object storage (`collection=g3`, `key=progress`; authority/backup when logged in) — stores the **bare snapshot** (without the envelope).
- **outbox**: `user://save.outbox.json` (queue of snapshots pending upload, idempotency key + ordered replay).
- Snapshots carry `version`; the server side is hosted by Nakama (see internal platform doc).

## 2. Snapshot structure

```json
{
  "version": 1,
  "player": {
    "stats_id": "stats_player",
    "level": 1, "xp": 0, "hp": 100,
    "inventory": ["item_steel_sword"],
    "equipped": { "weapon": "", "armor": "", "artifact": "" }
  },
  "quests": {
    "active": { "quest_grunt": [0] },
    "pending": {},
    "done": {}
  },
  "kv": { "any_key": "value" }
}
```

| Field | Description |
|---|---|
| `version` | Save version (monotonically increasing, currently `1`) |
| `player.stats_id` | Base stats Def id (`stats`) |
| `player.level/xp/hp` | Level / XP / current HP |
| `player.inventory` | List of item ids |
| `player.equipped` | Slot → item id; slot ids are declared by content `equipment.slots[]` (§25 content-schema) |
| `quests.active` | Quest id → array of progress per objective |
| `quests.pending` | Set of quests pending turn-in |
| `quests.done` | Set of completed quests |
| `kv` | Generic KV (content-layer cross-session persistence, `SaveService.get_value/set_value`); backward compatible, may be absent in old saves |

## 3. Modes and authority

| Mode | Location | Authority |
|---|---|---|
| Offline | `user://save.json` | Local |
| Logged-in | Nakama `g3/progress` | Cloud (local is a cache) |

## 4. Read/write timing (implementation)

- **Startup**: `SaveService.restore_on_boot()` reads the local save and restores it.
- **Level up / quest turn-in**: `SaveService.autosave()` writes locally; when online, also `save_cloud()`.
- **Login / reconnect**: the `net_connected` event triggers `flush_outbox()`; cloud uploads go through RPC `save_write` with the current `revision` (CAS). On a revision conflict the server snapshot wins (see §9), so a stale local save cannot silently overwrite cloud/administrative changes.

## 5. Versions and conflicts

- `version` is monotonically increasing; loading an old save upgrades it via the **migration chain** (currently only `1`).
- Conflict granularity: **whole-save**, resolved by **server-authoritative CAS** (`save_write` + `revision`, §9): stale writers are rejected and adopt the server snapshot. No field-level merge.

## 6. Integrity

- **Atomic write**: write `user://save.tmp` → rotate the backup slot → `rename` to replace the main save; an interruption does not produce a half-written main save.
- **Hash validation**: on read, validate the snapshot against the envelope's `hash`; a mismatch is treated as corruption.
- **Corruption fallback**: when the main save is missing/corrupted (parse failure or hash mismatch), fall back to `save.bak`; only if the backup is also unavailable, fall back to defaults without blocking startup.
- **Migration chain**: migrate level by level by `version` (currently only `1`); migration and default-field backfilling are **idempotent** and can be repeated.

## 7. Implemented / Deferred

**Implemented** (P23): atomic write (temp file → rename), hash validation, backup-slot fallback, migration chain (idempotent), outbox (idempotency key + ordered replay, `flush_outbox()`).

**Deferred (design reserved)**:

- Local **SQLite** multi-tables (`settings` / `profile` / `save_slot` / `world_state` / `cache`).
- Field-level conflict merge (currently whole-save LWW).
- Account data encryption / export / deletion (privacy compliance).

## 8. Management writes (control plane)

> Applies to the operator panel (`G_Admin`). See [admin-panel.md](admin-panel.md), ADR-0004.

- Management writes to `g3/progress` MUST go through the **server-authoritative path** (BFF → Nakama), never a raw client upload.
- Concurrency: **optimistic** using the Nakama storage object `version` as a compare-and-swap token (`StorageWriteRetry`). A write with a stale version is rejected (`ErrStorageRejectedVersion`) and retried/refused.
- Online targets follow `onlinePolicy` (`live` push / `block` / `kick_then_write` / `at_rest`); see [admin-panel.md](admin-panel.md) §5.
- Because the client-authored save is whole-save LWW, a management edit can be overwritten by a later client save; the authoritative path (or a pushed state update) is required for durable edits.
- All management writes are audited (`admin` schema).

## 9. Server-authoritative cloud write

> Supersedes direct client writes to `g3/progress` (see [protocol.md](protocol.md) §8 RPCs).

- Cloud writes MUST go through RPC **`save_write`** (`{snapshot, expected_revision}`); the storage object is written with **`permission_write=0`** so clients cannot write it directly.
- **`save_read`** returns `{exists, revision, snapshot}`; `revision` is the Nakama storage object version and acts as the CAS token.
- On revision mismatch the server rejects the write and returns the authoritative snapshot (`err="conflict"`); the client **adopts the server snapshot (server wins)**. This makes administrative corrections durable against stale client writes.
- Local save (`user://save.json` + outbox) is a cache/offline buffer; the cloud remains authoritative when online.