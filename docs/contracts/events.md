# 契约：事件目录

- 状态：生效中
- 更新日期：2026-09-17

> 类型化事件的**权威清单**（名称 + payload + 触发）。逻辑事件同步有序，表现事件异步可丢；内容只能**订阅白名单事件**。
> `App/autoload/event_bus.gd` 的 `signal` 集合必须与「已实现」表 + 「暂缓」清单一致（CI 校验）。

## 已实现

### 内容 / 加载

| 事件 | payload | 触发 |
|---|---|---|
| `content_loaded` | `{}` | Def DB 就绪 |
| `pack_loaded` | `{ packId }` | 单个内容包加载成功 |
| `pack_rejected` | `{ packId, reason }` | 内容包被拒（manifest 非法 / 缺 id / 能力不满足） |

### 场景

| 事件 | payload | 触发 |
|---|---|---|
| `scene_loaded` | `{ sceneId }` | 场景加载完成 |
| `scene_enter` | `{ sceneId }` | 进入场景 |
| `scene_exit` | `{ sceneId }` | 离开场景 |

### 世界 / 玩家

| 事件 | payload | 触发 |
|---|---|---|
| `player_landed` | `{ poiId, entryPointId }` | 飞行着陆 |
| `player_spawned` | `{ entityId }` | 玩家实体进入世界/场景 |

### 实体（`entity.spawn` / `entity.query`）

| 事件 | payload | 触发 |
|---|---|---|
| `entity_spawned` | `{ entityId, defId }` | 运行时实体创建 |
| `entity_despawned` | `{ entityId }` | 运行时实体移除 |

### 区域（`region.trigger`）

| 事件 | payload | 触发 |
|---|---|---|
| `region_enter` | `{ regionId }` | 进入 `RegionTrigger` 区域 |
| `region_exit` | `{ regionId }` | 离开 `RegionTrigger` 区域 |

### 交互

| 事件 | payload | 触发 |
|---|---|---|
| `interact_done` | `{ entityId, interactionId }` | 完成一次交互 |

### 任务 / 对话（`quest.core` / `dialog.core`）

| 事件 | payload | 触发 |
|---|---|---|
| `quest_state_changed` | `{ questId, state }` | 任务状态变化（`active`/`ready`/`done`） |
| `dialog_started` | `{ dialogId }` | 对话开始 |
| `dialog_ended` | `{ dialogId }` | 对话结束 |

### 战斗

| 事件 | payload | 触发 |
|---|---|---|
| `ability_cast` | `{ sourceId, abilityId }` | 释放技能 |
| `entity_damaged` | `{ targetId, amount, sourceId }` | 造成伤害 |
| `entity_died` | `{ entityId, killerId? }` | 实体死亡 |
| `effect_applied` | `{ targetId, effectId }` | 效果被施加 |

### 存档 / 网络

| 事件 | payload | 触发 |
|---|---|---|
| `save_done` | `{}` | 存档完成 |
| `load_done` | `{}` | 读档完成 |
| `net_connected` | `{}` | 网络连接建立 |
| `net_disconnected` | `{}` | 网络断开（socket closed） |

### 本地化

| 事件 | payload | 触发 |
|---|---|---|
| `locale_changed` | `{ locale }` | 语言切换完成 |

### 配置 / 画质（`config.flag` / `quality.tier`）

| 事件 | payload | 触发 |
|---|---|---|
| `flag_changed` | `{ flagKey, value }` | 特性开关变更 |
| `quality_changed` | `{ tier }` | 画质档位变更 |

## 暂缓（已定义未触发 或 设计保留）

（无——已定义事件均有触发点）

设计保留（尚未定义）：`instance_enter` · `instance_exit`

## 非 EventBus 信号

`NetClient.chat_message(username, content)`：世界聊天消息（社交专用，不进入事件总线）。

## 规则

- payload 随 Content Schema **版本化**；只增字段。
- 新增事件须登记本目录并说明触发时机；未实现项归入「暂缓」。
