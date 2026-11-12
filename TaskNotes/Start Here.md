# 从这里开始使用 TaskNotes

这篇笔记会带你在自己的库中快速了解 TaskNotes。你可以直接在这里按步骤操作，之后留作参考，或在不再需要时删除。

将下面三件事连接起来，就能更顺畅地使用 TaskNotes：

1. 快速记录任务。
2. 将任务保存为普通的 Markdown 笔记。
3. 通过 Bases 视图查看任务。

## 开始之前

- [ ] 启用 Obsidian **Bases** 核心插件。
- [ ] 确认 TaskNotes 已在 `TaskNotes/Views/` 中创建默认的 `.base` 文件。
- [ ] 打开命令面板并运行 **TaskNotes：打开任务视图**。

启用**自动创建默认文件**后，TaskNotes 会在启动时自动创建缺失的默认 `.base` 文件。如果文件不存在，请前往**设置 -> TaskNotes -> 常规 -> 视图和 Base 文件 -> 创建文件**。

## 1. 从命令面板创建任务

运行 **TaskNotes：创建新任务**，创建一项你近期打算完成的真实任务。

可以试试这样的标题：

```text
Review my TaskNotes setup tomorrow #admin
```

保存后，TaskNotes 会为该任务创建一篇 Markdown 笔记。打开任务笔记，看看顶部的 YAML 属性。Bases 正是通过这些属性进行筛选、排序、分组，并确定日历中的位置。

## 2. 将行内任务转换为 TaskNotes

下面的复选框是普通的 Markdown 任务。使用所在行的行内操作或命令面板，将它们逐一转换为 TaskNotes 任务笔记。

- [ ] 创建一个明天截止的 TaskNotes 收件箱任务 #admin
- [ ] 计划下周一花 20 分钟做规划 #planning
- [ ] 创建一个关联到进行中项目笔记的任务

转换后，请留意：

- 原有的任务行可以留在这篇笔记中作为上下文。
- 实际任务现在作为独立的 Markdown 笔记存在，因此会出现在 TaskNotes 视图中。
- 转换后的任务行包含指向任务笔记的链接。右键点击该链接打开任务菜单，试着更改状态、优先级、截止日期、计划日期或项目等属性。

## 3. 查看你的任务

下面嵌入的视图使用 TaskNotes 的默认 Bases 文件。

### 任务列表

![[TaskNotes/Views/tasks-default.base]]

想直接查看所有未完成任务时，使用这个视图。

### 日程

![[TaskNotes/Views/agenda-default.base]]

想关注截止日期、计划日期以及近期需要处理的事项时，使用这个视图。

### 迷你日历

![[TaskNotes/Views/mini-calendar-default.base]]

想在笔记旁边放一个紧凑的日历时，使用这个视图。

### 日历

![[TaskNotes/Views/calendar-default.base]]

想用更大的日历查看计划工作、截止日期、时间块和日历事件时，使用这个视图。

### 看板

![[TaskNotes/Views/kanban-default.base]]

想让任务在未完成、进行中和已完成等状态间推进时，使用这个视图。

## 4. 只调整妨碍你使用的设置

默认设置足以让你开始使用。等知道自己想改变什么，再去设置中调整：

- 如果任务文件保存到了错误的位置，请检查**设置 -> TaskNotes -> 常规**。
- 如果转换后的行内任务应该创建在其他位置，请检查**设置 -> TaskNotes -> 行内任务设置**。
- 如果任务对话框显示的字段过多或过少，请检查**设置 -> TaskNotes -> 对话框字段**。
- 如果你的库使用不同的属性名、状态、优先级或自定义字段，请检查**设置 -> TaskNotes -> 任务属性**。

## 5. 按自己的方式使用

试着给任务添加一个自定义属性，例如 `energy`、`client` 或 `area`。任务是 Markdown 笔记，因此你可以直接在 Bases 中使用自定义属性，不必依赖单独的任务数据库。

## 6. 根据笔记内容显示视图

你可以在每日笔记、项目笔记、会议笔记或任务笔记中嵌入 Bases 视图。高级 Bases 筛选器可以把当前笔记用作上下文，让同一个视图显示与嵌入位置相关的任务。

TaskNotes 的任务笔记“关联关系”小组件采用的就是这种方式。准备好自定义视图时，可以参阅 TaskNotes 文档中的示例。

接下来可以尝试：

- 复制 `TaskNotes/Views/tasks-default.base`，创建一个针对特定项目的视图。
- 在 Base 视图中按状态或优先级分组。
- 添加筛选器，显示本周截止的任务。
- 将任务关联到项目笔记，并使用反向链接查看它在哪里被提及。

## 更多帮助与扩展

- 文档：[tasknotes.dev](https://tasknotes.dev/)
- 遇到问题或错误：[在 GitHub 上提交问题](https://github.com/callumalpass/tasknotes/issues)
- 配套插件：
  - [Canvas Bases](https://tasknotes.dev/companion-plugins/canvas-bases/) 为 Bases 视图添加画布式看板和 JSON Canvas 快照，并可选显示 TaskNotes 徽标、操作和关联边。
  - [TaskNotes Workflows](https://tasknotes.dev/companion-plugins/tasknotes-workflows/) 为 TaskNotes 事件、定时计划、手动命令和任务操作添加基于 Markdown 定义的自动化功能。

## 完成

现在你已了解核心思路：任务是笔记，属性使它们可供查询，Bases 则将它们变成视图。
