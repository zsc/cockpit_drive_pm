# Chapter 7｜驾舱一体的“统一代理”与共享状态设计

## 1. 开篇段落

在“驾舱分离”的时代，座舱（IV）和智驾（AD）就像两个居住在同一屋檐下却语言不通的室友：座舱不知道智驾此时正在处理紧急避让，还在大声播报娱乐新闻；智驾不知道用户刚刚在座舱里抱怨“想找个厕所”，依然按照既定路线死板行驶。

**驾舱一体**的核心，不是将两套代码合并，而是建立**统一的意图理解代理（Unified Agent）**和**单一事实来源（Single Source of Truth）**。对于产品经理和架构师而言，本章的目标是设计一套机制，使得**用户的意图（Intent）**能无损地转化为**车辆的行动（Action）**，同时**车辆的状态（State）**能实时地调优**用户的体验（Experience）**。我们将重点讨论如何解耦架构、定义共享状态的数据结构、处理多模态冲突以及设计高可用的端云协同策略。

---

## 2. 文字论述

### 7.1 一体化边界：统一哪些、解耦哪些

**Rule-of-Thumb**: **控制面（Control Plane）统一，数据面（Data Plane）隔离。**

驾舱一体并不意味着我们要把 Android（座舱）和 RTOS（智驾）合并到一个操作系统中。相反，我们需要在它们之间架设一座高效的桥梁。

*   **必须解耦（隔离）**：
    *   **算力**：AD 芯片跑感知规控（保命），IV 芯片跑模型渲染（体验）。
    *   **操作系统**：AD 使用高实时性 OS（QNX/Linux RT），IV 使用富生态 OS（Android/Linux）。
    *   **传感器数据流**：激光雷达/摄像头的原始庞大数据直接进 AD，不经过 IV（避带宽阻塞）。
*   **必须统一（融合）**：
    *   **Trip Agent（行程代理）**：用户面对的只有一个“管家”，而不是“导航助手”和“控车助手”两个。
    *   **状态总线（State Bus）**：关于“车在哪、车在干嘛、人想干嘛”的数据必须只有一份拷贝。
    *   **交互策略（HMI Policy）**：AD 的报警级别必须能直接抑制 IV 的娱乐音量。

```ascii
[ User ] --(Voice/Touch)--> [ IV Domain (Android) ] <=====(Ethernet/DDS)=====> [ AD Domain (RTOS) ]
                                   |                                                 |
+----------------------------------|-------------------------------------------------|----------------+
|                          LOGICAL ARCHITECTURE: THE UNIFIED BRIDGE                                   |
|                                                                                                     |
|  1. Unified Trip Agent (Brain)       2. Shared State Bus (Spine)       3. Policy Guard (Shield)     |
|  +-------------------------+        +---------------------------+      +-------------------------+  |
|  | - Intent Understanding  | <----> | [Topic: Vehicle_Status]   | <--> | - Safety Constraints    |  |
|  | - Task Planning         |        | [Topic: Trip_Progress]    |      | - Arbitration Rules     |  |
|  | - Tool Routing          |        | [Topic: User_Context]     |      | - Workload Manager      |  |
|  +-------------------------+        +---------------------------+      +-------------------------+  |
+-----------------------------------------------------------------------------------------------------+
```

### 7.2 统一 Trip Agent 责任划分

Trip Agent 是运行在座舱域（或云端协同）的一个逻辑实体，它负责协调“对话”与“驾驶”。

*   **输入层**：接收多模态输入（语音指令、视线追踪、屏幕点击）。
*   **状态层**：维护 `Conversation Session`（对话上下文）和 `Trip Session`（行程全生命周）。
    *   *区别*：对话可能结束了，但行程还在继续（如导航中）。Agent 需要在后台持续监控行程状态。
*   **决策层**：
    *   **路由（Routing）**：判断指令是查天气（云端工具）、调空调（车控工具）还是改路线（AD 工具）。
    *   **参数提取**：将自然语言“避开那个拥堵路段”转化为 AD 可理解的结构化参数 `avoid_area_id` 或 `replan_strategy=fastest`。
*   **执行层**：调用具体的 Tool Executor，并追踪执行结果（Pending -> Running -> Success/Fail）。

### 7.3 共享状态同步机制 (Shared State)

这是驾舱一体的灵魂。我们必须定义标准化的数据契约（Schema），通常通过 DDS（Data Distribution Service）或 SOME/IP 实现发布/订阅。

#### 关键状态对象模型

| 对象域 | 状态名 (Key) | 说明与示例 | 生产者 (Writer) | 消费者 (Reader) |
| :--- | :--- | :--- | :--- | :--- |
| **Trip (行程)** | `Target.Destination` | 最终目的坐标、POI ID | IV (Map Engine) | AD (Planning) |
| | `Target.Waypoints` | 途经点列表 | IV | AD |
| | `Route.ETA` | 预计到达时间 | IV/AD | IV (UI显示) |
| | `Route.Constraints` | 偏好（不走高速、省电优先） | IV (User Setting) | AD (Planning) |
| **Vehicle (车辆)** | `AD.Mode` | Manual, ACC, LCC, NOA, Valet | AD | IV (UI/LLM) |
| | `AD.Status` | Available, Active, Handover_Req | AD | IV (HMI) |
| | `Chassis.Speed` | 车速 (km/h) | Chassis | All |
| **Context (上下文)**| `User.LastIntent` | "FindRestaurant" | IV (LLM) | IV |
| | `System.Confirmation`| Pending_Confirm (等待用户说是/否) | IV | IV |
| **HMI (交互)** | `Driver.Workload` | Low, Med, High, Critical | AD (Perception) | IV (Policy) |

### 7.4 冲突与仲裁 (Arbitration & Priority)

当 AD 系统和 IV 系统同时发出指令，或者用户指令与车辆安全状态冲突时，必须有仲裁机制。

**优先级金字塔（由高到低）：**
1.  **Safety Guard (安全红线)**：AEB 触发、车辆控、底盘故障。
    *   *动作*：强制接管屏幕显示警报，切断所有娱乐语音，拒绝所有非安全相关的用户指令。
2.  **Driver Physical Action (驾驶员物理操作)**：转动方向盘、踩刹车。
    *   *动作*：立即中断 AD 的自动巡航，中断语音助手的“正在为您变道”操作（如果还在规划阶段）。
3.  **Explicit Command (明确指令)**：用户点击屏幕上的“取消导航”或语音明确说“退出”。
4.  **Implicit/Smart Suggestion (智能建议)**：LLM 主动建议“探测到疲劳，建议去服务区”。
    *   *动作*：仅在低负载时弹出，不强制执行。

### 7.5 “驾驶状态”驱动交互策略 (Workload Management)

驾舱一体的体验不仅是“人控车”，更是“车懂人”。AD 系统通过传感器感知环境复杂度，计算**驾驶员负荷（Workload）**，实时调整座舱的交互策略。

*   **场景 A：高速巡航，路况良好 (Workload = Low)**
    *   **LLM 策**：开放式闲聊，允许长文本播报，允许视频播放，允许复杂的点餐推荐交互。
*   **场景 B：大雨天，复杂路口，AD 信心不足 (Workload = High)**
    *   **LLM 策略**：**静默模式（Inhibition）**。
    *   暂停非紧急的语音播报（如短信朗读）。
    *   屏幕弹窗冻结（不弹无关通知）。
    *   如果用户此时发起语音请求，回复应极简：“正在通过复杂路口，请稍后。”
*   **场景 C：接管报警 (Workload = Critical)**
    *   **LLM 策略**：完全静音。音频通道被 AD 报警音独占。

### 7.6 跨域联动模板

为了减少“意大利面条式”的代码，我们定义标准的跨域联动流程（SOP）。
**案例：用户说“我饿了”（Service -> Map -> AD）**

1.  **Trigger**：IV 收到语音，LLM 解析意图 `Find_Food`。
2.  **Service Domain**：调用美团/大众点评 API，根据当前位置推荐 Top 3 餐厅。
3.  **Interaction**：IV 展示卡片，用户选择“第二个”。
4.  **Action 1 (Map)**：IV 将餐厅位置设为目的地，发起路线规划，计算 ETA。
5.  **Action 2 (AD)**：用户语音确认“出发”。Trip Agent 将路线坐标下发至 AD 的 `Planning_Interface`。
6.  **Action 3 (Body)**：Trip Agent 检查到餐厅较远（>1小时），自动生成建议“需要为您开启座椅按摩吗？”

### 7.7 隐私与账号体系

*   **多音区与权限**：共享状态中必须包含 `Source_Seat_ID`。
    *   驾驶员（Driver）：有权修改导航、驾驶模式。
    *   后排乘客（Passenger）：仅有权修改音乐、自身座椅、或“建议”目的地（需驾驶员确认）。
*   **数据隔离**：AD 采集的行车视频（Dashcam）属于敏感数据。未经用户授权，LLM 不得读取视频内容进行分析（如“刚才路边那是谁”）。

### 7.8 性能与可用性

*   **端云协同（Hybrid Architecture）**：
    *   **Cloud LLM**：处理复杂推理、长上下文、非实时查询（如这款车怎么换雨刮水”）。
    *   **Edge LLM/Rule**：处理高频控车（“打开车窗”）、核心导航指令（“取消导航”）。
    *   **降级策略**：当网络 RTT > 1000ms 或断网时，Trip Agent 自动切断云端链路，仅使用车端离线引擎。此时共享状态总线仍在车内局域网正常工作，保证“无网也能控车”。
*   **冷启动**：车辆上电（Power On）瞬间，共享状态需从非易失存储（NVRAM）恢复关键信息（如上次导航是否未完成），确保体验连续性。

### 7.9 全链路可追溯 (Traceability)

为了调试“为什么车没听我的”，需要建立跨域的 Trace ID。
*   **SessionID**：一次语音会话的 ID。
*   **SpanID**：
    *   Span A: ASR 识别结果。
    *   Span B: LLM 意图解析结果。
    *   Span C: 工具调用参数（Tool Call）。
    *   Span D: AD 系统接收到的指令快照。
    *   Span E: AD 执行结果或拒绝原因（Reason Code）。
*   **要求**：所有子系统日志必须携带 Trace ID，上传至云端日志平台进行关联分析。

---

## 3. 本章小结

*   **架构哲学**：物理上分离（保证安全与性能），逻辑上统一（保证体验）。
*   **三大支柱**：**统一 Trip Agent**（大脑）、**共享状态总线**（神经）、**仲裁与安全策略**（免疫系统）。
*   **核心机制**：通过定义清晰的 Vehicle/Trip/Context 状态模型，实现 AD 与 IV 的数据握手；通过驾驶负荷（Workload）管理，实现从“人适应车”到“车适应人”的转变。
*   **底线**：安全永远高于智能。任何 LLM 的输出在进入 AD 执行层前，必须经过确定性的校验与仲裁。

---

## 4. 练习题

### 基础题 (50%)

**Q1. 简述“共享状态（Shared State）”中，AD 域和 IV 域分别作为“生产者（Producer）”的主要数据内容是什么？**

<details>
<summary>点击查看参考答案</summary>

*   **AD 域生产**：车辆物理状态（速度、位、车门）、自动驾驶模式（ACC/NOA状态）、驾驶员负荷状态（Workload Level）、感知到的环境风险（Risk Level）。
*   **IV 域生产**：用户意图（导航目的地、途经点）、偏好设置（避开高速）、用户指令（变道请求）、媒体播放状态（用于降低音量）。
</details>

**Q2. 为什么在驾舱一体架构中，不建议将激光雷达（LiDAR）的点云数据直接写入共享状态总线供座舱使用？**

<details>
<summary>点击查看参考答案</summary>

*   **带宽占用**：点云数据量极大，会阻塞 DDS/以太网总线，影响控制指令的实时性。
*   **处理能力**：座舱芯片（如 8295）虽然强，但主要用于渲染，不适合处理原始感知数据。
*   **架构原则**：共享状态应仅包含“逻辑状态”和“元数据”。如果座舱需要显示感知道路（SR），AD 应处理完后输出精简的 Object List（对象列表）给座舱渲染，而非原始数据。
</details>

**Q3. 在“驾驶负荷管理”中，当 AD 检测到 Workload = High（高负荷）时，座舱交互应该发生什么变化？请举例。**

<details>
<summary>点击查看参考答案</summary>

*   **抑制主动交互**：不主动推送非紧急通知（如“我想到了一个笑话”、“这里有家好吃的”）。
*   **简化被动响应**：用户问询时，回答简短有力，不念长文。
*   **视觉降噪**：关闭复杂的动效，保留核心仪表信息。
*   **音频独占**：暂停音乐或将其压低（Duck），确保导航或安全提示音清晰。
</details>

### 挑战题 (50%)

**Q4. 场景设计：用户在 NOA 状态下通过语音要求“向左变道超车”，但 AD 系统检测到左后方有高速来车，不具备变道条件。请利用本章的架构知识，描述完整的处理与反馈流程。**

<details>
<summary>点击查看参考答案</summary>

1.  **Intent**：IV 解析意图 `Action: Lane_Change`, `Direction: Left`。
2.  **Request**：Trip Agent 将请求写入共享状态 `Request.LaneChange = Left`。
3.  **Check (AD)**：AD 的规划模块（Planning）读取请求，结合感知数据进行安全性校验（Safety Check）。
4.  **Reject**：AD 判定风险过高，拒绝请求，更新共享状态 `Response.LaneChange = Rejected`, `Reason = Left_Rear_Traffic`.
5.  **Feedback (IV)**：Trip Agent 收到拒绝状态和原因。
6.  **Generation**：LLM 生成自然语言回复（结合原因）：“左后方有来车，暂时无法变道，安全后我会提示您。”（或保持静默，视策略而定）。
</details>

**Q5. (开放题) 随着端到端（End-to-End）自动驾驶模型的发展，未来的 AD 系统可能不再输出明确的“规划轨迹”，而是直接输出“控制信号”。这对目前的“共享状态架构”带来了什么挑战？如何调整？**

<details>
<summary>点击查看提示</summary>

*   **挑战**：当前的架构依赖 AD 告诉 IV “我打算怎么走”（Waypoints），以便 IV 在地图上画线给用户看。如果 E2E 模型是黑盒，IV 无法获取未来轨迹，导致“所见即所得”失效（用户不知道车要往哪开）。
*   **调整思路**：
    *   **World Model 解码**：要求 E2E 模型不仅输出 Control，还要输出其内部的 World Model 预测轨迹用于可视化（即专门训练一个用于可视化的 Head）。
    *   **置信度可视化**：座舱不再显示确定的蓝线，而是显示“意图光束”或“概率云”，表达 AI 的行驶意向。
    *   **架构变更**：共享状态中增加 `Predicted_Intent_Trajectory` 字段，专门用于 HMI 渲染，允许其与实际控制有微小偏差。
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 幽灵状态 (Ghost State / Desync)
*   **现象**：座舱大屏显示“导航已取消”，但 HUD 还在显示箭头，或者车还在按原路线跑。
*   **原因**：IV 和 AD 各自维护了一份状态，且没有强制同步。IV 改了自己的内存，没发给 AD；或者 AD 发了更新，IV 没收到。
*   **调试技巧**：确保实现**ACK（确认）机制**。IV 发出“取消”指令后，UI 显示“正在取消...”，直到收到 AD 返回的 `Status=Idle` 确认包后，才更新为“已取消”。

### 5.2 循环触发 (Feedback Loop)
*   **现象**：AD 减速 -> 座舱检测到减速 -> LLM 询问“为什么减速” -> AD 收到询问占用资源 -> AD 进一步减速。
*   **原因**：状态监听没有设置阈值或死区（Deadband）。
*   **对策**：在 Trip Agent 中设置**事件过滤器**。只有状态发生显著变化（如 ETA 变化 > 5分钟，速度变化 > 20km/h）或状态码跳变时，才触发 LLM 推理，避免高频抖动触发。

### 5.3 令牌消耗失控 (Token Burn)
*   **现象**：将车辆的所有实时传感器数据（如每秒 10 次的坐标刷新）都塞进 LLM 的 Prompt Context 中。
*   **后果**：Token 消耗巨大，且模型响应变慢。
*   **对策**：**状态快照（Snapshot**。LLM 仅在需要生成回复的那一刻，去共享状态总线拉取最新的**一次**快照。不要把状态流作为 Prompt 输入。

### 5.4 权限越界 (The "Valet" Attack)
*   **现象**：代客泊车模式（Valet Mode）下，泊车小哥通过语音助手查看了车主的历史导航记录或打开了手套箱。
*   **对策**：共享状态中引入 `Security_Context`。当 `AD.Mode == Valet` 时，Trip Agent 自动加载“受限策略”，屏蔽涉及隐私和资产安全的 Tool Calling。
