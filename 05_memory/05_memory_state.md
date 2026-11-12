# 05_memory：状态机、因果账本与长效记忆架构全景剖析

> **文件归属**：[`simpleclaw/memory/base.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/base.py)、[`ledger.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/ledger.py)、[`ledger_store.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/ledger_store.py)、[`session.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/session.py) 
> **模块定位**：会话窗口和记忆账本。账本记下原文、消息序号、改前改后。这不是数据库 WAL，记录本身也可以被后续整理改掉。 
> **推荐阅读顺序**：**第 5 篇**（在掌握上下文与工具后，剖析智能体如何记住用户、管理状态并持久化因果链路）

---

## 一、记忆要能回答「这句话从哪来」

只把「用户容易长痘」存成一条文本，下次能召回，但改错了不知道是哪几轮对话推出来的。提炼记忆如果堵在用户等待回复的那条路上，体感会变慢。同一件事前后说法相反时，两条都塞进提示词，模型会打架。

LangChain 的记忆组件、向量库、LangGraph 检查点都可以留下对话和状态。检查点还能从某一步继续。账本多记的是：这段原文、消息序号、写入前和写入后。

记忆提炼不放在用户等待的那条回复里。检索不是零模型：`TextMemoryRetriever` 默认会再调一次模型，从索引里挑条目。`fast_hybrid_only` 为真时才只走关键词。证据检索还用了 BM25。不要说成每次都是向量检索，也不要说成热路径完全不调模型。没有 10 毫秒这种测量。

| 要解决的事 | 我们的默认做法 |
| :--- | :--- |
| 回复时不要等记忆提炼 | 写入和归并放到后面。检索证据时仍可能先调一次挑选模型。 |
| 记错了要能查来源 | 账本记下原文切片 `source_chunk`、消息序号 `message_seq_start` / `message_seq_end`，以及是谁触发的。序号用的就是循环里那个「从头数的第几条」。 |
| 改错了要能看改前改后 | 写入前拍 `memory_before`，写入后拍 `memory_after`，具体改动放在 `memory_actions`。 |
| 同一事实不要存成互相打架的好几条 | 用稳定的 key（如 `user.allergy.alcohol`）覆盖旧值。同名事实只留一份。 |
| 白天的流水还要以后再整理 | 账本上的睡梦状态从待处理走到候选，再走到已应用。整理规则在睡梦模块，不堵主回复。 |

### 面试时可以怎么说

> 回复前不在这条路上现写长记忆。要查「之前说过什么」时，检索器可能再调一次模型挑条目。每条账记下原文、消息序号、改前改后。同一 key 覆盖旧值。检查点和向量库也能存状态。产品证据检索当前是索引加 BM25，再按配置决定要不要让模型挑选，不是默认的向量库。

---

## 二、核心类与数据协议全景

### 1. 记忆实体与接口（定义于 [`simpleclaw/memory/base.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/base.py)）

#### (1) `MemoryItem`（长效记忆数据单元）
```python
@dataclass
class MemoryItem:
    key: str              # 唯一主题标识符 (例如 "user.skin.barrier_status", "user.allergy.alcohol")
    content: str          # 完整事实文本正文
    description: str = "" # 单行超短摘要 (用于在目录区简短展示)
    metadata: dict = ...  # 扩展元数据 (置信度、更新时间、来源seq等)
```

#### (2) `Memory`（抽象基类契约）
- `async def store(self, key: str, content: str, *, description="", metadata=None) -> None`:
  - 同一个 key 覆盖旧内容，避免同一个 key 下留两份相反事实。两个不同 key 仍可能描述相反的事，覆盖解决不了这个。
- `async def retrieve(self, query: str = "", top_k: int = 20) -> list[MemoryItem]`:
  - 魔镜证据检索是 `TextMemoryRetriever`：索引预览，默认再让模型选编号，也可以只走关键词。另有 BM25。不要写成毫秒级向量检索。
- `async def as_section(self, query: str = "", top_k: int = 20) -> str`:
  - 组装为结构化 Markdown 区块直接注入动态尾部；若无记忆安全返回空字符串，不产生多余占位符。

---

### 2. 因果账本模型（定义于 [`simpleclaw/memory/ledger.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/ledger.py)）

#### (1) `MemoryLedgerRecord`（因果账本核心记录）
记录一次记忆提取事件的前因后果全貌：
- `ledger_id`: 带有 `memledger_` 业务前缀的全局唯一主键。
- `trigger_type`: 触发原因（`"context_compression"`, `"in_turn_compression"`, `"manual"`, `"dream"`）。
- `status`: 账本流转状态（`"queued"` -> `"extracting"` -> `"applied"` / `"failed"` / `"skipped"`）。
- **因果与截断切片**：
  - `message_seq_start` / `message_seq_end`: 被丢弃消息在全局会话中的绝对序号区间。
  - `dropped_count`: 被丢弃的消息条数。
  - `source_chunk`: 提取时留下的原文切片。热窗口裁掉以后，还能从账本看到这段来源。它不是会话全量历史的永久备份。
- **快照与变更追踪（Diffability）**：
  - `memory_before`: 提取前的完整记忆快照（`MemorySnapshot`）。
  - `memory_actions`: 本次执行的原子操作列表（如 `[{"op": "add", "key": "...", "val": "..."}]`）。
  - `memory_after`: 提取并覆盖后的最新记忆快照。
- **离线治理**：
  - `dream_status`: 梦境处理状态（`"pending"`, `"candidate"`, `"reviewed"`, `"applied"`）。

---

## 三、核心流转状态机与存储契约

### 1. 记忆账本生命周期流转状态机

```mermaid
stateDiagram-v2
    [*] --> queued: 滑窗压缩丢弃旧消息<br/>生成初始账本与 source_chunk 备份
    queued --> extracting: 后台 Worker 认领任务<br/>调用大模型分析丢弃的对话切片
    
    extracting --> applied: 成功提炼新事实<br/>原子写入 Memory 库并记录 after 快照
    extracting --> skipped: 经评估丢弃文本全为寒暄废话<br/>无需提炼长期事实，安全跳过
    extracting --> failed: 模型超时或 JSON 损坏<br/>记录 last_error 便于告警重试
    
    applied --> [*]
    skipped --> [*]
    failed --> queued: 故障自愈重试
```

### 2. 存储协议与接口 (`MemoryLedgerStore`)
定义于 [`simpleclaw/memory/ledger_store.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/memory/ledger_store.py)：
- `create_ledger(record)`: 初始入库（状态 `queued`）。
- `update_ledger(ledger_id, status, memory_actions, ...)`: 增量补丁更新状态与快照。
- `list_dream_pending(tenant_key, limit)`: 批量拉取 `dream_status == "pending"` 的候选记录供夜间离线反思。

---

## 四、全场景实战时序图（Mermaid 8.14.0+ 规范）

### 场景 1：热读路径：检索后注入动态尾巴

展示用户提问时，热路径只做检索，再把结果放进动态尾巴：

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant AgentLoop as ReAct核心循环
participant Builder as ContextBuilder
participant Mem as Memory存储后端
participant LLM as 大语言模型(LLM)

Note over User, AgentLoop: 用户发起提问
User->>AgentLoop: "我最近脸颊又红了，可以用之前买的精华吗？"
AgentLoop->>Builder: build_runtime_items(history, query="脸颊又红 精华")

Note over Builder, Mem: 热读路径 (只做检索与拼装，不在主回复路径上现提炼记忆)
Builder->>Mem: retrieve(query="脸颊又红 精华", top_k=5)
Mem->>Mem: 快速检索: 命中 "user.skin.barrier_status" 与 "user.avoid.acid"
Mem-->>Builder: 返回 MemoryItem 列表: [酒精过敏, 屏障轻度受损禁用酸类]

Note over Builder, LLM: 渲染为 Markdown 区块并注入动态尾部
Builder->>Builder: as_section() -> 生成 "# Memory\n- user.skin..."
Builder->>Builder: 组装进入 Dynamic Tail (不破坏 Stable Prefix 缓存)
Builder-->>AgentLoop: 返回装配好的 RuntimeInputItems
AgentLoop->>LLM: 发起推理流 (大模型具备精准上下文记忆)
LLM-->>User: "您之前有屏障受损和酒精过敏史，那款含高浓度水杨酸的精华目前切忌使用！"
```

---

### 场景 2：冷写路径：滑窗压缩触发因果账本生成与异步记忆提取流

展示当历史消息超限被丢弃时，因果账本如何实现源头备份、异步提取与两阶段状态流转：

```mermaid
sequenceDiagram
autonumber
participant AgentLoop as ReAct核心循环
participant Comp as ContextCompressor
participant Worker as 后台异步Worker
participant Store as MemoryLedgerStore
participant Mem as Memory存储后端
participant LLM as 大语言模型(LLM)

Note over AgentLoop, Comp: 上下文滑动压缩切片
AgentLoop->>Comp: compress_window() 发现历史超标 (Token=6500)
Comp->>Comp: 切割切片: dropped = messages[0:4] (第0~3条消息)
Comp->>Worker: 异步后台触发 on_compress(dropped)
Comp-->>AgentLoop: 立即返回截短后的窗口，主循环继续回复用户 (零阻塞)

Note over Worker, Store: 步骤 1：创建初始因果账本 (状态: queued)
Worker->>Store: create_ledger(record with status='queued',<br/>seq_start=0, seq_end=3, source_chunk=dropped)
Store-->>Worker: 账本已持久化入库 (ledger_id="memledger_01")

Note over Worker, LLM: 步骤 2：异步提炼事实 (状态推进: extracting)
Worker->>Store: update_ledger(status='extracting')
Worker->>Mem: 拍摄当前记忆快照 -> memory_before
Worker->>LLM: 提炼指令: "分析下方丢弃的4条对话，提取用户长期事实"<br/>+ dropped文本
LLM-->>Worker: 提炼结果: {"action": "set", "key": "user.allergy.alcohol", "content": "酒精过敏"}

Note over Worker, Mem: 步骤 3：写入 Memory 终态库并更新账本 (状态: applied)
Worker->>Mem: store(key="user.allergy.alcohol", content="酒精过敏")
Worker->>Mem: 拍摄写入后记忆快照 -> memory_after
Worker->>Store: update_ledger(status='applied', memory_actions=[...],<br/>memory_before=..., memory_after=..., completed=True)
Store-->>Worker: 账本因果链路归档完毕
```

---

### 场景 3：记忆自愈与等值覆盖 (Upsert) 冲突消解流

展示用户修改自身习惯时，系统如何通过同一语义 Key 强覆盖消除认知自相矛盾：

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant AgentLoop as ReAct核心循环
participant Mem as Memory存储后端

Note over User, Mem: 历史初始状态
Note over Mem: 当前库中记录: key="user.preference.water_temp",<br/>content="习惯用冷水洗脸"

Note over User, AgentLoop: 用户主动纠正自身习惯
User->>AgentLoop: "我发现用冷水洗脸脸颊反而更紧绷，我改成温水洗脸了。"
AgentLoop-->>User: "好的，已为您更新习惯，以后建议统一使用温水洁面。"

Note over AgentLoop, Mem: 异步提取与权威覆盖 (Upsert)
AgentLoop->>Mem: store(key="user.preference.water_temp",<br/>content="习惯用温水洗脸，自述冷水导致紧绷")
Mem->>Mem: 发现已存在相同的 key="user.preference.water_temp"
Mem->>Mem: 【原子覆盖】直接用新文本替换旧文本！
Note over Mem: 同一个 key 覆盖后只留「习惯用温水洗脸」。<br/>换一个 key 仍可能写出相反事实，覆盖不管那个。
```

---

### 场景 4：记忆快照比对与审计回滚流（Diff & Rollback）

展示当发现某次提取被恶意或错误污染时，如何依靠快照与原子动作执行反向回滚：

```mermaid
sequenceDiagram
autonumber
actor Admin as 运维/安全专家
participant Store as MemoryLedgerStore
participant Mem as Memory存储后端

Admin->>Store: get_ledger("memledger_01")
Store-->>Admin: 返回完整账本详情:<br/>- memory_before: [未记录疾病]<br/>- memory_actions: [{"op": "add", "key": "user.disease", "content": "重度痤疮"}]<br/>- memory_after: [含重度痤疮]

Note over Admin: 审查发现：用户其实是调侃朋友长痘，<br/>大模型产生幻觉误记为用户自身患病！
Admin->>Admin: 生成反向回滚补丁 (Inverse Patch):<br/>将 key="user.disease" 执行 delete 操作

Admin->>Mem: 删除污染的记忆条目 key="user.disease"
Admin->>Store: update_ledger("memledger_01", status="skipped",<br/>metadata={"rollback": True, "reason": "误提取调侃语"})
Store-->>Admin: 记忆状态已恢复至 memory_before 快照基线
```

---

### 场景 5：梦境队列拉取与离线知识提纯流（`list_dream_pending`）

展示夜间闲时，离线梦境引擎如何批量拉取待处理账本进行全局去重与知识升华：

```mermaid
sequenceDiagram
autonumber
participant DreamEngine as 离线梦境引擎(Dream)
participant Store as MemoryLedgerStore
participant LLM as 离线大模型推理
participant Mem as Memory存储后端

Note over DreamEngine: 示意。协议里的 cron 注释写过「例如凌晨 3 点」，代码没有把 03:00 写成排期。条数也是举例。

DreamEngine->>DreamEngine: 汇总这 12 条账本中的碎片事实与原话上下文
DreamEngine->>LLM: "对这批分散的肤况对话进行全局知识升华，消除冗余，归纳核心长期画像"
LLM-->>DreamEngine: 产出精炼的高阶知识: "用户具有季节性敏感特征，换季期耐受度极低"

DreamEngine->>Mem: store(key="user.profile.seasonal_sensitivity",<br/>content="季节性敏感特征，换季期耐受度极低")
DreamEngine->>Store: 批量更新 12 条账本 dream_status="applied"
Store-->>DreamEngine: 离线知识提纯完成
```
