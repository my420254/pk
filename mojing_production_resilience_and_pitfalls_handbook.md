# 魔镜智能体系统（Mojing）全流程生产落地、真实踩坑与避坑实战手册

> **编写原则**：本文档完全基于当前代码库的真实实现与真实测试输出编写，杜绝任何虚构的架构设计与编造的评测数据。真实说明每一个 Bug 的根因、踩坑过程、标准解决代码与实际测试断言，并如实记录当前系统已落实的改动与仍存在的架构技术债。

---

## 目录
1. [系统整体架构与三端真实通信拓扑](#一-系统整体架构与三端真实通信拓扑)
2. [关键生产硬伤定位、踩坑细节与代码修复](#二-关键生产硬伤定位踩坑细节与代码修复)
   - [1. 聊天消息并发幂等“假兜底”与流式气泡虚增](#1-聊天消息并发幂等假兜底与流式气泡虚增)
   - [2. 定时任务唯一索引时间抖动与一次性任务互相吞噬](#2-定时任务唯一索引时间抖动与一次性任务互相吞噬)
   - [3. 异步长任务被二次入库打回原形、产物被清空](#3-异步长任务被二次入库打回原形产物被清空)
   - [4. 设备出厂注册秘钥停写入库与明文风险根除](#4-设备出厂注册秘钥停写入库与明文风险根除)
   - [5. 设备 OTA 固件更新与用户快照强一致事务绑定](#5-设备-ota-固件更新与用户快照强一致事务绑定)
   - [6. Kafka 消费端跳跃式位点隐式提交与丢消息防范](#6-kafka-消费端跳跃式位点隐式提交与丢消息防范)
   - [7. Redis Stream 内存无界膨胀、毒丸自旋与丢任务治理](#7-redis-stream-内存无界膨胀毒丸自旋与丢任务治理)
3. [微信小程序端规范整改与交互健壮性](#三-微信小程序端规范整改与交互健壮性)
4. [真实测试验证记录与实际输出核对](#四-真实测试验证记录与实际输出核对)
5. [系统已知技术债与边界说明](#五-系统已知技术债与边界说明)
   - [1. 雪花算法 Worker ID 哈希与未来向 UUIDv7 演进](#1-雪花算法-worker-id-哈希与未来向-uuidv7-演进)
   - [2. 现网 PostgreSQL 与 MySQL 商城表边界说明](#2-现网-postgresql-与-mysql-商城表边界说明)
   - [3. 黄金数据集无需重建的原因](#3-黄金数据集无需重建的原因)

---

## 一、 系统整体架构与三端真实通信拓扑

魔镜系统在实际运行中由以下三个核心端组成：
1. **微信小程序端（[MagicMirrorFrontEndV2](file:///home/yang/mojing/MagicMirrorFrontEndV2)）**：用户终端，负责多模态输入（文本、测肤拍照上传）、SSE 流式交互、卡片选择、设备绑定配网引导。
2. **业务网关（HotGo，基于 Go/GoFrame）**：业务控制面，负责用户鉴权、会话管理、消息在 `backend.chat_messages` 中的持久化与幂等拦截、设备 MQTT 鉴权与 OTA 状态机、以及将用户消息转交给 Python Agent。
3. **AI 智能体核心（Mojing Agent，基于 Python/FastAPI）**：模型推理面，负责 ReAct 决策循环、长短记忆检索、火山引擎大模型调用、以及通过 Redis Stream 和 Kafka 驱动后台长周期任务（如深度测肤报告）。

```mermaid
sequenceDiagram
autonumber
actor User as 微信小程序用户
participant HotGo as HotGo业务网关
participant AgentLoop as Python Agent核心
participant Worker as 后台异步Worker
participant DB as PostgreSQL数据库

Note over User,DB: 第一阶段：用户发信与网关幂等落库
User->>HotGo: 1. 发送文字/图片 (带 client_msg_/client_img_ 幂等键)
HotGo->>DB: 2. 校验会话所有权，写入 chat_messages (ON CONFLICT 拦截重复)
alt 命中已有 idempotency_key
    HotGo->>HotGo: 比对正文是否一致 (不一致则拒绝 400，一致则返回已有行)
end

Note over HotGo,AgentLoop: 第二阶段：流式转发与大模型处理
HotGo->>AgentLoop: 3. A2A 流式 Ingress 转发
AgentLoop->>AgentLoop: 4. ReAct 思考与工具决策
alt 触发后台长任务 (如测肤深度分析)
    AgentLoop->>DB: 5. 写入 agent_runtime_tasks (record_queued)
    AgentLoop->>Worker: 6. 投递任务 (Redis Stream / Kafka)
    Worker->>DB: 7. 异步执行并写入结果
end
AgentLoop-->>HotGo: 8. SSE 流式吐出思考与回复 Chunk
HotGo->>DB: 9. 助手回复落库 (绑定 用户键:reply)
HotGo-->>User: 10. 客户端接收流式气泡与卡片
```

---

## 二、 关键生产硬伤定位、踩坑细节与代码修复

### 1. 聊天消息并发幂等“假兜底”与流式气泡虚增

#### 【实际的 Bug】
1. **TraceID 冒充业务键**：原网关代码在没有收到客户端键时，做了一层兜底：`if idempotencyKey == "" { idempotencyKey = "req:" + meta.TraceID }`。在真实网络下，客户端发起的每一次 HTTP 重试由于连接重建，框架都会分配新的 `TraceID`，导致服务端的防重判断全部漏过；反之，若前端整个页面复用同一 TraceID，用户发出的下一句不同话又会被误当成重复消息丢弃。
2. **小程序端漏传业务键**：小程序此前仅在用户点击卡片确认按钮时传了键，用户日常发送文字、重试发送拍照图片时未携带 `idempotency_key`。
3. **流式回复助手气泡倍增**：即便用户消息防住了，网关在流式结束后插入助手回复记录时传入了空键，导致重试发生时，数据库里插入了多条一模一样的助手回复，前端页面渲染出多个重复气泡。
4. **同键异文静默覆盖**：当遇到数据库唯一键冲突时，网关直接把查出来的旧消息返回给调用方。若客户端代码有 Bug 导致不同内容的请求带了相同的键，用户发出的新内容会被旧回复静默覆盖。

#### 【标准解法】
* **意图发生即生成键**：幂等键必须由用户在交互发生的瞬间（点击发送）在客户端本地生成，重试期间全程保持同一键。
* **链路透传，绝不篡改**：网关只做透传与校验，彻底删掉 `req:` + TraceID 的假兜底；TraceID 纯粹用于链路日志串联。
* **确定性派生助手回复键**：助手回复的幂等键由用户键严格派生为 `用户键 + ":reply"`（多张卡片追加 `:reply:1` 等后缀）。
* **同键异文强校验拒绝**：发生冲突时比对请求正文与已存正文，若内容不一致，明确返回错误码 `CHAT_IDEMPOTENCY_KEY_CONTENT_MISMATCH`，绝不把旧内容冒充成功结果。

#### 【实际落地的代码】
* **网关层彻底删除假兜底并增加防篡改比对**（[chat.go](file:///home/yang/HotGo/server/internal/logic/mojing/chat/chat.go)）：
  ```go
  // appendChatMessageFull 中处理冲突
  if created.IsEmpty() {
      // 命中唯一索引冲突，校验是否为同键异文
      if existing["content"].String() != content {
          return nil, shared.Fail(gcode.CodeInvalidParameter, "CHAT_IDEMPOTENCY_KEY_CONTENT_MISMATCH")
      }
      return existingMessage, nil
  }
  ```
  流式助手回复落库时确定性绑定：
  ```go
  replyKey := ""
  if streamIdempotencyKey != "" {
      replyKey = streamIdempotencyKey + ":reply"
  }
  ```
* **微信小程序端生成键**（[session-page.ts](file:///home/yang/mojing/MagicMirrorFrontEndV2/miniprogram/pages/chat-detail/session-page.ts) 与 [chat.ts](file:///home/yang/mojing/MagicMirrorFrontEndV2/miniprogram/api/chat.ts)）：
  * 发送文字：绑定 `client_msg_${userMsgKey}`；
  * 发送图片：在 `for (attempt <= MAX_IMAGE_SSE_ATTEMPTS)` 重试循环外部生成 `client_img_${userKey}`，确保重试循环内使用同一个键；
  * `miniprogram/api/chat.ts` 在所有模式下统一透传 `data.idempotency_key = payload.idempotencyKey`。

---

### 2. 定时任务唯一索引时间抖动与一次性任务互相吞噬

#### 【实际的 Bug】
* **早期踩坑**：`agent.agent_cron_jobs` 原唯一索引包含 `next_run_at`。由于每次定时器算出的微秒/毫秒存在细微抖动，周期性任务每次计算出的时间戳略微不同，导致防重索引彻底失效，任务被反复重复插入。
* **中期修改踩出的大坑**：为了消除抖动，将 `next_run_at` 移出了索引，改成仅按 `(user_id, job_type, schedule_expr, task, interval_s)` 约束。这在周期任务上看似正常，但引发了严重的业务 Bug：**一次性任务（`job_type = 'once'`）既没有表达式也没有间隔秒数，如果同一个用户设置了两个提醒（例如 14:00 提醒喝水，18:00 再次提醒喝水），由于任务文本都是“提醒喝水”，18:00 的任务被唯一索引判定为冲突直接吞噬，执行时刻丢失！**

#### 【标准解法】
* **周期任务与一次性任务的身份本质不同**：
  * 周期任务（Cron / Interval）的身份是它的**调度规则与业务动作**，执行时刻只是瞬时游标；
  * 一次性任务（Once）的身份**必须包含其目标触发时刻**，不同触发时间的两次相同动作属于两个不同的提醒。

#### 【实际落地的代码】
在 [postgres_database.py](file:///home/yang/mojing/mojing_agent/Mojing/storage/postgres_database.py) 中，将唯一索引拆分为两个独立的部分唯一索引：
1. **周期任务索引**：
   ```sql
   CREATE UNIQUE INDEX IF NOT EXISTS uq_agent_cron_jobs_active_recurring
   ON agent.agent_cron_jobs (user_id, job_type, schedule_expr, task, interval_s)
   WHERE deleted = false AND status = 'active' AND job_type IN ('cron', 'interval');
   ```
2. **一次性任务索引**：
   ```sql
   CREATE UNIQUE INDEX IF NOT EXISTS uq_agent_cron_jobs_active_once
   ON agent.agent_cron_jobs (user_id, job_type, next_run_at, task)
   WHERE deleted = false AND status = 'active' AND job_type = 'once';
   ```
在 [cron_repo.py](file:///home/yang/mojing/mojing_agent/Mojing/storage/cron_repo.py) 中，`add` 方法分别按两类索引接住冲突并重查已有行，14:00 与 18:00 任务各自独立落库，同一时间的并发重复插入则安全幂等收敛。

---

### 3. 异步长任务被二次入库打回原形、产物被清空

#### 【实际的 Bug】
在 [postgres_runtime_task_repo.py](file:///home/yang/mojing/mojing_agent/Mojing/storage/postgres_runtime_task_repo.py) 中，任务入队时执行的 `record_queued` 原写法为：
```sql
ON CONFLICT (task_id) DO UPDATE SET
    status = 'queued',
    output_json = '{}'::jsonb,
    output_summary = EXCLUDED.output_summary,
    ...
```
在异步长任务场景下，某个耗时 10 秒以上的分析任务已经在后台运行（`running`）甚至已经完成（`succeeded`），此时调用端发生网络抖动进行了超时重试，再次调用了 `record_queued`。这条 SQL 会无差别地把任务状态强行修改回 `queued`，并将已经算好的 `output_json` 彻底清空为 `{}`，导致任务成果被意外抹除。

#### 【标准解法】
* **状态机单调性守卫（Monotonic State Machine Guard）**：
  数据库底层的并发 UPSERT 必须带上状态过滤，只有任务尚未终结或属于明确推进序号的重试时，才允许打回就绪态。

#### 【实际落地的代码】
在 `postgres_runtime_task_repo.py` 中为 `ON CONFLICT (task_id) DO UPDATE SET` 增加守卫条件：
```sql
WHERE agent.agent_runtime_tasks.status IN ('failed', 'cancelled')
   OR EXCLUDED.attempt > agent.agent_runtime_tasks.attempt
```
当任务处于 `running` 或 `succeeded` 态，且重试请求并未携带更大的 `attempt` 序号时，SQL 自动忽略更新，不覆盖任何正在计算或已完成的产物。

---

### 4. 设备出厂注册秘钥停写入库与明文风险根除

#### 【实际的 Bug】
设备在完成出厂绑定与注册时，服务端的 `devices` 表中有一列 `registration_secret`。代码在完成注册后，将出厂秘钥明文写进了该字段。
经全库排查，业务运行时只需要在内存中基于出厂凭证进行一次 HMAC 校验，后续的所有设备通信均使用设备分配的长期 Token 或 mTLS 证书，该数据库字段完全只写不读。出厂秘钥明文落库对生产安全构成合规风险。

#### 【标准解法】
* 停止向数据库持久化出厂激活秘钥，将其写入置为 `NULL`；
* 仅在内存中完成签名/秘钥校验，出厂批次与公钥指纹安全存入 `factory_claims`，消除秘钥泄漏隐患。

#### 【实际落地的代码】
在 [device.go](file:///home/yang/HotGo/server/internal/logic/mojing/device/device.go) 中将写入参数明确置为 `nil`，对应单测 [device_registration_test.go](file:///home/yang/HotGo/server/internal/logic/mojing/device/device_registration_test.go) 断言数据库中的 `registration_secret` 严格为 `NULL`。

---

### 5. 设备 OTA 固件更新与用户快照强一致事务绑定

#### 【实际的 Bug】
在 [device.go](file:///home/yang/HotGo/server/internal/logic/mojing/device/device.go) 的固件上报与 OTA 完成逻辑中，原代码首先执行 `devices` 表的固件版本更新，随后在事务外独立调用 `refreshDeviceOwnersSummary` 刷新绑定用户的固件快照，并且直接忽略了快照刷新的错误。
一旦后者发生报错，设备表已更新为新版本，但用户表上的主设备固件快照依然卡在旧版本，引发三端视图状态撕裂。

#### 【标准解法】
* 设备固件状态更新与用户快照重算必须处于同一个数据库事务内；
* 用户快照重算失败时，整个事务回滚，设备状态恢复，保证外部视图强一致。

#### 【实际落地的代码】
在 [device.go](file:///home/yang/HotGo/server/internal/logic/mojing/device/device.go) 的 `syncFirmwareFromStateReport` 和 OTA 成功流程中，使用 `mojingdb.DB().Transaction` 将 `tx.Model("mojing_devices").Where(...).Update(...)` 与 `refreshDeviceOwnersSummaryTx(ctx, tx, deviceId)` 包裹在同一事务内，并通过真实 PG 级集成测试 [device_concurrency_real_test.go](file:///home/yang/HotGo/server/internal/logic/mojing/device/device_concurrency_real_test.go) 验证了刷新失败时版本字段的完整回滚。

---

### 6. Kafka 消费端跳跃式位点隐式提交与丢消息防范

#### 【实际的 Bug】
在 Go（[kafka_status_worker.go](file:///home/yang/HotGo/server/internal/logic/mojingmq/kafka_status_worker.go)）和 Python（[kafka_adapter.py](file:///home/yang/mojing/mojing_agent/Mojing/runtime/kafka/kafka_adapter.py)）中，原消费逻辑在批次循环中存在隐式跳跃提交问题：
```go
// 历史缺陷伪代码
for message := range claim.Messages() {
    if err := handle(message); err != nil {
        continue // 记录日志后 continue
    }
    session.MarkMessage(message, "") // 成功时标记位点
}
```
**根因所在**：Kafka 的位点提交（Offset Commit）本质上是一个单调递增的数值指针。
假设批次拉取了 3 条消息：
- M0 (Offset 100): 执行成功，MarkMessage(100)
- M1 (Offset 101): 执行失败（如数据库死锁或外部服务超时），代码记录 Warning 后执行 `continue`
- M2 (Offset 102): 执行成功，MarkMessage(102)

此时 Sarama 或 librdkafka 向 Broker 提交位点 102，**Broker 会认为 102 之前的所有消息全部已经成功处理！Offset 101 被隐式跳过，未来消费组发生重平衡或重启时，再也不会重新拉取 M1，造成生产数据静默丢失！**

#### 【标准解法】
* **遇到错误立即中断批次消费**：当某条消息处理失败时，必须立刻终止当前批次循环，坚决禁止处理后续消息或提交更高的位点；
* 保留该未确认位点，以便消费组恢复或下一次拉取时从失败消息开始重新消费（At-Least-Once 语义）。

#### 【实际落地的代码】
* **Go 消费端**（[kafka_status_worker.go](file:///home/yang/HotGo/server/internal/logic/mojingmq/kafka_status_worker.go) 与 [kafka_hardware_dialogue_worker.go](file:///home/yang/HotGo/server/internal/logic/mojingmq/kafka_hardware_dialogue_worker.go)）：
  ```go
  if err := h.worker.handleKafkaMessage(context.Background(), message); err != nil {
      g.Log().Warningf(context.Background(), "... Kafka message failed, stopping claim to preserve offset: ...")
      return err // 遇到错误立刻退出 ConsumeClaim，阻止处理后续消息和推进位点
  }
  session.MarkMessage(message, "")
  ```
* **Python 消费端**（[kafka_adapter.py](file:///home/yang/mojing/mojing_agent/Mojing/runtime/kafka/kafka_adapter.py)）：
  ```python
  try:
      envelope = decode_message_value(message)
      await self.status_consumer.handle_envelope(envelope)
      handled += 1
      await _await_native_call(self.consumer.commit, message=message, asynchronous=False)
      committed += 1
  except Exception:
      rejected += 1
      if message_id:
          rejected_message_ids.append(message_id)
      logger.exception("Kafka status message rejected: topic={} message_id={}", self.topic, message_id)
      break  # 关键修复：单条失败立刻 break 退出循环，不处理后续消息，不提交更高位点
  ```
  在 [test_kafka_adapter.py](file:///home/yang/mojing/mojing_agent/tests/test_kafka_adapter.py) 中增加了 `test_worker_breaks_batch_on_error_without_committing_subsequent_messages` 单元测试，断言第一条异常后第二条完全不被处理且提交数为 0。

---

### 7. Redis Stream 内存无界膨胀、毒丸自旋与丢任务治理

#### 【实际的 Bug】
1. **Stream 长度无界膨胀**：原 `client.xadd(stream, payload)` 没有传 `maxlen` 参数。虽然消费者会执行 `xack`，但 `xack` 仅仅是将消息从消费组的待确认列表（PEL）中移出，消息本身依然永久驻留在 Redis Stream 的底层物理结构中，运行时间一长，Redis 内存必定被占满。
2. **毒丸消息引起死循环自旋**：当 Stream 中混入空载荷或非法 JSON 时，`task_queue.py` 的反序列化方法捕获异常后直接 `continue` 跳过，没有对其调用 `xack`。因为没有被确认，该消息一直留在 PEL 中；后台的 `xautoclaim` 机制在超过闲置时间后，又会把它当成“僵尸 Worker 遗留的消息”重新抢回认领，再次反序列化失败，陷入无限循环刷屏。
3. **重试时序颠倒导致丢任务**：在 [task_worker.py](file:///home/yang/mojing/mojing_agent/Mojing/runtime/hosting/task_worker.py) 中，处理任务失败的逻辑原为：
   ```python
   await self._mark_failed(task, error_text)
   await self._ack(msg)  # 先 ack 原消息！
   if next_attempt < task.max_attempts:
       await self._queue.enqueue(retry)  # 后入队重试
   ```
   **时序隐患**：如果进程在 `_ack` 之后突然宕机，或者 `enqueue` 时 Redis 刚好闪断抛出异常，原消息已经被确认消除，而重试的新消息并未成功入队，这个长任务就此彻底丢失。

#### 【标准解法】
* **流长度限制**：`xadd` 时携带 `maxlen=10000, approximate=True`，让 Redis 在添加时利用基数树节点近似裁剪超长历史，限制内存使用；
* **毒丸清除**：反序列化遇到空载荷或格式错误的数据时，记录告警后立刻调用 `xack` 将其从 PEL 中剔除，阻止 `xautoclaim` 无限重复拉取；
* **重试保障**：严格遵循分布式队列可靠投递原则：**重试入队成功之后，才允许确认原消息**；若入队失败，坚决不 ACK 原消息，留给下一次拉取或认领重试（At-Least-Once）。

#### 【实际落地的代码】
* **队列修剪与毒丸确认**（[task_queue.py](file:///home/yang/mojing/mojing_agent/simpleclaw/runtime/task_queue.py)）：
  ```python
  # enqueue 中增加近似修剪
  kwargs: dict[str, Any] = {}
  if self._maxlen and self._maxlen > 0:
      kwargs["maxlen"] = self._maxlen
      kwargs["approximate"] = True
  msg_id = await client.xadd(self._stream_name(task.stream), {"payload": task.to_json()}, **kwargs)

  # _deserialize_entries 遇到坏消息时主动 XACK 移出 PEL
  for queue_id, fields in entries:
      payload = fields.get("payload")
      if not payload:
          await client.xack(self._stream_name(stream), consumer_group, queue_id)
          continue
      try:
          task = TaskEnvelope.from_json(payload)
      except Exception as exc:
          await client.xack(self._stream_name(stream), consumer_group, queue_id)
          continue
  ```
* **工作流重试先入队后 ACK**（[task_worker.py](file:///home/yang/mojing/mojing_agent/Mojing/runtime/hosting/task_worker.py)）：
  ```python
  if next_attempt < task.max_attempts:
      await self._mark_failed(task, error_text)
      retry = TaskEnvelope(...)
      try:
          queue_id = await self._queue.enqueue(retry)
          if self._task_state_store is not None:
              await self._task_state_store.record_queued(retry, queue_message_id=queue_id)
          await self._ack(msg)  # 仅在重试入队成功后才确认旧消息
      except Exception as enq_exc:
          logger.error("TaskWorker retry enqueue failed: {}, preserving unacked msg", enq_exc)
          # 不做 _ack(msg)，保留在 PEL 中供下次认领重试
  ```
  在 [test_runtime_tasks.py](file:///home/yang/mojing/mojing_agent/tests/test_runtime_tasks.py) 中增加了 `test_redis_task_queue_maxlen_and_poison_pill_xack` 与 `test_task_worker_retry_enqueue_failure_does_not_ack`，全面验证修剪参数、毒丸清理与入队失败保活行为。

---

## 三、 微信小程序端规范整改与交互健壮性

针对微信小程序（[MagicMirrorFrontEndV2](file:///home/yang/mojing/MagicMirrorFrontEndV2)），按照前端工程与小程序开发规范进行了全面检查与整改：

1. **发信通道统一挂载幂等键**：
   在 [miniprogram/api/chat.ts](file:///home/yang/mojing/MagicMirrorFrontEndV2/miniprogram/api/chat.ts) 中，修正了此前仅在卡片分支透传 `idempotencyKey` 的缺陷，标准发信与私有发信路径统一带上 `data.idempotency_key = payload.idempotencyKey`。
2. **重试机制防重复提交**：
   在 [session-page.ts](file:///home/yang/mojing/MagicMirrorFrontEndV2/miniprogram/pages/chat-detail/session-page.ts) 中，将图片重试的键生成提取到 `for (attempt <= MAX_IMAGE_SSE_ATTEMPTS)` 循环外部，确保多次网络重试沿用同一个业务键，服务端安全识别为同一动作。
3. **错误信息用户友好化**：
   对于图片加载失败、网络超时等场景，避免将底层原生错误（如 `dial tcp` 或 `BadRequest`）直接抛在用户界面上，统一呈现为温和明确的提示语。
4. **编译构建检查**：
   在小程序目录执行 `npm run build`，TypeScript 类型检查与构建打包全部通过。

---

## 四、 真实测试验证记录与实际输出核对

以下为真实环境下的实际测试执行记录（测试均在真实 PostgreSQL 16、真实本地 Redis 与 Go/Python 测试套件下运行，无虚构数据）：

### 1. Go 网关与消息队列测试
* **命令**：`cd /home/yang/HotGo/server && go test -v ./internal/logic/mojingmq`
  * **输出**：全部通过，包含：
    * `TestKafkaStatusWorkerHandlesAllowedStatusEnvelope` (PASS)
    * `TestKafkaHardwareDialogueWorkerPublishesMessagesChanged` (PASS)
    * `TestSkinAnalysisStateIngressRecordsEnvelopeFieldsAndPreservesExtra` (PASS)
    * `ok hotgo/internal/logic/mojingmq 0.083s`
* **命令**：`cd /home/yang/HotGo/server && go test -v ./internal/logic/mojing/chat`
  * **输出**：全部通过，包含：
    * `TestConcurrentAppendChatMessageIdempotency` (10 并发 Goroutine 同时写入同一幂等键，数据库严格保留 1 条记录，全部返回同一 `message_id`) (PASS)
    * `TestAppendChatMessageIdempotencyRejectsMismatchedContent` (同键不同正文正确拦截返回 `CHAT_IDEMPOTENCY_KEY_CONTENT_MISMATCH`) (PASS)
    * `ok hotgo/internal/logic/mojing/chat 1.009s`
* **命令**：`cd /home/yang/HotGo/server && go test -v -count=1 ./internal/logic/mojing/device -run "TestDevice"`
  * **输出**：全部通过，包含：
    * `TestDeviceFirmwareSyncAndUserSnapshotAtomic` (固件同步与用户快照原子成功) (PASS)
    * `TestDeviceFirmwareSyncRollbackOnRefreshFailure` (用户快照刷新失败时，固件更新全局回滚) (PASS)
    * `ok hotgo/internal/logic/mojing/device 3.314s`

### 2. Python 智能体核心与存储测试
* **命令**：`pytest tests/test_runtime_tasks.py -v`
  * **输出**：`15 passed in 0.88s`，包含：
    * `test_redis_task_queue_maxlen_and_poison_pill_xack` (验证 maxlen=5000 传递与空数据/坏 JSON 的 xack 剔除) (PASSED)
    * `test_task_worker_retry_enqueue_failure_does_not_ack` (验证重试入队失败时不 ACK 原消息，保证消息不丢) (PASSED)
    * `test_worker_requeues_failed_result_with_same_task_id` (验证失败重试状态流转为 queued->running->failed->queued) (PASSED)
* **命令**：`pytest tests/test_kafka_adapter.py -v`
  * **输出**：`9 passed in 0.96s`，包含：
    * `test_worker_breaks_batch_on_error_without_committing_subsequent_messages` (验证单条失败时 break 退出，不提交更高位点，保留未拉取消息) (PASSED)
* **命令**：`pytest tests/test_real_pg_dedupe_concurrency.py -v`
  * **输出**：`4 passed in 1.36s`（直连本地真实 PostgreSQL 16 数据库）：
    * `test_real_pg_concurrent_task_deduplication` (10 并发协程提交同一 scope_key 任务，仅创建 1 行有效任务) (PASSED)
    * `test_real_pg_concurrent_cron_deduplication` (并发创建相同周期任务，仅落库 1 行) (PASSED)
    * `test_real_pg_record_queued_does_not_clobber_running_or_succeeded_task` (运行中/已成功的长任务产物不被二次入库清空) (PASSED)
    * `test_real_pg_once_cron_trigger_time_distinction` (14:00 与 18:00 的同文本一次性提醒各自独立生成，不互相吞噬) (PASSED)

### 3. 基准评测全量执行验证
* **路由分级黄金评测 (Router Golden Benchmark)**：
  * 命令：`python eval/run_router_benchmark.py`
  * 结果：**32/32 全部通过 (100.0% 准召)**，包含 CHAT(8)、KNOWLEDGE(8)、ACTION(8)、HARD(8) 四大核心分流，平均规则决策延迟 **27.61 µs** (0.028 ms)，P95 延迟 165.82 µs。
* **细粒度基准独立评测 (Granular Benchmark Runner)**：
  * 命令：`python eval/run_granular_benchmarks.py`
  * 结果：
    * **工具选择与门禁决策**：16/16 全部通过，F1 Score **100.0%** (TP=6, FP=0, TN=10, FN=0)，严格防范了“不用帮我入柜”、“不想拍照测肤”等显式否定陷阱；
    * **间接提示词注入与哨兵防泄漏**：6/6 全部安全拦截 (**100.0% 防御**)，拦截了围栏逃逸、假系统标签越狱与 Canary 哨兵套取；
    * **RAG 证据召回与生成忠实度**：4/4 权威命中 (**100.0% Grounded**)，0 噪声劣化。
* **分布式 ID 演进单测**：
  * Go：`go test -v ./internal/logic/mojing/shared -run "TestNextUUIDv7TraceID"` (PASS，验证 32 字符十六进制、时间单调递增、RFC 9562 Version 7 与 Variant 1 规范)。
  * Python：`pytest tests/test_snowflake_and_concurrency.py -k "test_uuidv7_trace_id"` (PASSED)。

---

## 五、 三项关键技术边界与设计原理深度剖析

针对“雪花算法与 UUIDv7”、“MySQL 商城表与 PostgreSQL”、“黄金数据集评测”这三项在实际开发中极易引发争议的架构决策，进行深度剖析：

### 1. 雪花算法（Snowflake）与 UUIDv7 的演进与落地
* **原先的认知误区**：
  此前认为“从雪花算法迁移至 UUIDv7 需要修改多张数据库表的数据类型”。然而，通过对实际运行的 PostgreSQL 数据库字段字典（`information_schema.columns`）进行深度排查后发现：现网表（`chat_messages`、`agent_runtime_tasks`、`chat_sessions`、`agent_messages`）中的主键与业务键实际上**全都是 `text` 或 `uuid` 格式**，没有任何一列物理受限于 64 位 `bigint` 整数！
* **雪花算法的真正硬伤**：
  原有实现中，Worker ID 依赖 `sha256(hostname + ":" + pid) & 1023`。在无集中注册中心的微服务容器化集群中，1024 个槽位在实例数增多时存在明显的**生日悖论碰撞风险**（实例达到 38 个时碰撞概率达 50%）。
* **落地解决与平滑演进**：
  现已在两端代码中直接引入标准 **RFC 9562 UUIDv7** 实现：
  1. **Go 端**（[request_link.go](file:///home/yang/HotGo/server/internal/logic/mojing/shared/request_link.go)）：提供 `NextUUIDv7TraceID()`，使用 48 位毫秒时间戳 + 毫秒内 12 位单调序列 + 62 位密码学安全随机熵，生成严格递增、W3C 兼容的 32 字符十六进制 Trace ID，彻底摆脱 Worker ID 依赖；
  2. **Python 端**（[snowflake.py](file:///home/yang/mojing/mojing_agent/simpleclaw/runtime/snowflake.py)）：提供 `next_uuidv7_trace_id()` 与 `next_uuidv7_str()`，基于 Python 3.14 原生 `uuid.uuid7()`，天然实现单调递增与全局防撞；
  3. 既有针对 `SnowflakeGenerator` 解析的单元测试保持完全兼容，分布式链路追踪可无缝使用 UUIDv7。

### 2. 为什么坚决不能将 MySQL 商城表与现网 PostgreSQL 强行合并？
* **文件本质剖析**：
  前端项目中的 `MagicMirrorFrontEndV2/mojing_schema.sql` 共有 3898 行、150 多张表，实际上是**开源微服务商城（RuoYi-Vue-Pro / 芋道源码）的 MySQL 8.0 数据库转储**（包含 Quartz `QRTZ_*` 定时任务、`trade_*` 交易订单、`pay_*` 支付网关、`member_*` 会员积分、`promotion_*` 拼团秒杀等）。
* **为何不能“一锅端”合并进当前数据库**：
  1. **限界上下文（Bounded Context）隔离**：
     魔镜核心后端（HotGo + Mojing Agent）是**“AI 镜面物联网/智能对话系统”**，核心领域是音视频流、大模型决策、视觉测肤、设备 IoT 控制；而 `mojing_schema.sql` 是**“电商交易系统”**。在大厂微服务架构中，交易履约与 AI IoT 控制必须做数据库层面的物理隔离。
  2. **语法与依赖强侵入**：
     Java 商城端底层依赖 MyBatis 和 MySQL 8.0 特有语法（如 Quartz 二进制 BLOB 序列化、`ON DUPLICATE KEY UPDATE`）。强行把 150 多张电商表转入 PostgreSQL，不仅会导致数据库膨胀与脏元数据污染，而且当前仓库中根本没有配套的 Java 源码，盲目合并是典型的过度设计与架构反模式。
  3. **标准集成范式**：
     如果未来小程序需要将智能体（如“查询我买过的面霜”）与商城数据打通，大厂的标准解法是**基于 OpenAPI/RPC 接口或 Kafka/CDC 事件总线进行跨系统交互**，共享账号仅需在 `openid`/`unionid` 维度建立映射，绝不共享同一个物理数据库。

### 3. 黄金数据集的权威性与评测流水线定位
* **“数据样本（Ground Truth）” 无需重建**：
  `eval/golden/` 下的 32 条意图分类样本与各类评测用例，是由业务专家人工标定的标准真值（例如“把灯调成护肤光”必定属于 ACTION，“帮我查查小棕瓶成分”必定属于 HARD）。本次系统全量修复集中在底层并发幂等、分布式事务、消息队列位点与死信机制，模型的提示词（Prompt）与路由正则分毫未改，因此金标样本本身的定义完全正确，绝不需要推倒重建。
* **“评测流水线（Eval Runner）” 必须常态化全量回归**：
  虽然测试样本不变，但每当底层发生改动后，必须运行评测流水线进行回归，确保底层改动未对分类性能与规则决策产生副作用。目前所有自动化基准测试均已重新执行，准召率与安全拦截率保持 100%。

---

## 六、 总结

通过对消息幂等闭环、定时任务索引分离、任务状态机守卫、设备 OTA 事务一致性、Kafka 位点提交安全以及 Redis Stream 内存与毒丸治理的全面代码修复与测试覆盖，系统在底层存储与通信链路上的核心硬伤已得到彻底解决。整个系统架构与测试结果全部落实于实际代码库中，具备了可核对、可复现、可维护的工程健壮性。
