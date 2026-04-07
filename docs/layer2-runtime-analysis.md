# Layer 2：运行机制分析 — 一次请求的完整流动

> 分析时间：2026-04-07  
> 分析范围：`graph.py` 全文 + `middleware/` 下六个核心中间件  
> 核心原则：跟踪一次 `agent.invoke({"messages": [...]})` 调用，从入口到输出的完整路径

---

## 一、`graph.py` 全貌：装配流水线

`create_deep_agent()` 本质上是一个**工厂函数**，不执行任何业务逻辑，只做两件事：

1. **构建 general-purpose subagent**（固定的第一步）
2. **装配主 agent 的中间件栈**（顺序严格固定）

### 1.1 完整装配顺序（实际代码顺序）

```
create_deep_agent() 调用时：

  ① resolve_model(model)         ← 字符串 → BaseChatModel
  ② StateBackend() (默认)        ← 文件存储后端

  ── 构建 general-purpose subagent ──
  ③ gp_middleware = [
       TodoListMiddleware(),
       FilesystemMiddleware(backend),
       create_summarization_middleware(model, backend),
       PatchToolCallsMiddleware(),
       SkillsMiddleware(...)       ← 仅当 skills != None
       AnthropicPromptCachingMiddleware(),
     ]
  ④ general_purpose_spec = {name, description, system_prompt,
                              model, tools, middleware=gp_middleware}

  ── 处理用户提供的 subagents ──
  ⑤ for spec in subagents:
       AsyncSubAgent → async_subagents 列表
       CompiledSubAgent → inline_subagents（直接用）
       SubAgent → 填充 model/tools/middleware 默认值 → inline_subagents

  ⑥ 如果 inline_subagents 中没有 "general-purpose":
       inline_subagents.insert(0, general_purpose_spec)

  ── 装配主 agent 中间件栈 ──
  ⑦ deepagent_middleware = [
       TodoListMiddleware(),
       SkillsMiddleware(...)       ← 仅当 skills != None（注意：主agent比GP先加Skills）
       FilesystemMiddleware(backend),
       SubAgentMiddleware(backend, inline_subagents),  ← 编译所有子agent
       create_summarization_middleware(model, backend),
       PatchToolCallsMiddleware(),
       AsyncSubAgentMiddleware(async_subagents),        ← 仅当有异步子agent
       *middleware,                                      ← 用户自定义（追加在此）
       AnthropicPromptCachingMiddleware(),               ← 倒数第三（Caching边界）
       MemoryMiddleware(backend, memory),                ← 仅当 memory != None
       HumanInTheLoopMiddleware(interrupt_on),          ← 仅当 interrupt_on != None
     ]

  ⑧ system_prompt 处理：
       None     → BASE_AGENT_PROMPT
       str      → system_prompt + "\n\n" + BASE_AGENT_PROMPT
       SystemMessage → 内容块合并（用户前置，BASE后置）

  ⑨ create_agent(model, ..., middleware=deepagent_middleware)
       .with_config({
           "recursion_limit": 9_999,   ← 极高上限，防止复杂 agent 被截断
           "metadata": { "ls_integration": "deepagents", ... }
       })
```

### 1.2 中间件顺序不可改变的原因

| 顺序约束 | 原因 |
|---------|------|
| `AnthropicPromptCachingMiddleware` 必须在 `MemoryMiddleware` 之前 | Memory 内容每轮可能变化，放在 Caching 之后会使 Anthropic 的 prompt cache prefix 失效 |
| `PatchToolCallsMiddleware` 必须在所有工具中间件之前 | 需要在 LLM 看到消息前修复「悬空工具调用」 |
| `SubAgentMiddleware` 在 `FilesystemMiddleware` 之后 | 子 agent 可以访问同一 backend 的文件，顺序保证文件工具已注入 |
| 用户 `middleware` 插入在 Caching/Memory 之前 | 用户逻辑不应影响缓存边界 |

---

## 二、`FilesystemMiddleware` — 文件系统工具绑定

### 2.1 工具创建时机

所有 7 个工具在 `__init__` 阶段就**全部创建完毕**，通过闭包捕获 `self.backend` 引用：

```
__init__ 创建：
  ls          → 列目录
  read_file   → 分页读文件
  write_file  → 创建新文件（防覆盖）
  edit_file   → 精确字符串替换（要求先 read）
  glob        → 文件名匹配（带 20s 超时保护）
  grep        → 文本内容搜索
  execute     → shell 命令（仅 SandboxBackendProtocol 支持）
```

### 2.2 `wrap_model_call` — 每次 LLM 调用前的三件事

```
wrap_model_call(request, handler):

  1. 动态过滤 execute 工具
     检测 backend 是否实现 SandboxBackendProtocol
     → 不支持则从 request.tools 中剔除 execute
     → 同时从 system_prompt 中不添加执行相关说明

  2. 注入文件系统系统提示
     FILESYSTEM_SYSTEM_PROMPT（固定）
     + EXECUTION_SYSTEM_PROMPT（仅当 execute 可用时追加）

  3. 处理超大 HumanMessage 驱逐
     阈值：50,000 tokens（约 200KB）
     → 写入 /conversation_history/{uuid}.md
     → 消息打上 "lc_evicted_to" 标记
     → 后续每次调用替换为头尾预览 + 文件引用
```

### 2.3 `wrap_tool_call` — 工具结果过大驱逐

工具调用结果超过 **20,000 tokens**（约 80KB）时自动驱逐到文件系统：

```
工具结果 → 写入 /large_tool_results/{sanitized_tool_call_id}
LLM 收到 → 截断预览（头5行 + 尾5行）+ 文件路径引用
```

以下工具**排除**在驱逐机制之外（有各自的截断/限制策略）：
- `ls`, `glob`, `grep` — 内置输出截断
- `read_file` — 分页读取本身已处理
- `edit_file`, `write_file` — 返回内容极少

### 2.4 glob 的线程池保护

```python
# sync 版本：
with concurrent.futures.ThreadPoolExecutor(max_workers=1) as executor:
    future = executor.submit(lambda: ctx.run(resolved_backend.glob, ...))
    glob_result = future.result(timeout=GLOB_TIMEOUT)  # 20s
```

设计动机：glob 在大型文件系统可能扫描数千目录，此机制防止挂死主循环。

---

## 三、`SubAgentMiddleware` — 子智能体编译与调用

### 3.1 编译时机

所有子 agent 在 `SubAgentMiddleware.__init__` 时**全部编译**：

```
SubAgentMiddleware.__init__:
  for spec in subagents:
    CompiledSubAgent → 直接使用 runnable
    SubAgent →
      resolve_model(spec.model)
      build middleware list (caller-provided)
      create_agent(model, system_prompt, tools, middleware)
      → CompiledStateGraph
```

**启动成本前置**：编译发生在 `create_deep_agent()` 调用时，不是在运行时。

### 3.2 `task` 工具的执行流程

```
agent.task(description="...", subagent_type="general-purpose")
  │
  ├─ 查找 subagent_graphs[subagent_type]
  │
  ├─ 准备 subagent_state（继承父 agent 状态，排除隔离键）
  │   _EXCLUDED_STATE_KEYS = {
  │     "messages",          ← 子 agent 从全新消息开始
  │     "todos",             ← 任务列表不共享
  │     "structured_response",
  │     "skills_metadata",   ← 子 agent 加载自己的 skills
  │     "memory_contents",   ← 子 agent 加载自己的 memory
  │   }
  │
  ├─ subagent_state["messages"] = [HumanMessage(content=description)]
  │   └─ 子 agent 的「用户消息」就是 task() 的 description 参数
  │
  ├─ subagent.invoke(subagent_state)  或  await subagent.ainvoke(...)
  │
  └─ 结果处理：
      取 result["messages"][-1]（子 agent 最后一条消息）
      → 包装为 ToolMessage（主 agent 的工具返回值）
      → state_update 含子 agent 返回的非隔离状态键
      → 返回 Command(update={...})
```

### 3.3 状态继承与隔离的精确范围

| 状态键 | 子 agent 是否继承 | 备注 |
|--------|-----------------|------|
| `files` (FilesystemState) | ✅ 继承 | 子 agent 看到父 agent 的文件系统 |
| `messages` | ❌ 隔离 | 子 agent 只有一条消息：task description |
| `todos` | ❌ 隔离 | 子 agent 有独立任务列表 |
| `skills_metadata` | ❌ 隔离 | 子 agent 自己通过 SkillsMiddleware 加载 |
| `memory_contents` | ❌ 隔离 | 子 agent 自己通过 MemoryMiddleware 加载 |
| 用户自定义状态键 | ✅ 继承（排除上述之外）| 可通过子 agent 向主 agent 传递状态 |

---

## 四、`SummarizationMiddleware` — 上下文压缩机制

### 4.1 触发条件（双模式）

```
compute_summarization_defaults(model):

  模型有 profile.max_input_tokens（如 Claude）:
    trigger = ("fraction", 0.85)   → 占用 85% 上下文时触发
    keep    = ("fraction", 0.10)   → 保留最近 10% 的 token 对应消息

  模型无 profile（未知模型）:
    trigger = ("tokens", 170000)   → 超过 17 万 token 时触发
    keep    = ("messages", 6)      → 保留最后 6 条消息
```

**兜底机制**：即使未到阈值，若 LLM 返回 `ContextOverflowError`，立即回退到摘要路径。

### 4.2 完整压缩流程

```
wrap_model_call(request, handler):

  Step 0: 应用历史摘要事件（恢复 effective_messages）
    _summarization_event 在 private state 中存储：
    → cutoff_index: 历史截断点
    → summary_message: 摘要 HumanMessage
    effective_messages = [summary_message] + messages[cutoff_index:]

  Step 1: 工具参数截断（轻量优化）
    对 write_file / edit_file 的旧调用参数截断到 2000 字符
    触发：与摘要阈值相同（85% 或 17 万 token）
    保留：最近 10% 或最后 20 条消息

  Step 2: 判断是否需要摘要
    → 不需要：直接 handler(request) 继续
    → ContextOverflowError → 强制进入摘要

  Step 3: 执行摘要
    partition_messages(cutoff_index):
      to_summarize = effective_messages[:cutoff_index]
      preserved    = effective_messages[cutoff_index:]

    异步模式（awrap_model_call）：
      offload + summary 并发执行（asyncio.gather）
      ← 节省一次顺序等待

    offload_to_backend(to_summarize):
      → 写入 /conversation_history/{thread_id}.md
      → 追加格式（每次摘要为新的 markdown section）
      → 过滤掉之前的摘要消息（避免重复存储）

    create_summary(to_summarize):
      → LLM 调用（使用同一个 model）生成摘要文本

  Step 4: 构建新消息列表
    HumanMessage(content="...summary... see {file_path} for full history")
    + preserved_messages

  Step 5: 返回 ExtendedModelResponse
    → model_response: 以压缩后消息调用 handler 的结果
    → command: Command(update={"_summarization_event": new_event})
    ← 状态更新写入 LangGraph state（私有键）
```

### 4.3 摘要历史文件格式

路径：`/conversation_history/{thread_id}.md`，每次摘要**追加**一个 section：

```
## Summarized at 2026-04-07T10:00:00+00:00
Human: <初始消息>
AI: <回复>
...（原始对话内容）...

## Summarized at 2026-04-07T10:30:00+00:00
...（下一次摘要的内容）...
```

**链式摘要**：多次摘要事件叠加时，只追加「原始消息」，不重复存储之前的摘要消息（通过 `_filter_summary_messages` 过滤）。

---

## 五、`MemoryMiddleware` — AGENTS.md 记忆注入

### 5.1 加载时机：`before_agent` 钩子

```
before_agent(state, runtime, config):
  if "memory_contents" already in state:
    return None   ← 幂等，只加载一次

  backend.download_files(sources)  ← 批量并行下载
  → "file_not_found" → 静默跳过（文件不存在不是错误）
  → 其他错误 → 抛出 ValueError

  return MemoryStateUpdate(memory_contents={path: content, ...})
  ← 存入 private state（不出现在 checkpointer 中）
```

### 5.2 注入时机：每次 LLM 调用前

```
wrap_model_call → modify_request:
  读取 state["memory_contents"]
  → 格式化：每个 source 路径 + 内容
  → 包裹在 <agent_memory> 标签中
  → append_to_system_message
```

### 5.3 记忆内容的生命周期

```
加载时机：before_agent（每个 thread 的第一次 invoke）
更新方式：agent 主动调用 edit_file 修改 AGENTS.md 文件
失效方式：只要文件变化，下次新 thread 启动时会重新加载
           ← 同一 thread 内的更改在当前 thread 不可见！
```

**安全警告**：`MEMORY_SYSTEM_PROMPT` 明确禁止存储 API keys、密码等敏感信息。

---

## 六、`PatchToolCallsMiddleware` — 悬空工具调用修复

### 6.1 问题背景

Anthropic API 要求：每个 `AIMessage.tool_calls` 中的 tool call，都必须在后续消息中有一个对应的 `ToolMessage`。否则 API 会报错。

**触发场景**：人类用户在 tool call 执行期间发送了新消息（中断了 tool call 的执行），导致 tool call 没有对应的结果消息。

### 6.2 修复机制

```python
before_agent(state, runtime):
  for i, msg in enumerate(messages):
    if isinstance(msg, AIMessage) and msg.tool_calls:
      for tool_call in msg.tool_calls:
        # 在后续消息中查找对应的 ToolMessage
        corresponding = next(
          (m for m in messages[i:] if m.type == "tool" and m.tool_call_id == tool_call["id"]),
          None
        )
        if corresponding is None:
          # 注入合成的 ToolMessage
          patched_messages.append(ToolMessage(
            content=f"Tool call {tool_call['name']} with id {tool_call['id']} was cancelled "
                    "- another message came in before it could be completed.",
            name=tool_call["name"],
            tool_call_id=tool_call["id"],
          ))

  return {"messages": Overwrite(patched_messages)}  ← 完全覆盖消息列表
```

关键：返回 `Overwrite(patched_messages)` 而非追加，这会**完全替换**消息列表，保证 ID 一致性。

### 6.3 长期 vs 临时设计

这不是临时 hack——它是长期设计：
- 人类中断（human-in-the-loop）是 deepagents 的核心特性
- `HumanInTheLoopMiddleware` 需要配合此中间件才能安全使用
- Anthropic 的协议约束要求 tool_call 和 tool_result 成对出现

---

## 七、中间件链的执行模型

### 7.1 Pipeline 模式（非责任链）

```
middleware[0].wrap_model_call(request, handler=lambda req:
  middleware[1].wrap_model_call(req, handler=lambda req:
    middleware[2].wrap_model_call(req, handler=lambda req:
      ...
      middleware[N].wrap_model_call(req, handler=actual_llm_call)
    )
  )
)
```

每个中间件收到的 `handler` 是**下一个中间件的 wrap_model_call**，最内层 handler 是真实的 LLM 调用。

### 7.2 三个钩子时机

| 钩子 | 调用时机 | 典型用途 |
|------|---------|---------|
| `before_agent` | agent 启动时（整个 invoke 只调用一次） | 加载 Memory、修复悬空 tool calls |
| `wrap_model_call` | 每次 LLM 调用前/后 | 注入 system prompt、过滤工具、修改消息 |
| `wrap_tool_call` | 每次工具执行前/后 | 大结果驱逐、工具参数验证 |

### 7.3 中断机制

中间件**不能中断**调用链（没有责任链的 abort 语义），但可以：
- 返回 `ExtendedModelResponse`：携带 state 更新（用于 Summarization）
- 不调用 `handler`：返回自定义响应（理论上可行，但内置中间件不这样做）

---

## 八、`_utils.py` — 系统提示的追加策略

```python
def append_to_system_message(system_message, text):
  """追加文本块到 SystemMessage 的 content_blocks 列表。"""
  new_content = list(system_message.content_blocks) if system_message else []
  if new_content:
    text = f"\n\n{text}"  # 非首个块时加双换行分隔
  new_content.append({"type": "text", "text": text})
  return SystemMessage(content_blocks=new_content)
```

系统提示使用 **content_blocks 列表**（非字符串拼接），这支持：
- 多媒体内容块（图片等）
- 更精细的 token 计数
- Anthropic prompt caching prefix 的精确控制

---

## 九、一次请求的完整时序图

```
用户调用 agent.invoke({"messages": [HumanMessage("帮我写一个排序算法")]})

  ① before_agent 阶段（顺序执行）
    PatchToolCallsMiddleware.before_agent → 检查悬空 tool calls（无则跳过）
    MemoryMiddleware.before_agent         → 加载 AGENTS.md 到 state（首次）

  ② LLM 调用循环 开始 ==========

    第 1 轮（主 agent 思考）
    ┌─ wrap_model_call 链（由外到内）─────────────────────────────────┐
    │  TodoListMiddleware         → 注入 write_todos 工具             │
    │  SkillsMiddleware           → 注入 skills 工具列表              │
    │  FilesystemMiddleware       → 注入 fs 工具 + 驱逐大消息         │
    │  SubAgentMiddleware         → 注入 task 工具 + 说明             │
    │  SummarizationMiddleware    → 检查 token 使用，必要时压缩       │
    │  PatchToolCallsMiddleware   → （wrap_model_call 无操作）        │
    │  AnthropicPromptCachingMW   → 注入 cache_control 标记          │
    │  MemoryMiddleware           → 从 state 读取 memory 注入提示    │
    └─────────────────────────────────────────────────────────────────┘
    → 调用 Anthropic API（claude-sonnet-4-6）
    ← 返回 AIMessage(tool_calls=[task(description=..., subagent_type="general-purpose")])

    第 1.5 轮（tool call 执行）
    ┌─ wrap_tool_call 链 ─────────────────────────────────────────────┐
    │  FilesystemMiddleware.wrap_tool_call → 大结果驱逐               │
    └─────────────────────────────────────────────────────────────────┘
    → 执行 task 工具：
        subagent = general_purpose_compiled_graph
        subagent.invoke({..., "messages": [HumanMessage("帮我写一个排序算法")]})
        ↓
        [子 agent 内部完整循环：read_file / write_file / ...]
        ↓
        返回 result["messages"][-1].text
    → 包装为 ToolMessage，写入 state

    第 2 轮（主 agent 综合子 agent 结果）
    ... 同第 1 轮 wrap_model_call 链 ...
    → 调用 LLM 生成最终文本回复
    ← 返回 AIMessage(content="这是一个快速排序实现...")

  ③ 返回最终 state（messages 列表 + 更新的 files 等）
```

---

## 十、Layer 2 总结：关键设计决策

```
✅ 中间件顺序固定        顺序本身是设计的一部分，不可随意调整
✅ 编译成本前置          子 agent 图在启动时全部编译，运行时零开销
✅ 状态隔离精准          子 agent 继承文件状态，隔离任务/记忆/技能
✅ 驱逐机制透明          大文件自动转存，LLM 收到引用而非原始内容
✅ 摘要不修改 state      摘要事件通过私有 state 追踪，不影响 checkpointer
✅ Caching 边界精确      Memory/HumanInTheLoop 在 Caching 之后，防止 cache 失效
✅ 悬空工具调用容错      PatchToolCalls 在 before_agent 时修复，每轮 invoke 生效
✅ 异步路径并发优化      摘要的 offload + LLM 生成并发（asyncio.gather）
```

---

## 十一、遗留问题（转交 Layer 3）

- [ ] `AsyncSubAgentMiddleware` 的异步调度机制：如何并行执行多个子 agent？
- [ ] `SkillsMiddleware` 的技能文件格式：SKILL.md 的结构和加载策略
- [ ] `CompositeBackend` 的读写路由策略：如何按路径前缀分发？
- [ ] `backends/state.py` — StateBackend 存储在 LangGraph state 的哪个 key？
- [ ] `SummarizationToolMiddleware` — `compact_conversation` 工具的触发阈值（50% of trigger）
- [ ] `AnthropicPromptCachingMiddleware` 的具体实现：cache_control 如何注入？

---

*上一步：[Layer 1 - 契约层分析](./layer1-contract-analysis.md)*  
*下一步：[Layer 3 - 扩展机制分析](./layer3-extension-analysis.md)（待创建）*
