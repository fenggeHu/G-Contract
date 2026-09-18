# 契约：Engine SDK

- 状态：生效中
- 更新日期：2026-09-17

> 引擎组提供给内容层的**唯一接口**。变更遵循 平台内部文档 的兼容策略。
> **范围**：预研只实现「启用能力」，其余标「暂缓」。见 平台内部文档。

## 1. 版本

- SDK 用 SemVer；仅 MAJOR 破坏性变更。
- **能力自身带版本**（`MAJOR.MINOR`）。Manifest 声明：`"<name>"`（任意版本）或 `"<name>@>=X.Y"`（下限）。

## 2. 能力注册表（唯一权威）

### 2.1 启用（已实现·可被内容依赖）

| 能力 | 版本 | 实现 | 说明 |
|---|---|---|---|
| `world.stream` | 1.0 | `engine/runtime/world_stream.gd` | 分块流式加载 + 卸载；离线贴图渲染 |
| `world.coords` | 1.0 | `WorldStream.world_to_px/px_to_world` | 世界米 ↔ 像素换算（PPU） |
| `scene.switch` | 1.0 | `autoload/scene_manager.gd` | 场景切换 + 淡入淡出 |
| `net.client` | 1.0 | `autoload/net_client.gd` | Nakama 客户端（连接/对局/聊天/排行/组队/云存档） |
| `stats.core` | 1.0 | `autoload/combat.gd` | 属性与装备修正器 |
| `effect.apply` | 1.0 | `Combat.damage/apply_damage` | 伤害/效果结算 |
| `camera.control` | 1.0 | `engine/runtime/camera_follow.gd` | 相机跟随 / 旋转 / 距离(缩放) / 碰撞处理 / 震动 |
| `input.action` | 1.0 | `CharacterController` + `FlightPlayer` + Godot Input actions | 动作化输入（键盘 `ui_*` / `move_*` + 触屏摇杆 `touch_dir`） |
| `movement.character` | 1.0 | `engine/runtime/character_controller.gd` | 角色移动（步行/冲刺/跳跃/重力/地面检测/攀爬/飞行/坐骑），Def 驱动 |
| `physics.query` | 1.0 | `engine/runtime/physics_query.gd` | 碰撞/范围/射线/地面查询 |
| `region.trigger` | 1.0 | `engine/runtime/region_trigger.gd` | 区域进入/离开（Area2D） |
| `path.find` | 1.0 | `engine/runtime/path_finder.gd` | 高度感知网格 A*（`navgrid` 位图 + 高程；陡壁/峡谷） |
| `save.slot` | 1.0 | `autoload/save_service.gd` | 进度读写（本地 + Nakama 云存档） |
| `script.gdscript` | 1.0 | 内容 `*.gd` | GDScript 内容脚本 |
| `event.bus` | 1.0 | `autoload/event_bus.gd` | 类型化事件（目录见 events.md） |
| `localization.key` | 1.0 | `autoload/localization.gd` | 本地化 key 解析 + 语言切换（回退 current→fallback→key） |
| `quest.core` | 1.0 | `autoload/quests.gd` | 任务状态机（inactive→active→ready→done）；事件 `quest_state_changed` |
| `dialog.core` | 1.0 | `autoload/dialog.gd` | 对话节点/选项动作；事件 `dialog_started` / `dialog_ended` |
| `ui.core` | 1.0 | `autoload/ui.gd` | 通用 HUD 挂载（`UI.hud(parent)`） |
| `config.flag` | 1.0 | `autoload/config.gd` | 特性开关（运行时/环境变量/ProjectSettings）；事件 `flag_changed` |
| `conditions.core` | 1.0 | `autoload/conditions.gd` | 通用条件求值（all/any/not/flag/quest/item/stat/level/kv；见 content-schema §27） |
| `spatial.query` | 1.0 | `autoload/spatial.gd` | 均匀网格空间索引（AOI / 目标选取加速） |
| `telemetry.event` | 1.0 | `autoload/telemetry.gd` | 结构化遥测事件落盘（JSONL，有界） |
| `world.seed` | 1.0 | `autoload/rng.gd` | 运行时确定性随机（以 world `seed` 为根派生稳定子流） |
| `quality.tier` | 1.0 | `autoload/quality.gd` | 画质档位 low/medium/high（auto 探测）；事件 `quality_changed` |
| `targeting.query` | 1.0 | `autoload/targeting.gd` | 目标选取纯查询（nearest / in_radius / best） |
| `interaction.core` | 1.0 | `autoload/interaction.gd` | 交互距离判定与动作执行（teleport/give/dialog/quest） |
| `audio.play` | 1.0 | `autoload/audio_service.gd` | `sound_event` 事件播放（资源池/冷却/最大实例） |
| `audio.music` | 1.0 | `autoload/audio_service.gd` | 音乐播放/停止（`play_music` / `stop_music`） |
| `audio.mixer` | 1.0 | `autoload/audio_service.gd` | 总线音量（Master/SFX/Music/UI） |
| `fx.play` | 1.0 | `autoload/fx.gd` | 按 `fx` Def 实例化表现节点（纯表现） |
| `render.light2d` | 1.0 | `autoload/render.gd` | 按 `lighting_profile` 应用环境光（CanvasModulate） |
| `render.post` | 1.0 | `autoload/render.gd` | 后处理（glow/调整）配置 |
| `entity.spawn` | 1.0 | `autoload/entities.gd` | 运行时实体注册（Def→运行态；事件 `entity_spawned`） |
| `entity.query` | 1.0 | `autoload/entities.gd` | 实体查询（按 def/type/alive/范围内；`entity_despawned`） |
| `combat.timeline` | 1.0 | `autoload/combat.gd` | 技能时序：castMs/冷却/冷却组/射程校验 + 效果结算 |

> 与 `App/engine/sdk/capabilities.json` 的 `enabled` 集合必须一致（CI 校验）。

### 2.2 暂缓（设计保留，未实现）

（无）

> 原则：**规则进内容**；引擎只提供原语（属性/效果/伤害管线/网络）。玩法系统（任务/对话/战斗规则）在内容层与 `autoload`。

## 3. API（内容可见面 = 唯一接口）

> 内容只通过下列单例/运行时访问引擎（`App/autoload/*`、`App/engine/runtime/*`）；缺失项返回 `null`（NotFound）。
> 异步函数须 `await`；全部在**主线程**。

```text
# 内容与数据
ContentRuntime.get_def(type, id) -> Variant        # Def 查询（缺: null）
ContentRuntime.def_db / load_order / schema_version
ContentRuntime.pack_dir(pack_id) / resolve_def_path(def, rel) / first_def_id(type) / reload()
# 事件（目录见 events.md）
EventBus.<signal>.connect(handler)                 # 订阅
# 场景
SceneManager.switch_to(path, entry := "") -> void
SceneManager.to_menu() ; go(path, entry?) ; consume_pending()
SceneManager.to_world(entry := "", poi_id := "") -> void   # 返回世界（内容不引用 main/ 路径）
# UI
UI.hud(parent) -> Node                             # 挂载通用 HUD（内容不引用 res://ui）
# 世界（引擎通用场景 engine/runtime/world/world_scene.tscn，由 world Def 驱动）
WorldStream.configure(world_def) ; world_to_px(m) ; px_to_world(px) ; chunks
FlightPlayer.speed_px / auto / touch_dir             # 飞行移动（auto/touch/键盘）
# 角色移动（movement.character；参数来自 character Def，米/秒，经 ppu 换算像素）
CharacterController.configure(def) ; current_speed() ; params()
CharacterController.intent / sprint / climb / climbable / flying / mounted / altitude
CharacterController.state / on_ground / jumps_left ; state_changed/jumped/landed 信号
CharacterController.jump_velocity(height, gravity) ; gravity_step(vy, gravity, max_fall, dt) ; approach(v, t, d)
# 通用可玩角色（引擎通用复合节点，含 CharacterController + FollowCamera；Def id 由内容设置）
PlayerCharacter.character_def / camera_def ; set_intent(v) ; body / camera
# 物理查询（physics.query；只读，无空间返回空）
PhysicsQuery.overlap_circle(node, center, radius, mask, exclude) ; raycast(node, from, to, mask, exclude) ; ground_check(body)
# 寻路（path.find；高度感知网格 A*，数据来自 navgrid Def）
PathFinder.find_path(navgrid_def, from_cell, to_cell, opts) ; search(walk, alt, cols, rows, from, to, opts) ; cell_to_m(cell, cellM)
# 区域（region.trigger）
RegionTrigger.region_id / body_mask ; entered/exited 信号   # 同时发 EventBus.region_enter/region_exit
# 相机（camera.control）
FollowCamera.target / smooth / rotate_with_target / free_rotation / set_zoom_level(z) / zoom_by(d) / rotate_by(dyaw) / shake(strength)
# 触屏输入（ui/touch_controls.tscn）
TouchControls.move(dir) / land_pressed ; set_land_available(v) ; TouchJoystick.calc(center, point, radius)
# 战斗
Combat.stats(def_id) ; item_stats(item_id)
Combat.damage(src, tgt, effect_id) ; apply_damage(rt, dmg, source_id)
Combat.in_radius(list, origin, radius, exclude_id) ; loot_roll(stats_def) ; xp_reward(stats_def)
# 玩家 / 任务 / 对话
Player.setup(stats_id) ; derived() ; max_hp() ; add_item(id) ; add_xp(n)
Player.export_state() / import_state(d)
Quests.state(id) ; start(id) ; progress(kind, target, n) ; turn_in(id)
Quests.export_state() / import_state(d)
Dialog.run(dialog_id)
# 存档
SaveService.get_value/set_value ; snapshot/apply
SaveService.save_local/load_local ; save_cloud/load_cloud ; autosave
# 网络（net.client）
NetClient.connect_to() ; join() ; send_input(dx, dy) ; attack()
NetClient.submit_score/top_scores ; add_friend_by_username/list_friends ; create_party
NetClient.cloud_save_write/cloud_save_read
# 音频（audio.play/music/mixer）
AudioService.play(event_id, params?) ; play_music(event_id) ; stop_music()
AudioService.set_bus_volume(bus, linear) ; bus_volume(bus)
# 特效 / 渲染（fx.play / render.light2d / render.post）
Fx.play(fx_id, parent, position?) ; Render.apply_lighting(profile_id) ; Render.set_ambient(color) ; Render.set_post(post)
# 本地化（localization.key）
Localization.text(key, params?) ; has(key) ; locale() ; set_locale(code) ; detect_locale()
# 配置开关（config.flag）
Config.flag(key, default?) ; set_flag(key, value) ; flags()
Config.setting(key, default?) ; require(key) ; validate(required[]) ; set_policy(key, severity) ; policy(key)
# 条件（conditions.core；表达式见 content-schema §27）
Conditions.eval(spec, ctx?)
# 空间索引（spatial.query；通用加速）
Spatial.configure(cell) ; insert(id, pos) ; move(id, pos) ; remove(id) ; has(id) ; position(id) ; count() ; clear()
Spatial.query_circle(center, radius, exclude_id?) ; query_rect(rect, exclude_id?) ; nearest(center, radius?, exclude_id?)
# 遥测（telemetry.event）
Telemetry.event(name, props?) ; lines() ; clear() ; flush() ; export_events(limit?)
# 遥测回流（PS-04；须先 Privacy.consent()）
NetClient.upload_telemetry(limit?)
# 确定性随机（world.seed）
Rng.root_seed() ; seed_for(key) ; rng(key) ; randf(key) ; randi(key, from, to)
# 画质档位（quality.tier）
Quality.tier() ; set_tier(t) ; tier_settings()
# 目标选取（targeting.query）
Targeting.nearest(candidates, origin, radius?, exclude_id?) ; in_radius(...) ; best(candidates, origin, score)
# 交互（interaction.core）
Interaction.range_of(def_id) ; in_range(pos, target, def_id) ; run(def_id, actor?)
# 实体（entity.spawn / entity.query）
Entities.spawn(def_id, pos?, props?) ; query(filters) ; entity(id) ; despawn(id) ; all() ; count() ; clear()
# 技能时序（combat.timeline）
Combat.begin_cast(ability_id, caster_rt, target_rt) ; cooldown_ready(caster_id, ability_id)
# 生命周期 dispatch（engine-sdk §5；底层为 Hooks）
Lifecycle.dispatch(method, args?)     # 调用 g3_lifecycle 组内同名方法
# 钩子（hooks.md；内容可选用）
Hooks.register(domain, hook, cb, priority?) ; emit(domain, hook, args?) ; is_known(domain, hook) ; count(domain, hook)
# 隐私（PS-06；上传类须先同意）
Privacy.consent() ; set_consent(v) ; delete_local_data()
# 日志 / 指标（observability.md §7；内容可选用）
Log.info(cat, msg, fields?) ; warn ; error ; debug ; trace ; fatal ; set_level(cat, level) ; is_enabled(level, cat)
Metrics.inc(name, n?) ; counter(name) ; observe(name, ms) ; timer_stats(name) ; snapshot() ; flush()
```

> 新增内容可见 API 属**引擎能力变更**：走 §9 能力请求流程，并同步 §2 与 `capabilities.json`。

## 4. 错误码

| 码 | 含义 |
|---|---|
| `NotFound` | Def/实体/场景不存在 |
| `InvalidArg` | 参数非法 |
| `NotLoaded` | 依赖未加载 |
| `Rejected` | 缺少能力或版本不满足 |
| `NoPath` | 无法寻路 |
| `Timeout` | 异步超时 |
| `SandboxDenied` | 沙箱权限不足 |
| `RuntimeError` | 其他运行时错误 |

## 5. 生命周期回调（脚本，设计保留）

`on_content_loaded` · `on_scene_enter` · `on_scene_exit` · `on_region_enter` · `on_save` · `on_load` · `on_tick`（可选）

> **当前状态**：已由 `Hooks`（`autoload/hooks.gd`，域/优先级/索引派发）+ `Lifecycle` 适配层实现（`g3_lifecycle` 组）。内容脚本可加入 `g3_lifecycle` 组按名实现回调，或经 `Hooks.register` 直接注册。目录见 [hooks.md](hooks.md)。

## 6. 脚本边界

- 可信内容：GDScript，可热重载。
- 不可信内容（未来 UGC）：WASM 沙箱 + 配额 + 白名单 API。
- 脚本只在主线程；禁止阻塞与长任务。

## 7. 错误约定

- 明确 `Result`/`Error`，不静默。
- 内容脚本异常被捕获并降级，不影响引擎与其他内容。

## 8. 禁止事项

- 内容访问引擎内部实现或平台 API。
- 引擎为单一内容/玩法开专用 API。
- 任何“仅某内容包用”的接口都是设计失败信号。

## 9. 能力请求流程

世界组提需求 → 引擎组评估通用性 → 实现为通用能力（升版本）→ 写入注册表与 Schema → 发布并更新契约测试。

## 10. 能力矩阵输出

构建时生成 `capabilities.json`，供内容校验器与 CI 读取；格式见 [content-tooling.md](content-tooling.md) §3。