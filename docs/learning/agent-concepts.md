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

## Deep Agents Architecture

### 一句话解释

Deep Agents 的架构可以理解为三层：

```text
Deep Agents = opinionated harness：默认能力、middleware、backends、profiles
LangChain = agent abstraction：model + tools + middleware -> agent loop
LangGraph = runtime：state、checkpoints、streaming、interrupts
```

Deep Agents 不重新发明 runtime，而是在 LangChain `create_agent()` 和 LangGraph runtime 之上，提供一套更完整、更适合长任务的 agent harness。

### 三层架构

#### LangGraph：运行时层

LangGraph 负责 Agent 或 workflow 的运行机制：

- 状态在步骤之间如何传递。
- checkpoint 如何保存。
- streaming 如何暴露执行过程。
- interrupt 如何暂停和恢复任务。

它关注的是“这个 Agent 系统如何可靠运行”。

#### LangChain：Agent 抽象层

LangChain 的 `create_agent()` 在 LangGraph 之上提供 agent 抽象。

调用者主要描述：

- 使用什么 model。
- 有哪些 tools。
- 使用哪些 middleware。

LangChain 会构建 agent loop：

```text
调用模型
→ 模型决定是否调用工具
→ 执行工具
→ 工具结果回到消息历史
→ 再次调用模型
→ 直到模型输出最终答案
```

它关注的是“模型、工具和 middleware 如何组成一个 Agent”。

#### Deep Agents：Opinionated Harness 层

Deep Agents 位于最上层，提供默认组装好的长任务 Agent 能力：

- planning。
- filesystem。
- subagents。
- skills。
- memory。
- summarization。
- backend。
- profiles。
- human-in-the-loop。

它关注的是“如何把常见长任务 Agent 所需能力打包好，让用户开箱即用”。

### Construction：构建阶段

当应用代码调用：

```python
from deepagents import create_deep_agent

agent = create_deep_agent(...)
```

就进入 construction 阶段。

`create_deep_agent()` 大致会做这些事：

1. 解析请求的 chat model，以及适用的 provider profile / harness profile。
2. 解析 backend，用于 filesystem、skills、memory 和 `execute`。
3. 组装主 Agent 的 middleware stack。
4. 构建默认 `general-purpose` subagent 和调用者传入的 subagents。
5. 组合 system prompt：用户指令、SDK 默认提示词、profile 提示词。
6. 调用 LangChain 的 `create_agent(...)`，生成最终可运行的 LangGraph graph。

因此，源码阅读的主入口是：

```text
libs/deepagents/deepagents/graph.py
```

### Execution：执行阶段

当返回的 graph 被 `invoke()`、`stream()` 或其他运行接口调用时，就进入 execution 阶段。

执行流程可以理解为：

```text
LangGraph 读取当前 state
→ 准备 messages、system prompt、tools
→ 模型生成回答或 tool call
→ 如果有 tool call，则执行工具
→ 工具结果写回 state
→ 再次调用模型
→ 直到模型输出最终答案
```

Deep Agents 主要通过 middleware 改变这个执行过程。

### Middleware

Middleware 是插入 agent loop 中的行为扩展点。

它和普通 tool 的区别很重要：

```text
tool = 模型选择调用后才执行
middleware = 可以影响模型调用前后、工具执行前后、state 准备过程
```

Middleware 可以做普通 tool 做不到的事情，例如：

- 在模型请求前增加或移除 tools。
- 向 system prompt 注入 filesystem、memory、skills、subagent 说明。
- 在上下文过长时 summarization / compaction。
- 把大型工具输出卸载到文件。
- 给 graph state 增加 typed fields。
- 在文件系统工具执行前做权限检查。

### Middleware Stack

Deep Agents 的 middleware stack 可以分为三段：

#### Base scaffolding

基础脚手架，提供 Deep Agent 的默认能力：

- planning。
- filesystem access。
- subagent delegation。
- summarization。
- request cleanup。

#### Caller middleware

调用者传入的自定义 middleware。

用于在不重写整个 harness 的情况下扩展行为。

#### Profile and tail middleware

根据最终模型、provider 和工具表面做调整：

- provider-specific behavior。
- tool exclusions。
- prompt caching。
- memory injection。
- human approval。

需要注意：subagents 也有自己的 middleware stack。如果某个行为只在 delegated work 中出现，应先确认是 main-agent stack、declarative subagent、compiled subagent，还是 async / remote subagent 触发的。

### Tool Surface

Tool surface 指模型在一次请求中“看得见、可以选择调用”的工具集合。

Deep Agents 中，tool surface 来自多个层：

- built-in middleware 注入标准工具：
  - todo management。
  - filesystem tools。
  - subagent delegation。
- 调用者通过 `tools=` 传入自定义工具。
- backend 决定 shell execution 是否可用。
- harness profiles 可以通过 `excluded_tools` 隐藏工具。
- filesystem permissions 决定文件工具调用是否允许、拒绝或中断。

调试工具问题时，可以按这个规则判断：

```text
工具缺失：检查 middleware assembly 和 profile exclusions。
工具可见但调用失败：检查 backend capability 和 permission enforcement。
```

例如：

- `execute` 工具不可用，可能是 backend 不支持 shell execution。
- `read_file` 可见但失败，可能是路径、权限或 backend 实现问题。

### Filesystem Access

Deep Agents 的文件系统能力不是直接等同于本机磁盘读写，而是由 backend 决定。

常见情况：

- `StateBackend`：文件存在 LangGraph state 中，通常是 thread-scoped。
- `FilesystemBackend`：文件映射到本地磁盘。
- `StoreBackend`：文件或 memory 可存入 LangGraph store。
- sandbox backend：文件和 shell execution 位于远程或隔离环境中。

因此，看到 Agent 有 `read_file` / `write_file` 工具时，要继续追问：

```text
这些文件实际存在哪里？
这个 backend 是否支持持久化？
这个 backend 是否支持 execute？
权限在哪里检查？
```

### State 与 Persistence

Deep Agents 的 state  lives in LangGraph。

它扩展了 LangChain 的 `AgentState`，定义了 `DeepAgentState`。其中 `messages` 使用 `DeltaChannel` reducer，使长线程中的 checkpoint 增长保持线性，而不是每次复制完整消息历史导致快速膨胀。

持久化要分成两类理解：

#### Graph state / checkpoints

来自 LangGraph。

保存内容包括：

- conversation state。
- message history。
- interrupts。
- resumability。

#### Filesystem / memory persistence

来自 Deep Agents backends。

保存内容包括：

- Agent 写入的文件。
- memory。
- backend 路由的数据。
- sandbox 或本地 filesystem 中的内容。

两者区别：

```text
LangGraph 持久化的是 graph 执行状态。
Deep Agents backend 持久化的是文件、memory、shell 执行环境相关内容。
```

### Profiles

Profiles 用来针对不同 provider 或 model 调整 harness 行为。

例如某些模型可能需要：

- 更短或不同格式的工具描述。
- 隐藏部分工具。
- 附加模型特定提示词。
- 启用 prompt caching。

因此，当你看到同样的 Agent 换模型后表现不同，除了模型本身能力差异，也要检查是否有 profile 改变了 prompt、middleware 或 tool surface。

### 源码阅读入口

根据架构文档，阅读 Deep Agents 源码可以从这些位置开始：

```text
graph.py        Agent 构建、middleware 顺序、prompt 组装、默认模型行为
middleware/     工具可见性、prompt 注入、请求时行为
backends/       文件持久化、shell 支持、路径路由
profiles/       provider 或 model 特定的 harness 调整
__init__.py     公共 API 和兼容性边界
```

阅读方法：

1. 从 `create_deep_agent()` 的某个公开参数开始。
2. 找到它安装了哪个 middleware、backend 或 profile。
3. 继续跟踪该组件在 execution 阶段如何参与运行。

### 简短总结

Deep Agents 的架构核心是：

```text
LangGraph 负责可靠运行。
LangChain 负责 agent loop 抽象。
Deep Agents 负责把长任务 Agent 所需能力默认组装好。
```

理解 Deep Agents 时，最重要的不是只看 tools，而是看 `middleware + backend + profile` 如何共同塑造 Agent 的行为。
