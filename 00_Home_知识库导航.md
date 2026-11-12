# 🏠 SimpleClaw & Mojing 架构知识库控制台

> [!TIP]
> **欢迎来到个人工程与架构知识库！**
> 本知识库已打通 Windows Obsidian 与 WSL2 终端环境，支持双向实时同步、Mermaid 架构渲染与 Git 版本备份。
> 当前主干涵盖 **AI Agent 核心循环、上下文工程、高可用消息队列（Kafka/Redis）、Go 业务微服务、事务发件箱演进及大厂面试真题**。

> [!NOTE] 🎨 **架构全景白板现已就绪**
> 👉 **强烈推荐打开（Obsidian 官方原生现代白板）**：[[SimpleClaw_Agent架构全景.canvas|🌟 SimpleClaw Agent 核心全景架构图 (Canvas 现代白板)]]
> 👉 **手绘风白板**：[[SimpleClaw_Agent整体架构手绘图.excalidraw|🎨 SimpleClaw Agent 手绘架构图 (Excalidraw)]]
> *(点击 Canvas 白板即可获得现代卡片化排版，字迹清晰、连线整洁、支持无限缩放和鼠标拖拽)*

---

## 🗺️ 架构全局学习与阅读主干

```
                         【智能体心跳中枢】
                 01_core ([[01_react_loop.md]])
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
    【上下文装配引擎】              【大模型适配驱动】
02_context ([[02_context_engineering.md]])   03_llm ([[03_volcengine.md]])
            │                               │
            └───────────────┬───────────────┘
                            ▼
                    【工具治理与发件箱】
                   04_tools ([[04_tools.md]])
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
     【短期会话与长期记忆】          【并发控制与状态机】
05_memory ([[05_memory.md]])  06_runtime ([[06_concurrency_runtime.md]])
            │                               │
            └───────────────┬───────────────┘
                            ▼
                    【异步睡梦蒸馏整理】
                  07_dream ([[07_dream.md]])
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
     【子智能体多Agent协同】         【长任务工作项检查单】
08_subagent ([[08_subagent.md]]) 09_workitem ([[09_workitem.md]])
            │                               │
            └───────────────┬───────────────┘
                            ▼
                    【技能插件与SDK总线】
                  10_skills & 11_sdk_api
```

---

## 📚 知识库四大核心专区直达

### 1. 核心智能体框架 (01 ~ 11)
- [[01_core/01_react_loop.md|01. ReAct 核心循环与打断回滚机制]]
- [[02_context/02_context_engineering.md|02. 上下文工程与动态 Token 滑动窗口裁剪]]
- [[06_runtime/06_concurrency_runtime.md|06. 高并发运行时、分布式排他锁与任务队列]]
- [[07_dream/07_dream_pipeline.md|07. Dream 异步睡梦沉淀与记忆整理]]
- [[08_subagent/08_subagent_runtime.md|08. 多 Agent 协同、上下文隔离与交接契约]]

### 2. 中间件与高可用架构专项 (MQ & Storage)
- [[15_redis_stream_kafka_double_hop_to_transactional_outbox.md|⭐ 15. 深度复盘：从 Redis Stream 双跳到 PostgreSQL 事务发件箱演进]]
- [[kafka.md|Kafka 工业级架构、单分区保序与死信队列全景]]
- [[redis+kafka.md|Redis 与 Kafka 的职责边界与极致互补]]
- [[redis stream.md|Redis Streams 内部原理与消费者组自愈机制]]
- [[go.md|Go 语言并发编程、GMP 调度器与微服务实战]]

### 3. 系统容灾、运维与实战手册
- [[mojing_production_resilience_and_pitfalls_handbook.md|魔镜生产韧性与排障踩坑手册]]
- [[整体架构.md|魔镜系统端到端全链路数据流转总览]]
- [[云端服务与代码对照.md|云端火山引擎服务与本地代码一一对应映射]]
- [[K8S+Prometheus.md|云原生容器化部署与监控告警指标]]
- [[RAG.md|企业级检索增强生成 (RAG) 调优实录]]

### 4. 大厂求职与面试真题复盘
- [[14_architecture_audit.md|14. Agent 与 HotGo 真实代码选型与造轮子复盘]]
- [[13_production_gaps_and_answers.md|13. 生产差距诊断与面试官必问硬核解答]]
- [[12_architecture_selection_and_interview_guide.md|12. 架构选型与面试应答通用指南]]

---

## ⚡ Dataview 动态笔记库追踪 (最近更新)

```dataview
TABLE file.mtime AS "最后修改时间", file.folder AS "所在目录", file.size AS "大小(字节)"
FROM ""
WHERE file.name != "00_Home_知识库导航"
SORT file.mtime DESC
LIMIT 10
```

---

## 🛠️ 知识库快捷工作流指南
* **全局代码与全文检索**：按 `Ctrl + Shift + F` 或打开左侧 **Omnisearch** 搜索任意技术点；
* **查看神经元网状图谱**：按 `Ctrl + G` 打开 **Graph View**，即可直观查看技术体系的相互引用；
* **画架构草图**：随时在左侧点击 **Excalidraw** 新建交互式白板画图；
* **Git 自动备份**：已初始化 Git 仓库，可在左下角状态栏点击 Git 图标手动拉取或提交。
