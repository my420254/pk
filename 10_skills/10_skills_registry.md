# 10_skills：动态技能插件、渐进式暴露与 SOP 物化全景剖析

> **文件归属**：[`simpleclaw/skills/base.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/skills/base.py)、[`registry.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/skills/registry.py)、[`tools/builtin/skill.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/tools/builtin/skill.py)  
> **模块定位**：SimpleClaw 智能体内核的“动态技能插件与 SOP 挂载中枢”，负责基于文件即配置（`SKILL.md`）的规范解析、轻量索引常驻稳定前缀（Progressive Disclosure）、大模型自主按需激活以及正文场景化多轮物化。  
> **推荐阅读顺序**：**第 10 篇**。魔镜仓库里当前是 5 个 `SKILL.md`：护肤柜、深度报告场景、设备连接、日记场景、日记关注点回顾。目录变长会进稳定前缀，不要说成几十个或永远只有几十 token。

---

## 一、技能目录常驻，正文用到再加载

把十几篇长流程全写进系统提示词，token 和注意力都会差。只靠向量检索，用户说的是口语，标题对不上时会漏。

「目录很短、正文按需读」现在已经是常见做法，不少代码代理也用 `SKILL.md` 这一套。我们接进自己的上下文：目录放稳定前缀，正文放到动态尾巴。目录变了，前缀就变，缓存要重新算。不要说成目录一放进前缀就 100% 命中，也不要说成目录永远只有几十 token。技能一多，目录会变长。

`scene` 是进入一个场景后多轮都留着，结束时用 `unload_skill` 卸掉。`observation` 是这次看完就走。`unload_skill` 的 `needs_followup` 是 `False`，主循环不等它再开一轮。

| 要解决的事 | 我们的默认做法 |
| :--- | :--- |
| 流程正文太长，不能全塞进每轮提示词 | 文件是 `SKILL.md`：头上是名字、一句话、何时使用；正文用到再读。 |
| 模型得知道有哪些技能，又不能每次带全文 | 目录放稳定前缀。模型调用 `use_skill(name=...)` 再加载正文。 |
| 有的流程要跨几轮，有的看一眼就行 | `scene` 留到场景结束；`observation` 只在当次使用。 |
| 目录尽量稳定，好让前缀缓存还有机会命中 | 正文不进前缀。目录本身一改，哈希变了，缓存失效，这是正常的。 |

### 面试时可以怎么说

> 技能用 `SKILL.md`。系统提示词里只放目录：名字、一句话、什么时候用。模型决定要用时再把正文加载进来。场景类技能会留几轮，看完即走的只在当次出现。这和现在常见的渐进式技能是同一类做法。我们多接的是：目录放稳定前缀，正文放动态尾巴，并且和 `use_skill` / `unload_skill` 接在同一个循环里。目录变了，前缀缓存就会失效。

---

## 二、核心类与数据协议全景

### 1. 技能实体与描述符（定义于 [`simpleclaw/skills/base.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/skills/base.py)）

#### (1) `SkillDescriptor`（轻量元数据描述符）
`SKILL.md` 头部 YAML Frontmatter 的内存投射，体积极小：
```python
@dataclass(slots=True)
class SkillDescriptor:
    name: str                                    # 唯一英文标识名 (如 "objection_handling")
    description: str                             # 一句话功能简介
    when_to_use: str                             # 触发时机明确界定 (模型研判的关键依据)
    path: Path                                   # 物理文件路径
    materialization: SkillMaterialization = "observation"  # 物化模式: "scene" | "observation"
    metadata: dict[str, Any] = ...               # 业务自定义扩展字段 (category, tags 等)
```

#### (2) `SkillDocument`（完整技能文档）
包含元数据 `descriptor` 与剥离了 YAML 头部后的纯 Markdown 正文 `body`。正文仅在技能被激活时才会被加载。

#### (3) 物理文件规范 (`SKILL.md` 示例)
```markdown
---
name: objection_handling
description: 处理用户对护肤品价格或繁琐流程的疑虑
when_to_use: 当用户表达嫌贵、担心刺激或怀疑效果时使用
materialization: scene
category: negotiation
---
## 异议处理四步法核心 SOP:
1. 【共情倾听】: 认同用户的预算担忧，切忌当场反驳；
2. 【价值拆解】: 将产品总价换算为每日单次使用成本 (如单日仅需 3 元)；
3. 【成分证明】: 强调高浓度活性成分与第三方人体功效报告；
4. 【阶梯替代】: 若用户依然犹豫，推荐平价备选方案。
```

---

## 三、`SkillRegistry` 注册表与渐进式物化

定义于 [`simpleclaw/skills/registry.py`](file:///home/yang/mojing/mojing_agent/simpleclaw/skills/registry.py)：

### 1. 核心方法列表
- `discover()`: 递归扫描 `self._roots` 目录下的所有 `SKILL.md`，自动解析并缓存至 `_documents` 字典。
- `render_index()`: 导出所有可用技能的精简 YAML 目录（仅含名称、简述与触发时机），例如：
  ```yaml
  skills:
    - name: objection_handling
      description: 处理用户对护肤品价格的疑虑
      when_to_use: 用户嫌贵或怀疑效果时
  ```
  该目录放进 `ContextBuilder` 的稳定前缀。目录内容不变、服务端缓存还在时，这段才有机会命中。目录一改，前缀就变。
- `render_skill_body(name)`: 提取技能完整 Markdown 正文，供动态挂载。

---

## 四、全场景实战时序图（Mermaid 8.14.0+ 规范）

### 场景 1：技能渐进式暴露全景时序流（目录常驻 -> 自主决策 -> 按需激活）

展示整个渐进式暴露的经典闭环流程：

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant AgentLoop as ReAct核心循环
participant Builder as ContextBuilder
participant Tool as SkillTool(use_skill)
participant Reg as SkillRegistry
participant LLM as 大语言模型(LLM)

Note over User, LLM: 阶段 1：初始状态 (仅有微型索引，命中前缀缓存)
User->>AgentLoop: "你们推荐的精华液太贵了，几十毫升要上千块，根本不值！"
AgentLoop->>Builder: build_runtime_items()
Builder->>Reg: render_index()
Reg-->>Builder: 仅返回技能 YAML 摘要目录 (名称 + 一句话 + 触发条件)
Builder-->>AgentLoop: Prompt 前缀中包含 index: [objection_handling, ...]
AgentLoop->>LLM: 发起推理。前缀没变且服务端缓存还在时，这段前缀可以少算

Note over LLM, Tool: 阶段 2：模型自主研判并调用技能工具
LLM-->>AgentLoop: 发现用户嫌贵，自主发起工具调用: use_skill(skill_name="objection_handling")
AgentLoop->>Tool: execute(skill_name="objection_handling")
Tool->>Builder: activate_skill("objection_handling")
Builder->>Reg: require("objection_handling")
Reg-->>Builder: 返回 SkillDocument (materialization="scene")
Builder->>Reg: render_skill_body("objection_handling")
Reg-->>Builder: 提取《异议处理四步法》完整正文
Builder->>Builder: 存入 _active_scene_skills 字典
Tool-->>AgentLoop: ToolResult(ok=True, content="Skill 'objection_handling' activated")

Note over AgentLoop, LLM: 阶段 3：正文动态挂载至 Dynamic Tail，模型依指南答复
AgentLoop->>Builder: build_runtime_items() (下一轮推理)
Builder->>Builder: 将《异议处理四步法》完整 SOP 追加在 Dynamic Tail 末端
Builder-->>AgentLoop: 组装好的 Runtime Items
AgentLoop->>LLM: 驱动大模型遵循四步法生成答复
LLM-->>User: "完全理解您的预算考虑！这款精华单次用量仅需两滴，折算下来每天只需3块钱，且添加了高纯度玻色因..."
```

---

### 场景 2：场景常驻技能 (`materialization="scene"`) 多轮挂载流

展示常驻技能一旦激活，如何在后续多轮对话中持续生效：

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant AgentLoop as ReAct核心循环
participant Builder as ContextBuilder
participant LLM as 大语言模型(LLM)

Note over AgentLoop, Builder: 轮次 1：激活场景技能 (如: skin_diary_guidance 肤况打卡指南)
AgentLoop->>Builder: activate_skill("skin_diary_guidance")
Builder->>Builder: 登记入 _active_scene_skills 列表

Note over User, LLM: 轮次 2：用户继续追问后续细节
User->>AgentLoop: "那我今晚应该涂什么？"
AgentLoop->>Builder: build_runtime_items()
Builder->>Builder: 检测到 active_skills 包含 "skin_diary_guidance"<br/>自动提取其正文追加在 Dynamic Tail！
Builder-->>AgentLoop: 返回包含打卡指南的上下文
AgentLoop->>LLM: stream(...)，模型严格按打卡规范提醒用户拍照记录
LLM-->>User: 输出推荐步骤，并在末尾提醒：“请拍摄一张自然光下的脸颊自拍，为您存入护肤日记！”

Note over User, LLM: 轮次 3：用户回答后，技能指南依然常驻生效
User->>AgentLoop: "照片拍好了"
AgentLoop->>Builder: build_runtime_items()
Builder->>Builder: 依旧自动注入打卡指南，直到调用 deactivate_skill 注销！
```

---

### 场景 3：单次观测技能 (`materialization="observation"`) 查阅流

展示单次查询类技能（如成分速查）如何用完即焚，不污染后续提示词：

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant AgentLoop as ReAct核心循环
participant Tool as SkillTool(use_skill)
participant Reg as SkillRegistry
participant LLM as 大语言模型(LLM)

User->>AgentLoop: "烟酰胺的主要禁忌是什么？"
AgentLoop->>LLM: stream(...)
LLM-->>AgentLoop: 发起调用: use_skill(skill_name="niacinamide_guide")

Note over AgentLoop, Tool: 查阅单次观测技能
AgentLoop->>Tool: execute(skill_name="niacinamide_guide")
Tool->>Reg: require("niacinamide_guide")
Reg-->>Tool: 返回 materialization="observation" (单次观测！)
Tool->>Reg: render_skill_body("niacinamide_guide")
Reg-->>Tool: 提取烟酰胺禁忌说明正文

Note over Tool, AgentLoop: 正文作为工具结果返回，不写进稳定前缀
Tool-->>AgentLoop: ToolResult(ok=True, content="烟酰胺禁忌：不能与高浓度水杨酸同时使用...")
AgentLoop->>LLM: stream(messages 包含 ToolResult)
LLM-->>User: "烟酰胺的核心禁忌是不能与强酸同时使用，因为酸性环境会导致烟酰胺水解为烟酸，引发皮肤刺痛红肿。"

Note over AgentLoop: 下一轮对话中，该技能正文不再存在于 System Prompt 中，Token 立即释放！
```

---

### 场景 4：技能动态注销与上下文释放流 (`deactivate_skill`)

展示场景交互结束后，如何主动卸载技能释放 Token：

```mermaid
sequenceDiagram
autonumber
actor User as 用户
participant AgentLoop as ReAct核心循环
participant Tool as SkillTool(unload_skill)
participant Builder as ContextBuilder
participant LLM as 大语言模型(LLM)

User->>AgentLoop: "这个方案很清晰，我决定购买了，谢谢！"
AgentLoop->>LLM: stream(...)
LLM-->>AgentLoop: 发现异议已解决，主动发起: unload_skill(skill_name="objection_handling")

Note over AgentLoop, Builder: 动态注销技能
AgentLoop->>Tool: execute(unload_skill)
Tool->>Builder: deactivate_skill("objection_handling")
Builder->>Builder: 从 _active_scene_skills 字典中移除该技能！
Tool-->>AgentLoop: ToolResult(ok=True, content="Skill unloaded")

Note over AgentLoop, Builder: 检验注销效果
AgentLoop->>Builder: build_runtime_items()
Builder->>Builder: Dynamic Tail 中不再包含《异议处理四步法》正文！
Builder-->>AgentLoop: 已卸载该场景技能正文，释放尾部空间
LLM-->>User: "祝您护肤愉快，期待您的皮肤早日焕新！"
```

---

### 场景 5：本地磁盘递归扫描与容错标准化解析流

展示系统启动时，`SkillRegistry` 如何从磁盘递归发现技能并进行语法容错：

```mermaid
sequenceDiagram
autonumber
participant App as 应用启动入口
participant Reg as SkillRegistry
participant FS as 本地文件系统
participant Parser as parse_skill_markdown

App->>Reg: 实例化 SkillRegistry(roots=["/app/skills"])
Reg->>Reg: discover() 触发自动发现流程
Reg->>FS: 遍历 root.rglob("SKILL.md")
FS-->>Reg: 返回文件路径列表 [/app/skills/objection/SKILL.md, ...]

loop 逐个文件解析
    Reg->>Parser: parse_skill_markdown(path)
    Parser->>Parser: _split_frontmatter: 切分 YAML 头部与 Markdown 正文
    Parser->>Parser: _normalize_materialization: 将 "scene_overlay" 映射为 "scene"
    Parser->>Parser: 打包生成 SkillDescriptor 与 SkillDocument
    Parser-->>Reg: 返回完整的 SkillDocument 实体
    Reg->>Reg: register_document(): 存入 _documents 字典
end

Reg-->>App: 注册表初始化完成，共加载 14 个业务技能
```
