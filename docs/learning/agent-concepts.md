# Agent Concepts

本文档用于记录学习 LangChain、LangGraph、Deep Agents 和相关开源项目时遇到的重要概念。后续可以持续追加新的术语、理解笔记和示例。

## Agent Harness

### 一句话解释

Agent harness 是把大语言模型包装成“能持续完成任务的 Agent”的运行框架和工程脚手架。

Deep Agents 在根目录 `README.md` 中把自己称为：

```text
The batteries-included agent harness.
```

这里的 `batteries-included` 表示它不只是提供一个最小 agent loop，而是默认带上规划、文件系统、上下文管理、子代理、skills、memory、human-in-the-loop、部署与观测等能力。

### 为什么需要 Agent Harness

单独的大语言模型主要负责根据输入生成文本。它本身并不天然具备以下工程能力：

- 调用外部工具。
- 维护任务状态。
- 规划多步骤任务。
- 读写文件。
- 调用子代理。
- 处理长上下文。
- 保存长期记忆。
- 在关键工具调用前等待人工确认。
- 支持流式输出、追踪、部署和恢复运行。

Agent harness 的作用，就是把这些能力组织到一起，让模型从“回答问题”升级为“执行复杂任务”。

### 组成部分

可以粗略理解为：

```text
LLM = 会推理和生成内容的大脑
Tools = 外部能力，例如搜索、数据库、文件系统、命令行
Runtime = 执行图、状态管理、持久化、恢复运行的环境
Prompt / Memory = 任务规则、长期偏好、上下文
Agent Harness = 把 LLM、tools、runtime、prompt、memory、workflow 组织起来的框架
```

### 在 Deep Agents 中的体现

在 Deep Agents 中，`create_deep_agent()` 创建的不是一个简单聊天机器人，而是一个完整的 agent harness。

它会基于 LangChain 和 LangGraph 组装以下能力：

- 默认系统提示词。
- 任务规划工具 `write_todos`。
- 文件系统工具，例如 `ls`、`read_file`、`write_file`、`edit_file`、`grep`。
- 命令执行工具 `execute`。
- 子代理工具 `task`。
- skills。
- memory。
- summarization。
- human-in-the-loop。
- backend。
- checkpointer / store / cache 等 LangGraph 运行时能力。

示意：

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="openai:gpt-5.5",
    system_prompt="You are a research assistant.",
    tools=[],
)
```

这里得到的 `agent` 不是只会聊天的模型，而是一个具备任务执行能力的 Agent 系统。

根目录 README 中的 Quickstart 也说明了这一点：用户只需要调用 `create_deep_agent()`，就能得到一个可以规划、读写文件、管理上下文并调用工具的 Agent。

### 对比普通 LLM

普通 LLM：

```text
用户：帮我分析这个项目。
模型：只能根据用户贴出来的内容回答。
```

带 agent harness 的 Agent：

```text
用户：帮我分析这个项目。
Agent：
1. 制定分析计划。
2. 搜索项目文件。
3. 阅读 README、配置和源码。
4. 调用工具分析依赖和入口。
5. 必要时调用子代理分析不同模块。
6. 总结项目结构。
7. 将结果保存为 Markdown。
```

### 与 LangChain / LangGraph / Deep Agents 的关系

```text
LangChain = 提供模型、工具、agent 抽象等基础组件。
LangGraph = 提供状态图、持久化、streaming、interrupt、恢复运行等 runtime。
Deep Agents = 在 LangChain / LangGraph 之上提供更完整、更有默认能力的 agent harness。
```

换句话说：

- LangChain 提供组件。
- LangGraph 提供运行时。
- Deep Agents 提供面向长任务、多工具、多上下文的完整 harness。

更具体地说：

```text
LangGraph = runtime，负责状态、持久化、streaming、checkpoint、interrupt。
LangChain = model、tool、agent 等基础抽象。
Deep Agents = 在 LangChain / LangGraph 之上封装更完整的 agent harness。
```

### 判断一个项目是否有 Agent Harness

可以看它是否提供这些能力：

- 是否有统一的 Agent 构建入口？
- 是否管理模型和工具的调用循环？
- 是否有状态管理？
- 是否支持多轮任务执行？
- 是否能读写外部资源？
- 是否能插入 middleware 或 workflow？
- 是否支持观测、恢复、部署或评测？

能力越完整，越接近一个成熟的 agent harness。

### 简短总结

Agent harness 是让 LLM 从“文本生成器”变成“任务执行系统”的工程框架。

Deep Agents 的核心价值，就是提供一个开箱即用、可扩展、面向长任务的 agent harness。

### 安全提醒

Deep Agents 的 README 明确提到它遵循 “trust the LLM” 模型。也就是说，Agent 可以执行它的 tools 允许它执行的任何操作。

因此安全边界不应该依赖模型“自觉”，而应该放在：

- tool 权限。
- filesystem permission。
- sandbox。
- backend。
- human-in-the-loop。
- 部署环境隔离。

设计 Agent 时要先问：如果模型错误调用了某个工具，最坏会发生什么？然后在工具层和运行环境层限制损害范围。

## LangGraph Runtime

### 一句话解释

LangGraph 是 Agent 或工作流的运行时引擎，负责状态、持久化、streaming、checkpoint 和 interrupt。

也可以理解为：LangGraph 负责 Agent “怎么运行、怎么保存、怎么恢复、怎么观察过程”。

### Runtime 是什么

Runtime 指代码或系统真正执行时所依赖的运行环境和执行机制。

在 Agent 应用中，runtime 不只是“运行一段函数”，还要负责：

- 维护多轮任务状态。
- 在多个节点或步骤之间传递数据。
- 保存执行进度。
- 支持失败恢复。
- 支持流式输出。
- 支持人工中断和恢复。

因此，LangGraph 可以看作 Agent 应用的 runtime。

### State：状态

状态是一次 Agent 或工作流执行过程中持续传递和更新的数据。

例如，一个 Agent 运行时可能维护：

```text
messages = 用户、模型、工具之间的消息历史
files = Agent 创建或修改的文件状态
todos = 当前任务计划
current_step = 当前执行到哪一步
tool_results = 工具调用结果
```

普通函数调用结束后，局部变量通常就消失了。LangGraph 会把这些状态放在图执行过程中持续传递，让后续节点可以基于之前的结果继续工作。

### Persistence：持久化

持久化是把运行状态保存到内存之外的位置，例如数据库、文件、远程存储或 checkpoint store。

持久化后可以做到：

```text
用户今天聊到一半
明天继续同一个 thread
Agent 还能接着之前的上下文运行
```

这对长任务、后台任务、生产环境恢复非常重要。

### Streaming：流式输出

Streaming 是边执行边返回过程，而不是等整个 Agent 全部结束后再返回最终结果。

例如，一个 Agent 执行时可以实时展示：

```text
模型开始生成
调用 search 工具
工具返回结果
调用 read_file 工具
模型继续回答
输出最终结果
```

Streaming 对聊天 UI、调试和观测很重要，因为用户和开发者可以实时看到 Agent 正在做什么。

### Checkpoint：检查点

Checkpoint 是执行过程中的存档点。

每执行到某些步骤，LangGraph 可以把当前状态保存下来。后续如果中断、失败或需要恢复，就可以从 checkpoint 继续。

可以类比为：

```text
游戏存档
代码执行断点
数据库事务日志
```

在 Agent 场景中，checkpoint 尤其重要，因为 Agent 任务可能很长，可能会调用工具、等待人工确认，也可能中途失败。

### Interrupt：中断

Interrupt 是在执行过程中暂停，等待外部输入或人工决策。

例如 Agent 准备执行一些高风险操作：

```text
删除文件
执行 shell 命令
调用付款 API
发送邮件
创建 Pull Request
```

LangGraph 可以在这些步骤暂停，让用户或系统决定：

```text
approve = 批准执行
reject = 拒绝执行
edit = 修改参数后继续
```

确认后，Agent 可以从暂停点继续运行。

### 与 Deep Agents 的关系

Deep Agents 负责提供更完整的 Agent 能力，例如：

- filesystem。
- skills。
- memory。
- subagents。
- summarization。
- human-in-the-loop。

LangGraph 负责这些能力背后的运行机制，例如：

- 状态如何保存。
- 多轮执行如何继续。
- 工具调用过程如何流式输出。
- 人工审批点如何暂停和恢复。
- 长任务如何 checkpoint。

可以这样理解：

```text
Deep Agents = 给 Agent 配好工具、能力和默认行为。
LangGraph = 让这些能力以可靠、可恢复、可观测的方式运行。
```

### 简短总结

LangGraph 是构建 Agent 应用时的运行时层。它让 Agent 不只是一次性函数调用，而是可以拥有状态、可持续运行、可中断、可恢复、可观察的系统。
