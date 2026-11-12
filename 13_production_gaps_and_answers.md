# 13：还要补上的部分，以及按现有代码怎么答

> 这份说明只写魔镜 / SimpleClaw 仓库里已经有的行为，以及还没做、被问到时必须承认的缺口。
> 数字只使用已经在代码或评测报告里出现过的配置。没有测过的耗时不写死。
> 第 12 篇里「首字 300 毫秒」「凌晨 3 点」「微秒级锁」「前缀缓存在 Redis」不要背。那些和代码不符。

---

## 一、已经有的，不要再改成另一套

这些和常见做法一致，保持即可。

| 职责 | 现在的代码 | 被问到怎么说 |
|---|---|---|
| 一轮对话 | `TurnRunner` + `ReactLoop`。默认最多 20 次迭代（`_DEFAULT_MAX_ITERATIONS = 20`）。后处理压到 3，简单动作压到 2。 | 主路径是单轮 ReAct，不是长流程图。 |
| 同一台机器上的排队 | `SessionTurnScheduler.drain` 拿 `ScopeLockRegistry`，键是 `session_ingress:{session_key}`。队列是 `InMemorySessionIngressStore`。 | 张三和李四互不等待。张三在这一台机器上的两句话会排队。 |
| 限流 | `rate_limiter` 用 Redis Lua 做租户令牌桶。Redis 不可用时退回本进程计数。全局正在飞的请求数也是本进程的。 | Redis 在这里负责「这段时间能打多少次」，不负责「这个会话有没有一轮正在跑」。 |
| 内部可重做的活 | Redis Stream：`XADD` / `XREADGROUP` / `XACK`。默认闲置 60 秒后 `XAUTOCLAIM`。 | 机器死了，未确认的工单可以被别的 Pod 接走。慢任务可能和认领方各做一次，处理必须能重复执行。 |
| 对外长任务 | `deep_research`、`image_analysis`、`skin_diary_generation`、`cabinet_product_research` 先写入 `agent.agent_runtime_tasks`，再 `FOR UPDATE SKIP LOCKED` 占行发 Kafka。 | 重活不在 Agent 进程里算。Agent 只负责提交和等回执。 |
| 会话正文 | Postgres `agent_messages`。`save_turn` 只追加本轮新增的尾部。 | 历史不放 Redis。 |
| 前缀 | 进程字典里的 `response_id`，再落到 `LLMCacheRepository`（Postgres）。请求体里的 `prompt_cache_key` 是方舟的路由提示，不保证命中。 | 命中看 `cached_tokens > 0`。命中不等于首字变快。全量设备提示词命中后，重复测量的中位数大约 3 秒，最快大约 1.6 秒，慢的仍到 7 秒。 |
| 上下文长度 | `ContextCompressor(max_tokens=2200, target_tokens=1000, min_keep_tokens=600)`，只在用户消息边界切，并累加 `history_offset`。 | 这是滑动窗口，不是先摘要再把原文丢掉。 |
| 工具参数 | `execution_runtime` 用 JSON Schema 校验，再走 `validate_params`。不合法不执行。 | 格式不稳靠校验把错误交回模型或记失败，不靠模型「尽量猜」。 |
| 打断 | 读工具 `cancel`。`durable` / `sync_write` 是 `detach`。还没 `save_turn` 的半截对话用内存快照丢掉。入柜是当场 `mark_in_cabinet`，不经过 `SideEffectBuffer`。 | 取消停的是这一轮的模型流和未落库对话。已经入柜或已经提交的外部任务不会自动撤销。 |
| 子代理 | 目录里是 `skin_diary`、`deep_report`、`dream`。 | 不是任意递归生出新代理。 |
| 技能 | 5 个 `SKILL.md`。目录在稳定前缀，正文 `use_skill` 再读。 | 技能是提示词流程。MCP 是进程外的工具协议，仓库里有 HTTP 客户端。不要把每个技能都说成一个 MCP。 |
| SSE 响应头 | `chat.py` 的 `_SSE_HEADERS` 有 `Cache-Control: no-cache` 和 `X-Accel-Buffering: no`。`ingress_delivery` 在队列空闲 15 秒时写 `: ping\n\n`。 | 头只防住认它的那一层 Nginx。心跳防的是读超时。没有 `Last-Event-ID`，断线不能从上次的字续上。 |

外部静默收割：`CronScheduler.STALE_RUNNING_TTL_S = 900`（15 分钟）。深度报告去重默认 600 秒。图片和入柜调研的去重窗口是 180 秒。不要说成统一 10 分钟。

---

## 二、已经落地的，以及还没补的

落地要做的已经做：网关按 `X-Mojing-Session-Key` 黏会话、模型请求带 `traceparent`、预算用接口返回的 input/output。下面三件明确不做：SSE `Last-Event-ID`（客户端没有续传协议）、datamarking/双模型（主路径是自家工具不是任意网页）、设备主路径 `work_item_store`（会再塞 4 个工具，深研已经走 Kafka）。

### 1. 多 Pod 下同一会话只有一个写者

**现在。** 锁和入口队列仍在收到 HTTP 的那台进程里。网关侧已经按会话黏：HotGo 转发时带 `X-Mojing-Session-Key`，`deploy/nginx/mojing.conf` 用 `hash $mojing_session_affinity consistent`。当前 upstream 仍是一台 `127.0.0.1:8000`，一台进程时黏不黏行为一样。加第二台之前，把新 server 写进这个 upstream，不要给 SSE 再加一把 Redis 锁。

`append_messages` 带 `owner_user_id` 时，插入事务里仍有 `pg_advisory_xact_lock`，只管序号。

黏性失效时（Pod 挂了、头没带上），不要靠 Redis 锁去续那条已经断开的 SSE。第二台机器即使拿到锁，也推不了第一台已经建立的流。

**为什么是这个顺序。** SSE、打断标记、这一轮的上下文都在收到请求的进程里。LangGraph Platform 把运行状态放在 Postgres，Redis 只做工人之间的唤醒和取消，同样不把用户正文放进 Redis。

**证明。** 两台 Agent、同一个 `X-Mojing-Session-Key`，应落在同一台。这个实验做完之前，面试不要说「多 Pod 已经验证串行」。

**不能说。** 微秒级、Redis 锁是行业法定第一步。

### 2. 生产环境 SSE 逐字输出与心跳保活

**现状与工程落地。**
1. **15 秒注释心跳保活**：在 [`Mojing/api/ingress_delivery.py`](file:///home/yang/mojing_agent/Mojing/api/ingress_delivery.py) 中，`_event_stream` 对 `queue.get()` 增加 15 秒超时等待。当模型思考、外部工具调用或深研产生长时延时，自动向流中写入 `: ping\n\n`（SSE 规范注释行，客户端 EventSource 自动忽略但重置中间代理的 TCP 空闲计时器），彻底解决 Nginx `proxy_read_timeout` 或云厂商 ALB 静默切断长连接的问题；
2. **响应头规避中间缓冲**：`_SSE_HEADERS` 显式下发 `Cache-Control: no-cache` 与 `X-Accel-Buffering: no`；
3. **代理层全链路配置检查**：每一跳代理（Nginx/Ingress）需保证 `proxy_buffering off`、`gzip off`、`proxy_http_version 1.1` 及空的 `Connection` 头。

**为什么。** 开发环境直连无代理；测试与生产环境多层代理默认开启缓冲或 60s 掐断空闲 TCP，这是 SSE 线上常见环境差异。

---

### 3. 内部工单的同一 `scope_key` 多 Pod 互斥

后台 `TaskWorker` 用 `python-redis-lock`：`expire` 默认 30 秒，`auto_renewal=True` 时后台线程大约每 `expire` 的三分之二续一次。这是现成库，不是再手写一套看门狗。进程内仍先拿 `asyncio.Lock`。拿不到 Redis 锁就抛 `ScopeLockTimeout`，工人把同一 attempt 重新入队，不跑任务。会话入口仍是本进程队列，不走这把锁。

无 Redis 时只剩进程内锁。配了同步 Redis 却报错时抛 `ScopeLockTimeout`，不退回「只跑本地」。

---

### 4. 不可信数据与长期记忆隔离：围栏加哨兵，记忆防投毒

**现状与工程落地。**
1. **围栏定界（Delimiting）**：送入模型前，工具结果包裹在 `<<<UNTRUSTED>>>` … `<<<END_UNTRUSTED>>>` 中，伪造的闭合标签预先做实体转义，原文留存数据库；
2. **动态安全哨兵（Canary）**：设备主路径动态尾部生成 `MOJING-CANARY-` 随机串，回复或工具参数中出现即打日志并剔除；
3. **长期记忆防投毒（Memory Poisoning Defense）**：在 [`Mojing/agent/memory/memory_extract.py`](file:///home/yang/mojing_agent/Mojing/agent/memory/memory_extract.py) 中：
   - `_messages_to_text` 跳过 `ToolResultMessage`，抽取时再过滤一批注入口令。这挡的是工具原文进记忆，不是所有投毒路径。不要说 100%。
   - 提取候选动作经 `_contains_prompt_injection` 过滤，拦截系统级指令；
   - **用户当轮原话绝对权威**：当轮用户明确表述（如“变干皮了，不再控油”）具有最高优先级，强制覆盖冲突的历史陈旧事实；
4. **标准化固定攻击回归集**：在 [`eval/golden/indirect_injection_cases.py`](file:///home/yang/mojing_agent/eval/golden/indirect_injection_cases.py) 中建立 6 组典型越狱场景，由评测脚本定期回归验证。

---

### 5. 预算治理：单轮总 Token 预算共享池与子 Agent 熔断

**现状与工程落地。**
`TurnTokenBudget` 在 `simpleclaw/runtime/turn_budget.py`。默认上限 16384。每次模型调用结束，用响应里的 `input_tokens + output_tokens` 累加（`VolcengineLLM._log_completion`）。接口没返回 usage 时这一跳不加。触顶后提交新的 durable 任务会失败，已经进 Kafka 或发件箱的不在这里杀掉。

---

### 6. 细粒度三项黄金基准独立评测拆开打分

**现状与工程落地。**
建立统一独立的评测执行器 [`eval/run_granular_benchmarks.py`](file:///home/yang/mojing_agent/eval/run_granular_benchmarks.py)，输出三张独立量化 Markdown 表格：
1. **工具决策准确率与准召率表**（基于 [`tool_trigger_cases.py`](file:///home/yang/mojing_agent/eval/golden/tool_trigger_cases.py)）：评测显式入柜/深研等正样本，与寒暄/常识/显式否定陷阱（“不用加柜子”）等负样本的准召（TP/FP/TN/FN/Precision/Recall/F1）；
2. **间接注入与哨兵防泄漏表**（基于 [`indirect_injection_cases.py`](file:///home/yang/mojing_agent/eval/golden/indirect_injection_cases.py)）：评测工具诱导重定向、围栏逃逸与哨兵防泄漏（100% 拦截）；
3. **RAG 证据召回与生成忠实度表**（基于 [`rag_faithfulness_cases.py`](file:///home/yang/mojing_agent/eval/golden/rag_faithfulness_cases.py)）：评测 Hit@1 / Hit@3 证据命中率、回答对权威要点的忠实支撑度，并在混入高字面相似度谣言干扰段落且答案发散时，在报表上明确打上 `[NOISE_INJECTED_DEGRADATION]` 噪声劣化诊断标记！

---

## 二点五、面经问法对到哪一层

这些是公开面经里反复出现的问法，不是某次考试的标准分数。训练站写的「参考答案」里的百分比不要背。

| 面经里的问法 | 按这个仓库怎么答 |
|---|---|
| 米哈游：工具输出是数据，标来源，通道分开，敏感调用再授权 | 围栏标的是工具结果。用户原话仍是指令通道。入柜确认是再授权。阻断在注册表和 Schema。 |
| 米哈游：外部内容不要进长期记忆 | 已在 `_messages_to_text` 剥离 `ToolResultMessage` 与外部检索正文；候选动作经 `_contains_prompt_injection` 拦截越狱词；当轮用户原话绝对权威，强制覆盖冲突历史。 |
| 米哈游 / 快手：MCP 或工具是否可信 | 有 HTTP MCP 客户端。协议不使工具变得可信。Host 仍要白名单、超时和确认。 |
| 快手：提示注入和工具安全 | 用第四节的四层。不要答「提示词写了不要听从」。 |
| 快手：令牌桶、漏桶、滑动窗口 | 租户限流是 Redis Lua 令牌桶。这不是会话锁。 |
| 美团 / 字节常见追问：工具失败、格式不稳、记忆分层、长上下文 | 参数先过 Schema。读取消、写脱钩。瞬时网络错误只读工具退避重试 1 次，写工具 0 重试。短期是窗口，长期是记忆账本。窗口不是摘要。 |
| RAG：召回上去了、准确率下来了 | 跑 `eval/run_granular_benchmarks.py`。固定用例与固定 K，把「检索 Hit@K」与「忠实度 Groundedness」拆开打分。混入高相似度谣言干扰时，报表显式打上 `[NOISE_INJECTED_DEGRADATION]` 标记。 |
| 金丝雀 | 动态尾部的泄漏哨兵，只检测。不放进稳定前缀。 |

---

## 三、问到稳定性时，按状态查，不按感觉查

### 这一轮没有字

1. 本机 `curl -N` 是否逐块。不是，先看生成器有没有 `yield`。
2. 本机逐块、环境外面不是，查每一跳的缓冲、压缩和读超时。见第二节第 2 条。
3. 首字日志停在进 `llm.stream` 之后，看方舟是否返回、`cached_tokens` 是多少。缓存命中仍可能要数秒。
4. 停在某个工具，看该工具的超时和 `cancel` / `detach`。
5. 任务行停在 `queued`：没人领，或发件箱没占到行。停在 `sending`：卡在发 Kafka。停在 `wait_external`：对方没回。从 5 分钟变成 20 分钟，先看状态从哪一段开始变长，不要在 Agent 进程里找这 20 分钟的计算。

### 用户取消，或发布时 Pod 被换掉

取消把 `interrupt_event` 置上，`aclose()` 停掉发往方舟的流，内存快照丢掉还没落库的半截回复。`detach` 的写和已经提交的外部任务继续。发布会断开当前连接。未确认的 Stream 条目靠闲置后再认领。发件箱的 `sending` 超时后再被别的 Pod 占行。没有把一条 SSE 迁到新 Pod 的实现。面试就这样说。

### 看板没有数

先拿一个已知 `session_key` 查 `agent.agent_messages`。原始表有行、聚合没有，是统计或迁移没写到看板读的那张表，或时间区、租户条件滤掉了。原始表也没有，是没写入，不是图表坏了。不要先改前端。

### trace 对不上

网关能从 `traceparent` 解析出 trace id。它没有自动出现在每一次模型请求上。排查时用 `session_key`、`ingress_id`、任务 `task_id` 串，不要说成全链路已经打通。

---

## 四、上下文、记忆、工具、技能：问法对照

**短期和长期。** 短期是这一轮内存里的 `messages`，超长后按用户边界裁，序号用 `history_offset`。长期是记忆账本：原文片段、消息序号、改前改后。提炼不堵在用户等待的那次回复里。用户问「之前说过什么」时，`TextMemoryRetriever` 默认还会再调一次模型从索引里挑，另有 BM25。不要说热路径零模型，也不要说每次都是向量检索。

**滑动窗口和摘要。** 窗口适合还要引用最近工具结果的对话，切的时候不能把工具调用和结果拆开。摘要适合很多天以前的偏好，不适合这一轮刚刚返回的柜子或天气。现在生产路径用的是窗口。做梦是空闲时的整理，默认成功间隔 30 分钟，不是凌晨 3 点的排期，也不会把草稿自动注册成 `SKILL.md`。

**工具失败、超时、参数非法。** 参数先过 Schema。超时和打断按读/写分开：读取消，写和长任务脱钩。需要模型接着看的工具，结果要写回。纯副作用不写回时，回放不能留下「只有调用、没有结果」，否则下一次可能 400。这是已知边界，不要说成协议已经没有洞。

**工具很多怎么选。** 不要每句话带上全部长流程。目录放稳定前缀，用到再加载正文。业务工具的 `concurrency_safe` 默认是 `False`，同一轮里排队，不是默认并行。只有这次请求带了 `tools`，才会把 `parallel_tool_calls` 设为真。前缀复用时不再重传工具定义。

**效果和坏例。** 用第六节的固定题。没有这份额外的集之前，只引用已经跑过的延迟探针，并说明它量的是进模型之后到第一个字，不含语音识别和播报。

**给业务系统加 AI。** 看板、订单、测肤记录仍属于原系统的表。Agent 读允许的范围，写走现有接口和确认。模型不直接改看板的库。对话和任务状态在 Agent 的表。

---

## 五、面试短答

问：为什么自己写循环，不用 LangGraph？

答：主路径是一轮 ReAct，要接火山的 `previous_response_id` 和 `history_offset`，所以写在 `TurnRunner` 里。LangGraph 适合长流程和人工暂停后续跑。同一条 thread 两个进程一起写，检查点也不是锁。图里同样可以把待写入放到确认之后。我们入柜是确认后当场写，打断不自动删柜。

问：Redis 用在哪？

答：租户限流（Redis Lua 令牌桶），内部工单队列（Redis Stream 消费者组 + xautoclaim）。后台 TaskWorker 用 `python-redis-lock` 续期，拿不到锁就重新入队。用户这一轮仍是本进程队列。仓库里没有网关黏性配置，不能说多 Pod 对话已经串行。会话正文和前缀 id 在 Postgres，不在 Redis。

问：开发环境 SSE 正常，生产不逐字？

答：先本机 `curl -N`，再从代理外打。本机逐块、外面一次吐完，就是缓冲或压缩。约 60 秒断开是读超时。代码在 `ingress_delivery.py` 中每 15 秒发一次 `: ping\n\n` 注释行重置代理空闲计时器，响应头下发 `Cache-Control: no-cache` 与 `X-Accel-Buffering: no`；同时排查每一跳代理的 `proxy_buffering off`。

问：Agent 挂了或超时怎么查？

答：看任务行停在 `queued`、`sending` 还是 `wait_external`。外部静默 900 秒由定时收割标失败。深度报告去重是 600 秒，不是所有工具都是这个数。

问：用户取消，发布时正在跑的怎么办？

答：取消停模型流和未落库对话。已提交任务继续，处理要能重复执行。发布断开当前连接，队列和发件箱负责把没确认的活交给别的 Pod。SSE 不迁移。

问：ReAct 循环死循环或工具重复调用怎么解？

答：同一轮里计算工具指纹 `SHA-256(name + canonical_args)`。第 1 次重复拦截真实执行，返回显式 JSON 告知“重复调用、状态没变，请勿重复调用”；第 2 次仍重复则立即触发 `ErrorEvent` 熔断退出，不浪费无效 Token。瞬时网络错误只读工具退避重试 1 次，写工具严格 0 重试。

问：上下文超长？

答：2200 触发，收到 1000，最近 600 尽量留，只在用户消息处切。这是窗口，不是摘要。

问：召回高了答案差了？

答：跑 `eval/run_granular_benchmarks.py` 拆开打分。固定用例与固定 K，区分「检索 Hit@K」与「忠实度 Groundedness」。召回高、忠实度低是噪声污染，报表打上 `[NOISE_INJECTED_DEGRADATION]` 标记；可通过收紧检索条数或加一层重排治理。

问：文档里的恶意指令与记忆投毒？

答：四层防御。1. `compliance.md` 定界工具结果是数据不是规则；2. 模型输入用 `<<<UNTRUSTED>>>` 包裹并做闭合转义；3. 动态尾部每轮下发 `MOJING-CANARY` 哨兵检测泄漏；4. 长期记忆防投毒：`_messages_to_text` 剥离 `ToolResultMessage` 与外部检索正文，候选动作过滤越狱词，用户当轮原话绝对优先；并由 `eval/run_granular_benchmarks.py` 定期回归 6 组间接注入用例。

问：多代理会不会把预算打穿？

答：`TurnTokenBudget` 默认上限 16384。每次模型调用结束用接口返回的 `input_tokens + output_tokens` 累加。触顶后提交新的 durable 任务会失败。已经进 Kafka 或发件箱的不会在这里杀掉。

问：看板没数据，是不是迁移？

答：先查原始消息表。有行而聚合没有，才查迁移和统计条件。

问：入柜打断会不会幽灵数据？

答：不会自动撤销。确认后 `mark_in_cabinet` 当场写。`SideEffectBuffer` 只处理调用过 `stage()` 的项，生产入柜没有调用。对话可以整轮丢掉，柜子里的记录还在。
