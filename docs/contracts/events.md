# Contract: Event Catalog

- Status: Active
- Updated: 2026-09-17

> The **authoritative list** of typed events (name + payload + trigger). Logical events are synced and ordered; presentation events are asynchronous and lossy; content may only **subscribe to whitelisted events**.
> The `signal` set in `App/autoload/event_bus.gd` must match the "Implemented" table + the "Deferred" list (CI validation).

## Implemented

### Content / Loading

| Event | payload | Trigger |
|---|---|---|
| `content_loaded` | `{}` | Def DB ready |
| `pack_loaded` | `{ packId }` | A single content pack loaded successfully |
| `pack_rejected` | `{ packId, reason }` | Content pack rejected (invalid manifest / missing id / capability not satisfied) |

### Scene

| Event | payload | Trigger |
|---|---|---|
| `scene_loaded` | `{ sceneId }` | Scene loading complete |
| `scene_enter` | `{ sceneId }` | Enter scene |
| `scene_exit` | `{ sceneId }` | Leave scene |

### World / Player

| Event | payload | Trigger |
|---|---|---|
| `player_landed` | `{ poiId, entryPointId }` | Flight landing |
| `player_spawned` | `{ entityId }` | Player entity enters the world/scene |

### Entity (`entity.spawn` / `entity.query`)

| Event | payload | Trigger |
|---|---|---|
| `entity_spawned` | `{ entityId, defId }` | Runtime entity created |
| `entity_despawned` | `{ entityId }` | Runtime entity removed |

### Region (`region.trigger`)

| Event | payload | Trigger |
|---|---|---|
| `region_enter` | `{ regionId }` | Enter a `RegionTrigger` region |
| `region_exit` | `{ regionId }` | Leave a `RegionTrigger` region |

### Interaction

| Event | payload | Trigger |
|---|---|---|
| `interact_done` | `{ entityId, interactionId }` | Complete an interaction |

### Quest / Dialog (`quest.core` / `dialog.core`)

| Event | payload | Trigger |
|---|---|---|
| `quest_state_changed` | `{ questId, state }` | Quest state changed (`active`/`ready`/`done`) |
| `dialog_started` | `{ dialogId }` | Dialog started |
| `dialog_ended` | `{ dialogId }` | Dialog ended |

### Combat

| Event | payload | Trigger |
|---|---|---|
| `ability_cast` | `{ sourceId, abilityId }` | Cast an ability |
| `entity_damaged` | `{ targetId, amount, sourceId }` | Damage dealt |
| `entity_died` | `{ entityId, killerId? }` | Entity died |
| `effect_applied` | `{ targetId, effectId }` | Effect applied |

### Inventory / Progression / Loot / Vendor (`action.core` / `loot.core` / `progression.tree` / `vendor.core`)

| Event | payload | Trigger |
|---|---|---|
| `item_added` | `{ itemId, count }` | Item added to the player bag |
| `ability_learned` | `{ abilityId, source }` | Ability learned (`learned`/`item`/`granted`/`quest`/`progression`) |
| `progression_changed` | `{ treeId }` | Talent/skill tree allocation or reset |
| `loot_rolled` | `{ tableId, drops }` | A loot table was evaluated |
| `loot_picked` | `{ itemId }` | Ground item picked up (`action.pick_up`) |
| `vendor_transaction` | `{ action, itemId, amount }` | Buy/sell completed |
| `teleported` | `{ worldId }` | Teleport action applied (`action.teleport`) |

### Save / Network

| Event | payload | Trigger |
|---|---|---|
| `save_done` | `{}` | Save complete |
| `load_done` | `{}` | Load complete |
| `net_connected` | `{}` | Network connection established |
| `net_disconnected` | `{}` | Network disconnected (socket closed) |

### Instance / cross-authority (`scene.switch`)

| Event | payload | Trigger |
|---|---|---|
| `instance_enter` | `{ instanceId }` | Crossed from world into an instance (authority switched) |
| `instance_exit` | `{ instanceId }` | Left an instance back to the world |

### Localization

| Event | payload | Trigger |
|---|---|---|
| `locale_changed` | `{ locale }` | Language switch complete |

### Config / Quality (`config.flag` / `quality.tier`)

| Event | payload | Trigger |
|---|---|---|
| `flag_changed` | `{ flagKey, value }` | Feature flag changed |
| `quality_changed` | `{ tier }` | Quality tier changed |

## Deferred (defined but not triggered, or design reserved)

(None — all defined events have a trigger point)

Design reserved (not yet defined): `instance_enter` · `instance_exit`

## Non-EventBus Signals

`NetClient.chat_message(username, content)`: world chat message (social-purpose only, does not enter the event bus).

## Rules

- payload is **versioned** along with the Content Schema; fields may only be added.
- New events must be registered in this catalog with their trigger timing; unimplemented items go under "Deferred".