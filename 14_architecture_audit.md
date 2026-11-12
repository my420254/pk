# 14：Agent 与 HotGo 选型复盘

> 对照仓库里的代码，不是对照愿望。数字只写代码或评测里出现过的。文档目录是 `simpleclaw_docs`，不是 `mojing_agent_docs`。面试以第 13 篇和第 14 篇为准，第 12 篇里还有旧句子，不要背。

---

## 一、先说结论

主路径这样拆是对的，不要推倒重来：

| 层 | 现在干什么 | 不要改成 |
|---|---|---|
| 魔镜 Agent（Python） | 一轮 ReAct、工具、记忆抽取、内部 Redis Stream、对外 Postgres 发件箱 | 不要把 Agent 整坨改写成 Go 只为了分布式锁 |
| HotGo（Go） | 用户、设备、后台、把硬件对话转到 Agent、Kafka 消费状态 | 不要在 Go 里再写一套 ReAct |
| Postgres | 对话正文、任务行、前缀 id、发件箱 | 不要把聊天历史改成 Redis 主存 |
| Redis | 租户令牌桶、内部 Stream、后台工单租约 | 不要当 prompt 缓存，不要当会话正文 |
| Kafka | 外部长任务 | 不要把后处理、记忆抽取也塞进去 |

本地配置里两边 Redis **不是同一个实例**：Agent `.env` 是 `redis://127.0.0.1:6379/0`，HotGo `config.local-http-cloud.yaml` 是 `127.0.0.1:6380` db `2`。生产如果合成一个 Redis，必须用不同 key 前缀。不要面试时说「已经共用 6379 db2」。

---

## 二、造轮子对照

该用现成库的地方，不要再手写。该自研的地方，写清楚为什么。

| 能力 | 现成做法 | 仓库现在 | 判定 |
|---|---|---|---|
| Python 分布式锁看门狗 | `python-redis-lock`（`expire` + `auto_renewal`） | TaskWorker 走这个库，默认 30 秒过期、自动续期 | 对。不要再写 `SET NX` + 自己续期 |
| Java Redisson 看门狗 | 后台续期，进程死了等 TTL | 同一类问题，Python 用上面那个库 | 对得上口，不要说 Python 没有 |
| Go 分布式锁 | `github.com/bsm/redislock` | HotGo `hgrds/lock` 自己用 Lua `SET` + goroutine 续期 | 框架自带的。只用在集群配置同步，先别换库 |
| Redis 红锁 RedLock | `redsync` | 两边都没用 | 正确。一台 Redis 不要上红锁 |
| 会话 SSE 锁 | 网关黏性，或 LangGraph Platform 把 run 放 Postgres | `InMemorySessionIngressStore` + 进程内 `asyncio.Lock` | 有意没上 Redis 会话锁。SSE 搬不走 |
| 对话存储 | Postgres / 厂商 thread | `agent_messages` | 对 |
| 限流 | Redis Lua / 网关 | `MultiTierAgentRateLimiter` Lua 令牌桶，Redis 挂了退回本进程 | 对 |
| 内部可重做工单 | Redis Stream 或队列库 | `XADD` / `XREADGROUP` / `XACK` / 闲置 60 秒 `XAUTOCLAIM` | 对。至少一次，处理必须能重复执行 |
| 外部长任务 | Outbox + Kafka | `FOR UPDATE SKIP LOCKED` 占行再发 | 对 |
| 前缀缓存 | 方舟 `previous_response_id` / `prompt_cache_key` | 进程字典 + Postgres `LLMCacheRepository` | 对。命中看 `cached_tokens > 0` |
| ReAct 循环 | LangGraph / 自己写 | `TurnRunner` + `ReactLoop` | 主路径是单轮循环，接了方舟前缀和 `history_offset`，所以写在自己循环里。不要说 LangGraph 做不到 |
| 缓存防击穿 | 单飞 + 空值 TTL + 抖动 | `RedisCacheReplica` **只有测试在用** | 不要接到对话账本上。个人量级对话走 Postgres 即可 |
| HotGo 业务缓存 | GoFrame `gcache` Redis 适配器 | `library/cache` | 后台配置用，不是聊天历史 |
| HotGo 延迟队列 | Redis List 或 Kafka | 配置里 `mq.driver` 可选 disk/redis/kafka；魔镜子路径是 Kafka | 不要和 Agent 的 Stream 混为一谈 |

---

## 三、魔镜 Agent：每一块现在怎么做

数据从用户进到落库，大致是：

```
设备 / App
  → HotGo（鉴权、转发）
  → Mojing HTTP（限流）
  → SessionTurnScheduler（本进程排队）
  → TurnRunner（本轮预算、canary、前缀）
  → ReactLoop（模型 ↔ 工具）
  → save_turn → Postgres
  → 内部活 → Redis Stream → TaskWorker（python-redis-lock）
  → 外部活 → Postgres 行 → Kafka → 外部服务 → 回执
```

### 1. 循环

`ReactLoop` 默认最多 20 次迭代。后处理压到 3，简单动作压到 2。工具指纹 `SHA-256(name + 规范化参数)`：第一次重复不真执行，回一条 JSON；再重复就熔断。熔断前先给还没回执的 `tool_calls` 补 `ToolResultMessage`（带 `deadlock=true`），否则下一轮模型会 400。

只读工具网络失败可以再试 1 次，写工具 0 次。

### 2. 上下文

`ContextCompressor(max_tokens=2200, target_tokens=1000, min_keep_tokens=600)`，只在用户消息边界切。这是滑动窗口，不是摘要之后丢掉原文。工具结果进模型前用 `<<<UNTRUSTED>>>` 围栏。泄漏哨兵放在动态尾部，不要放进可缓存的前缀。

### 3. 会话并发

入口是 `InMemorySessionIngressStore`。`SessionTurnScheduler` 用 `ScopeLockRegistry()`，**不传 redis_url**。张三李四互不等。同一会话在这一台机器上排队。

HotGo 转发带 `X-Mojing-Session-Key`。`deploy/nginx/mojing.conf` 按这个头 `hash ... consistent`。当前 upstream 仍是一台进程，加第二台时把 server 写进同一 upstream。不要给 SSE 加 Redis 锁。

`append_messages` 带 `owner_user_id` 时，插入事务里有 `pg_advisory_xact_lock`，只管序号。

### 4. 后台工单锁

`InfrastructureBundle` 里 `ScopeLockRegistry(redis_url=task_queue._url)` 只给 TaskWorker。`python-redis-lock` 拿不到或 Redis 报错抛 `ScopeLockTimeout`，工人把同一 attempt 重新入队，**不跑任务**。这不是会话锁。

### 5. 预算

`TurnTokenBudget` 默认 16384。每次 `VolcengineLLM` 流结束，用响应里的 `input_tokens + output_tokens` 累加。触顶后提交新的 durable 任务失败。已经进 Kafka / 发件箱的不在这里杀掉。

### 6. 记忆

抽取时 `_messages_to_text` 跳过 `ToolResultMessage`。再过滤一批注入口令。用户当轮原话覆盖旧事实。这不是所有投毒路径都封死。

### 7. SSE

`Cache-Control: no-cache`、`X-Accel-Buffering: no`。`ingress_delivery` 队列空闲 15 秒写 `: ping\n\n`。没有 `Last-Event-ID`。

### 8. 不要接到生产上的东西

`RedisCacheReplica` 实现了空值 TTL、抖动、同 key 单飞。测试在用。对话主路径不要接：聊天不是商品详情页，缓存穿透那一套用不上。

---

## 四、HotGo：每一块现在怎么做

HotGo 是后台和入口，不是第二套 Agent。

| 块 | 做法 | 注意 |
|---|---|---|
| 集群锁 | `hgrds/lock`：Lua `SET` + 看门狗 goroutine 按 TTL/3 续期，释放时核对随机值 | 用在 `PublishClusterSync`（系统配置、黑名单、超管），不是聊天锁 |
| `TryLockFunc` | 只有 `TryLock` 成功才跑回调 | 已经按这个写，不要改回去 |
| 缓存 | `gcache` Redis 适配器 | 后台缓存 |
| 队列 | `mq.driver` 可选；魔镜相关 worker 走 Kafka | 本地 yaml 里 Kafka 和 Agent 用同一套 broker 注释 |
| 健康 | Redis `PING` | 连的是 HotGo 自己的 Redis |
| 数据库 | 后台 MySQL（`hotgo` 库）+ 魔镜垂直切片用 Postgres `backend` | 和 Agent 的 `agent_messages` 不是同一张表 |

HotGo 这把锁是框架里手写的，和 Java Redisson 同类，只是没有换成 `bsm/redislock`。集群配置同步量小，先保持。不要上 RedLock。不要让 HotGo 和 Agent 抢同一把会话锁。

---

## 五、这次修掉的坑

| 坑 | 原来会怎样 | 现在 |
|---|---|---|
| `python-redis-lock.Lock(..., strict=False)` | 构造函数没有这个参数，异常被吃掉，退回本地锁，看门狗等于没上 | 不再传 `strict` |
| 抢锁失败仍 `yield` 进临界区 | 没锁也跑工单 | `ScopeLockTimeout`，TaskWorker 原 attempt 重入队 |
| Redis 报错退回本地锁 | 多 Pod 在 Redis 抖动时同时跑同一 `scope_key` | 配了同步 Redis 就失败关闭，不跑 |
| 工具死锁只发 `ErrorEvent` | 缺 ToolResult，下一轮模型 400 | 先补 `deadlock` 回执再熔断 |
| 预算只在 ContextVar 里 | Redis Worker 看不见，新 durable 仍能提交 | `_submit_runtime_task` 提交前检查 |
| 子 Agent 从 `turn_runner` 拿预算 | 类挪了之后容易引错 | 改为 `simpleclaw.runtime.turn_budget` |

对应测试：`tests/test_production_hardening.py`、`tests/test_loop_fingerprint_dedupe_and_truncation.py`、`tests/test_tool_execution_runtime.py`（超额预算不提交 durable）。

---

## 六、落地要做的，和明确不做的

落地要做的（已做）：

1. **同一会话进同一台 Agent。** 网关黏性，不是 Redis 锁。HotGo 带头，nginx `hash`。现在仍是一台 upstream，加副本时改 nginx，不要改会话锁。
2. **每一次模型 HTTP 带上本轮 trace。** `traceparent` / `X-Trace-Id` / `X-Client-Request-Id`。方舟日志才能对上 HotGo 和 Agent。
3. **预算用接口返回的 token。** 不是字符除以 3。

明确不做：

1. **SSE `Last-Event-ID`。** 心跳已经防读超时。续传要 HotGo、硬件、小程序都认 id 并重放。现在断线就是重说一句，不要在服务端空做缓冲。
2. **datamarking / 双模型。** 主路径工具结果来自自己的柜和档案。围栏 + 哨兵 + 记忆跳过工具原文够用。双模型会把每一轮成本翻倍，硬件路径扛不住。
3. **设备主路径 `work_item_store`。** 那是 SDK 里给长研究任务用的检查单工具。镜子主路径已经有设备工具和 Kafka 深研。再挂 4 个 work_item 工具会抢模型注意力，产品也没有这张清单 UI。

还要承认的缺口：

- `RedisCacheReplica` 未接入生产，不要接到聊天账本。
- 评测脚本在仓库里，不要说已经按发布节奏天天跑。
- 黏性没有用两台 Pod 打过。面试不要说「多 Pod 已经验证串行」。

---

## 七、经典场景（面试和线上都容易踩）

### 1. 开发环境 SSE 逐字，生产一次吐完或 60 秒断开

本机 `curl -N` 对照。本机逐块、外面一把吐完，是缓冲或 gzip。大约一分钟断开是 `proxy_read_timeout`。代码侧已有 15 秒 `: ping` 和 `X-Accel-Buffering: no`。还要查每一跳代理。

### 2. 同一会话进了两台 Pod

两台都会调模型，Postgres 里字交错。黏性优先。Redis 锁锁不住已经建好的那条 SSE。

### 3. 用户说停，柜子里已经有货

入柜是确认后当场 `mark_in_cabinet`，不进 `SideEffectBuffer`。打断丢掉的是还没 `save_turn` 的对话。不要说成事务回滚。

### 4. Kafka 重复

占行后进程死了，可能发两次。下游认 `idempotency_key`。行锁消不掉至少一次投递。

### 5. Redis Stream 慢任务被认领两次

闲置 60 秒 `XAUTOCLAIM` 只说明很久没 ACK，不能证明原进程死了。处理必须幂等。scope 锁降低同 key 并行，代替不了幂等。

### 6. 工具调用缺 result 导致 400

打断、熔断、并行里有一个失败，都要补 `ToolResultMessage`。死锁路径已经补。独立跑 `ReactLoop` 时，取消路径也会补 `cancelled=True`。

### 7. 前缀没命中还说首字一定快

命中条件是响应里 `cached_tokens > 0`。`prompt_cache_key` 只是路由提示。全量设备提示词命中后，重复测量的中位数大约 3 秒，最快大约 1.6 秒，慢的仍到 7 秒。不要背 300 毫秒。

### 8. 缓存穿透击穿雪崩

那是 Redis 当 DB 前面的只读副本时的题。对话主路径不是这种用法。限流 Redis 挂了退回本进程计数，表现是限流变松，不是把 Postgres 打穿。

### 9. 新对话按钮

手机会话可以加「先聊到这里」的边界。不要新开 `sessionId` 去清硬件 `main:{device}` / `rtc:{room}`，也不要清柜子、日记、已经进 Kafka 的活。

### 10. Python 没有看门狗吗？

有。后台工单用 `python-redis-lock`。HotGo 用自研 Lua + goroutine。会话 SSE 故意不用。

---

## 八、和常见开源项目怎么对齐着说

- **LangGraph**：长流程、人工暂停后续跑。检查点不是锁。图里也可以把写入放到确认节点。我们入柜是确认后当场写。不用 LangGraph 是因为主路径已经接在 `TurnRunner` 和方舟前缀上，不是因为对方会写出幽灵柜。
- **LangGraph Platform**：run 放 Postgres，Redis 做工人唤醒。用户正文不放 Redis。和我们「对话在库、工单在 Redis」同一方向。
- **AutoGPT**：同一会话两路写库之后才加 `SET NX EX`。我们还没走到必须给 SSE 加 Redis 租约那一步。
- **Dify / Coze**：工作流产品。不要贬，也不要说我们覆盖了他们的编辑器。

---

## 九、改文档时对上的文件

| 文档 | 改了什么 |
|---|---|
| `README.md` | 第 6、11、14 篇和代码一致 |
| `01_core` | 死锁先补 ToolResult |
| `04_tools` | Redis 职责，去掉手写 120s `SET NX` |
| `06_runtime` | TaskWorker 用 `python-redis-lock`，会话仍是内存队列 |
| `11_sdk_api` | 15 秒心跳，仍无 `Last-Event-ID` |
| `12` | 锁的说法改掉；仍不要当标准答案 |
| `13` | 已落地 / 未落地分开；预算是估算 |
| `redis+kafka.md` | 指向本篇，去掉「测试全绿」那种空话 |
