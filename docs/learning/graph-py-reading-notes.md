# `libs/deepagents/deepagents/graph.py` 阅读笔记

本文档用通俗方式梳理 `libs/deepagents/deepagents/graph.py` 的内容。这个文件是 Deep Agents SDK 的核心装配入口，最重要的公开函数是 `create_deep_agent()`。

## 一句话总结

`graph.py` 是 Deep Agents 的“总装配线”。

它负责把这些部件组装成一个可运行的 LangGraph Agent：

- model。
- system prompt。
- tools。
- middleware。
- backend。
- skills。
- memory。
- subagents。
- permissions。
- interrupt。
- state schema。
- checkpointer / store / cache。

最终它调用 LangChain 的 `create_agent(...)`，得到一个真正可运行的 LangGraph graph。

## 文件整体结构

可以把 `graph.py` 分成几块：

1. 导入依赖。
2. 定义 `DeepAgentState`。
3. 定义默认系统提示词 `BASE_AGENT_PROMPT`。
4. 定义若干辅助函数。
5. 定义受保护 middleware。
6. 定义主入口 `create_deep_agent()`。
7. 在 `create_deep_agent()` 末尾调用 LangChain `create_agent(...)`。

## `DeepAgentState`

```python
class DeepAgentState(AgentState):
    messages: Required[
        Annotated[
            list[AnyMessage],
            DeltaChannel(_messages_delta_reducer, snapshot_frequency=50),
        ]
    ]
```

`DeepAgentState` 是 Deep Agents 对 LangChain `AgentState` 的扩展。

它最重要的地方是：`messages` 使用了 `DeltaChannel`。

通俗理解：

```text
普通方式：
每次 checkpoint 都可能保存完整 messages，长对话越跑越大。

DeltaChannel：
只保存消息变化量，并定期做快照，让 checkpoint 增长更可控。
```

这样可以降低长线程中 checkpoint 的增长压力。

## `BASE_AGENT_PROMPT`

`BASE_AGENT_PROMPT` 是所有 Deep Agent 默认携带的基础系统提示词。

它告诉模型：

- 你是一个可以使用工具完成任务的 deep agent。
- 回答要直接、简洁。
- 不要说太多空话。
- 遇到任务时先理解、再行动、再验证。
- 出问题时要分析原因。
- 对不明确请求只问必要问题。
- 长任务要给进度更新。

也就是说，Deep Agents 不只是提供工具，还通过默认 prompt 规定了 Agent 的工作风格。

## System Prompt 的组合方式

最终传给模型的 system prompt 不是只有一段，而是由多层组合：

```text
USER   = 用户传入的 system_prompt
BASE   = SDK 默认的 BASE_AGENT_PROMPT
CUSTOM = HarnessProfile.base_system_prompt
SUFFIX = HarnessProfile.system_prompt_suffix
```

顺序是：

```text
USER -> BASE 或 CUSTOM -> SUFFIX
```

重要原则：

- 用户传入的 `system_prompt` 总是在最前面，优先级最高。
- profile 的 suffix 放在最后，更贴近模型实际看到的对话。
- 如果用户传入的是 `SystemMessage`，会保留其中的 content blocks 和 cache control。

## 默认模型相关逻辑

文件里有两个相关函数：

```python
def _build_default_model() -> ChatAnthropic
def get_default_model() -> ChatAnthropic
```

默认模型是：

```text
claude-sonnet-4-6
```

但依赖默认模型已经被标记为 deprecated。也就是说，未来更推荐用户显式传入模型：

```python
create_deep_agent(model="openai:gpt-5.5")
```

而不是：

```python
create_deep_agent()
```

## Prompt Caching Middleware

相关函数：

```python
_create_bedrock_prompt_caching_middleware()
_append_prompt_caching_middleware()
```

作用：

- 默认加入 Anthropic prompt caching middleware。
- 如果安装了 `langchain-aws`，再尝试加入 Bedrock prompt caching middleware。

通俗理解：

```text
长 prompt 很贵，也很慢。
Prompt caching middleware 尝试让支持缓存的 provider 复用 prompt 前缀。
```

## 文件权限和 interrupt 合并

相关函数：

```python
_merge_fs_interrupt_on(...)
```

Deep Agents 有两类可能触发 human-in-the-loop 的来源：

1. 用户显式传入的 `interrupt_on`。
2. 文件系统权限规则中 `mode="interrupt"` 生成的 interrupt。

这个函数负责把它们合并。

规则：

```text
如果同一个工具两边都有配置，用户显式传入的 interrupt_on 优先。
```

## 自定义 Middleware 合并

相关函数：

```python
_apply_custom_middleware(...)
```

它负责把用户传入的 middleware 合并进默认 middleware stack。

规则：

1. 如果自定义 middleware 的 `.name` 和已有 middleware 相同：
   - 替换默认 middleware。
   - 保持原来的位置。

2. 如果是全新的 middleware：
   - 插入到核心 middleware 之后。
   - 放在 profile、prompt caching、memory 等 tail middleware 之前。

通俗理解：

```text
用户可以替换默认零件，也可以加新零件。
但新零件会被放在一个相对安全、可预期的位置。
```

## 受保护 Middleware

文件中定义了：

```python
_REQUIRED_MIDDLEWARE = (
    (FilesystemMiddleware, ()),
    (SubAgentMiddleware, ()),
)
```

这表示有些 middleware 不能被 profile 随便排除。

原因：

- `FilesystemMiddleware` 支撑内置文件工具，并负责文件权限检查。
- `SubAgentMiddleware` 支撑 `task` 工具。

如果 profile 错误地排除了这些核心 middleware，Deep Agents 会抛错，而不是悄悄生成一个残缺 Agent。

## `create_deep_agent()` 是什么

`create_deep_agent()` 是本文件最核心的函数，也是 Deep Agents SDK 的主入口。

它负责创建一个完整的 Deep Agent。

默认工具包括：

```text
write_todos
ls
read_file
write_file
edit_file
glob
grep
execute
task
```

注意：

- `execute` 是否真正可用，取决于 backend 是否支持 sandbox execution。
- `task` 是否存在，取决于是否有同步 subagent。

## `create_deep_agent()` 的重要参数

### `model`

指定模型。

可以是：

- `"provider:model"` 字符串。
- 已初始化的 `BaseChatModel`。
- `None`，但这种默认模型用法已 deprecated。

### `tools`

用户自定义工具。

这些工具会追加到 Deep Agents 内置工具上。

重点：

```text
tools 是 additive 的。
传入 tools 不会移除内置工具。
```

如果想移除工具，要通过 profile 的 `excluded_tools`。

### `system_prompt`

用户自定义系统提示词。

它会放在 SDK 默认 prompt 前面，因此优先级更高。

### `middleware`

用户自定义 middleware。

可以替换默认 middleware，也可以新增 middleware。

### `subagents`

子代理配置。

支持三种：

1. `SubAgent`
   - 声明式同步子代理。
   - 通过 `task` 工具调用。

2. `CompiledSubAgent`
   - 已经编译好的 runnable。
   - 也通过 `task` 工具调用。

3. `AsyncSubAgent`
   - 远程或后台子代理。
   - 通过 `AsyncSubAgentMiddleware` 暴露启动、检查、取消等工具。

### `skills`

skill 来源路径列表。

Deep Agents 会通过 `SkillsMiddleware` 按需加载这些技能。

### `memory`

memory 文件路径列表，通常是 `AGENTS.md`。

Deep Agents 会通过 `MemoryMiddleware` 把这些记忆加入系统 prompt。

### `permissions`

文件系统权限规则。

支持：

```text
allow
deny
interrupt
```

其中 `interrupt` 会自动接入 human-in-the-loop。

### `backend`

文件、memory、shell execution 使用的后端。

默认是：

```python
StateBackend()
```

如果需要真正执行 shell 命令，需要传入支持 sandbox execution 的 backend。

### `interrupt_on`

指定哪些工具调用需要暂停等待人工确认。

例如：

```python
interrupt_on={"edit_file": True}
```

表示每次编辑文件前都暂停。

### `state_schema`

自定义 LangGraph state schema。

如果传入，应继承或兼容 `DeepAgentState`，以保留 `DeltaChannel` 的 messages 行为。

### `checkpointer` / `store` / `cache`

这些直接传给 LangChain / LangGraph：

- `checkpointer`：保存 graph 状态。
- `store`：持久化存储。
- `cache`：缓存。

## 主函数内部流程

`create_deep_agent()` 内部大致可以分成以下步骤。

### 1. 解析模型

如果 `model is None`：

- 发出 deprecation warning。
- 使用默认 Anthropic 模型。

否则：

- 调用 `resolve_model(model)` 解析模型。

然后根据模型找到对应 harness profile：

```python
_profile = _harness_profile_for_model(model, _model_spec)
```

profile 会影响：

- prompt。
- extra middleware。
- excluded middleware。
- excluded tools。
- tool description overrides。
- general-purpose subagent。

### 2. 准备工具和 backend

用户传入的 tools 会先应用 tool description overrides：

```python
_tools = _apply_tool_description_overrides(...)
```

backend 默认是：

```python
StateBackend()
```

### 3. 处理用户传入的 subagents

函数会把 subagents 分成两类：

```text
inline_subagents = 同步 subagents
async_subagents = 远程 / 后台 subagents
```

判断逻辑：

- 有 `graph_id`：认为是 `AsyncSubAgent`。
- 有 `runnable`：认为是 `CompiledSubAgent`。
- 否则：认为是声明式 `SubAgent`。

对于声明式 `SubAgent`，函数会：

1. 解析 subagent model。
2. 找 subagent 对应 profile。
3. 继承或覆盖 permissions。
4. 构建 subagent 自己的 middleware stack。
5. 加入 skills。
6. 加入 prompt caching。
7. 应用 excluded middleware。
8. 合并自定义 middleware。
9. 应用 excluded tools。
10. 合并 interrupt 配置。
11. 继承或覆盖 tools。
12. 应用 profile prompt。
13. 加入 `inline_subagents`。

重点：

```text
子代理不是简单 prompt。
每个声明式子代理都有自己的 model、tools、middleware、permissions、skills、interrupt。
```

### 4. 自动添加默认 general-purpose subagent

如果用户没有提供名为 `general-purpose` 的同步子代理，并且 profile 没禁用默认子代理，Deep Agents 会自动添加一个默认 general-purpose subagent。

它的用途是：

```text
处理通用复杂任务、搜索文件、执行多步骤任务。
```

这个默认子代理也会有自己的 middleware stack：

- `TodoListMiddleware`
- `FilesystemMiddleware`
- summarization middleware
- `PatchToolCallsMiddleware`
- skills
- profile middleware
- prompt caching
- tool exclusion

然后插入到 `inline_subagents` 最前面。

### 5. 构建主 Agent middleware stack

主 Agent 的 middleware stack 大致是：

```text
TodoListMiddleware
SkillsMiddleware                如果传入 skills
FilesystemMiddleware
SubAgentMiddleware              如果有 inline_subagents
SummarizationMiddleware
PatchToolCallsMiddleware
AsyncSubAgentMiddleware         如果有 async_subagents
profile extra middleware
prompt caching middleware
MemoryMiddleware                如果传入 memory
HumanInTheLoopMiddleware        如果有 interrupt_on 或 interrupt permission
custom middleware
ToolExclusionMiddleware         如果 profile 排除工具
```

注意：

- 用户 middleware 会插入在核心 middleware 之后、tail middleware 之前。
- profile excluded middleware 会执行两次过滤：自定义 middleware 合并前后各一次。
- tool exclusion 最后执行，避免被后续 middleware 恢复。

### 6. 处理 private state keys

函数会收集所有 middleware 的 private state 字段：

```python
private_state_keys = private_state_field_names(...)
```

然后传给 `SubAgentMiddleware`。

作用：

```text
避免主 Agent 的内部 middleware state 泄漏给 subagents。
```

### 7. 组合最终 system prompt

先应用 profile prompt：

```python
base_prompt = _apply_profile_prompt(_profile, BASE_AGENT_PROMPT)
```

然后：

- 如果用户没有传 `system_prompt`，直接使用 `base_prompt`。
- 如果传的是 `SystemMessage`，追加 base prompt 到 content blocks。
- 如果传的是字符串，把用户 prompt 放在前面，再拼 base prompt。

结果：

```text
用户 prompt 总是在 SDK 默认 prompt 前面。
```

### 8. 调用 LangChain `create_agent(...)`

最后一步：

```python
return create_agent(
    model,
    system_prompt=final_system_prompt,
    tools=_tools,
    middleware=deepagent_middleware,
    ...
)
```

然后再调用：

```python
.with_config(...)
```

配置内容包括：

- `recursion_limit = 9999`
- LangSmith / LangChain metadata：
  - `ls_integration = "deepagents"`
  - deepagents 版本号
  - agent name

## 一张流程图

```text
create_deep_agent(...)
        |
        v
解析 model
        |
        v
匹配 harness profile
        |
        v
准备 tools 和 backend
        |
        v
处理用户 subagents
        |
        v
必要时添加 general-purpose subagent
        |
        v
组装主 Agent middleware stack
        |
        v
合并 memory / interrupt / custom middleware / tool exclusion
        |
        v
组合最终 system prompt
        |
        v
调用 LangChain create_agent(...)
        |
        v
返回可运行的 LangGraph graph
```

## 如何理解这个文件

不要把 `graph.py` 理解为“执行任务的地方”。

它更像：

```text
配置解析器 + 组件装配器 + LangChain create_agent 包装器
```

真正执行文件读写的是：

```text
middleware/filesystem.py
backends/
```

真正执行 subagent 委派的是：

```text
middleware/subagents.py
middleware/async_subagents.py
```

真正执行 memory 注入的是：

```text
middleware/memory.py
```

真正执行 skills 加载的是：

```text
middleware/skills.py
```

`graph.py` 的职责是把这些能力按正确顺序装到同一个 Agent 上。

## 阅读源码时的抓手

可以按这些问题读 `graph.py`：

1. 用户传入的参数被谁消费？
2. 哪些参数会影响 middleware stack？
3. 哪些参数会影响 system prompt？
4. 哪些参数会传给 LangChain `create_agent()`？
5. 哪些能力是 main agent 和 subagent 都有的？
6. 哪些能力只属于 main agent？
7. profile 在哪里改变行为？
8. backend 在哪里决定工具能力？

## 最重要的理解

`create_deep_agent()` 的核心不是“创建一个模型”，而是：

```text
把模型、工具、状态、backend、middleware、subagents、skills、memory 和 prompt 组装成一个完整的任务执行系统。
```

也就是说，它创建的是一个 Deep Agent harness，而不是一个普通聊天机器人。
