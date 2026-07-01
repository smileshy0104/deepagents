# LangChain 与 Deep Agents 项目化学习路线

本文档面向已经初步学习过 LangChain 和 Deep Agents 的开发者，目标是通过开源项目阅读与实战项目，把 LangChain、LangGraph、Deep Agents、LangSmith 串成一条完整的学习路径。

## 总体目标

学习完成后，应能独立完成一个具备以下能力的 Agent 项目：

- 使用 LangChain / LangGraph 构建有状态 Agent。
- 使用 Deep Agents 提供规划、文件系统、skills、memory、subagents 等能力。
- 接入 RAG、SQL、代码阅读、自定义 tools 等业务能力。
- 使用 LangGraph Server 或 Deep Agents CLI 进行本地运行与部署验证。
- 使用 LangSmith 做 tracing、评测和回归分析。
- 能阅读 Deep Agents 源码，理解核心 middleware、backend 和 deploy CLI。

## README 阅读笔记：Deep Agents 的项目定位

根目录 `README.md` 给 Deep Agents 的定位是：

```text
The batteries-included agent harness.
```

可以理解为：Deep Agents 是一个“开箱即用、默认拥有（langchain和langgraph）能力较完整的 Agent 框架”。它不是替代 LangChain 或 LangGraph，而是在它们之上提供一层更完整的 agent harness。

三者关系：

```text
LangGraph = runtime，负责状态、持久化、streaming、checkpoint、interrupt。
LangChain = model、tool、agent 等基础抽象。
Deep Agents = 在 LangChain / LangGraph 之上封装更完整的 agent harness。
```

其中 `runtime` 可以理解为 Agent 应用真正执行时的运行时层：它负责状态如何传递、执行进度如何保存、过程如何流式输出、任务如何中断和恢复。详细概念见 `docs/learning/agent-concepts.md` 中的 `LangGraph Runtime`。

README 中强调的核心原则：

- Opinionated：默认配置面向“长任务、多步骤”复杂任务。
- Extensible：可以替换或覆盖任意部分，不需要 fork 项目。
- Model-agnostic：只要模型支持 tool calling，就可以接入 frontier、open-weight 或本地模型。
- Production-ready：生产能力来自 LangGraph 的 streaming、persistence、checkpointing，以及 LangSmith 的 tracing、evaluation、deployment。

README 中列出的核心能力：

- Sub-agents：把任务委派给上下文隔离的子代理。
- Filesystem：通过可插拔 backend 读、写、编辑、搜索文件。
- Context management：总结长线程，并把大型工具输出卸载到文件。
- Shell access：在 sandbox 中执行命令。
- Persistent memory：跨会话记忆。
- Human-in-the-loop：工具调用前允许人工批准、编辑或拒绝。
- Skills：按需加载可复用行为。
- Tools：支持自定义函数和 MCP server。

README 的 Quickstart 说明主入口是 `create_deep_agent()`：

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="openai:gpt-5.5",
    tools=[my_custom_tool],
    system_prompt="You are a research assistant.",
)
result = agent.invoke({"messages": "Research LangGraph and write a summary"})
```

因此阅读源码时，应该围绕 `create_deep_agent()` 去理解它如何组装：

- model。
- system prompt。
- tools。
- middleware。
- backend。
- subagents。
- skills。
- memory。
- LangGraph runtime 参数。

README 的安全提醒也很关键：Deep Agents 遵循 “trust the LLM” 模型。Agent 可以做它的 tools 允许它做的任何事，所以安全边界应该放在 tool、permission、sandbox 和 backend 层，而不是期待模型自我约束。

源码阅读入口：

- `libs/deepagents/deepagents/graph.py`：`create_deep_agent()` 主入口。
- `libs/deepagents/deepagents/middleware/filesystem.py`：文件系统工具。
- `libs/deepagents/deepagents/middleware/subagents.py`：子代理。
- `libs/deepagents/deepagents/middleware/skills.py`：skills。
- `libs/deepagents/deepagents/middleware/memory.py`：memory。
- `libs/deepagents/deepagents/backends/`：backend 抽象和实现。

## 推荐开源项目

建议按照学习价值和难度递进阅读：

1. [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)
   - 当前主仓库。
   - 重点学习 `create_deep_agent`、middleware、backend、skills、subagents、deploy CLI。

2. [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)
   - Deep Agents 底层运行时。
   - 重点学习状态图、持久化、streaming、interrupt、deployment。

3. [langchain-ai/react-agent](https://github.com/langchain-ai/react-agent)
   - 最小 ReAct agent 模板。
   - 适合理解 agent loop、tool calling、LangGraph Studio。

4. [langchain-ai/retrieval-agent-template](https://github.com/langchain-ai/retrieval-agent-template)
   - RAG agent 模板。
   - 适合从普通 RAG 过渡到 agentic RAG。

5. [langchain-ai/rag-research-agent-template](https://github.com/langchain-ai/rag-research-agent-template)
   - 面向研究场景的 RAG agent 模板。
   - 适合学习检索、问题拆解、证据引用和回答合成。

6. [langchain-ai/chat-langchain](https://github.com/langchain-ai/chat-langchain)
   - 生产级文档问答项目。
   - 适合学习真实文档问答系统、guardrails、前后端集成和 LangGraph 应用结构。

7. [langchain-ai/open_deep_research](https://github.com/langchain-ai/open_deep_research)
   - 开源 deep research agent。
   - 适合学习多步骤 research、搜索、规划、报告生成。

8. [langchain-ai/agent-chat-ui](https://github.com/langchain-ai/agent-chat-ui)
   - Agent 聊天 UI。
   - 适合给 LangGraph / Deep Agents 应用接可视化界面。

9. [langchain-ai/create-agent-chat-app](https://github.com/langchain-ai/create-agent-chat-app)
   - 快速生成 agent chat app。
   - 适合学习前后端联调、streaming、tool calls 展示。

10. [langchain-ai/langgraph-supervisor-py](https://github.com/langchain-ai/langgraph-supervisor-py)
    - 多 Agent supervisor 架构。
    - 适合理解主 Agent 调度多个专家 Agent 的模式。

11. [langchain-ai/open-agent-platform](https://github.com/langchain-ai/open-agent-platform)
    - 面向 LangGraph agents 的开放平台。
    - 适合后期学习 Agent 产品化和平台化管理。

12. [langchain-ai/open-swe](https://github.com/langchain-ai/open-swe)
    - 异步 coding agent。
    - 基于 LangGraph 和 Deep Agents，包含云 sandbox、GitHub、Slack、Linear、自动 PR 等能力。
    - 复杂度较高，建议最后阅读。

## 学习顺序

### 第 0 阶段：建立技术地图

建议时间：1 天

目标：

- 明确 LangChain、LangGraph、Deep Agents、LangSmith 的边界。
- 能解释 Deep Agents 为什么构建在 LangGraph 之上。

阅读内容：

- `README.md`
- `libs/ARCHITECTURE.md`
- `libs/deepagents/deepagents/graph.py`
- LangGraph README 和官方入门文档

需要掌握：

- LangChain：模型、tools、agent 抽象。
- LangGraph：状态图、持久化、streaming、interrupt、部署。
- Deep Agents：planning、filesystem、subagents、skills、memory、summarization。
- LangSmith：tracing、evaluation、monitoring。

阶段产出：

- 一页架构笔记，说明这四者之间的关系。

### 第 1 阶段：最小 ReAct Agent

建议时间：2-3 天

参考项目：

- `langchain-ai/react-agent`

项目任务：

- 构建一个最小 Agent。
- 添加 2-3 个工具，例如天气、计算器、简单搜索。
- 使用 LangGraph Studio 或 `langgraph dev` 本地运行。

学习重点：

- tool calling。
- agent loop。
- message state。
- LangGraph 本地调试。

阶段产出：

- 一个能调用自定义工具的最小 LangGraph Agent。
- 一份工具调用链路笔记。

### 第 2 阶段：Deep Agents 最小实践

建议时间：3-5 天

参考项目：

- 当前仓库 `libs/deepagents`

项目任务：

- 写一个文件整理 Agent。
- 让 Agent 读取一个目录。
- 总结目录中的文件。
- 生成 `index.md`。
- 支持写入和编辑文件。

重点阅读：

- `libs/deepagents/deepagents/graph.py`
- `libs/deepagents/deepagents/middleware/filesystem.py`
- `libs/deepagents/deepagents/backends/`

学习重点：

- `create_deep_agent()` 的参数。
- `StateBackend` 与 `FilesystemBackend` 的区别。
- Deep Agents 内置 filesystem tools 的工作方式。
- `execute` 工具为什么依赖 sandbox backend。

阶段产出：

- 一个基于 `create_deep_agent()` 的本地 demo。
- 一份 backend 对比笔记。

### 第 3 阶段：RAG 与文档问答 Agent

建议时间：1 周

参考项目：

- `retrieval-agent-template`
- `rag-research-agent-template`
- `chat-langchain`

项目任务：

- 做一个项目文档问答 Agent。
- 读取 Markdown、Python 文件或 README。
- 建立检索索引。
- 用户提问时先检索，再回答。
- 回答中附带来源文件。

学习重点：

- retriever tool。
- agentic RAG。
- query rewrite。
- relevance grading。
- 引用来源与防止幻觉。

阶段产出：

- 一个可以问本地 `deepagents` 源码的问答 Agent。
- 10 个固定测试问题和期望答案。

### 第 4 阶段：Skills 与 Memory

建议时间：3-5 天

参考项目：

- `examples/content-builder-agent`
- `examples/deploy-content-writer`

项目任务：

- 将上一阶段的文档问答 Agent 升级为源码学习助教。
- 添加 skills：
  - `code-reading`
  - `api-analysis`
  - `test-analysis`
  - `architecture-summary`
- 添加 `AGENTS.md` 作为学习偏好 memory。
- 每次学习结果保存到 `notes/`。

学习重点：

- skill 的触发机制。
- `SKILL.md` frontmatter。
- memory 如何进入系统提示词。
- Agent 如何长期积累上下文。

阶段产出：

- 一个能按不同学习任务加载不同 skill 的 Agent。
- 一组可复用 `SKILL.md`。

### 第 5 阶段：Subagents 与多 Agent 协作

建议时间：1 周

参考项目：

- Deep Agents `SubAgentMiddleware`
- `langgraph-supervisor-py`
- `open_deep_research`

项目任务：

- 将源码学习助教升级为多 Agent 协作系统。
- 设计至少 3 个子代理：
  - `reader`：阅读代码和文档。
  - `summarizer`：总结模块职责。
  - `critic`：检查回答是否遗漏关键点。

学习重点：

- supervisor / worker 模式。
- 子代理上下文隔离。
- 主 Agent 如何拆解任务。
- 子 Agent 结果如何汇总。

阶段产出：

- 输入“解释 Deep Agents 的 backend 系统”，主 Agent 自动分派任务并生成结构化学习笔记。

### 第 6 阶段：SQL 与数据分析 Agent

建议时间：3-5 天

参考项目：

- `examples/text-to-sql-agent`
- LangGraph SQL agent 文档

项目任务：

- 构建一个 Text-to-SQL Agent。
- 使用 SQLite 或业务测试数据库。
- 支持自然语言问题。
- 自动探索 schema。
- 生成 SQL、执行 SQL、解释结果。

学习重点：

- 数据库 schema 探索。
- SQL 生成与校验。
- 工具安全边界。
- 错误恢复。

阶段产出：

- 一个能查询 SQLite 数据库的数据分析 Agent。
- 一份 SQL safety checklist。

### 第 7 阶段：UI 与部署

建议时间：1 周

参考项目：

- `agent-chat-ui`
- `create-agent-chat-app`
- `chat-langchain`
- 当前仓库 `libs/cli`

项目任务：

- 给源码学习助教接入 UI。
- 后端使用 `langgraph dev`。
- 前端使用 Agent Chat UI。
- 展示 streaming、tool calls、历史消息。
- 使用 `deepagents deploy --dry-run` 理解部署 payload。

学习重点：

- LangGraph Server。
- 前后端 streaming。
- tool calls 可视化。
- `agent.json`、`AGENTS.md`、`tools.json`。
- Deep Agents CLI 的部署模型。

阶段产出：

- 一个可通过网页使用的 Agent。
- 一份部署流程笔记。

### 第 8 阶段：评测与观测

建议时间：3-5 天

参考项目：

- `libs/evals`
- LangSmith evaluation 文档

项目任务：

- 给源码学习助教加入评测。
- 设计 10-20 个固定问题。
- 定义评分规则。
- 记录是否引用正确文件。
- 记录是否出现幻觉。
- 比较 prompt、skills、subagents 调整前后的效果。

学习重点：

- LangSmith tracing。
- datasets。
- regression eval。
- 评分器设计。
- Agent 改动后的质量对比。

阶段产出：

- 一套可重复运行的 Agent eval。
- 一份调优记录。

### 第 9 阶段：最终大项目

建议时间：2-3 周

推荐项目：Codebase Wiki Deep Agent

功能目标：

- 输入一个 Git 仓库路径。
- 自动扫描代码结构。
- 生成 `wiki/index.md`。
- 为每个模块生成说明文档。
- 支持用户问源码问题。
- 支持持续更新 wiki。
- 支持 skills：
  - 读 API。
  - 读测试。
  - 读架构。
  - 生成学习题。
- 支持 subagents：
  - reader。
  - researcher。
  - critic。
  - writer。
- 支持 UI。
- 支持 eval。
- 可选择部署到 LangGraph 或 Managed Deep Agents。

核心学习收益：

- 串联 LangChain tools、LangGraph state、Deep Agents filesystem / skills / subagents。
- 形成一个真实可展示的作品。
- 具备继续阅读 `open-swe` 这类复杂 Agent 系统的基础。

## 推荐执行顺序一句话版

```text
react-agent
→ deepagents 最小 demo
→ RAG template
→ chat-langchain
→ skills / memory
→ subagents
→ SQL agent
→ Agent Chat UI
→ evals
→ open_deep_research
→ open-swe
```

## 基于开源项目的主线学习路线

如果希望完全围绕开源项目推进，建议不要只看概念文档，而是每个阶段都选一个主项目，完成“阅读源码 → 跑通示例 → 改一个小功能 → 输出笔记”的闭环。

### 路线 A：从最小 Agent 到 Deep Agents

主线项目：

1. `langchain-ai/react-agent`
2. `langchain-ai/langgraph`
3. `langchain-ai/deepagents`

学习顺序：

1. 先跑通 `react-agent`，理解 ReAct agent、tool calling、message state。
2. 再阅读 `langgraph` 的基础示例，理解状态图、节点、边、checkpoint、streaming。
3. 最后回到 `deepagents`，重点看它如何在 LangGraph 之上封装 planning、filesystem、skills、subagents。

建议实践：

- 第一次改造：给 `react-agent` 增加一个自定义工具。
- 第二次改造：用 LangGraph 写一个两节点工作流。
- 第三次改造：用 `create_deep_agent()` 写一个能读写本地文件的 Agent。

阶段目标：

- 能说清楚 LangChain agent、LangGraph graph、Deep Agents harness 的区别。
- 能独立写一个最小 Deep Agent。

### 路线 B：从 RAG 到生产级文档助手

主线项目：

1. `langchain-ai/retrieval-agent-template`
2. `langchain-ai/rag-research-agent-template`
3. `langchain-ai/chat-langchain`
4. 当前仓库 `examples/deploy-mcp-docs-agent`

学习顺序：

1. 先用 `retrieval-agent-template` 理解基础检索问答。
2. 再用 `rag-research-agent-template` 学习 agentic RAG：检索、判断、改写、再检索。
3. 然后阅读 `chat-langchain`，重点看生产级文档助手如何组织后端、前端、guardrails 和 LangGraph。
4. 最后回到 `examples/deploy-mcp-docs-agent`，把文档助手改造成 Deep Agents 可部署项目。

建议实践：

- 用 `deepagents` 仓库本身作为文档语料。
- 建立一个“问 Deep Agents 源码”的文档问答 Agent。
- 回答必须附带文件路径或来源。
- 加入固定测试问题，检查回答是否引用正确文件。

阶段目标：

- 能完成一个真实可用的项目文档问答系统。
- 能理解普通 RAG 与 Agentic RAG 的差异。

### 路线 C：从 Skills / Memory 到源码学习助教

主线项目：

1. 当前仓库 `examples/content-builder-agent`
2. 当前仓库 `examples/deploy-content-writer`
3. 当前仓库 `libs/deepagents/deepagents/middleware/skills.py`
4. 当前仓库 `libs/deepagents/deepagents/middleware/memory.py`

学习顺序：

1. 先跑通 `content-builder-agent`，理解 `AGENTS.md`、`skills/`、subagents 的文件组织方式。
2. 再阅读 `deploy-content-writer`，理解文件式 Agent 项目如何被部署。
3. 然后读 skills 和 memory middleware 源码，理解它们如何把外部文件变成 Agent 可用上下文。
4. 最后做一个“源码学习助教”，让 Agent 根据任务加载不同 skill。

建议实践：

- 新建 `skills/code-reading/SKILL.md`。
- 新建 `skills/test-analysis/SKILL.md`。
- 新建 `skills/architecture-summary/SKILL.md`。
- 让 Agent 输出学习笔记到 `notes/`。

阶段目标：

- 能设计可复用 skill。
- 能让 Agent 具备长期学习偏好和项目记忆。

### 路线 D：从多 Agent 到 Deep Research

主线项目：

1. `langchain-ai/langgraph-supervisor-py`
2. `langchain-ai/open_deep_research`
3. 当前仓库 `examples/deep_research`
4. 当前仓库 `libs/deepagents/deepagents/middleware/subagents.py`

学习顺序：

1. 先读 `langgraph-supervisor-py`，理解 supervisor / worker 的多 Agent 模式。
2. 再读 `open_deep_research`，学习 research agent 如何规划、搜索、反思、写报告。
3. 然后跑通当前仓库的 `examples/deep_research`。
4. 最后回到 Deep Agents 的 subagents middleware，理解 Deep Agents 如何暴露 `task` 工具来委派任务。

建议实践：

- 给源码学习助教增加 3 个 subagents：
  - `reader`：负责读代码。
  - `researcher`：负责查文档和资料。
  - `critic`：负责检查遗漏和不确定点。
- 主 Agent 负责拆解问题和整合最终答案。

阶段目标：

- 能构建多 Agent 协作系统。
- 能理解子代理上下文隔离、任务委派、结果合成。

### 路线 E：从 UI 到可部署 Agent 产品

主线项目：

1. `langchain-ai/agent-chat-ui`
2. `langchain-ai/create-agent-chat-app`
3. `langchain-ai/open-agent-platform`
4. 当前仓库 `libs/cli`

学习顺序：

1. 先用 `agent-chat-ui` 给本地 LangGraph Agent 接 UI。
2. 再用 `create-agent-chat-app` 学习快速搭建 Agent 应用。
3. 然后阅读 `open-agent-platform`，理解多个 Agent 的平台化管理方式。
4. 最后阅读当前仓库 `libs/cli/deepagents_cli/deploy`，理解 Deep Agents 项目如何被打包为平台 payload。

建议实践：

- 给“源码学习助教”接一个聊天 UI。
- 展示 streaming 输出。
- 展示 tool calls。
- 展示生成的 wiki / notes。
- 使用 `deepagents deploy --dry-run` 检查部署 payload。

阶段目标：

- 能把 Agent 从命令行 demo 做成可交互产品。
- 能理解部署配置、远端运行、MCP 工具注册等概念。

### 路线 F：最终挑战：异步 Coding Agent

主线项目：

1. `langchain-ai/open-swe`
2. 当前仓库 `libs/code`
3. 当前仓库 `examples/deploy-coding-agent`
4. 当前仓库 `libs/partners/*`

学习顺序：

1. 先使用 `deepagents-code` 或阅读 `libs/code`，理解本地 coding agent 的交互方式。
2. 再跑通 `examples/deploy-coding-agent`，理解可部署 coding agent 的最小结构。
3. 然后阅读 `open-swe`，重点看云 sandbox、GitHub 集成、异步任务、subagent orchestration、自动 PR。
4. 最后阅读 partner sandbox 包，理解 Agent 执行环境如何抽象。

建议实践：

- 做一个“小型自动修 bug Agent”。
- 输入 issue 描述。
- Agent 读取代码。
- 修改文件。
- 运行测试。
- 输出 patch 和解释。

阶段目标：

- 能理解生产级 coding agent 的核心架构。
- 能看懂 `open-swe` 这种复杂项目。

## 基于开源项目的推荐总顺序

```text
1. react-agent
   学最小 Agent 和 tool calling

2. langgraph
   学状态图、持久化、streaming、human-in-the-loop

3. deepagents
   学 planning、filesystem、skills、memory、subagents

4. retrieval-agent-template / rag-research-agent-template
   学 RAG 和 agentic RAG

5. chat-langchain
   学生产级文档助手

6. examples/content-builder-agent
   学 skills、memory、文件式 Agent 配置

7. examples/deep_research / open_deep_research
   学多步骤 research Agent

8. langgraph-supervisor-py
   学多 Agent supervisor 架构

9. agent-chat-ui / create-agent-chat-app
   学 Agent UI 和流式交互

10. libs/cli/deepagents_cli/deploy
    学 Deep Agents 部署 payload 和项目结构

11. libs/evals
    学 Agent 评测和回归测试

12. open-swe
    学异步 coding agent、sandbox、GitHub 集成和自动 PR
```

## 每个开源项目的阅读方法

阅读任何一个开源 Agent 项目时，都按同一套问题拆解：

1. 入口在哪里？
   - CLI 入口。
   - API 入口。
   - LangGraph graph 定义。

2. Agent 状态是什么？
   - message state。
   - 自定义 state。
   - checkpoint / store。

3. 工具有哪些？
   - 普通 Python tools。
   - MCP tools。
   - filesystem tools。
   - sandbox execution。

4. 上下文从哪里来？
   - system prompt。
   - memory。
   - skills。
   - RAG 检索。
   - 用户输入。

5. 是否有多 Agent？
   - supervisor。
   - subagents。
   - async subagents。
   - worker graph。

6. 如何观测和评测？
   - LangSmith tracing。
   - eval datasets。
   - regression tests。
   - benchmark。

7. 如何部署？
   - `langgraph dev`。
   - LangGraph Platform。
   - Deep Agents CLI。
   - Docker / self-hosted service。

按这 7 个问题读项目，比从头到尾扫源码更有效。

## 当前仓库中的重点阅读文件

建议按以下顺序阅读：

1. `README.md`
2. `libs/ARCHITECTURE.md`
3. `libs/deepagents/deepagents/graph.py`
4. `libs/deepagents/deepagents/middleware/`
5. `libs/deepagents/deepagents/backends/`
6. `examples/content-builder-agent/`
7. `examples/text-to-sql-agent/`
8. `examples/deep_research/`
9. `libs/cli/deepagents_cli/deploy/`
10. `libs/evals/`

## 建议从哪里开始

最推荐从当前仓库直接开始做：

**Deep Agents 源码学习助教**

第一版只需要完成：

- 能读取当前仓库文件。
- 能回答“某个模块是干什么的”。
- 能输出一份 Markdown 学习笔记。
- 能引用具体文件路径。

之后再逐步加入：

- RAG 检索。
- skills。
- memory。
- subagents。
- UI。
- eval。
- 部署。

这样学习路线最稳，也最贴近 Deep Agents 的真实使用场景。
