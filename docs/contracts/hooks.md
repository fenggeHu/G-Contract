# 契约：平台钩子目录（Hooks）

- 状态：生效中
- 更新日期：2026-09-17

> 用途：内容组按本目录**注册钩子**扩展行为；平台**不修改内容层**，只提供接口与文档。
> 与事件的区别：**钩子 = 有序、可拦截、同步**；**事件 = 通知、不可拦截、可异步**（见 [events.md](events.md)）。
> 实现：`autoload/hooks.gd`（`Hooks` 注册表：优先级 + 按 `(domain, hook)` 索引派发）；`autoload/lifecycle.gd` 为**适配层**——把 `EventBus` 事件桥接为钩子，并派发到 `g3_lifecycle` 组内同名方法。
> **全部列出的域均已接通触发**（world/scene/region/entity/combat/quest/dialog/net/save/tick）。
> 归属：平台实现（`G_Engine`）；本目录为**共用契约**。

## 1. 注册模型（已实现：autoload/hooks.gd）

```text
Hooks.register(domain, hook, callable, priority := 0)   # 注册（平台 API）
Hooks.emit(domain, hook, args...)                        # 派发（平台内部）
```

- 同一 `(domain, hook)` 多个订阅者按 `priority` 升序调用；同 priority 按注册顺序。
- 按钩子建索引：**未实现该钩子的订阅者不被遍历**。
- 钩子同步执行；耗时应短，禁止阻塞与长任务。

## 2. 域与钩子（规划）

| 域 | 钩子 | 签名 | 时机 |
|---|---|---|---|
| `world` | `on_content_loaded` | `()` | Def DB 就绪 |
| `scene` | `on_scene_enter` / `on_scene_exit` | `(scene_id)` | 场景进出 |
| `region` | `on_region_enter` | `(region_id)` | 区域进入 |
| `entity` | `on_entity_spawned` / `on_entity_died` | `(entity_id[, killer_id])` | 实体创建 / 死亡 |
| `combat` | `on_ability_cast` / `on_entity_damaged` | `(source_id, ability_id)` / `(target_id, amount, source_id)` | 战斗结算 |
| `quest` | `on_quest_state_changed` | `(quest_id, state)` | 任务状态变化 |
| `dialog` | `on_dialog_started` / `on_dialog_ended` | `(dialog_id)` | 对话开始/结束 |
| `net` | `on_net_connected` / `on_net_disconnected` | `()` | 网络连接 |
| `save` | `on_save` / `on_load` | `()` | 存档读写 |
| `tick` | `on_tick` | `(dt)` | 每帧（**谨慎使用**） |

## 3. 边界

- 钩子只承载**通用引擎时机**；玩法规则仍属内容层（Def / 脚本），平台不定义玩法模型。
- 不新增“仅某个内容包需要”的钩子（设计失败信号，见 [engine-sdk.md](engine-sdk.md) §8）。