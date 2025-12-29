# Chapter 4｜驾舱一体目标架构总览

## 4.1 开篇与目标

### 本章概要
在完成了需求定义后，我们面临的最大工程挑战是：**“如何让概率性的 LLM（大语言模型）安全地指挥确定性的 AD（自动驾驶）系统？”**

传统的软硬隔离架构（Cockpit domain vs. AD domain）已无法满足“驾舱一体”的需求。我们需要构建一个**“共享认知架构”**。本章将定义系统的逻辑视图、数据流向、端云边界以及核心的状态管理机制。

### 学习目标
1.  **全局观**：理解从语音输入到车辆执行的全链路架构（Interactive -> Agent -> Policy -> Execution）。
2.  **解耦与融合**：掌握“统一状态机（Unified State Machine）的设计，解决驾舱状态不同步的顽疾。
3.  **安全边界**：明确 **Policy Guard（策略守卫）** 如何作为 LLM 与车辆控制之间的绝对防火墙。
4.  **接口定义**：定义座舱 Agent 与 Vision-to-Action 自驾模型之间的宏观指令接口。

---

## 4.2 总体架构：云端-车端-移动端协同

驾舱一体架构本质上是一个 **Hybrid-AI System（混合人工智能系统）**，它结合了云端的强大推理能力（Reasoning）和端端的实时反射能力（Reflex）。

### 4.2.1 逻辑架构视图 (Logical Architecture View)

```ascii
[ Cloud Layer: Heavy Reasoning & Knowledge ]
       |
       +--> [ LLM Service / RAG Knowledge Base / User Profile Graph ]
       |         ^
       |         | (Async Sync / Context Upload)
       v         |
[ Edge Layer (Vehicle): Real-time & Safety & Privacy ]
+-----------------------------------------------------------------------------------+
|  1. Interaction & Perception Layer (多模态感知)                                   |
|     [Mic Array] -> [ASR] --> +                                                    |
|     [Touch/Gaze] ----------> | Multi-modal Fusion -> (Standardized Intent Event)  |
|     [Cabin Camera] --------> +                                                    |
+-----------------------------------------------------------------------------------+
       | (Intent Event)
       v
+-----------------------------------------------------------------------------------+
|  2. Unified Agent Orchestrator (统一编排器)                                       |
|     +-------------------------+    +-----------------------+                      |
|     |  Router (分发路由)       | -> | Fast Path (Rule/SLM)  | -> (Simple Cmd)      |
|     +-----------+-------------+    +-----------------------+                      |
|                 | (Complex)                                                       |
|                 v                                                                 |
|     +-------------------------+    +-----------------------+                      |
|     |  Cognitive Planner      | <->| Memory & Context Mgr  |                      |
|     |  (LLM Based Reasoning)  |    | (Short/Long Term)     |                      |
|     +-----------+-------------+    +-----------------------+                      |
+-----------------------------------------------------------------------------------+
       | (Planned Tool Calls)
       v
+-----------------------------------------------------------------------------------+
|  3. Policy & Safety Guard (安全守卫 - The "Air Gap")                              |
|     [ Schema Validator ] -> [ Permission Check ] -> [ Risk Evaluator ]            |
|     -> [ Confirmation Logic (HMI) ] -> [ Rate Limiter ]                           |
+-----------------------------------------------------------------------------------+
       | (Authorized Action)
       v
+-----------------------------------------------------------------------------------+
|  4. Execution Layer (执行层)                                                      |
|     +----------------+   +------------------+   +----------------+                |
|     | Trip Executor  |   | Vehicle Executor |   | Service Executor|               |
|     +-------+--------+   +--------+---------+   +-------+--------+                |
|             | (Route)             | (Control)           | (API)                   |
|             v                     v                     v                         |
|     [ Nav Engine ]       [ AD System (E2E)]     [ Cloud 3rd Party ]               |
|                          [   VCU / BCM    ]                                       |
+-----------------------------------------------------------------------------------+
       ^              ^              ^
       |              |              |
+------+--------------+--------------+----------------------------------------------+
|  5. Shared State Bus (统一状态总线 - Pub/Sub)                                     |
|     Topics: /trip/state, /vehicle/telemetry, /user/auth, /ad/perception           |
+-----------------------------------------------------------------------------------+
```

---

## 4.3 分层架构详细设计

### 4.3.1 交互层 (Interaction Layer)：输入的归一化
本层的核心任务是**“消噪”**与**“融合”**。
*   **多模态融合**：单纯的语音往往指代不明。例如用户指着窗外说“那是哪里？”，系统需结合视线追踪（Gaze Tracking）或手指指向与外部摄像头画面进行坐标映射。
*   **标准化输出**：无论输入源是语音、APP还是按钮，都转化为统一的 `UserIntentEvent` 投递给编排器。

### 4.3.2 Agent 编排层 (Orchestration Layer)：大脑
这是架构中最复杂的部分，采用了 **Router-Planner** 模式。
*   **Router (路由)**：
    *   **Fast Path**: 对于“打开车窗”、“调高音量”等高频低风险指令，**不走云端 LLM**，直接由端侧 NLU（Natural Language Understanding）或小模型（SLM）处理，保证 < 500ms 的端到端响应。
    *   **Slow Path**: 对于“我想去个能看海的地方吃海鲜”、“帮我规划一条不堵车的路”等复杂指令，路由至云端 LLM 进行推理。
*   **Cognitive Planner (认知规划)**：
    *   负责任务拆解（Chain-of-Thought）。例如：“先找海边的海鲜餐厅” -> “确认是否营业” -> “规划路线” -> “检查电量是否足够”。
    *   **Memory Manager**: 维护 `ConversationContext`（当前聊什么）和 `UserPreferences`（用户画像）。

### 4.3.3 工具执行层 (Tool / Executor Layer)：手脚
Agent 不产生直接作用，它只是生成“工具调用请求”。工具层负责具体的业务逻辑封装。
*   **原子化封装**：将底层复杂的 CAN 信号或 API 封装为语义化的函数。
    *   *Raw*: `CAN_ID_0x123, byte2=1`
    *   *Tool*: `set_window(seat='driver', action='open')`
*   **接口标准化**：所有工具必须有明确的 OpenAPI 格式定义（Schema），包括参数类型、取值范围和描述，以便 LLM 理解。

### 4.3.4 共享状态层 (Shared State Layer)：神经系统
**这是驾舱一体的“命门”**。
*   **现状问题**：App设置了导航，车机不知道；语音改了目的地，AD 还在按旧路线跑。
*   **解决方案**：建立 **Unified State Machine (USM)**。
    *   所有子系统（Nav, AD, Media）通过 Pub/Sub 模式（如 MQTT/DDS）向总线汇报状态。
    *   Agent 不直接查询底层硬件，而是查询 USM 中的状态快照（State Snapshot）。

### 4.3.5 安全与策略层 (Policy Layer)：免疫系统
**设计原则：Rule-of-Thumb 4.1 —— LLM 负责“建议”，代码负责“批准”。**
*   **拦截器模式**：所有来自 Agent 的 Tool Call 在执行前，**强制**经过此层。
*   **关键检查项**：
    1.  **Schema Check**: 参数是否符合定义？（防止 LLM 编造参数）
    2.  **Permission Check**: 当前说话人（声纹识别结果）是否有权执行此操作？（e.g., 后排儿童禁止开关车门）
    3.  **Safety Check**: 当前车辆状态是否允许？（e.g., 车速 > 10km/h 禁止开启后备箱；AD 模式下修改目的地需二次确认）。
    4.  **Confirmation Injection**: 如果判定为高风险，策略层挂起执行，返回 `NeedConfirmation` 事件，触发语音/UI 询问用户。

---

## 4.4 关键数据对象（统一状态模型）

为了让 LLM 能够“理解”物理世界，必须定义清晰的数据结构。

### 4.4.1 Trip State (行程状态)
```json
{
  "trip_id": "uuid_v4",
  "phase": "NAVIGATING", // PLANNING, NAVIGATING, ARRIVED, PAUSED
  "destination": {
    "poi_id": "map_provider_id",
    "name": "静安嘉里中心",
    "coordinate": {"lat": 31.22, "lon": 121.45},
    "entry_point": "P3停车场入口" // E2E AD 需要精确的入口坐标
  },
  "route_constraints": {
    "avoid_highway": false,
    "avoid_congestion": true,
    "energy_optimized": true
  },
  "ad_handover_status": "AUTO_CRUISE" // AD系统当前对路线的执行状态
}
```

### 4.4.2 Vehicle State (车辆融合状态)
```json
{
  "motion": {
    "speed": 65.5,
    "gear": "D",
    "heading": 180.0
  },
  "cabin": {
    "temperature": 24.0,
    "windows": {"fl": 0, "fr": 0, "rl": 100, "rr": 100}, // 0=closed
    "occupancy": ["driver", "passenger_rr"] // 座椅占用传感器
  },
  "autonomy": {
    "level": "L2_PLUS",
    "available_modes": ["LCC", "NOA"],
    "confidence": 0.85 // AD系统的自信度，低时语音提示接管
  }
}
```

### 4.4.3 Conversation Context (会话上下文)
```json
{
  "session_id": "sess_123",
  "turns": 5,
  "active_intent": "navigate_with_stopover",
  "slot_buffer": {
    "destination": "机场",
    "stopover": null // 等待填充
  },
  "last_system_action": "ask_clarification" // "请问是去虹桥还是浦东？"
}
```

---

## 4.5 与自动驾驶 (AD) 栈的边界与接口

端到端（E2E）自驾模型（Vision -> Action）通常是一个黑盒。座舱不能干预其体的转向角度，但可以进行**宏观意图注入**。

### 接口清单 (Interface Specifications)

| 方向 | 接口名称 | 描述 | 数据载体 | 频次 |
| :--- | :--- | :--- | :--- | :--- |
| **座舱 -> AD** | `SetTargetTrajectory` | 设定宏观导航路径点（Waypoints） | Protobuf / ROS msg | 事件触发 |
| **座舱 -> AD** | `SetDrivingStyle` | 设定驾驶风格（激进/保守） | Enum {AGGRESSIVE, COMFORT} | 事件触发 |
| **座舱 -> AD** | `TriggerAction` | 触发特定战术动作（如：换道、泊入） | Enum {CHANGE_LANE_LEFT, PARK_IN} | 事件触发 |
| **AD -> 座舱** | `ReportEgoState` | 上报车辆位姿、速度、加速度 | Struct | 50Hz (高频) |
| **AD -> 座舱** | `ReportEnvironment` | 感知到的环境（用于 SR 渲染，人车路） | List<Object> | 30Hz |
| **AD -> 座舱** | `ReportDecisionReason` | **(关键)** 解释为何做此动作（如：为何减速？） | Enum/String ("Obstacle", "TrafficLight") | 变化时触发 |

> **Gotcha**: AD 系的 `ReportDecisionReason` 对“对话上下文导航”至关重要。如果车突然停了，用户问“怎么了？”，座舱必须能从 AD 获取到“检测到前方有行人”的状态，而不是回答“我不知道”。

---

## 4.6 端云协同与容灾策略

| 能力 | 部署位置 | 策略描述 | 异常（断网）预案 |
| :--- | :--- | :--- | :--- |
| **ASR** | Hybrid | 优先云端（高精度），端侧并行流式识别（低延迟）。 | 自动降级为纯端侧 ASR，词库受限。 |
| **NLU/Agent** | Hybrid | 复杂推理走云端 LLM；简单指令走端侧 Parser/SLM。 | 启用端侧规则引擎，仅支持固定指令集（导航回家、空调、窗户）。 |
| **Map/Search** | Cloud | POI 搜索依赖云端实时数据。 | 降级为端侧离线地图搜索（数据陈旧，无路况）。 |
| **Policy Guard** | **Edge Only** | 安全校验必须在本地，不可依赖网络。 | N/A (本地必须始终可用)。 |
| **AD Planning** | **Edge Only** | E2E 模型在 Orin/Thor 等算力平台上运行。 | N/A。 |

---

## 4.7 可观测性与回放 (Observability)

为了支持 P4/P5 阶段的排查，架构必须内置 Trace 能力。

*   **Trace ID**: 生成唯一的 `trace_id`，贯穿 `ASR` -> `NLU` -> `Policy` -> `AD Interface`。
*   **Shadow Mode (影子模式)**: 在云端回放用户的语音指令，对比新旧版本模型的规划结果，用于迭代评估。
*   **Blackbox Data**: 关键的“确认”动作（用户点击 Yes）和 AD 的执行日志必须同步落盘，用于事故定责。

---

## 4.8 本章小结

1.  **分层解耦**：架构分为交互层、Agent编排层、Policy安全层、执行层。
2.  **安全核心**：**Policy Layer** 是系统的核心防火墙，任何 AI 生成的决策在执行前必须经过确定性的代码校验。
3.  **状态统一**：通过 **Unified State Machine** 消除座舱与自驾的信息差。
4.  **端云互补**：云端负责“聪明”（复杂语义），端侧负责“快”和“稳”（安与高频）。

---

## 4.9 练习题

### 基础题
1.  **[架构]** 为什么我们需要将 Agent 的“路由（Router）”功能区分 Fast Path 和 Slow Path？全部用大模型不可以吗？
    <details>
    <summary>Hint</summary>
    考虑一下开窗户这种操作如果延迟3秒用户会怎么想？再考虑一下成本。
    </details>
    <details>
    <summary>参考答案</summary>
    <b>答案：</b>
    1. <b>时延体验</b>：控车类指令（如开窗、静音）要求 < 500ms 的响应，云端 LLM 通常需要 1-3秒，体感无法接受。<br>
    2. <b>稳定性</b>：网络抖动不应影响基础功能。<br>
    3. <b>成本</b>：高频简单的指令使用大模型是算力和费用的浪费。
    </details>

2.  **[接口]** 在 `Trip State` 对象中，为什么需要包含 `ad_handover_status`（AD交接状态）？
    <details>
    <summary>Hint</summary>
    如果导航说“到达目的地”，但 AD 还在以 60km/h 巡航，会发生什么？
    </details>
    <details>
    <summary>参考答案</summary>
    <b>答案：</b> 导航软件的逻辑状态（Navigating/Arrived）与 AD 系统的物理控制状态（Engaged/Disengaged）需要同步。例如，当导航即将结束时，Agent 需要知道 AD 是否处于接管状态，如果是，需要提前通知 AD 系统准备“泊车”或“靠边停车”，而不是直接结束导航导致 AD 突然退出或漫无目的巡航。
    </details>

3.  **[数据]** `Vehicle State` 中的 `confidence`（置信度）字段对于座舱交互有什么具体作用？
    <details>
    <summary>Hint</summary>
    人车共驾时的信任建立。
    </details>
    <details>
    <summary>参考答案</summary>
    <b>答案：</b> 用于通过 HMI 管理用户预期。当 AD 置信度高时，座舱保持静默或仅显示蓝色光效；当置信度下降（如在大雨天或标线不清），座舱应通过语音或视觉（黄色/红色）提前提示驾驶员“请注意路况”或“准备接管”，实平滑的人机共驾。
    </details>

### 挑战题

4.  **[场景设计]** 场景：用户正在 NOA（自动导航辅助驾驶）模式下行驶。突然说：“前面那辆大车太慢了，超过去。”
    请详述数据流向：从语音输入到 AD 执行变道，中间经过了哪些层？Policy Layer 在这里要做什么检查？
    <details>
    <summary>Hint</summary>
    这不仅仅是一个导航指令，这是一个战术指令。
    </details>
    <details>
    <summary>参考答案</summary>
    <b>流程分析：</b><br>
    1. <b>Interaction</b>: ASR 转录文本。<br>
    2. <b>Agent</b>: LLM 识别意图 `vehicle_control`，Action=`overtake_vehicle`，Target=`front_vehicle`。<br>
    3. <b>Policy (关键)</b>:
       - 检查当前 AD 模式是否支持主动变道指令（L2+支持，L2不支持）。
       - 检查当前车速与周围环境（通过 AD Perception 接口查询左侧车道是否有车）。
       - <b>风险判定</b>: 这是一个中高风险动作。<br>
    4. <b>Decision</b>:
       - 如果左侧安全：通过接口 `TriggerAction(CHANGE_LANE_LEFT)` 发送给 AD。
       - 如果左侧不安全：Policy 拦截，返回失败原因。TTS 播报“左侧有车，暂时无法变道”。<br>
    5. <b>Execution</b>: AD 系统执行变道轨迹规划。
    </details>

5.  **[架构演进]** 当前架构是 E2E AD + LLM Cockpit。未来如果升级为 **End-to-End Driving with Language (VLA - Vision Language Action)** 模型（即一个大模型同时看路和听懂人话），架构层级中最先被淘汰或合并的是哪一层？为什么？
    <details>
    <summary>Hint</summary>
    现在的架构是因为“脑子”和“小脑”是分开的。
    </details>
    <details>
    <summary>参考答案</summary>
    <b>答案：</b> 最先被合并的是 <b>Agent 编排层与 AD 规划层的边界</b>，以及 <b>交互层的语义转译</b>。
    <br>在 VLA 架构下，视觉 token 和语言 token 进入同一个 Transformer，模型直接输出驾驶动作。此时，独立的“工具调用（Tool Calling）”和显式的“意图识别”将不再需要，因为语言直接调节了驾驶的 Latent Space。但 <b>Policy & Safety Guard</b> 层永远不能淘汰，它必须作为独立于 AI 黑盒之外的“安全监控器（Monitor）”存在。
    </details>

---

## 4.10 常见陷阱与错误 (Gotchas)

| 陷阱 (Gotcha) | 现象描述 | 调试/规避技巧 |
| :--- | :--- | :--- |
| **Idempotency Fail (幂等性缺失)** | 用户因为网络卡顿连说两次“打开后备箱”，结果系统打开后又试图关闭，或者报错。 | 工具层必须实现幂等。`open_trunk()` 如果已经是 open 状态，应直接返回 Success，而不是执行 toggle 动作。 |
| **Race Condition (状态竞争)** | 语音指令“去公司”刚下发，用户手动在屏幕点击“取消”。语音指令处理慢了一拍，在取消后又把导航开启了。 | 引入 **Lamport Timestamps** 或简单的序列号（Sequence ID）。执行层只接受比当前状态 ID 更新的指令，丢弃过期的指令。 |
| **Context Thrashing (上下文抖动)** | LLM 记住了一小时前乘客随口说的“我想喝咖啡”，在驾驶员说“导航回家”时，错误地把咖啡店设为途经点。 | **Session Management** 策略至关重要。定义明确的 Session 结束边界（如：车辆熄火、用户明确说“退出”、或长时间无交互）。Trip 结束后自动清除短期偏好。 |
| **Over-Guarding (过度防御)** | Policy 层设置太严，导致 LLM 很聪明，但车子看起来很笨（什么都不让做）。 | 建立 **Policy 分级制度**。
- Level 1 (Info): 随便调（切歌、查天气）。
- Level 2 (Comfort): 限制范围（空调、座椅）。
- Level 3 (Safety): 严格校验 + 二次确认（导航变更、驾驶模式）。 |

---
**Rule-of-Thumb 4.2**: *在驾舱一体架构中，唯一可信的真理是“车辆物理状态”和“Policy 校验结果”。LLM 的输出只能被视为“请求（Request）”而非“指令（Command）”。*
