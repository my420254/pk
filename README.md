<<<<<<< HEAD
# pk
个人知识库
=======
# SimpleClaw 文档：循环、上下文和周边模块怎么读

> 按源码把 SimpleClaw 的循环、上下文、工具和周边模块串起来。
> 每篇先讲要解决的问题，再对到具体代码。能讲清楚取舍就行，不要背「开源做不到」。

---

## 一、全局阅读路线图（标号学习序列：先看哪个，后看哪个）

学习一个现代智能体运行时，必须按照**“心跳驱动 -> 上下文组装 -> 模型推理 -> 工具治理 -> 状态与记忆 -> 高级异步睡梦与子智能体”**的递进链路阅读：

```
                        【智能体心跳中枢】
                 01_core (01_react_loop.md)
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
    【上下文装配引擎】              【大模型适配驱动】
 02_context (02_context.md)      03_llm (03_volcengine.md)
            │                               │
            └───────────────┬───────────────┘
                            ▼
                    【工具生命周期与门禁】
                    04_tools (04_tools.md)
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
     【短期会话与长期记忆】          【并发控制与任务状态机】
    05_memory (05_memory.md)      06_runtime (06_runtime.md)
            │                               │
            └───────────────┬───────────────┘
                            ▼
                    【异步睡梦蒸馏整理】
                    07_dream (07_dream.md)
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
     【子智能体多Agent协同】         【长任务工作项检查单】
   08_subagent (08_subagent.md)   09_workitem (09_workitem.md)
            │                               │
            └───────────────┬───────────────┘
                            ▼
                    【技能插件与SDK总线】
                  10_skills & 11_sdk_api
```

### 推荐阅读序列表

| 序号 | 模块目录 | 核心文件 / 模块 | 核心解决的问题与技术要点 | 推荐理由 |
| :---: | :--- | :--- | :--- | :--- |
| **01** | **`01_core/`** | `loop.py`, `messages.py`, `events.py` | ReAct 循环。打断丢掉还没落库的对话；`sync_write` 是 detach，入柜不经过 `SideEffectBuffer`。不用 LangGraph 是因为链路已经接在 `TurnRunner` 和前缀缓存上。 | **先看这篇**。后面的模块都挂在这圈循环上。 |
| **02** | **`02_context/`**| `builder.py`, `compressor.py`, `providers.py` | **上下文工程与滑动压缩**：系统提示词拆解、Token 动态预算评估（Mojing 2200/1000/600）、轮次边界保护性丢弃、注意力包注入 | 理解提示词是如何被动态渲染并在超限时平滑压缩的。 |
| **03** | **`03_llm/`** | `volcengine.py`, `adapters/` | 方舟前缀缓存：`previous_response_id` 先放进程字典，再放库。命中看 `cached_tokens`。`prompt_cache_key` 是文档里的路由提示，不保证同一台机器。 | 本地字典命中只是取出 id。创建前缀的耗时不要写成固定几百毫秒。 |
| **04** | **`04_tools/`** | `execution_runtime.py`, `registry.py`, `catalog.py`, `outbox.py` | **工具生命周期、执行运行时与队列二分治理**：JSON Schema 校验、强转自愈、两阶段持久化审计、动态权限审批、跨服务外发（Transactional Outbox）与内部工单（Redis Stream）治理 | 大模型如何安全、幂等、可控地调用外部工具，外部长任务如何可靠派发。 |
| **05** | **`05_memory/`** | `session.py`, `ledger.py`, `ledger_store.py` | 账本记下原文、消息序号、改前改后。提炼不堵在主回复里。证据检索默认还会再调一次模型挑条目。 | 不要写成不可篡改的 WAL，也不要写成热路径零模型。 |
| **06** | **`06_runtime/`** | `scope_lock.py`, `side_effects.py`, `services.py`, `task_queue.py`, `outbox.py` | 内部工单用 `python-redis-lock` 续期。用户这一轮仍是本进程队列。对外四类任务占行后发 Kafka。租户令牌桶用 Redis Lua。 | 分清工单锁和会话入口。不要把看门狗说成已经锁住了 SSE。 |
| **07** | **`07_dream/`** | `scheduler.py`, `executor.py`, `policy.py` | 空闲时整理。默认成功间隔 30 分钟。`cron` 不是写死的凌晨 3 点。`skill_candidate` 不会自动注册成技能。 | 用户在说话就让路。不要写成夜间自进化或释放本机 GPU。 |
| **08** | **`08_subagent/`** | `runtime.py` | 魔镜注册了 `skin_diary`、`deep_report`、`dream`。工具白名单缩小权限。没有接 A2A。 | 白名单不是安全证明。示意里的转账、删数据工具不在目录里。 |
| **09** | **`09_workitem/`** | `store.py`, `protocol.py`, `prompt.py` | `simpleclaw/workitem` 里有目标和证据。完成前可以拒绝没有证据的 `complete`。 | 魔镜装配器丢掉了 `work_item_store`，设备主路径目前没装上。 |
| **10** | **`10_skills/`** | `registry.py`, `base.py` | 目录进稳定前缀，正文用 `use_skill` 再读。当前魔镜是 5 个 `SKILL.md`。 | 目录一变，前缀哈希就变。不要说成 100% 命中。 |
| **11** | **`11_sdk_api/`** | `sdk/assembler.py`, `api/server.py` | 启动时装配，每一轮再装工具和提示词。主路径 SSE 有 15 秒注释心跳。模型请求带本轮 `traceparent`。没有 `Last-Event-ID`。 | 心跳不是断线续传。 |
| **12** | **`12_architecture_selection`** | `12_architecture_selection_and_interview_guide.md` | 选型草稿。残留句子不要背。 | 以第 13、14 篇为准。 |
| **13** | **`13_production_gaps_and_answers.md`** | 同上 | 已经落地的行为、还要补的部分、不可信数据的分层、面经对照、面试短答。 | 问稳定性、多 Pod、SSE、注入、上下文和评测时看这篇。 |
| **14** | **`14_architecture_audit.md`** | Agent 与 HotGo | 选型、造轮子对照、两边 Redis 各自管什么、已修的坑、还没补的洞。 | 完整复盘看这篇。 |
| **15** | **`15_redis_stream_kafka_double_hop_to_transactional_outbox.md`** | 异步队列演进复盘 | 从 Redis Stream + Kafka “双跳架构” 演进到 PostgreSQL 本地事务发件箱（Transactional Outbox）。详述分布式双写不一致深坑、PEL 位移重复投递、`FOR UPDATE SKIP LOCKED` 抢占与多端 WebSocket 同步真相。 | **必读硬核复盘**。搞清 Redis 与 Kafka 位移分工与真实踩坑演进。 |

---

## 二、阅读文档结构规范

每篇文档按这个顺序写：
1. **取舍怎么讲**：先说要解决的具体问题，再说我们做成了什么默认行为。框架里多半也能做，不要说成开源天生做不到。
2. **所有类、属性与方法全景解析**：不留死角，详细列出每个参数的类型、意义与业务防线。
3. **真实中间件与业务配置**：结合项目实际用到的 PostgreSQL、Redis、火山方舟等，给出真实参数与配置。
4. **全场景时序图与实例**：严格遵循 Typora & Mermaid 8.14.0+ 规范，每种场景配套一个独立图与实际案例，全面覆盖正常路径与异常熔断边界。
>>>>>>> b490457 (init: bootstrap simpleclaw knowledge base vault with homepage and obsidian configurations)
