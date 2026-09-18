# 契约：协议与共享模型

- 状态：生效中
- 更新日期：2026-09-17

> `shared/` 是 **Client 与 Server 的唯一共用源**；协议与共享模型**只此一处**，两端只读。

## 1. 共享层职责

```
shared/
├── protocol/   消息定义（版本化）+ 编解码
├── model/      实体 / 组件 / Def 的共享类型
└── contract/   生成物：content schema、capabilities、协议 IDL
```

> **当前状态**：已有**版本化 IDL**——`shared/protocol/protocol-1.0.json`、`shared/model/model-1.0.json`（源：`App/engine/sdk/{protocol,model}/`，由 `content sync` 同步）。`content gen` 生成两端常量与编解码辅助（`App/engine/sdk/generated/gdscript/g3_protocol.gd` 等，漂移门 `arch_check` R8）。预研阶段传输仍为内联 JSON（Nakama Realtime match state，见 §8）；二进制（Protobuf）为后续独立项。

## 2. 消息分类

| 类别 | 方向 | 传输 | 示例 |
|---|---|---|---|
| `world` | 双向 | 可靠 + 稀疏广播 | 位置、AOI 进入/离开、实体状态 |
| `instance` | 双向 | 实时（预测） | 移动、技能、命中、状态同步 |
| `social` | 双向 | 可靠 | 聊天、组队、好友、邮件 |
| `system` | 双向 | 可靠 | 登录、版本协商、错误、心跳 |

## 3. 传输

| 通道 | 用途 | 特性 |
|---|---|---|
| 世界通道 | 存在/AOI | 可靠、低频 |
| 副本通道 | 实时动作 | 低延迟、可丢快照 + 预测 |
| 社交通道 | 聊天/组队 | 可靠、可跨区 |

- 世界与副本**用不同连接/协议**，互不拖累。

## 4. 序列化与版本

- 优先**二进制**（紧凑、快）；调试可用 JSON 镜像。
- 每条消息带**类型 ID + 版本**；遵循兼容策略（只增字段，删除须升 MAJOR）。
- 握手做**版本协商**：不兼容则拒绝并提示。

## 5. 共享模型

`shared/model` 定义两端共识的**实体/组件/Def ID** 与枚举：

| 模型 | 说明 |
|---|---|
| `Entity` | `{ id, archetype(defRef), components[] }` |
| 组件 | `Transform/Movement/Appearance/Presence/Social/Replicable`（副本侧另有战斗组件） |
| Def 引用 | 内容包 id + Def id（跨包引用需声明） |

> **Def = 静态原型**；**Entity = 运行时状态**。原型来自内容，运行时状态在服务端权威。

## 6. 生成物（`shared/contract`）

- `content-<ver>.json`：内容 Schema。
- `capabilities.json`：引擎能力矩阵。
- 协议 IDL → 两端代码（类型安全、防漂移）：`content gen` 产出 `g3_protocol.gd` / `g3_model.gd` / `g3_protocol.gen.go` / `g3_protocol_gen.lua`（含 `T_*` 常量、`MESSAGES`、`encode/decode`）。漂移门见 [content-tooling.md](content-tooling.md) §6、`arch_check` R8。
- **唯一生成源**；生成物不得手改。

## 7. 契约测试

- 协议兼容性测试（新旧版本互通）。
- 共享模型与两端实现的一致性校验。
- 接入 CI 引擎矩阵（见 平台内部文档）。

## 8. 预研消息（P6，当前实现）

传输：**Nakama 内建 Realtime match state**（JSON 对象），`t` 为类型。加入对局用 Nakama **presence join**（无 `t="join"` 数据消息）。

### RPC

| RPC | 入参（JSON） | 出参（JSON） | 说明 |
|---|---|---|---|
| `create_match` | `{mode:"pve"\|"pvp", ...}` | `{match_id, mode}` | 创建通用对局；缺省 `mode="pve"` |

- `pvp` 可选参数：`max_players`（默认 4）· `respawn_seconds`（默认 3）· `score_target`（先达者胜，0=不限）· `time_limit`（tick 上限，0=不限）· `arena{w,h}`（坐标边界）。
- `aoi`（可选，**AOI 定向广播**）：`{ enabled?, cell, radius, hysteresis?, maxRadius? }`（单位与 `arena` 一致=像素；`radius` 按 `maxRadius` clamp，且**下限 = 交战距离**（`RANGE`/`combat` 投影里的最大技能 `range`，防“隐身攻击者”）；提供即启用，`enabled:false` 显式关闭）。**默认关闭**；世界层建议开启。
- `combat`（可选，通用战斗投影）：`{ abilities: { <abilityId>: { cooldownMs, range, damage, cooldownGroup? } } }`。由内容产出（规则/数值在内容层），服务器据此**权威结算** `cast`，服务端不含玩法数值、无每内容代码。客户端可用 `NetClient.build_combat_data()` 从 Def 生成。
- 分队：客户端 join metadata 带 `team`；无 `team` 即 FFA（互敌）。

### Match state

| 方向 | 消息 | 字段 | 实现 |
|---|---|---|---|
| C→S | `input` | `dx`、`dy`（-1..1，服务端 clamp） | ✅ |
| C→S | `attack` | — （固定常数额外机制；无 `combat` 投影时的兼容路径） | ✅ |
| C→S | `cast` | `ability`（内容 ability id）、`target`（可选实体 id） | ✅ |
| S→C | `snapshot` | `proto`、`mode`、`tick`、`players{}`、（`pve` 含 `enemy{}`）；AOI 启用时**只含该玩家兴趣半径内实体**，并带 `enter[]`/`leave[]` 差量 | ✅ |
| S→C | `welcome` | `id`、`mode`、`tick`、`proto`、`arena` | ✅ |

**AOI（兴趣裁剪）**：`aoi.enabled` 时，服务端对每个 presence 只广播其半径内实体（九宫格 `aoi.lua`），并给出 `enter`/`leave` 差量；**迟滞**：进入需 `d ≤ radius`，**保留**允许 `d ≤ radius×(1+hysteresis)`（消除边界抖动）；**只裁剪广播、不裁剪模拟**，交互双方（攻击/受击/目标）始终互达。

**加入被拒**：满员 → `match_full`；版本不符 → `proto_mismatch`（Nakama join 返回失败；客户端发 `match_join_failed(reason)` 信号）。

**版本协商（握手）**：`proto` 为对局协议版本（当前 `1`）。客户端 `join metadata` 带 `{"proto": <n>}`；服务端 `match_join_attempt` 不匹配则拒绝（`proto_mismatch`），未携带则放行（兼容旧端）。`welcome` / `snapshot` 均回带 `proto`，客户端校验不一致即报错。

实体字段：`id` · `x` · `y` · `hp` · `max_hp` · `dead`；`pvp` 另含 `team` · `kills` · `deaths` · `respawn_at`。
**服务端权威**：HP/伤害/重生/胜负由服务器结算（online-and-instances.md）：
- `pve`：`attack` 命中范围内敌人。
- `pvp`：`attack` 命中范围内最近的敌对玩家（同 `team` 免伤）；死亡按 `respawn_seconds` 重生。
