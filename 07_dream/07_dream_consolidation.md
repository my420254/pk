# 07_dream：空闲时的记忆整理

> **文件归属**：[`simpleclaw/dream/protocol.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/protocol.py)、[`scheduler.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/scheduler.py)、[`executor.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/executor.py)、[`policy.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/policy.py)、[`signal.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/signal.py)、[`store.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/store.py)  
> **模块定位**：空闲时整理记忆。用户正在说话就推迟。产物类型写在协议里，包括偏好、教训和技能草稿。草稿不会自动变成已加载的 `SKILL.md`。  
> **推荐阅读顺序**：**第 7 篇**（记忆账本之后。这是后台整理，不是主回复的一部分。）

---

## 一、整理记忆要让着正在说话的用户

每轮都把原文塞进检索，时间一长会堆成流水账：今天用了 A，明天换了 B，后天觉得 B 油。模型看得到细节，归纳不出「这个人屏障弱，先不要强酸」。工具参数填错了，如果没有地方记下这次教训，下次还是靠模型临场发挥。

定时任务完全可以先问「会话忙不忙」再跑。我们把这个判断写成准入策略：用户在说话或会话忙，整理就推迟或取消。两次成功整理之间默认至少隔 30 分钟（`min_interval_ms = 30 * 60 * 1000`）。同一作用域不要重入。

产出是结构化结果，例如偏好、失败教训、工具纠错、技能草稿，不是只打一段总结日志。生产环境走任务队列；本地可以直接把已就绪的任务排空，方便调试。

| 要解决的事 | 我们的默认做法 |
| :--- | :--- |
| 整理不能打断用户正在进行的对话 | 会话忙或用户新消息到了，这次整理不跑，或者协作取消。 |
| 不要空闲时连着打模型 | 默认 30 分钟冷却，加上同一作用域互斥。 |
| 整理结果要能进后面的记忆和技能，而不是一段没法用的摘要 | 写成明确类型的产物，再决定写回记忆、纠正工具，或留下技能草稿。 |
| 线上和本地不要两套逻辑 | 同一套任务。线上进 Redis Streams，本地可以自己排空。 |

### 面试时可以怎么说

> 睡梦是空闲时的记忆整理，不是主回复的一部分。用户在说话就让路。默认 30 分钟内不重复跑。它要把流水账收成偏好、教训这种能写回的结果。Cron 也能先判断会话忙不忙，我们是把让路、冷却和产物类型写成这一条策略。

---

## 二、核心类与数据协议全景

### 1. 梦境实体流转契约（定义于 [`simpleclaw/dream/protocol.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/protocol.py)）

#### (1) 七大触发契机 (`DreamTrigger`)
- `"idle_session"`: 会话超时闲置（如用户 30 分钟未发言）。
- `"memory_threshold"`: 记忆因果账本积压超阈值，需深度合并。
- `"runtime_task_completed"`: 复杂长耗时任务执行完毕。
- `"subagent_completed"`: 垂直调研子智能体汇报完成。
- `"cron"`: 定时触发。源码注释里的「凌晨 3 点」是例子，不是已经排好的任务。
- `"manual"`: 管理员或专家显式触发。
- `"system_monitor"`: 系统巡检发现数据异常。

#### (2) `DreamCandidate`（候选人）
信号到达后的第一站实体。计算去重键 `dedupe_key = f"{scope_key}:{trigger}:{source}"`，交给准入海关审批。

#### (3) `DreamJob`（正式梦境任务工单）
准入放行后生成的正式工单，携带分布式全局追踪 `trace_id`。可通过 `to_task_envelope()` 无缝转化为底层 `TaskEnvelope`，投递至后台队列。

#### (4) 七大认知成果物 (`DreamArtifactType`)
```python
DreamArtifactType = Literal[
    "memory_summary",       # 全局长期记忆综合摘要 (消解矛盾后的纯净状态)
    "preference_profile",   # 提炼的用户长期偏好画像 (如：对酒精过敏、喜爱轻盈乳液)
    "task_lesson",          # 复杂长任务规划与分解经验
    "tool_usage_lesson",    # 工具调用失败复盘与参数修正指南
    "failure_lesson",       # 业务严重故障教训复盘
    "skill_candidate",      # 协议里的技能草稿类型。不会自动写入 SKILL.md
    "context_cleanup",      # 上下文冗余垃圾信息剔除指令
]
```

---

## 三、准入风控海关与执行器

### 1. `DreamAdmissionPolicy`（准入风控策略）
定义于 [`simpleclaw/dream/policy.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/policy.py)：
默认让着正在说话的用户。`force=True` 时策略直接放行，不再看会话忙不忙：

```mermaid
flowchart TD
    Signal["梦境候选人到达 (DreamCandidate)"] --> CheckForce{"force == True?<br/>(管理员强制触发)"}
    CheckForce -- 是 --> Allow["签署批准 (Admitted)"]
    CheckForce -- 否 --> CheckBusy{"会话正忙?<br/>(session_busy == True)"}
    
    CheckBusy -- 是 --> Deny1["拒签 (session_busy)"]
    CheckBusy -- 否 --> CheckUser{"用户新消息到达队列?<br/>(user_ingress_pending)"}
    
    CheckUser -- 是 --> Deny2["拒签 (user_ingress_pending)"]
    CheckUser -- 否 --> CheckRunning{"本作用域已有做梦任务正在运行?<br/>(scope_already_running)"}
    
    CheckRunning -- 是 --> Deny3["拒签 (scope_already_running)"]
    CheckRunning -- 否 --> CheckCooldown{"距上次成功做梦 < 30分钟?<br/>(cooldown 保护)"}
    
    CheckCooldown -- 是 --> Deny4["拒签 (cooldown)"]
    CheckCooldown -- 否 --> Allow
```

### 2. `DreamExecutor`（双轨执行协调器）
定义于 [`simpleclaw/dream/executor.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/dream/executor.py)：
- **队列里的执行 (`execute`)**：消费 Redis Streams 上的 `TaskEnvelope`。业务异常记成 `failed` 并写下 `last_error`。接住异常不能避免进程被杀掉，OOM 或 SIGKILL 仍会中断这次执行。
- **本地自排空模式 (`drain_ready`)**：在单测或本地简易部署中，就地排空执行已准入工单，零外部 MQ 依赖。

---

## 四、全场景实战时序图（Mermaid 8.14.0+ 规范）

### 场景 1：会话空闲触发准入并生成正式工单流（空闲自整理）

展示用户离开 30 分钟后，系统自动触发整理并向分布式队列派发工单：

```mermaid
sequenceDiagram
autonumber
participant Monitor as 会话巡检器
participant Sched as DreamScheduler
participant Policy as AdmissionPolicy
participant Store as DreamStore
participant Queue as 任务队列/Worker

Note over Monitor, Sched: 探测到用户 30 分钟无交互
Monitor->>Sched: submit_signal(trigger="idle_session", session_key="s123")
Sched->>Store: 创建候选人记录 DreamCandidate(status='candidate')

Note over Sched, Policy: 提交准入海关评估
Sched->>Policy: evaluate(candidate, facts)
Policy->>Policy: 校验事实: 会话空闲, 无新用户输入, 超过30分钟冷却期
Policy-->>Sched: 签署准入裁决 DreamAdmissionDecision(allowed=True)

Note over Sched, Queue: 升华生成正式 Job 并投递分布式队列
Sched->>Store: 升级为正式工单 DreamJob(status='admitted')
Sched->>Queue: submit_task(job.to_task_envelope())
Queue-->>Sched: 返回 queue_id="task_dream_99"
Sched->>Store: 更新工单状态为 queued (等待离线 Worker 消费)
```

---

### 场景 2：准入海关拦截驳回流（会话忙碌 / 冷却时间内拒签）

展示当用户正在对话或刚做完梦不久时，准入海关如何严防算力浪费与打扰用户：

```mermaid
sequenceDiagram
autonumber
participant Signal as 外部触发信号
participant Sched as DreamScheduler
participant Policy as AdmissionPolicy
participant Store as DreamStore

Note over Signal, Sched: 场景 A：会话正在忙碌中
Signal->>Sched: submit_signal(trigger="memory_threshold")
Sched->>Policy: evaluate(candidate, facts=[session_busy=True])
Policy-->>Sched: 拒签 DreamAdmissionDecision(allowed=False, reason="session_busy")
Sched->>Store: 标记候选人 status="skipped" (注记: 会话忙碌，主动让路)

Note over Signal, Sched: 场景 B：防抖冷却时间内 (短时间频繁触发)
Signal->>Sched: submit_signal(trigger="idle_session")
Sched->>Policy: evaluate(candidate, facts=[last_succeeded=5分钟前, min_interval=30分钟])
Policy-->>Sched: 拒签 DreamAdmissionDecision(allowed=False, reason="cooldown")
Sched->>Store: 标记候选人 status="skipped" (注记: 距离上次成功不足30分钟)
```

---

### 场景 3：双轨执行器与多类型成果物落盘流（认知提炼）

展示独立 Worker 如何消费工单，调用大模型提取用户偏好与工具教训并批量落盘：

```mermaid
sequenceDiagram
autonumber
participant Worker as 离线Worker
participant Exec as DreamExecutor
participant Store as DreamStore
participant LLM as 离线大模型推理
participant Mem as Memory存储

Worker->>Exec: execute(TaskEnvelope: dream_job)
Exec->>Store: update_job_status("dreamjob_01", status="running")

Note over Exec, LLM: 驱动业务算法提炼深层知识
Exec->>LLM: "复盘近期长对话，提取核心用户偏好与踩坑教训"
LLM-->>Exec: 返回结构化数据:<br/>1. 偏好画像: 对酒精重度过敏，禁用强酸<br/>2. 工具教训: query_weather 工具在无城市参数时必须默认传北京

Note over Exec, Store: 批量持久化认知资产 (DreamArtifact)
Exec->>Store: save_artifacts([<br/>Artifact(type="preference_profile", status="validated"),<br/>Artifact(type="tool_usage_lesson", status="validated")<br/>])

Note over Exec, Mem: 正式生效高价值记忆
Exec->>Mem: store(key="user.profile.allergy", content="重度酒精过敏，禁用强酸")
Exec->>Store: update_job_status("dreamjob_01", status="succeeded", completed=True)
Exec-->>Worker: 返回 TaskExecutionResult(succeeded=True)
```

---

### 场景 4：技能草稿类型（协议里有，没有自动装上）

`DreamArtifactType` 包含 `skill_candidate`。执行器没有把草稿写成 `SKILL.md`，也没有调用 `SkillRegistry` 注册。下面的图只说明这个类型长什么样，不是已上线的自进化。

```mermaid
sequenceDiagram
autonumber
participant Exec as DreamExecutor
participant LLM as 离线大模型推理
participant Store as DreamStore
participant Reg as SkillRegistry

Note over Exec, LLM: 睡梦反思分析近一周用户密集提问
Exec->>LLM: "分析会话记录，提取用户高频提问但系统缺乏标准 SOP 的领域"
LLM-->>Exec: 识别出重复套路: 用户频繁询问"刷酸后如何安全泛红急救"<br/>提炼出结构化技能草稿: skill_acid_emergency

Note over Exec, Store: 生成技能候选成果物
Exec->>Store: save_artifacts([Artifact(<br/>type="skill_candidate",<br/>payload={"skill_name": "acid_rescue_sop", "markdown_body": "## 刷酸急救四步法..."}<br/>)])

Note over Store, Reg: 当前代码没有这一步。草稿停在产物里，不会自动进技能目录。
```

---

### 场景 5：前台紧急用户消息打断与做梦任务协作式取消流

用户进线时，准入可以拒绝新的整理，运行中的任务可以协作取消。整理调用的是远程模型，取消的是这个任务，不是释放一块本机显卡。

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant Ingress as 会话网关
participant Sched as DreamScheduler
participant Exec as DreamExecutor
participant Store as DreamStore

Note over Exec: 后台 Worker 正在计算复杂的周报总结梦境任务...
User->>Ingress: 发送消息: "紧急！我脸颊涂了新乳液大面积红肿了！"

Note over Ingress, Sched: 嗅探到用户高优先级进线
Ingress->>Sched: notify_user_active(session_key="s123")
Sched->>Sched: 发现该 session 对应的梦境工单处于 running 状态！

Note over Sched, Exec: 发送协作打断信号 (Cancelation)
Sched->>Exec: cancel_job("dreamjob_01", reason="user_barge_in")
Exec->>Exec: 捕获 Cancelled 信号，安全终止大模型请求
Exec->>Store: update_job_status("dreamjob_01", status="cancelled")

Note over Ingress: GPU 与会话锁瞬间全量释放给用户前台会话！
Ingress-->>User: 用户这条先处理。整理任务如果还在跑，按让路规则推迟或取消
```
