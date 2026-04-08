# 五、ACP（Agent Client Protocol）集成

本文分析 `deepagents/libs/acp` 中 Deep Agents 与 ACP 的桥接实现：服务端类、会话与配置、多模态输入转换、LangGraph 流式输出到 ACP 消息的映射、人机协同（HITL）与检查点行为。

## 概述

```
+------------------+     ACP JSON-RPC      +------------------------+
|  ACP Client      | <-------------------> |  acp.run_agent()       |
|  (e.g. Zed)      |                       |  + AgentServerACP      |
+------------------+                       |    (extends ACP Agent) |
        |                                  +-----------+------------+
        | session_update /                          |
        | request_permission                        | LangGraph
        v                                           v
+---------------+    thread_id=session_id    +----------------------+
| Client conn   | -------------------------> | CompiledStateGraph   |
| _conn         |                            | .astream(...)         |
+---------------+                            +----------+-----------+
                                                        |
                                             +----------v-----------+
                                             | MemorySaver (可选)   |
                                             | thread 级持久        |
                                             +----------------------+
```

**核心关联**：同一 `session_id` 既作为 ACP 会话 ID，也作为 LangGraph `configurable.thread_id`，从而把对话与检查点绑定到同一线程。

## 目录结构

| 路径 | 作用 |
|------|------|
| `deepagents_acp/server.py` | ACP `Agent` 实现：`AgentServerACP`，核心协议方法 |
| `deepagents_acp/utils.py` | ACP 内容块 → LangChain 多模态内容；命令安全与展示 |
| `examples/`、`run_demo_agent.sh` | Zed 等客户端演示入口 |
| `tests/` | 服务端、权限、危险模式等单元测试 |

---

## 5.1 ACP Server 实现（`AgentServerACP`）

类 `AgentServerACP` 继承 `acp.Agent`，实现 `initialize`、`new_session`、`set_session_mode`、`set_config_option`、`cancel`、`prompt` 等协议方法。

### 构造与初始化能力

```python
class AgentServerACP(ACPAgent):
    def __init__(
        self,
        agent: CompiledStateGraph | Callable[[AgentSessionContext], CompiledStateGraph],
        *,
        modes: SessionModeState | None = None,
        models: list[dict[str, str]] | None = None,
    ) -> None:
```

- **工厂 vs 预编译图**：接受 `CompiledStateGraph`（固定）或 `Callable`（工厂模式，允许传入 `modes` / `models` 做会话级配置）
- **初始化能力**：`initialize` 声明 `PromptCapabilities(image=True)`，表示支持图像类 prompt

---

## 5.2 会话管理与消息格式转换

### 新建会话（`new_session`）

```
new_session() 调用：
├── 生成 session_id = uuid4().hex
├── 记录 cwd（当前工作目录）
├── 初始化 mode/model 字典项
└── 可选返回 SessionConfigOption（mode / model 下拉列表）
```

### 用户 Prompt：ACP → LangChain 内容块

```
ACP ContentBlock                   LangChain content_blocks
──────────────                     ─────────────────────────
TextContentBlock      ──────────►  {"type": "text", "text": ...}
ImageContentBlock     ──────────►  {"type": "image_url", ...}
AudioContentBlock     ──────────►  NotImplementedError（未支持）
ResourceContentBlock  ──────────►  纯文本占位（URI + 说明）
EmbeddedResourceBlock ──────────►  data URI 文本说明
```

音频当前显式抛出 `NotImplementedError`，资源类块转为纯文本，避免把二进制直接喂入模型。

---

## 5.3 协议消息：流式输出映射

### LangGraph 流 → ACP 消息映射表

| LangGraph 流事件 | ACP 消息类型 | 处理逻辑 |
|-----------------|-------------|---------|
| `stream_mode == "messages"`，字符串 chunk | 助手文本 | `update_agent_message(text_block(text))` |
| `tool_call_chunks`（LLM 输出） | `ToolCallStart` | 按 index 累积 args_str，JSON 解析成功后触发 |
| `ToolMessage`（工具结果） | `ToolCallUpdate` 完成态 | `update_tool_call(..., status="completed")` |
| `stream_mode == "updates"`, `__interrupt__` | HITL 审批 | 进入 `_handle_interrupts` |
| `tools` 节点更新含 `todos` | `AgentPlanUpdate` | `_handle_todo_update` |

### ToolCallStart 工具种类映射

```python
kind_map = {
    "read_file":    "read",
    "write_file":   "edit",
    "edit_file":    "edit",
    "ls":           "read",
    "grep":         "search",
    "glob":         "search",
    # 默认 "other"
}
```

- **`edit_file`**：具备 old/new 字符串时使用 `start_edit_tool_call` 与 `tool_diff_content`，在客户端显示 diff
- **`execute` 工具**：结果经 `format_execute_result` 格式化为 Markdown 风格片段

### 核心流式循环

```python
while current_state is None or current_state.interrupts:
    async for stream_chunk in agent.astream(
        Command(resume={"decisions": user_decisions})
        if user_decisions
        else {"messages": [{"role": "user", "content": content_blocks}]},
        config=config,
        stream_mode=["messages", "updates"],
        subgraphs=True,
    ):
        # 处理各类流事件...
```

- `subgraphs=True`：三元组流格式（namespace, mode, data）
- 顶层 namespace 非空时跳过部分文本日志，避免子图重复刷屏

---

## 5.4 计划（Plan）与 `write_todos`

- 工具参数解析后若工具名为 `write_todos`，立即把 todos 转成 `AgentPlanUpdate` 发给客户端
- HITL 中对计划的批准/拒绝会与 `_session_plans`、`_clear_plan` 协同
- 拒绝计划时可附带反馈文案引导模型重规划

---

## 5.5 人机协同（HITL）与 ACP 限制

### 标准 HITL 流程

```
interrupt.value.action_requests
        │
        ├─ 为每个动作构造 PermissionOption
        │   ├── approve
        │   ├── reject
        │   └── approve_always（shell 工具特有）
        │
        └─ request_permission() → 等待用户决策
                │
                └─ Command(resume={"decisions": user_decisions})
```

### "始终允许"机制

- `extract_command_types` 解析命令签名，`_allowed_command_types` 存 `(tool_name, cmd_type)`
- `contains_dangerous_patterns` 防止在已授权命令中注入重定向、反引号等绕过

### ACP 不支持自由形式 interrupt

```
if isinstance(updates, dict) and "__interrupt__" in updates:
    # 非结构化 interrupt
    raise RequestError(
        -32600,
        "ACP limitation: this agent raised a free-form "
        "LangGraph interrupt(), which ACP cannot display."
    )
```

---

## 5.6 Checkpoint / 状态恢复机制

### MemorySaver 自动注入

- 首次 `prompt` 时若图无 `checkpointer`，自动注入 **`MemorySaver()`**
- 与 `thread_id=session_id` 配合实现**同会话多轮**的状态持久化

### 模式/模型切换行为

- `set_config_option` / `set_session_mode` 会 `_reset_agent` 重建图实例
- 当前设计侧重**会话内多轮 prompt**，而非跨模式长期共享同一张图实例

---

## 小结

| 能力 | 支持状态 |
|------|---------|
| 文本输入/输出 | ✅ 完整支持 |
| 图像输入 | ✅ 支持 |
| 音频输入 | ❌ NotImplementedError |
| 工具调用可视化 | ✅ ToolCallStart + ToolCallUpdate |
| Diff 展示（edit_file） | ✅ tool_diff_content |
| 计划可视化（todos） | ✅ AgentPlanUpdate |
| HITL 审批 | ✅ 结构化 PermissionOption |
| 自由形式 interrupt | ❌ RequestError（ACP 协议限制） |
| 多轮会话（checkpointer） | ✅ 自动注入 MemorySaver |

ACP 层完整实现了**多模态入参、流式文本、工具调用生命周期、计划可视化、HITL 权限**。与 LangGraph 的衔接点是 `astream` + `Command(resume=...)` + `thread_id`。
