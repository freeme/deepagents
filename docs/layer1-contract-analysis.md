# Layer 1：契约层分析 — DeepAgents 对外承诺了什么

> 分析时间：2026-04-07  
> 分析范围：`libs/deepagents/deepagents/` 核心包的公开接口层  
> 核心原则：不看实现，只看接口——框架对使用者承诺了什么

---

## 分析对象

| 文件 | 职责 |
|------|------|
| `deepagents/__init__.py` | 顶级导出符号 |
| `deepagents/graph.py`（签名部分） | `create_deep_agent` 工厂函数 |
| `deepagents/backends/protocol.py` | 后端抽象接口 |
| `deepagents/middleware/__init__.py` | 中间件架构说明 |
| `deepagents/_models.py` | 模型解析策略 |

---

## 一、顶级公开 API（`__init__.py`）

框架采用极简主义导出策略，只暴露 9 个符号：

```
create_deep_agent()              ← 唯一工厂函数

中间件（用户可选扩展点）：
  FilesystemMiddleware
  MemoryMiddleware
  SubAgentMiddleware
  SubAgent
  CompiledSubAgent
  AsyncSubAgentMiddleware
  AsyncSubAgent

__version__
```

**设计洞察**：`SkillsMiddleware` 和 `SummarizationMiddleware` **未被导出**到顶级 `__init__.py`（但在 `middleware/__init__.py` 中有导出）。这是有意为之——Skills 和 Summarization 是内部默认行为，用户无需直接操作；而 Memory/Filesystem/SubAgent 是用户可能需要自定义配置的扩展点。

---

## 二、`create_deep_agent()` 函数签名全解

```python
create_deep_agent(
    # 核心两参数（位置参数，最常用）
    model: str | BaseChatModel | None = None,     # 默认 claude-sonnet-4-6
    tools: Sequence[...] | None = None,           # 追加工具

    # 行为定制（keyword-only）
    system_prompt: str | SystemMessage | None,    # 自定义系统提示
    middleware: Sequence[AgentMiddleware] = (),   # 插入中间件（内置栈之后）
    subagents: Sequence[SubAgent|CompiledSubAgent|AsyncSubAgent] | None,
    skills: list[str] | None,                    # 技能文件路径列表
    memory: list[str] | None,                    # AGENTS.md 路径列表
    response_format: ...,                         # 结构化输出

    # 基础设施（LangGraph 原语直透）
    context_schema: type[ContextT] | None,
    checkpointer: Checkpointer | None,
    store: BaseStore | None,
    backend: BackendProtocol | BackendFactory | None,  # 默认 StateBackend()

    # 控制流
    interrupt_on: dict[str, bool | InterruptOnConfig] | None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph
```

### 参数层次结构

```
必填（隐式默认）            可选扩展              基础设施直透
──────────────────          ────────────────      ─────────────────
model ← claude-sonnet-4-6   tools                 checkpointer
backend ← StateBackend()    system_prompt         store
                            middleware            cache
                            subagents             context_schema
                            skills                response_format
                            memory
                            interrupt_on

      ↑                          ↑                      ↑
 零配置即可用               业务逻辑定制             LangGraph 原语透传
```

### 关键设计决策

- `model=None` 和 `backend=None` 在函数体内第一行被替换为默认值，零配置即可运行
- `middleware` 参数插入的位置在内置中间件栈之后、`AnthropicPromptCachingMiddleware` 和 `MemoryMiddleware` 之前，顺序是固定的
- `system_prompt` 总是被**前置**到 `BASE_AGENT_PROMPT` 之前（而非替换），保证基础行为不被覆盖
- `subagents` 同时支持三种形态：声明式 `SubAgent`、预编译 `CompiledSubAgent`、异步远程 `AsyncSubAgent`

---

## 三、`BackendProtocol` 接口全貌

后端接口是整个框架最精心设计的抽象之一，分为两层：

### 3.1 文件操作层（`BackendProtocol`）

```
读操作：
  ls(path)                        → LsResult
  read(path, offset, limit)       → ReadResult
  grep(pattern, path, glob)       → GrepResult
  glob(pattern, path)             → GlobResult

写操作：
  write(path, content)            → WriteResult    ← 文件不存在时才写（防覆盖）
  edit(path, old, new, all?)      → EditResult     ← 精确字符串替换

批量操作：
  upload_files([(path, bytes)])   → [FileUploadResponse]
  download_files([path])          → [FileDownloadResponse]

每个操作均有异步版本：als / aread / agrep / aglob / awrite / aedit / ...
```

### 3.2 执行层（`SandboxBackendProtocol`，继承自 `BackendProtocol`）

```
execute(command, timeout?)    → ExecuteResponse
aexecute(command, timeout?)   → ExecuteResponse
id: str (property)            ← sandbox 实例唯一标识
```

### 3.3 文件格式演进：v1 → v2

```
v1（legacy）                       v2（current）
─────────────────────              ──────────────────────────────
content: list[str]                 content: str（UTF-8 或 base64）
（按 \n 分割的行列表）              encoding: "utf-8" | "base64"
无 encoding 字段                   created_at: str（ISO 8601）
                                   modified_at: str（ISO 8601）
```

后端向下兼容 v1 格式（发出 `DeprecationWarning`），v0.7 起将移除。

### 3.4 错误处理哲学

所有操作返回 `XxxResult(error: str | None, data: ...)` 而非抛出异常。

错误码是面向 LLM 设计的字面量：
```
"file_not_found" | "permission_denied" | "is_directory" | "invalid_path"
```

LLM 可以读懂这些错误码并决定下一步行动（如：遇到 `file_not_found` 改用 `write`，遇到 `is_directory` 改用 `ls`）。

### 3.5 `write` 防覆盖设计

`write` 只能创建新文件，文件已存在时报错；修改已有文件必须用 `edit`（精确字符串替换）。这迫使 LLM 先 `read` 确认文件状态再决定使用哪个操作，防止意外覆盖。

---

## 四、中间件架构契约（`middleware/__init__.py`）

### 4.1 两条工具注入路径

```
路径 1: SDK middleware（AgentMiddleware.wrap_model_call 钩子）
         │
         ├─ 可以：每次 LLM 调用前动态过滤工具列表
         ├─ 可以：向系统提示注入内容
         ├─ 可以：变换消息历史（截断、摘要）
         └─ 可以：维护跨轮持久状态

路径 2: 用户传入的 tools=[] 列表
         │
         └─ 只能：被 LLM 调用时执行
            不能：主动修改任何上下文

两路都在 create_deep_agent() 内合并后提供给 LLM
```

### 4.2 何时用 middleware vs 普通 tool

| 需求 | 用什么 |
|------|--------|
| 每次 LLM 调用前修改系统提示 | middleware |
| 动态决定给 LLM 哪些工具 | middleware |
| 跨轮持久状态 | middleware |
| 所有 SDK 消费者都需要 | middleware |
| 无状态、独立执行 | 普通 tool |
| 只有特定消费者（如 CLI）需要 | 普通 tool |

---

## 五、模型解析策略（`_models.py`）

```
resolve_model(model: str | BaseChatModel)
              │
              ├─ 已是 BaseChatModel  → 直接返回（用户完全控制）
              │
              ├─ "openai:..."        → init_chat_model(use_responses_api=True)
              │                         默认走 Responses API，非 Chat Completions
              │
              ├─ "openrouter:..."    → 版本检查（>= 0.2.0）
              │                         + 注入归因 headers
              │                         （X-Title / HTTP-Referer）
              │                         env var 可覆盖
              │
              └─ 其他字符串          → init_chat_model(model)
                                        LangChain 标准多 provider 路由
```

**OpenRouter 归因机制**：使用 OpenRouter 时，框架默认注入 `X-Title: Deep Agents` 和 `HTTP-Referer: github.com/langchain-ai/deepagents`，用于 OpenRouter 使用量统计。用户可通过 `OPENROUTER_APP_URL` / `OPENROUTER_APP_TITLE` 环境变量覆盖。

---

## 六、附录：General-Purpose Subagent 注入机制

> 此部分在 Layer 1 探索过程中延伸分析，详见独立文档。

注入分三阶段：

1. **提前构建**：`graph.py` 在函数入口处构建 `general_purpose_spec`，继承主 agent 的 `model` 和 `tools`，但拥有独立的中间件栈
2. **条件注入**：遍历用户传入的 `subagents` 后，检查是否已存在 `name == "general-purpose"` 的 SubAgent，不存在则 `insert(0, general_purpose_spec)`
3. **编译成图**：`SubAgentMiddleware.__init__` 时调用 `create_agent()` 把所有 SubAgent 编译为 `CompiledStateGraph`，编译成本在启动时支付

**覆盖方法**：传入 `subagents=[{"name": "general-purpose", ...}]` 即可替换默认 GP 子智能体。

---

## 七、Layer 1 总结：核心承诺

```
✅ 零配置可用     create_deep_agent() 无参数即可运行
✅ Provider 无关  str 格式支持所有 LangChain 支持的模型
✅ Backend 可换   BackendProtocol 完全解耦存储和执行
✅ LLM 友好错误  文件操作返回标准错误码而非异常
✅ 同步/异步对称  每个 backend 方法都有 a* 异步版本
✅ 向前兼容      deprecated 方法保留到 v0.7，带 DeprecationWarning
✅ 扩展点明确    middleware / tool / backend 三个层次各有职责
```

---

## 八、遗留问题（转交 Layer 2）

- [ ] `AgentMiddleware.wrap_model_call` 的完整签名和执行时机
- [ ] `StateBackend` 默认行为：文件存储在哪里？会话级还是请求级？
- [ ] `SummarizationMiddleware` 的触发条件：token 数量阈值是多少？
- [ ] `PatchToolCallsMiddleware` 修复了 Anthropic 的什么格式问题？
- [ ] `BackendFactory` 类型的用途：什么时候需要工厂而不是实例？
- [ ] `AnthropicPromptCachingMiddleware` 必须放在最后的具体原因

---

*下一步：[Layer 2 - 运行机制分析](./layer2-runtime-analysis.md)（待创建）*
