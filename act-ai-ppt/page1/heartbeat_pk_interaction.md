# 心动PK 前后端交互文档

## 一、总体交互架构

### 1.1 用户侧入口

```mermaid
flowchart TB
    subgraph 用户可见入口
        E1[玩法Web页<br/>宣发用, 独立H5]
        E2[频道内挂件<br/>活动期间自动出现<br/>广播控制显示/隐藏]
        E3[频道内弹窗<br/>PC: 内嵌页H5<br/>App: H5弹窗]
        E4[玩法页H5<br/>玩法/榜单/任务数据<br/>查询后端接口]
    end

    E2 -- 点击 --> E3
    E2 -- 点击 --> E4
    E3 -. 内容同 .-> E4
```

| 入口 | 触发方式 | 载体 | 说明 |
|------|----------|------|------|
| 玩法Web页 | 运营配置链接 | 独立H5 | 宣发用，展示活动规则、奖励信息 |
| 频道内挂件 | 后端广播控制 | 房间模板 | 活动时间内自动显示，实时数据 |
| 频道内弹窗 | 挂件打开/单播触发 | PC内嵌页H5 / App H5弹窗 | 详细信息展示、交互操作 |
| 玩法页H5 | 挂件点击 | H5 | 玩法数据、榜单数据、任务数据等，查询后端接口 |

### 1.2 后端服务架构

```mermaid
flowchart TB
    subgraph 外部业务服务
        PK[zhuiya-pk<br/>跨厅PK服务]
    end

    subgraph Kafka
        K1[zy_pk_start_event<br/>跨厅PK开始<br/>TODO: 待新增]
        K2[zy_pk_end_event<br/>跨厅PK结束<br/>TODO: 待新增]
        K3[zy_pk_score_change_event<br/>PK分数变化<br/>TODO: 待新增]
    end

    subgraph 活动后端
        ACT[活动中控服务<br/>事件消费 / 前端交互<br/>频道广播 / 用户单播]
        LOTTERY[抽发奖服务<br/>奖包配置管理<br/>抽发奖逻辑执行]
        WIDGET[挂件服务<br/>控制显示/隐藏<br/>列表查询 / 变化广播]
    end

    subgraph 前端
        FE_WEB[玩法Web页<br/>宣发用]
        FE_H5[玩法页H5<br/>玩法/榜单/任务]
        FE_ROOM[房间模板<br/>挂件 + 弹窗]
    end

    PK -- 发布 --> K1 & K2 & K3
    ACT -- 消费 --> K1 & K2 & K3

    FE_WEB -- HTTP --> ACT
    FE_H5 -- HTTP 查询 --> ACT
    FE_ROOM -- HTTP --> ACT
    ACT -- 奖池id/奖包id --> LOTTERY
    ACT -- 挂件显隐通知 --> WIDGET
    WIDGET -- 挂件显隐广播 --> FE_ROOM
    ACT -- 频道广播(玩法数据) --> FE_ROOM
    ACT -- 用户单播/弹窗 --> FE_ROOM
```

| 服务 | 职责 | 对外交互 |
|------|------|----------|
| **活动中控服务** | 消费PK事件驱动活动逻辑，与前端交互，发广播和单播 | Kafka消费、前端HTTP、广播、单播 |
| **抽发奖服务** | 奖包配置管理、抽发奖逻辑执行 | 提供奖池id/奖包id供中控服务调用 |
| **挂件服务** | 控制挂件显示/隐藏，挂件列表查询，变化广播 | 中控通知 → 挂件广播推送至前端 |

### 1.3 PK事件（活动中控消费）

活动中控作为旁路服务，消费跨厅PK的 Kafka 事件，不侵入 PK 业务核心代码。

> **注意**：跨厅PK（`GameInvitationHandler` 邀请制）与跨业务PK（`CrossBizPkService`，topic 前缀 `hd_cross_biz_pk_*`）是两套不同的流程。
> 心动PK 基于**跨厅PK**，但跨厅PK 目前没有对外 Kafka 事件，以下三个事件均需在 `zhuiya-pk` 服务中**新增**。

事件来源：`zhuiya-pk` PK服务（需新增 Kafka Producer）

| Kafka Topic | Proto消息(建议) | 含义 | 触发时机(zhuiya-pk) | 心动PK用途 | 状态 |
|---|---|---|---|---|---|
| **`zy_pk_start_event`** | `ZyPkStartEvent` | 跨厅PK开始 | `PkStartService.startInvitePk()` 成功后 | 记录对局，开始监控PK值累计，单播心动模式介绍弹窗 | **TODO: 待新增** |
| **`zy_pk_end_event`** | `ZyPkEndEvent` | 跨厅PK结束 | `PkProgressService` 结算逻辑（正常结束/投降/超时） | terminateType=NORMAL_END时进入心动PK结算 | **TODO: 待新增** |
| **`zy_pk_score_change_event`** | `ZyPkScoreChangeEvent` | PK分数变化 | `PkProgressService` addScore后 | 实时获取双方当前分数，检测2000W阈值，更新挂件数据 | **TODO: 待新增** |

> **设计决策**：活动不消费原始送礼流水（`zhuiwan_prop_used`）自行算分，而是由 PK 服务在 `addScore()` 后发出分数变化事件。
> 好处：① 活动不重复 PK 的算分逻辑（加时礼物、减分礼物、全麦打赏等边界） ② 拿到的是 PK 服务的权威分数

### 1.4 通信方式

| 方式 | 触发方 | 目标 | 场景 |
|------|--------|------|------|
| Kafka消费 | PK服务(zhuiya-pk) | 活动中控服务 | PK生命周期事件(开始/结束/分数变化) |
| HTTP | 前端主动 | 活动中控服务 | 页面数据拉取、用户主动操作 |
| 频道广播 | 活动中控/挂件服务 | 房间全量用户 | 挂件显示/更新/隐藏、爱雨触发 |
| 用户单播 | 活动中控服务 | 指定用户 | 弹窗提示（心动介绍、PK开启、获奖通知） |

---

## 二、核心流程图

### 2.1 心动PK 完整生命周期

```mermaid
sequenceDiagram
    participant H as 主持(发起方)
    participant C1 as 房间A前端
    participant PK as PK/匹配服务(Kafka)
    participant ACT as 活动中控服务
    participant WDG as 挂件服务
    participant LOT as 抽发奖服务
    participant C2 as 房间B前端

    Note over H,C2: ===== 阶段1: 跨厅PK开始 =====
    PK->>ACT: Kafka: ZyPkStartEvent(roundId, 双方player信息)
    ACT->>ACT: 记录对局, 开始监控PK值累计
    ACT-->>H: 单播: 心动模式介绍弹窗

    Note over H,C2: ===== 阶段2: PK值累计, 检测阈值 =====
    loop 每次分数变化
        PK->>ACT: Kafka: ZyPkScoreChangeEvent(双方当前分数)
        ACT->>ACT: 检测双方总PK值 ≥ 2000W ?
    end

    Note over H,C2: ===== 阶段3: 心动PK触发(总PK值≥2000W) =====
    ACT->>WDG: 通知显示心动PK挂件
    WDG-->>C1: 挂件广播: 心动PK挂件显示
    WDG-->>C2: 挂件广播: 心动PK挂件显示
    ACT-->>C1: 单播(双方所有主持): 心动PK已开启弹窗
    ACT-->>C2: 单播(双方所有主持): 心动PK已开启弹窗

    Note over H,C2: ===== 阶段4: 心动PK进行中 =====
    loop 每次分数变化(心动模式内)
        PK->>ACT: Kafka: ZyPkScoreChangeEvent
        ACT->>ACT: 计算心动PK新增值, 更新TOP主持排名
        ACT-->>C1: 频道广播: 玩法数据更新(新增PK值/TOP1主持/预估奖励)
        ACT-->>C2: 频道广播: 玩法数据更新
    end

    Note over H,C2: ===== 阶段5: PK结束 =====
    PK->>ACT: Kafka: ZyPkEndEvent(endType=NORMAL_END)
    ACT->>ACT: 计算心动PK胜负, 确定获胜方贡献TOP3主持
    ACT->>LOT: 请求发放获胜主持奖励(奖池id/奖包id)
    LOT-->>ACT: 返回发奖结果
    ACT->>WDG: 通知隐藏挂件
    WDG-->>C1: 挂件广播: 心动PK结束 + 隐藏
    WDG-->>C2: 挂件广播: 心动PK结束 + 隐藏
    ACT-->>C1: 单播(获胜方贡献TOP3主持): 获奖弹窗

    Note over H,C2: ===== 阶段6: 心动爱雨(获胜房间) =====
    ACT-->>C1: 频道广播(获胜房间): 心动爱雨触发
```

### 2.2 心动爱雨交互流程

```mermaid
sequenceDiagram
    participant U as 游客(前端)
    participant C as 前端
    participant ACT as 活动中控服务
    participant LOT as 抽发奖服务

    ACT-->>C: 频道广播: 心动爱雨触发(3分钟倒计时)
    C->>U: 弹出确认弹窗【立即参与】
    U->>ACT: HTTP: 上报已看到爱雨弹窗(防止重进频道再次触发)

    alt 用户点击【立即参与】
        Note over U,LOT: 无需发请求, 前端直接开始掉落15个爱心(10s)
        loop 用户点击爱心
            U->>C: 点击爱心, toast提示已收集数量
        end

        Note over U,LOT: 掉落结束/用户主动关闭
        U->>ACT: HTTP: 提交抽奖(携带收集爱心数)
        ACT->>LOT: 调用抽奖(奖池id/奖包id, 收集数)
        LOT->>LOT: 判定中奖(≥1个可抽奖, 15个必中)
        LOT-->>ACT: 返回抽奖结果
        ACT-->>U: 返回抽奖结果

        alt 中奖
            C->>C: PC弹窗/移动端toast展示奖品
            U->>ACT: HTTP: 发送公屏感谢消息
        else 未中奖
            C->>C: 弹窗提示未中奖
        end

    else 用户关闭弹窗
        Note over U,LOT: 不参与本次爱雨, 退出重进也不再触发
    end
```

---

## 三、前端状态机

```mermaid
stateDiagram-v2
    [*] --> 普通PK: 发起随机匹配
    普通PK --> 心动PK进行中: 广播 widget_update(action=show)
    心动PK进行中 --> 心动PK进行中: 广播 widget_update(action=update)
    心动PK进行中 --> PK结束: 广播 widget_update(action=hide)
    PK结束 --> 心动爱雨: 广播 love_rain(获胜房间)
    PK结束 --> [*]: 非获胜房间
    心动爱雨 --> [*]: 3分钟超时/用户关闭
```

---

## 四、前端关键交互说明

### 4.1 挂件（广播驱动）

挂件服务只负责控制挂件的显示/隐藏。挂件内的玩法数据（PK值、TOP主持、预估奖励等）由活动中控服务直接广播到前端。

| 事件 | 来源 | 前端行为 |
|------|------|----------|
| 收到 `widget_update(show)` | 挂件服务 | 座位区右上角显示心动PK挂件 |
| 收到 `widget_update(hide)` | 挂件服务 | 隐藏并移除挂件 |
| 收到玩法数据广播 | 活动中控服务 | 更新挂件内玩法数据（双方PK值、TOP1主持、预估奖励） |
| 用户点击挂件 | — | 打开心动PK玩法页H5 |

### 4.2 弹窗（单播驱动）

| 单播类型 | 目标用户 | 前端行为 |
|----------|----------|----------|
| `heartbeat_pk_intro` | 发起匹配的主持 | 弹出心动模式介绍，点击详情跳转玩法页 |
| `heartbeat_pk_started` | PK双方所有主持 | 弹出"心动PK已开启"，关闭时向右上角挂件方向收起动画 |
| `heartbeat_pk_reward` | 获胜方TOP3主持 | 弹出获奖提示，展示排名和奖品 |

### 4.3 心动爱雨（广播触发 + HTTP交互）

| 步骤 | 交互 |
|------|------|
| 收到爱雨广播 | 弹出确认弹窗，仅本人可见 |
| 看到弹窗 | 前端上报已展示（HTTP），服务端记录，防止重进频道再次触发 |
| 点击【立即参与】 | 无需发请求，前端直接开始掉落爱心动画 |
| 点击爱心 | 前端本地计数，toast提示"已收集X/15" |
| 10s结束 / 点击关闭 | 调用 `love_rain/lottery` 提交收集数，展示结果 |
| 中奖 | 自动调用公屏接口发送感谢消息 |
| 关闭/不参与 | 标记本轮已结束，退出重进不再触发 |

---

## 五、异常与边界处理

| 场景 | 处理策略 |
|------|----------|
| 每厅每日心动PK上限(3场) | 服务端判断，超限后不再触发心动模式 |
| 全服奖池耗尽 | 礼物奖励改发"飘逸羚踪入场秀3天"，奖池归0后才切换 |
| 爱雨期间用户退出重进 | 前端弹窗展示时即上报，服务端记录，重进后不再推送 |
| 爱雨期间新进房用户 | 3分钟内进房的新用户也收到爱雨广播，可参与 |
| 用户提前关闭爱心特效 | 前端以当前已收集数调用lottery接口，不再展示特效 |
| heart_collected = 0 | 不参与抽奖，服务端直接返回无奖 |
| 爱雨奖品每日上限 | 服务端控制，达上限后该奖品概率归零，重新分配 |
