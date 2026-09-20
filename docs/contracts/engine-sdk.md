# Contract: Engine SDK

- Status: Active
- Updated: 2026-09-20

> The **sole interface** the engine team provides to the content layer. Changes follow the compatibility policy in architecture overview.
> **Scope**: pre-research implements only "enabled capabilities"; the rest are marked "deferred". See pre-research scope.

## 1. Version

- The SDK uses SemVer; only MAJOR changes are breaking.
- **Capabilities carry their own version** (`MAJOR.MINOR`). Manifest declares: `"<name>"` (any version) or `"<name>@>=X.Y"` (lower bound).

## 2. Capability Registry (Sole Authority)

### 2.1 Enabled (implemented · can be depended on by content)

| Capability | Version | Implementation | Notes |
|---|---|---|---|
| `world.stream` | 1.0 | `engine/runtime/world_stream.gd` | Chunked streaming load + unload; offline tile rendering |
| `world.coords` | 1.0 | `WorldStream.world_to_px/px_to_world` | World meters ↔ pixel conversion (PPU) |
| `scene.switch` | 1.0 | `autoload/scene_manager.gd` | Scene switching + fade in/out |
| `net.client` | 1.0 | `autoload/net_client.gd` | Nakama client (connect/match/chat/ranking/party/cloud save) |
| `world.presence` | 1.0 | `autoload/net_client.gd` + `G_Server/nakama/modules/world.lua` | Persistent-world entry/presence: `enter_world` → authoritative spawn, AOI snapshot replication, reconnect |
| `stats.core` | 1.0 | `autoload/combat.gd` | Stat and equipment modifiers |
| `effect.apply` | 1.0 | `Combat.damage/apply_damage` | Damage/effect resolution |
| `effect.aura` | 1.0 | `autoload/combat.gd` | Effect/aura runtime: `statModifiers` (flat/percent), `maxStacks`, `stackMode refresh/stack`, `durationMs` expiry |
| `camera.control` | 1.0 | `engine/runtime/camera_follow.gd` | Camera follow / rotation / distance (zoom) / collision handling / shake |
| `input.action` | 1.0 | `CharacterController` + `FlightPlayer` + Godot Input actions | Action-based input (keyboard `ui_*` / `move_*` + touch joystick `touch_dir`) |
| `movement.character` | 1.0 | `engine/runtime/character_controller.gd` | Character movement (walk/sprint/jump/gravity/ground check/climb/fly/mount), Def-driven |
| `appearance.apply` | 1.0 | `autoload/appearance.gd` | Parameterized appearance: normalize/validate `species.params` and apply to a presentation node; key = `species@hash` |
| `physics.query` | 1.0 | `engine/runtime/physics_query.gd` | Collision/area/raycast/ground queries |
| `region.trigger` | 1.0 | `engine/runtime/region_trigger.gd` | Region enter/exit (Area2D) |
| `terrain.tiled` | 1.0 | `engine/runtime/terrain_map.gd` | Tiled `.tmj` terrain: `ground`/`detail` via Godot `TileSet`+`TileMapLayer`; `height`/`pathing` queries; `pathing` blockers (static collision) |
| `sprite.atlas` | 1.0 | `engine/runtime/sprite_atlas.gd` | Atlas PNG + region table → regions/frames as `Texture2D` / `SpriteFrames` (content-schema §29) |
| `gm.command` | 1.0 | `autoload/gm.gd` | Dev/QA command registry (`register`/`execute`/`commands`); **disabled in release builds** |
| `gm.panel` | 1.0 | `autoload/gm.gd` | Dev-only command panel (`toggle_panel`); never built when disabled |
| `path.find` | 1.0 | `engine/runtime/path_finder.gd` | Height-aware grid A* (`navgrid` bitmap + elevation; steep walls/canyons) |
| `save.slot` | 1.0 | `autoload/save_service.gd` | Progress read/write (local + Nakama cloud save) |
| `script.gdscript` | 1.0 | content `*.gd` | GDScript content scripts |
| `event.bus` | 1.0 | `autoload/event_bus.gd` | Typed events (catalog see events.md) |
| `localization.key` | 1.0 | `autoload/localization.gd` | Localization key resolution + locale switching (fallback current→fallback→key) |
| `quest.core` | 1.0 | `autoload/quests.gd` | Quest state machine (inactive→active→ready→done); event `quest_state_changed` |
| `dialog.core` | 1.0 | `autoload/dialog.gd` | Dialog nodes/choice actions; events `dialog_started` / `dialog_ended` |
| `ui.core` | 1.0 | `autoload/ui.gd` | Generic HUD mounting (`UI.hud(parent)`) |
| `config.flag` | 1.0 | `autoload/config.gd` | Feature flags (runtime/env vars/ProjectSettings); event `flag_changed` |
| `conditions.core` | 1.0 | `autoload/conditions.gd` | Generic condition evaluation (all/any/not/flag/quest/item/stat/level/kv; see content-schema §27) |
| `spatial.query` | 1.0 | `autoload/spatial.gd` | Uniform grid spatial index (AOI / targeting acceleration) |
| `telemetry.event` | 1.0 | `autoload/telemetry.gd` | Structured telemetry event logging to disk (JSONL, bounded) |
| `world.seed` | 1.0 | `autoload/rng.gd` | Runtime deterministic randomness (derive stable substreams from world `seed` root) |
| `quality.tier` | 1.0 | `autoload/quality.gd` | Quality tiers low/medium/high (auto detection); event `quality_changed` |
| `targeting.query` | 1.0 | `autoload/targeting.gd` | Pure targeting queries (nearest / in_radius / best) |
| `interaction.core` | 1.0 | `autoload/interaction.gd` | Interaction range checks and action execution (teleport/give/dialog/quest) |
| `audio.play` | 1.0 | `autoload/audio_service.gd` | `sound_event` event playback (resource pool/cooldown/max instances) |
| `audio.music` | 1.0 | `autoload/audio_service.gd` | Music play/stop (`play_music` / `stop_music`) |
| `audio.mixer` | 1.0 | `autoload/audio_service.gd` | Bus volumes (Master/SFX/Music/UI) |
| `fx.play` | 1.0 | `autoload/fx.gd` | Instantiate presentation nodes by `fx` Def (presentation only) |
| `render.light2d` | 1.0 | `autoload/render.gd` | Apply ambient light by `lighting_profile` (CanvasModulate) |
| `render.post` | 1.0 | `autoload/render.gd` | Post-processing (glow/adjustment) configuration |
| `entity.spawn` | 1.0 | `autoload/entities.gd` | Runtime entity registration (Def→runtime state; event `entity_spawned`) |
| `entity.query` | 1.0 | `autoload/entities.gd` | Entity queries (by def/type/alive/in range; `entity_despawned`) |
| `combat.timeline` | 1.0 | `autoload/combat.gd` | Ability timeline: castMs/cooldown/cooldown group/range validation + effect resolution |
| `ability.grant` | 1.0 | `autoload/player.gd` | Abilities granted by equipped items (`item.grantsAbility`), active while equipped |

> Must match the `enabled` set in `App/engine/sdk/capabilities.json` (CI validation).

### 2.2 Deferred (design reserved, not implemented)

(none)

> Principle: **rules go into content**; the engine only provides primitives (stats/effects/damage pipeline/network). Gameplay systems (quests/dialog/combat rules) live in the content layer and `autoload`.

## 3. API (content-visible surface = sole interface)

> Content accesses the engine only through the following singletons/runtime (`App/autoload/*`, `App/engine/runtime/*`); missing entries return `null` (NotFound).
> Async functions require `await`; all run on the **main thread**.

```text
# Content and data
ContentRuntime.get_def(type, id) -> Variant        # Def query (missing: null)
ContentRuntime.def_db / load_order / schema_version
ContentRuntime.pack_dir(pack_id) / resolve_def_path(def, rel) / first_def_id(type) / reload()
# Events (catalog see events.md)
EventBus.<signal>.connect(handler)                 # Subscribe
# Scene
SceneManager.switch_to(path, entry := "") -> void
SceneManager.to_menu() ; go(path, entry?) ; consume_pending()
SceneManager.to_world(entry := "", poi_id := "") -> void   # Return to world (content does not reference main/ paths)
# UI
UI.hud(parent) -> Node                             # Mount generic HUD (content does not reference res://ui)
# World (engine generic scene engine/runtime/world/world_scene.tscn, driven by world Def)
WorldStream.configure(world_def) ; world_to_px(m) ; px_to_world(px) ; chunks
FlightPlayer.speed_px / auto / touch_dir             # Flight movement (auto/touch/keyboard)
# Character movement (movement.character; params from character Def, meters/second, converted to pixels via ppu)
CharacterController.configure(def) ; current_speed() ; params()
CharacterController.intent / sprint / climb / climbable / flying / mounted / altitude
CharacterController.state / on_ground / jumps_left ; state_changed/jumped/landed signals
CharacterController.jump_velocity(height, gravity) ; gravity_step(vy, gravity, max_fall, dt) ; approach(v, t, d)
# Generic playable character (engine generic composite node, includes CharacterController + FollowCamera; Def id set by content)
PlayerCharacter.character_def / camera_def / species_def ; set_intent(v) ; set_appearance(params) ; body / camera
# Appearance (appearance.apply; parameterized, content-declared params)
Appearance.normalize(species_def, params) -> Dictionary ; Appearance.key(species_id, params) -> String
Appearance.apply(species_def, params) -> Node ; Appearance.params(akey) ; Appearance.remember(akey, species, params)
# Physics queries (physics.query; read-only, no space returns empty)
PhysicsQuery.overlap_circle(node, center, radius, mask, exclude) ; raycast(node, from, to, mask, exclude) ; ground_check(body)
# Pathfinding (path.find; height-aware grid A*, data from navgrid Def)
PathFinder.find_path(navgrid_def, from_cell, to_cell, opts) ; search(walk, alt, cols, rows, from, to, opts) ; cell_to_m(cell, cellM)
# Region (region.trigger)
RegionTrigger.region_id / body_mask ; entered/exited signals   # Also emits EventBus.region_enter/region_exit
# Terrain (terrain.tiled; Tiled .tmj, read-only)
TerrainMap.terrain_path / load_tiled(path) ; size_cells() ; has_pathing()
TerrainMap.walkable_cell(cx, cy) / walkable_px(pos) ; height_cell(cx, cy)   # pathing=0 阻挡；越界阻挡
# Sprite atlas (sprite.atlas; atlas PNG + region table)
SpriteAtlas.from_id(id) ; configure(def) ; is_loaded() ; texture()
SpriteAtlas.region(name) / has_region(name) ; frame_textures(anim) ; sprite_frames()
# GM (gm.command / gm.panel; dev-only, disabled in release)
Gm.is_enabled() / set_enabled(v) ; register(name, handler, meta?) ; execute(name, args) ; commands()  # also NetClient.upload_crash_report()
# Crash reporting (trial stub; upload requires Privacy consent)
CrashGuard.build_report(reason) ; store_report(reason) ; has_report() ; read_report() ; clear_report()
NetClient.upload_crash_report()   # -> server `crash_report` (g3/crash_<user>)
Gm.toggle_panel(parent)   # dev-only; null when disabled
# Camera (camera.control)
FollowCamera.target / smooth / rotate_with_target / free_rotation / set_zoom_level(z) / zoom_by(d) / rotate_by(dyaw) / shake(strength)
# Touch input (ui/touch_controls.tscn)
TouchControls.move(dir) / land_pressed ; set_land_available(v) ; TouchJoystick.calc(center, point, radius)
# Combat
Combat.stats(def_id) ; item_stats(item_id) ; ability_cost(ability_id)   # ability.cost {resource, amount}
Combat.damage(src, tgt, effect_id) ; apply_damage(rt, dmg, source_id)
Combat.in_radius(list, origin, radius, exclude_id) ; loot_roll(stats_def) ; xp_reward(stats_def)
# Player / quests / dialog
Player.setup(stats_id) ; derived() ; max_hp() ; add_item(id) ; add_xp(n)
Player.resource(id) / resource_max(id) / can_pay(id, n) / spend(id, n) / regen(dt) / resources_state()
Player.export_state() / import_state(d)
Quests.state(id) ; start(id) ; progress(kind, target, n) ; turn_in(id)
Quests.export_state() / import_state(d)
Dialog.run(dialog_id)
# Save
SaveService.get_value/set_value ; snapshot/apply
SaveService.save_local/load_local ; save_cloud/load_cloud ; autosave
# Network (net.client)
NetClient.connect_to() ; join() ; send_input(dx, dy) ; attack()
NetClient.enter_world(prefer_poi?) ; leave_world() ; appearance_get(user_id, species)  # 持久世界房间 + 服务端权威出生点
NetClient.enter_instance(mode, params, location) ; exit_instance()   # 跨权威切换（世界↔副本）；事件 instance_enter/exit
NetClient.submit_score/top_scores ; add_friend_by_username/list_friends ; create_party
NetClient.cloud_save_write/cloud_save_read  # 经 RPC save_write/save_read；CAS + server wins（save-schema.md §9）
# Audio (audio.play/music/mixer)
AudioService.play(event_id, params?) ; play_music(event_id) ; stop_music()
AudioService.set_bus_volume(bus, linear) ; bus_volume(bus)
# FX / rendering (fx.play / render.light2d / render.post)
Fx.play(fx_id, parent, position?) ; Render.apply_lighting(profile_id) ; Render.set_ambient(color) ; Render.set_post(post)
# Localization (localization.key)
Localization.text(key, params?) ; has(key) ; locale() ; set_locale(code) ; detect_locale()
# Config flags (config.flag)
Config.flag(key, default?) ; set_flag(key, value) ; flags()
Config.setting(key, default?) ; require(key) ; validate(required[]) ; set_policy(key, severity) ; policy(key)
# Conditions (conditions.core; expressions see content-schema §27)
Conditions.eval(spec, ctx?)
# Spatial index (spatial.query; generic acceleration)
Spatial.configure(cell) ; insert(id, pos) ; move(id, pos) ; remove(id) ; has(id) ; position(id) ; count() ; clear()
Spatial.query_circle(center, radius, exclude_id?) ; query_rect(rect, exclude_id?) ; nearest(center, radius?, exclude_id?)
# Telemetry (telemetry.event)
Telemetry.event(name, props?) ; progress(step, props?) ; lines() ; clear() ; flush() ; export_events(limit?)
# Telemetry upload (PS-04; requires Privacy.consent() first)
NetClient.upload_telemetry(limit?) ; telemetry_report() -> Dictionary
# Deterministic randomness (world.seed)
Rng.root_seed() ; seed_for(key) ; rng(key) ; randf(key) ; randi(key, from, to)
# Quality tiers (quality.tier)
Quality.tier() ; set_tier(t) ; tier_settings()
# Targeting (targeting.query)
Targeting.nearest(candidates, origin, radius?, exclude_id?) ; in_radius(...) ; best(candidates, origin, score)
# Interaction (interaction.core)
Interaction.range_of(def_id) ; in_range(pos, target, def_id) ; run(def_id, actor?)
# Entities (entity.spawn / entity.query)
Entities.spawn(def_id, pos?, props?) ; query(filters) ; entity(id) ; despawn(id) ; all() ; count() ; clear()
# Effects / auras (effect.aura)
Combat.apply_effect(rt, effect_id, source_id?) ; auras(rt) ; aura_stacks(rt, effect_id)
Combat.stat_modifiers(rt) ; effective_stats(rt, base_stats)
# Granted abilities (ability.grant)
Player.active_abilities()
# Ability timeline (combat.timeline)
Combat.begin_cast(ability_id, caster_rt, target_rt) ; cooldown_ready(caster_id, ability_id)
# Lifecycle dispatch (engine-sdk §5; backed by Hooks)
Lifecycle.dispatch(method, args?)     # Calls the same-named method in the g3_lifecycle group
# Hooks (hooks.md; optional for content)
Hooks.register(domain, hook, cb, priority?) ; emit(domain, hook, args?) ; is_known(domain, hook) ; count(domain, hook)
# Privacy (PS-06; uploads require consent first)
Privacy.consent() ; set_consent(v) ; delete_local_data()
# Logging / metrics (observability.md §7; optional for content)
Log.info(cat, msg, fields?) ; warn ; error ; debug ; trace ; fatal ; set_level(cat, level) ; is_enabled(level, cat)
Metrics.inc(name, n?) ; counter(name) ; observe(name, ms) ; timer_stats(name) ; snapshot() ; flush()
```

> Adding content-visible APIs is an **engine capability change**: follow the §9 capability request process, and sync §2 and `capabilities.json`.

## 4. Error Codes

| Code | Meaning |
|---|---|
| `NotFound` | Def/entity/scene does not exist |
| `InvalidArg` | Invalid argument |
| `NotLoaded` | Dependency not loaded |
| `Rejected` | Missing capability or version not satisfied |
| `NoPath` | No path found |
| `Timeout` | Async timeout |
| `SandboxDenied` | Insufficient sandbox permission |
| `RuntimeError` | Other runtime error |

## 5. Lifecycle Callbacks (script, design reserved)

`on_content_loaded` · `on_scene_enter` · `on_scene_exit` · `on_region_enter` · `on_save` · `on_load` · `on_tick` (optional)

> **Current status**: implemented by `Hooks` (`autoload/hooks.gd`, domain/priority/index dispatch) + the `Lifecycle` adapter layer (`g3_lifecycle` group). Content scripts can join the `g3_lifecycle` group and implement callbacks by name, or register directly via `Hooks.register`. See [hooks.md](hooks.md) for the catalog.

## 6. Script Boundaries

- Trusted content: GDScript, hot-reloadable.
- Untrusted content (future UGC): WASM sandbox + quotas + whitelisted APIs.
- Scripts run only on the main thread; blocking and long tasks are prohibited.

## 7. Error Conventions

- Explicit `Result`/`Error`, never silent.
- Content script exceptions are caught and degraded, without affecting the engine or other content.

## 8. Prohibitions

- Content accessing engine internals or platform APIs.
- The engine opening dedicated APIs for a single content/gameplay.
- Any interface "used by only one content pack" is a signal of design failure.

## 9. Capability Request Process

World team raises a requirement → engine team evaluates generality → implement as a generic capability (bump version) → write into the registry and Schema → publish and update contract tests.

## 10. Capability Matrix Output

`capabilities.json` is generated at build time, read by the content validator and CI; see [content-tooling.md](content-tooling.md) §3 for the format.