# 契约：数据与存档 Schema

- 状态：生效中
- 更新日期：2026-09-17

> 覆盖**离线（本地权威）**与**登录（云存档）**两种模式。详见 平台内部文档。

## 1. 存储

- **本地**：`user://save.json`（离线权威、断网续玩）。文件为**信封**：`{ "snapshot": <快照>, "hash": "sha256:…" }`，`hash` 为快照规范化 JSON 的 SHA-256。
- **备份槽**：`user://save.bak`（上一份可用主存档，写入前轮转）；主存档损坏时回退。
- **云端**：Nakama 对象存储（`collection=g3`，`key=progress`；登录时权威/备份）——存**裸快照**（不含信封）。
- **outbox**：`user://save.outbox.json`（待上行快照队列，幂等键 + 顺序重放）。
- 快照带 `version`；服务端由 Nakama 承载（见 平台内部文档）。

## 2. 快照结构

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

| 字段 | 说明 |
|---|---|
| `version` | 存档版本（单调递增，当前 `1`） |
| `player.stats_id` | 基础属性 Def id（`stats`） |
| `player.level/xp/hp` | 等级 / 经验 / 当前血量 |
| `player.inventory` | 物品 id 列表 |
| `player.equipped` | 槽位 → 物品 id；槽位 id 由内容 `equipment.slots[]` 声明（§25 content-schema） |
| `quests.active` | 任务 id → 各目标进度数组 |
| `quests.pending` | 待交付任务集合 |
| `quests.done` | 已完成任务集合 |
| `kv` | 通用 KV（内容层跨会话持久化，`SaveService.get_value/set_value`）；向后兼容，旧档可缺省 |

## 3. 模式与权威

| 模式 | 位置 | 权威 |
|---|---|---|
| 离线 | `user://save.json` | 本地 |
| 登录 | Nakama `g3/progress` | 云端（本地为缓存） |

## 4. 读写时机（实现）

- **启动**：`SaveService.restore_on_boot()` 读本地并恢复。
- **升级 / 任务交付**：`SaveService.autosave()` 写本地；联网时同时 `save_cloud()`。
- **登录**：`NetClient` 连接后若本地有档则上行云端（`[Save] pushed local -> cloud`）；`net_connected` 事件触发 `flush_outbox()` 顺序重放待上行项。

## 5. 版本与冲突

- `version` 单调递增；加载旧档按**迁移链**升级（当前仅 `1`）。
- 冲突粒度：**整档 LWW**（预研不做字段级合并）。

## 6. 完整性

- **原子写**：写 `user://save.tmp` → 轮转备份槽 → `rename` 替换主存档；中断不产生半写主档。
- **哈希校验**：读取时按信封 `hash` 校验快照；不一致视为损坏。
- **损坏回退**：主存档缺失/损坏（解析失败或哈希不符）时回退 `save.bak`；备份亦不可用才回退默认，不阻塞启动。
- **迁移链**：按 `version` 逐级迁移（当前仅 `1`）；迁移与缺省字段补齐**幂等**，可重复执行。

## 7. 已实现 / 暂缓

**已实现**（P23）：原子写（临时文件→rename）、哈希校验、备份槽回退、迁移链（幂等）、outbox（幂等键 + 顺序重放，`flush_outbox()`）。

**暂缓（设计保留）**：

- 本地 **SQLite** 多表（`settings` / `profile` / `save_slot` / `world_state` / `cache`）。
- 字段级冲突合并（当前为整档 LWW）。
- 账号数据加密 / 导出 / 删除（隐私合规）。
