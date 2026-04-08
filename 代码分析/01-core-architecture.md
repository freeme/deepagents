# 一、核心架构与入口点

## 概述

DeepAgents 在 `deepagents.graph.create_deep_agent` 中把 **LangChain `create_agent`**（底层为 LangGraph 的 `CompiledStateGraph`）与一组 **固定顺序的中间件栈**、**子代理（含默认 `general-purpose`）**、**后端（默认 `StateBackend`）** 和 **可解析的模型字符串** 组装在一起，形成「深度代理」运行时。

整体数据流与分层（ASCII）：

```
                    ┌─────────────────────────────────────────┐
                    │  调用方：create_deep_agent(...)          │
                    └───────────────────┬─────────────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        │                               │                               │
        ▼                               ▼                               ▼
 resolve_model /              SubAgent 规格处理                 backend 默认
 get_default_model            (inline / async / compiled)      StateBackend
        │                               │                               │
        └───────────────────────────────┴───────────────────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │  主代理 middleware 栈（有序拼接）       │
                    │  Todo → Skills? → FS → SubAgent →      │
                    │  Summarize → Patch → Async? →          │
                    │  用户 middleware → AnthropicCache →     │
                    │  Memory? → HumanInTheLoop?             │
                    └───────────────────┬───────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │  system_prompt ⊕ BASE_AGENT_PROMPT      │
                    └───────────────────┬───────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │  langchain.agents.create_agent(...)     │
                    │  → CompiledStateGraph                   │
                    └───────────────────┬───────────────────┘
                                        │
                    ┌───────────────────▼───────────────────┐
                    │  .with_config({                        │
                    │    recursion_limit: 9999,              │
                    │    metadata: { ls_integration, ... }   │
                    │  })                                    │
                    └────────────────────────────────────────┘
```

公开 API 与中间件包的关系：

```
deepagents/__init__.py          deepagents/middleware/__init__.py
─────────────────────          ─────────────────────────────────
create_deep_agent  ◄──────────  文档说明：SDK 中间件 vs 用户 tools
__version__                     导出各 Middleware / SubAgent 类型
Async*, Compiled*, FS, Mem, Sub*
```

---

## 1.1 `create_deep_agent` 工厂函数

### 完整函数签名与参数

```python
def create_deep_agent(
    model: str | BaseChatModel | None = None,
    tools: Sequence[BaseTool | Callable | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware] = (),
    subagents: Sequence[SubAgent | CompiledSubAgent | AsyncSubAgent] | None = None,
    skills: list[str] | None = None,
    memory: list[str] | None = None,
    response_format: ResponseFormat[ResponseT] | type[ResponseT] | dict[str, Any] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    backend: BackendProtocol | BackendFactory | None = None,
    interrupt_on: dict[str, bool | InterruptOnConfig] | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph[AgentState[ResponseT], ContextT, _InputAgentState, _OutputAgentState[ResponseT]]:
```

### 参数表（默认值与类型）

| 参数 | 类型注解 | 默认值 | 说明 |
|------|----------|--------|------|
| `model` | `str \| BaseChatModel \| None` | `None` | `None` 时用 `get_default_model()`（Claude Sonnet 4.6）；否则 `resolve_model` |
| `tools` | `Sequence[...] \| None` | `None` | 传给 `create_agent`；子代理默认继承 `tools or []` |
| `system_prompt` | `str \| SystemMessage \| None` | `None` | 与 `BASE_AGENT_PROMPT` 合并 |
| `middleware` | `Sequence[AgentMiddleware]` | `()` | 插在主栈「业务中间件」之后、Anthropic 缓存与 Memory 之前 |
| `subagents` | 多种子代理规格序列 \| None | `None` | 分 `SubAgent` / `CompiledSubAgent` / `AsyncSubAgent` 三条路径 |
| `skills` | `list[str] \| None` | `None` | 非空则为主代理与默认子代理注入 `SkillsMiddleware` |
| `memory` | `list[str] \| None` | `None` | 非空则追加 `MemoryMiddleware`（在 Anthropic 缓存之后） |
| `response_format` | 多种 \| None | `None` | 结构化输出，透传 `create_agent` |
| `context_schema` | `type[ContextT] \| None` | `None` | 运行时上下文 schema |
| `checkpointer` | `Checkpointer \| None` | `None` | 状态持久化 |
| `store` | `BaseStore \| None` | `None` | 持久存储（部分 backend 需要） |
| `backend` | `BackendProtocol \| BackendFactory \| None` | `None` | `None` 时默认 `StateBackend()` |
| `interrupt_on` | `dict \| None` | `None` | 主代理：最终 `HumanInTheLoopMiddleware`；声明式子代理可继承或覆盖 |
| `debug` | `bool` | `False` | 透传 `create_agent` |
| `name` | `str \| None` | `None` | 透传 `create_agent`，并写入 `metadata["lc_agent_name"]` |
| `cache` | `BaseCache \| None` | `None` | 透传 `create_agent` |

### Middleware 堆栈构建顺序与优先级

**主代理 `deepagent_middleware`**（按 `append` 顺序拼接）：

```
序号  中间件                                         条件
───  ──────────────────────────────────────────    ──────────────
 1   TodoListMiddleware()                           无条件
 2   SkillsMiddleware(backend, sources=skills)      skills is not None
 3   FilesystemMiddleware(backend)                  无条件
 4   SubAgentMiddleware(backend, inline_subagents)  无条件
 5   create_summarization_middleware(model, backend) 无条件
 6   PatchToolCallsMiddleware()                     无条件
 7   AsyncSubAgentMiddleware(async_subagents)       存在异步子代理时
 8   用户传入的 middleware（extend）                 无条件
 9   AnthropicPromptCachingMiddleware(...)          无条件（放末尾）
10   MemoryMiddleware(backend, sources=memory)      memory is not None
11   HumanInTheLoopMiddleware(interrupt_on)         interrupt_on is not None
```

注释明确：**缓存与 memory 放在最后**，避免 memory 更新破坏 Anthropic prompt cache 前缀：

```python
# Caching + memory after all other middleware so memory updates don't
# invalidate the Anthropic prompt cache prefix.
deepagent_middleware.append(AnthropicPromptCachingMiddleware(unsupported_model_behavior="ignore"))
if memory is not None:
    deepagent_middleware.append(MemoryMiddleware(backend=backend, sources=memory))
if interrupt_on is not None:
    deepagent_middleware.append(HumanInTheLoopMiddleware(interrupt_on=interrupt_on))
```

**默认「general-purpose」子代理**的中间件序（更简化，无 SubAgentMiddleware 避免递归）：

```
TodoListMiddleware()
FilesystemMiddleware(backend)
create_summarization_middleware(model, backend)
PatchToolCallsMiddleware()
[SkillsMiddleware(...)]  # 若 skills is not None
AnthropicPromptCachingMiddleware(...)
```

**声明式 `SubAgent`** 在展开时的栈：Todo → FS → Summarize → Patch →（可选）Skills → **用户 spec 的 `middleware`** → AnthropicCache。

### `BASE_AGENT_PROMPT` 全文与设计取向

```
BASE_AGENT_PROMPT = """You are a Deep Agent, an AI assistant that helps users accomplish tasks using tools. You respond with text and tool calls. The user can see your responses and tool outputs in real time.

## Core Behavior
- Be concise and direct. Don't over-explain unless asked.
- NEVER add unnecessary preamble ("Sure!", "Great question!", "I'll now...").
- Don't say "I'll now do X" — just do it.
- If the request is ambiguous, ask questions before acting.
- If asked how to approach something, explain first, then act.

## Professional Objectivity
- Prioritize accuracy over validating the user's beliefs
- Disagree respectfully when the user is incorrect
- Avoid unnecessary superlatives, praise, or emotional validation

## Doing Tasks
When the user asks you to do something:
1. **Understand first** — read relevant files, check existing patterns.
2. **Act** — implement the solution.
3. **Verify** — check your work against what was asked.

Keep working until the task is fully complete. Don't stop partway...

## Progress Updates
For longer tasks, provide brief progress updates at reasonable intervals..."""
```

**设计哲学（归纳）**：
- 强调**少废话**：禁止 "Sure!", "Great question!" 等无效前言
- **先理解再行动**：先读文件、检查现有模式，再实施
- **客观纠错**：优先准确，不迎合用户偏见
- **长任务持续执行**：直到完成或真正阻塞才停止
- 与「工具型 coding agent」产品定位一致

### 与 `system_prompt` 的合并规则

```python
if system_prompt is None:
    final_system_prompt = BASE_AGENT_PROMPT
elif isinstance(system_prompt, SystemMessage):
    final_system_prompt = SystemMessage(content_blocks=[
        *system_prompt.content_blocks,
        {"type": "text", "text": f"\n\n{BASE_AGENT_PROMPT}"}
    ])
else:
    # String: 简单拼接
    final_system_prompt = system_prompt + "\n\n" + BASE_AGENT_PROMPT
```

### 最终调用与返回

```python
return create_agent(
    model,
    system_prompt=final_system_prompt,
    tools=tools,
    middleware=deepagent_middleware,
    ...
).with_config({
    "recursion_limit": 9_999,
    "metadata": {
        "ls_integration": "deepagents",
        "versions": {"deepagents": __version__},
        "lc_agent_name": name,
    },
})
```

---

## 1.2 LangGraph 集成与图结构

### `create_agent` 的调用方式

- **来源**：`from langchain.agents import create_agent`
- **传入**：已解析的 `model`、`final_system_prompt`、`tools`、组装好的 `middleware` 列表，以及其他配置
- **结果**：`create_agent` 返回 LangGraph 侧已编译的状态图；DeepAgents 再 **`with_config`** 设置递归上限与 LangSmith 元数据

DeepAgents **不**在本仓库内重新实现图节点边，而是依赖 LangChain Agent 抽象的 **`CompiledStateGraph`**。

### `AgentState` 类型体系

```
AgentState[ResponseT]          ← 运行时内部状态
├── _InputAgentState           ← 图对外输入视图（用户消息等）
└── _OutputAgentState[ResponseT] ← 图对外输出视图（含可选结构化响应）
```

三者均从 `langchain.agents` / `langchain.agents.middleware.types` 导入，本仓库仅消费类型定义。

### 默认配置意义

| 配置项 | 值 | 意义 |
|--------|-----|------|
| `recursion_limit` | `9_999` | 允许极深的 ReAct/工具循环，避免长任务过早被截断 |
| `ls_integration` | `"deepagents"` | LangSmith 追踪中标记集成来源 |
| `versions.deepagents` | `__version__` | 可追溯具体版本 |
| `lc_agent_name` | `name` | LangSmith 中对 agent 命名 |

---

## 1.3 模型解析与适配

### `resolve_model` 完整逻辑（`_models.py`）

```
        model
          │
          ├─ BaseChatModel ──────────────► 原样返回
          │
          ├─ str 以 "openai:" 开头 ───────► init_chat_model(..., use_responses_api=True)
          │
          ├─ str 以 "openrouter:" 开头 ──► 版本检查 ──► init_chat_model(..., **attribution)
          │
          └─ 其他字符串 ─────────────────► init_chat_model(model)
```

```python
def resolve_model(model: str | BaseChatModel) -> BaseChatModel:
    if isinstance(model, BaseChatModel):
        return model
    if model.startswith("openai:"):
        return init_chat_model(model, use_responses_api=True)
    if model.startswith("openrouter:"):
        check_openrouter_version()
        return init_chat_model(model, **_openrouter_attribution_kwargs())
    return init_chat_model(model)
```

### `provider:model` 格式

字符串通过 **`langchain.chat_models.init_chat_model`** 解析；约定为 **`provider:model`** 格式：
- `openai:gpt-4o`
- `anthropic:claude-opus-4-5`
- `openrouter:openai/gpt-4o`

`model_matches_spec` 通过 `partition(":")` 检查当前模型是否匹配规格字符串。

### OpenAI Responses API vs Chat Completions 切换策略

| 场景 | 策略 |
|------|------|
| 字符串 `"openai:..."` | 强制 `use_responses_api=True`（Responses API 默认） |
| 传入 `BaseChatModel` 实例 | 完全由用户配置决定，`resolve_model` 不改写 |
| 需要 Chat Completions | 先 `init_chat_model("openai:...", use_responses_api=False)` 再传入 |
| 需要限制数据保留 | 传入带 `store=False`、`include=[...]` 等参数的已构造模型 |

---

## 关键设计决策

1. **薄封装、厚中间件**：核心能力放在各 `AgentMiddleware` 的钩子里，`graph.py` 本身只做组装，不写业务节点
2. **默认子代理可覆盖**：若用户提供的 `subagents` 里已有名为 `general-purpose` 的项，则不再插入默认 GP 规格
3. **Anthropic 缓存位置固定在末尾**：注释明确说明避免 cache 失效的设计意图
4. **模型串的特例处理**：
   - `openai:` → Responses API 默认
   - `openrouter:` → 版本检查 + 归因 header（可被环境变量覆盖）
   - 其余走通用 `init_chat_model`
5. **运行规模**：`recursion_limit=9999` 体现「长链路工具调用」预期，与「做到完成为止」的 `BASE_AGENT_PROMPT` 相互呼应
6. **`None` 时的安全默认**：`backend=None` → `StateBackend()`（内存存储），无需外部依赖即可运行
