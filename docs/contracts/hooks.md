# Contract: Platform Hooks Catalog (Hooks)

- Status: Active
- Updated: 2026-09-17

> Purpose: The content team **registers hooks** from this catalog to extend behavior; the platform **does not modify the content layer**, only provides interfaces and documentation.
> Difference from events: **hook = ordered, interceptable, synchronous**; **event = notification, non-interceptable, may be asynchronous** (see [events.md](events.md)).
> Implementation: `autoload/hooks.gd` (`Hooks` registry: priority + dispatch indexed by `(domain, hook)`); `autoload/lifecycle.gd` is an **adapter layer** — bridges `EventBus` events into hooks and dispatches them to methods of the same name in the `g3_lifecycle` group.
> **All listed domains are already wired up for triggering** (world/scene/region/entity/combat/quest/dialog/net/save/tick).
> Ownership: platform implementation (`G_Engine`); this catalog is a **shared contract**.

## 1. Registration model (implemented: autoload/hooks.gd)

```text
Hooks.register(domain, hook, callable, priority := 0)   # register (platform API)
Hooks.emit(domain, hook, args...)                        # dispatch (platform internal)
```

- Multiple subscribers to the same `(domain, hook)` are called in ascending `priority` order; for equal priority, in registration order.
- Indexed by hook: **subscribers that do not implement that hook are not traversed**.
- Hooks execute synchronously; they should be short-running, blocking and long tasks are forbidden.

## 2. Domains and hooks (planned)

| Domain | Hook | Signature | Timing |
|---|---|---|---|
| `world` | `on_content_loaded` | `()` | Def DB ready |
| `scene` | `on_scene_enter` / `on_scene_exit` | `(scene_id)` | Scene enter/exit |
| `region` | `on_region_enter` | `(region_id)` | Region enter |
| `entity` | `on_entity_spawned` / `on_entity_died` | `(entity_id[, killer_id])` | Entity spawn / death |
| `combat` | `on_ability_cast` / `on_entity_damaged` | `(source_id, ability_id)` / `(target_id, amount, source_id)` | Combat resolution |
| `quest` | `on_quest_state_changed` | `(quest_id, state)` | Quest state change |
| `dialog` | `on_dialog_started` / `on_dialog_ended` | `(dialog_id)` | Dialog start/end |
| `net` | `on_net_connected` / `on_net_disconnected` | `()` | Network connection |
| `save` | `on_save` / `on_load` | `()` | Save read/write |
| `tick` | `on_tick` | `(dt)` | Every frame (**use with caution**) |

## 3. Boundaries

- Hooks only carry **generic engine timings**; gameplay rules still belong to the content layer (Def / scripts), and the platform does not define gameplay models.
- Do not add hooks that "only one content pack needs" (a design-failure signal; see [engine-sdk.md](engine-sdk.md) §8).