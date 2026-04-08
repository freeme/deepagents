# 二、Middleware 中间件系统

## 2.1 中间件架构总览

### 2.1.1 设计背景与两条工具路径

`middleware/__init__.py` 说明：LLM 看到的工具来自两条路径——

**(1) SDK 中间件**（本包）：在每次请求前拦截、动态过滤工具、注入系统提示、变换消息、维护跨轮状态；

**(2) 调用方传入的 `tools`**：由 `create_deep_agent(..., tools=...)` 合并进最终工具集。

普通 `tools=[]` 中的函数只能被模型调用，无法在 LLM 调用**之前**改提示或工具列表。

### 2.1.2 `AgentMiddleware` 与典型钩子语义

各中间件统一继承 `langchain.agents.middleware.types` 中的 `AgentMiddleware`，并按需实现：

| 钩子 | 典型语义（DeepAgents 中的用法） |
|------|--------------------------------|
| `wrap_model_call` / `awrap_model_call` | 在调用底层 LLM **之前**修改 `ModelRequest`（系统消息、工具列表），或包装 `handler` |
| `wrap_tool_call` / `awrap_tool_call` | 在工具执行**之后**处理 `ToolMessage`（如大结果落盘） |
| `before_agent` / `abefore_agent` | 图执行开始时加载状态（如 `memory_contents`、`skills_metadata`） |
| `modify_request` | 部分中间件抽取的纯函数式请求变换（如 Memory/Skills） |

### 2.1.3 `create_deep_agent` 中的组合顺序（ASCII）

主代理中间件栈（按 `append` 顺序）：

```
[TodoListMiddleware]
    -> [SkillsMiddleware]?          (若配置了 skills)
    -> [FilesystemMiddleware]
    -> [SubAgentMiddleware]
    -> [SummarizationMiddleware]    (create_summarization_middleware)
    -> [PatchToolCallsMiddleware]
    -> [AsyncSubAgentMiddleware]?   (若有远程 AsyncSubAgent)
    -> ... 用户传入的 middleware ...
    -> [AnthropicPromptCachingMiddleware]  (unsupported_model_behavior="ignore")
    -> [MemoryMiddleware]?           (若配置了 memory)
    -> [HumanInTheLoopMiddleware]?  (若配置了 interrupt_on)
```

**职责边界简述**：

| 中间件 | 核心职责 |
|--------|---------|
| TodoList | 规划与待办工具 |
| Skills | 会话开始前加载技能元数据，每轮通过系统提示注入（渐进披露） |
| Filesystem | 文件/可选执行工具 + 大工具结果/大消息外溢 |
| SubAgent | `task` 同步子代理 |
| Summarization | 自动摘要 + 历史落盘 |
| PatchToolCalls | 修复「悬空」tool call（无对应 ToolMessage） |
| AsyncSubAgent | 远程 LangGraph 后台任务工具集 |
| AnthropicPromptCaching | 提示缓存标记（放末尾使前缀稳定） |
| Memory | `AGENTS.md` 注入（在缓存之后） |
| HumanInTheLoop | 工具执行前可中断等待人工 |

---

## 2.2 FilesystemMiddleware — 文件系统工具

### 2.2.1 状态与 Backend

- 状态模式：`FilesystemState` 含 `files`，通过 `_file_data_reducer` 合并并支持 `None` 表示删除
- `FilesystemMiddleware._get_backend(runtime)`：支持 `BackendProtocol` 实例；若传入可调用 factory 会发 **DeprecationWarning**

### 2.2.2 工具参数设计（Pydantic Schema）

| 工具 | 核心参数 |
|------|----------|
| `ls` | `path`：必须为绝对路径 |
| `read_file` | `file_path`，`offset`（默认 0），`limit`（默认 100 行） |
| `write_file` | `file_path`，`content` |
| `edit_file` | `file_path`，`old_string`，`new_string`，`replace_all`（默认 false） |
| `glob` | `pattern`，`path`（默认 `"/"`） |
| `grep` | `pattern`（**字面量**，非正则），`path`（可选），`glob`（可选），`output_mode` |
| `execute` | `command`，`timeout`（可选；0 表示部分 backend 无超时） |

### 2.2.3 错误处理模式

- 路径校验：先 `validate_path`，失败返回 `f"Error: {e}"`
- Backend 返回：检查 `*.error` 字段
- `glob`：同步路径用线程池 + `GLOB_TIMEOUT`（20s），超时返回明确错误字符串
- 不支持 `SandboxBackendProtocol` 时，`execute` 工具不会暴露

### 2.2.4 行号格式化

`read_file` 使用 `format_content_with_line_numbers`（cat-n 风格，固定宽度 6）：

```python
def format_content_with_line_numbers(content: str | list[str], start_line: int = 1) -> str:
    """
    超长行（> MAX_LINE_LENGTH=5000）按块切分，续行用 "5.1", "5.2" 等形式
    """
    if len(line) <= MAX_LINE_LENGTH:
        result_lines.append(f"{line_num:{LINE_NUMBER_WIDTH}d}\t{line}")
    else:
        # 续行：continuation_marker = f"{line_num}.{chunk_idx}"
```

### 2.2.5 大结果外溢（防止上下文爆满）

```
wrap_tool_call：
  若工具结果超过近似 token 阈值
  → 写入 /large_tool_results/{sanitized_tool_call_id}
  → 消息替换为预览 + 路径说明

wrap_model_call：
  对超大 HumanMessage（最后一条、未打标）
  → 写入 /conversation_history/{uuid}.md
  → 在 state 打 lc_evicted_to 标记
  → 已打标消息在请求里替换为预览
```

---

## 2.3 SubAgentMiddleware — 同步子代理

### 2.3.1 `SubAgent` / `CompiledSubAgent` 字段

**`SubAgent`（TypedDict）**：

- 必填：`name`，`description`，`system_prompt`
- 可选：`tools`，`model`（字符串或 `BaseChatModel`），`middleware`，`interrupt_on`，`skills`

**`CompiledSubAgent`**：

- `name`，`description`，`runnable`（状态里必须含 `messages`）

### 2.3.2 `_build_task_tool` 流程

```
1. 构建 subagent_graphs: dict[name, Runnable] 与工具描述
2. task(description, subagent_type, runtime):
   - 校验类型
   - 从 runtime.state 剔除 _EXCLUDED_STATE_KEYS
   - 设 messages=[HumanMessage(content=description)]
3. invoke / ainvoke
4. _return_command_with_state_update:
   - 从结果取 messages[-1].text 作为唯一返回文本
   - 合并 state_update（排除 _EXCLUDED_STATE_KEYS）
   - 用 Command 更新并附 ToolMessage
```

```python
def _return_command_with_state_update(result: dict, tool_call_id: str) -> Command:
    state_update = {k: v for k, v in result.items() if k not in _EXCLUDED_STATE_KEYS}
    message_text = result["messages"][-1].text.rstrip() if result["messages"][-1].text else ""
    return Command(
        update={
            **state_update,
            "messages": [ToolMessage(message_text, tool_call_id=tool_call_id)],
        }
    )
```

### 2.3.3 `_EXCLUDED_STATE_KEYS`

```python
_EXCLUDED_STATE_KEYS = {
    "messages",
    "todos",
    "structured_response",
    "skills_metadata",
    "memory_contents"
}
```

用于：调用子代理时不传入父级这些键；返回时也不把这些键合并回主代理，避免 todos/结构化输出/技能元数据/记忆泄漏。

### 2.3.4 默认 general-purpose 子代理

`GENERAL_PURPOSE_SUBAGENT` 仅含 `name` / `description` / `system_prompt`；`create_deep_agent` 为其补全 `model`、`tools`、`middleware`（含 Todo、Filesystem、Summarization、Patch、可选 Skills、**最后** AnthropicPromptCachingMiddleware）。

---

## 2.4 AsyncSubAgentMiddleware — 异步/远程子代理

### 2.4.1 `AsyncSubAgent` TypedDict

- 必填：`name`，`description`，`graph_id`
- 可选：`url`（省略则 SDK 默认），`headers`（`_resolve_headers` 默认补 `x-auth-scheme: langsmith`）

### 2.4.2 工具集与状态

- 状态：`AsyncSubAgentState.async_tasks`，reducer 合并更新
- `_ClientCache`：按 `(url, headers)` 缓存 sync/async client 实例
- **工具集**：

| 工具 | 功能 |
|------|------|
| `start_async_task` | 建 thread + runs.create |
| `check_async_task` | runs.get + 成功时拉 thread values 取最后一条消息 |
| `update_async_task` | 同 thread 新 run，`multitask_strategy="interrupt"` |
| `cancel_async_task` | 取消运行中的 task |
| `list_async_tasks` | 可选拉取 live status |

---

## 2.5 SummarizationMiddleware — 上下文压缩

### 2.5.1 自动触发

- `wrap_model_call`：先 `_get_effective_messages`（应用已有 `_summarization_event`），再可选 **truncate_args**（旧消息里 `write_file`/`edit_file` 的大参数），再判是否摘要
- 若抛出 `ContextOverflowError` 则走摘要兜底

### 2.5.2 历史落盘

- 路径：`{history_path_prefix}/{thread_id}.md`（默认前缀 `/conversation_history`）
- 每次摘要追加一节 `## Summarized at {ISO}` + `get_buffer_string`
- 链式摘要时过滤 `lc_source=='summarization'` 的 HumanMessage，避免重复卸载

### 2.5.3 摘要插入位置

**不是**单独的 `SystemMessage`：摘要作为 **`HumanMessage`** 插入，且 `additional_kwargs["lc_source"] = "summarization"`。

```python
def _build_new_messages_with_path(self, summary: str, file_path: str | None) -> list[AnyMessage]:
    return [
        HumanMessage(
            content=content,
            additional_kwargs={"lc_source": "summarization"},
        )
    ]
```

### 2.5.4 `SummarizationToolMiddleware`

- 提供 `compact_conversation` 工具；与自动摘要共用 `_summarization_event`
- 手动压缩有 `_is_eligible_for_compaction`（约达到自动触发阈值的 **50%** 才允许）

---

## 2.6 MemoryMiddleware — 持久记忆

### 2.6.1 `AGENTS.md` 加载

- `before_agent` / `abefore_agent`：若 state 尚无 `memory_contents`，则 `backend.download_files` 按 `sources` 顺序加载
- `file_not_found` 跳过，其它错误抛 `ValueError`

### 2.6.2 路径与展示

多源按 `sources` 顺序，仅包含有内容的 path：

```python
sections = [f"{path}\n{contents[path]}" for path in self.sources if contents.get(path)]
memory_body = "\n\n".join(sections)
return MEMORY_SYSTEM_PROMPT.format(agent_memory=memory_body)
```

### 2.6.3 与系统提示集成

- `modify_request` → `append_to_system_message` 注入 `<agent_memory>` 与长段 `memory_guidelines`（学习反馈、何时更新记忆等）
- `wrap_model_call` 调用 `modify_request` 后转发 `handler`

---

## 2.7 SkillsMiddleware — Agent Skills

### 2.7.1 Progressive Disclosure（渐进式披露）

```
before_agent：
  扫描所有 sources 目录 → 加载 SkillMetadata（只读 frontmatter）
  → 写入 state（私有字段 skills_metadata）

wrap_model_call：
  → 注入技能名称 + 描述 + path（读全文用 read_file 工具）
  → 不注入 SKILL.md 全文（按需读取）
```

### 2.7.2 `SkillMetadata` 与 YAML frontmatter

字段：`path`, `name`, `description`, `license`, `compatibility`, `metadata`, `allowed_tools`

解析：`^---\s*\n(.*?)\n---\s*\n` 取 frontmatter，`yaml.safe_load`；最大文件 `MAX_SKILL_FILE_SIZE`（10MB）

### 2.7.3 名称校验 `_validate_skill_name`

- 长度 ≤64；小写字母/数字/单 `-`；不以 `-` 首尾、无 `--`
- **必须与父目录名一致**
- 违规时发 warning 仍可能加载（向后兼容）

### 2.7.4 多 source 叠加

- 按 `sources` 顺序扫描，字典 `all_skills[name] = skill`，**后者覆盖同名**（last wins）

---

## 2.8 PatchToolCallsMiddleware — 工具调用修正

- `before_agent`：遍历每条 `AIMessage.tool_calls`，若在后续消息中找不到同 `tool_call_id` 的 `ToolMessage`，则**追加**一条合成 `ToolMessage`，说明该调用被取消

```python
for i, msg in enumerate(messages):
    patched_messages.append(msg)
    if isinstance(msg, AIMessage) and msg.tool_calls:
        for tool_call in msg.tool_calls:
            corresponding_tool_msg = next(
                (msg for msg in messages[i:] if msg.type == "tool" and msg.tool_call_id == tool_call["id"]),
                None,
            )
            if corresponding_tool_msg is None:
                tool_msg = (
                    f"Tool call {tool_call['name']} with id {tool_call['id']} was "
                    "cancelled - another message came in before it could be completed."
                )
                patched_messages.append(
                    ToolMessage(content=tool_msg, name=tool_call["name"], tool_call_id=tool_call["id"])
                )

return {"messages": Overwrite(patched_messages)}
```

---

## 2.9 AnthropicPromptCachingMiddleware

### 在栈中的位置

放在 **Skills / Filesystem / SubAgent / Summarization / Patch（及用户 middleware）之后**，**Memory 与 HITL 之前**：

```python
# graph.py 注释：
# Caching + memory after all other middleware so memory updates don't
# invalidate the Anthropic prompt cache prefix.
deepagent_middleware.append(AnthropicPromptCachingMiddleware(unsupported_model_behavior="ignore"))
```

意图：先固定大块「行为说明 + 工具说明」的前缀以便缓存；记忆等易变内容放在缓存段之后，减少对缓存键的冲击。

### `unsupported_model_behavior="ignore"`

非 Anthropic 模型不支持该缓存语义时，**忽略缓存扩展**而非报错，使同一套 `create_deep_agent` 在多厂商模型上可用。

---

## 2.10 `_utils.py` — 系统消息拼接

`append_to_system_message`：在已有 `SystemMessage` 的 `content_blocks` 末尾追加文本块；若已有内容则在前面加 `\n\n` 分隔。

```python
def append_to_system_message(system_message: SystemMessage | None, text: str) -> SystemMessage:
    new_content: list[ContentBlock] = list(system_message.content_blocks) if system_message else []
    if new_content:
        text = f"\n\n{text}"
    new_content.append({"type": "text", "text": text})
    return SystemMessage(content_blocks=new_content)
```

---

## 2.11 关键设计决策总结

1. **中间件 vs 工具**：需要每轮改请求、跨轮状态、动态工具集时必须用 `AgentMiddleware`

2. **Filesystem**：Backend 协议统一存储与执行能力探测；大结果外溢路径约定 `/large_tool_results/`、`/conversation_history/`

3. **SubAgent**：状态键显式排除，避免子代理污染主图；`task` 只返回最终一条文本

4. **Summarization**：摘要以 **HumanMessage** 插入并带 `lc_source`；历史 markdown 按线程追加，避免重复卸载

5. **Skills**：元数据进 state、全文靠 `read_file` 工具按需读取，多源 **last wins**

6. **PatchToolCalls**：用合成 ToolMessage 满足 API 对 tool_call 成对约束（解决模型输出不完整时的错误）

7. **Anthropic 缓存**：置后 + `ignore` 非 Anthropic；Memory 再后置以降低缓存失效风险
