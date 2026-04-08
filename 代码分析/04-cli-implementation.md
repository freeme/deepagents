# 四、CLI 工具实现

本文基于 `/Volumes/Data/agent-frameworks/deepagents/libs/cli/deepagents_cli/` 下的源码，对 DeepAgents CLI 的结构与关键设计作系统梳理。

## 1. CLI 整体架构（ASCII）

```
                         ┌─────────────────────────────────────────┐
                         │           main.py :: cli_main()          │
                         │  parse_args → 分支: ACP / skills /       │
                         │  threads / update / -n / 交互式 TUI       │
                         └───────────────┬─────────────────────────┘
                                         │
           ┌─────────────────────────────┼─────────────────────────────┐
           │                             │                             │
           ▼                             ▼                             ▼
   ┌───────────────┐            ┌───────────────┐            ┌─────────────────┐
   │  --acp        │            │  -n / pipe    │            │  默认：交互式    │
   │  _run_acp_    │            │  non_         │            │  run_textual_   │
   │  cli_async    │            │  interactive  │            │  cli_async →    │
   │  进程内 agent  │            │  + 远程 Server │            │  run_textual_   │
   └───────────────┘            └───────┬───────┘            │  app (Textual)  │
                                      │                    └────────┬────────┘
                                      │                            │
                                      │                    LangGraph Server
                                      │                    子进程 + RemoteAgent
                                      ▼                            ▼
                         ┌────────────────────────────────────────────┐
                         │  agent.create_cli_agent / create_deep_agent  │
                         │  CompositeBackend + Middleware + interrupt_on │
                         └────────────────────────────────────────────┘
```

**要点**：
- 默认交互路径在 **Textual TUI**（`app.py`）中运行，**LangGraph 服务在子进程**中启动，TUI 通过 SDK 客户端连接
- **`-n` / 管道 stdin** 走 `non_interactive.py`，同样依赖远程图运行
- **`--acp`** 则在当前进程内直接 `create_cli_agent` 并跑 ACP 服务，不启动 Textual

---

## 4.1 CLI 入口与应用框架（`main.py` + `app.py`）

### Textual TUI 应用结构

```python
async def run_textual_app(
    *,
    agent: Any = None,
    assistant_id: str | None = None,
    backend: CompositeBackend | None = None,
    auto_approve: bool = False,
    ...
) -> AppResult:
    """
    当 server_kwargs 提供且 agent=None 时：
    - 应用立即启动，显示 "Connecting..." banner
    - 后台启动服务器（LangGraph dev 子进程）
    - 服务器清理在 app 退出后自动处理
    """
    app = DeepAgentsApp(...)
    try:
        await app.run_async()
    finally:
        if app._server_proc is not None:
            app._server_proc.stop()
    return AppResult(return_code=..., thread_id=..., session_stats=..., update_available=...)
```

### 交互式与非交互式（headless）切换

| 模式 | 触发条件 | 入口 |
|------|---------|------|
| 交互式 TUI | 默认（无 `-n`，无管道） | `run_textual_cli_async` → Textual App |
| 非交互 `-n TEXT` | 显式传入单次消息 | `run_non_interactive(...)` |
| 管道 stdin | 检测到 stdin 为管道 | `apply_stdin_pipe()` 合并输入后同上 |
| ACP 模式 | `--acp` 参数 | `_run_acp_cli_async`（进程内） |

**重要**：交互模式下 **`auto_approve`（`-y`）不传给服务端**。服务端始终配置完整 HITL interrupt，客户端用 `session_state.auto_approve` 在 `textual_adapter` 侧自动批准，从而支持会话中切换批准策略。

---

## 4.2 Agent 生命周期（`agent.py`）

### `create_cli_agent` 组装逻辑

```
1. 技能源（Skills sources）
   内置 → 用户 → 项目 → 实验性 Claude 目录等（多路径合并）

2. 记忆（Memory）
   MemoryMiddleware + FilesystemBackend
   来源包含用户与项目的 AGENTS.md 路径列表

3. 本地 Backend 组装
   ┌─ enable_shell=True ──► LocalShellBackend(root_dir, inherit_env=True)
   └─ enable_shell=False ─► FilesystemBackend(root_dir)

4. CompositeBackend（沙箱为 None 时）
   default:            本地 shell/fs backend
   /large_tool_results/:  独立临时目录 FilesystemBackend
   /conversation_history/: 独立临时目录 FilesystemBackend

5. HITL 配置
   interrupt_on 由 auto_approve / ShellAllowListMiddleware 决定

6. create_deep_agent(
       model, system_prompt, tools, backend=composite_backend,
       middleware, interrupt_on, checkpointer, subagents
   ).with_config(config)
```

### `ShellAllowListMiddleware`

在 `interrupt_shell_only` 且非 `auto_approve` 时：

- 在**执行前**用白名单检查 shell 命令
- 非法命令返回错误 `ToolMessage`，**不触发图 interrupt**
- 目的：保持 LangSmith 单次连续 trace，避免因 interrupt 导致轨迹碎片化

---

## 4.3 配置系统（`config.py`）

### 配置加载优先级

```
shell 环境变量
    > 项目 .env（DEEPAGENTS_CLI_* 前缀桥接到 LangSmith SDK 变量名）
    > ~/.deepagents/.env（override=False，不覆盖已有值）
```

首次访问 `settings` 时 `_ensure_bootstrap()`：加载 `.env` 文件、覆盖 `LANGSMITH_PROJECT` 以隔离 agent 追踪。

### Shell 白名单机制

```python
class _ShellAllowAll(list):
    """哨兵子类，表示无限制 shell 访问"""

SHELL_ALLOW_ALL: list[str] = _ShellAllowAll(["__ALL__"])

def parse_shell_allow_list(allow_list_str: str | None) -> list[str] | None:
    if allow_list_str.strip().lower() == "all":
        return SHELL_ALLOW_ALL      # 跳过 contains_dangerous_patterns 检查
    if allow_list_str.strip().lower() == "recommended":
        return list(RECOMMENDED_SAFE_SHELL_COMMANDS)  # 预置安全命令列表
    # 逗号分隔可合并 recommended
```

`is_shell_command_allowed(command, allow_list)` 核心逻辑：

1. 若 `allow_list` 是 `_ShellAllowAll` 实例 → 直接返回 `True`（跳过危险模式检测）
2. 否则先执行 `contains_dangerous_patterns`（重定向、`$()`、反引号、后台 `&` 等）
3. 再按管道/复合命令分段，用 `shlex.split` 取首 token 与白名单集合比对

### `get_default_coding_instructions`

从包内 `default_agent_prompt.md` 读取，与 AGENTS.md 长期记忆分离。

---

## 4.4 Hooks 系统（`hooks.py`）

**定位**：CLI 外部集成扩展点，**不是** LangChain 的 PreToolUse/PostToolUse/Stop 钩子。

### 机制

- 配置：`~/.deepagents/hooks.json`，定义 `command` + 可选 `events`
- 触发：匹配事件时向子进程 **stdin 写入 JSON**，5s 超时，线程池并发，失败只记日志

### 触发的事件名（部分）

| 事件 | 触发时机 |
|------|---------|
| `session.start` | 会话开始 |
| `user.prompt` | 用户输入 |
| `permission.request` | 工具审批请求 |
| `task.complete` | 任务完成 |
| `context.offload` | 上下文外溢 |
| `context.compact` | 摘要压缩 |
| `session.end` | 会话结束 |

---

## 4.5 MCP 工具集成（`mcp_tools.py` + `mcp_trust.py`）

### 配置格式与传输类型

支持 **Claude Desktop 格式** JSON（`mcpServers`）：

| 传输类型 | 配置方式 |
|---------|---------|
| stdio | `command` + `args` + `env` |
| SSE | `url` 字段 |
| HTTP | `url` 字段（映射为 `streamable_http`） |

### 配置发现顺序（低 → 高优先级）

```
~/.deepagents/.mcp.json
→ <project>/.deepagents/.mcp.json
→ <project>/.mcp.json
→ --mcp-config（显式文件，最高优先级）
```

`merge_mcp_configs` 按后者覆盖同名 server。

### 信任机制（`mcp_trust.py`）

- 项目级 stdio server 可执行本地命令
- 对项目配置路径内容做 **SHA-256 指纹**，写入 `~/.deepagents/config.toml` 的 `[mcp_trust.projects]`
- 指纹变化需重新批准（`_check_mcp_project_trust` 在启动 TUI 前做交互式确认）
- `trust_project_mcp=False` 时去掉 stdio，仅保留远程 SSE/HTTP

---

## 4.6 Sessions 与线程管理（`sessions.py`）

- 使用 **LangGraph SQLite 异步 checkpoint**（`AsyncSqliteSaver` + `aiosqlite`）
- 数据库默认 `~/.deepagents/sessions.db`
- 提供 `generate_thread_id`（UUID7）、`list_threads`、`delete_thread` 等
- 供 `threads` 子命令与 TUI `/threads` 使用

---

## 4.7 TUI Widgets（`widgets/`）

### `approval.py`（HITL 审批）

```
ApprovalMenu 按键绑定：
  1/y     → 批准（ApproveDecision）
  2/a     → 全局自动批准（ApproveAlwaysDecision）
  3/n     → 拒绝（RejectDecision）
  e       → 展开（shell 命令过长时）
  Enter   → 确认当前选择
```

- 与 `tool_renderers.get_renderer` 配合，按工具类型展示写入预览 / diff / 通用参数
- `Decided` message 携带 `decision` dict，供上层 `Future` 完成

### `tool_renderers.py`（渲染注册表）

| 工具 | Widget |
|------|--------|
| `write_file` | `WriteFileApprovalWidget` |
| `edit_file` | `EditFileApprovalWidget`（difflib 生成 unified diff） |
| 其他 | `GenericApprovalWidget` |

### `diff.py`

- `compose_diff_lines`：按行拆成 `Static`，增删行用 CSS class（`.diff-line-added` / `.diff-line-removed`）
- 主题色来自 `theme.get_theme_colors()`，支持行数上限与统计头

---

## 4.8 流式输出与 LangGraph 适配（`textual_adapter.py`）

### `execute_task_textual` 核心流程

```python
async for chunk in agent.astream(
    stream_input,
    stream_mode=["messages", "updates"],
    subgraphs=True,
    config=config,
    context=context,
    durability="exit",
):
    namespace, current_stream_mode, data = chunk
    
    if current_stream_mode == "updates":
        if "__interrupt__" in data:
            # HITL 中断处理
            ...
    elif current_stream_mode == "messages":
        if namespace == ():  # 只渲染主 namespace，子 agent 输出不混入
            # 渲染文本/工具调用
```

### 关键过滤规则

| 规则 | 目的 |
|------|------|
| 只渲染主 namespace（`ns_key == ()`） | 子 agent 输出不混入主聊天区 |
| 过滤 `lc_source == "summarization"` 的流 | 用 spinner 替代，不展示摘要模型内部输出 |
| `durability="exit"` | 图退出时自动 flush 剩余消息 |

### 客户端 `auto_approve`

```python
if session_state.auto_approve:
    decisions = [ApproveDecision(type="approve") for _ in action_requests]
    resume_payload[interrupt_id] = {"decisions": decisions}
else:
    # 展示审批 UI，等待用户决策
    future = await adapter._request_approval(action_requests, assistant_id)
    decision = await future
```

---

## 4.9 人机协作 HITL（`ask_user.py` / `_ask_user_types.py`）

### `AskUserMiddleware`

- 注册 `ask_user` 工具，内部调用 **`interrupt(ask_request)`**
- resume 后 `_parse_answers` 生成 `ToolMessage`

### `interrupt_on` 配置（`agent._add_interrupt_on`）

覆盖以下工具（按需配置）：

| 工具 | 默认中断条件 |
|------|------------|
| `execute` | 命令不在白名单时 |
| `write_file` | 始终（可配置 auto_approve） |
| `edit_file` | 始终（可配置 auto_approve） |
| `web_search` | 始终（可配置） |
| `task` | 可选 |
| `compact_conversation` | 可选 |

---

## 4.10 Skills CLI（`skills/`）

| 模块 | 职责 |
|------|------|
| `load.py` | `list_skills`：多目录合并（后者覆盖同名），`load_skill_content` 带路径 containment 防 symlink 逃逸 |
| `invocation.py` | `build_skill_invocation_envelope`：包装 SKILL.md 与用户请求，在 `additional_kwargs.__skill` 中持久化元数据 |
| `commands.py` | `deepagents skills list/create/info/delete` 等；`setup_skills_parser` 在 `main.parse_args` 注册 |

技能在 **Agent 运行时** 由 `SkillsMiddleware` 加载（`agent.py` 中的 `sources` 列表），CLI 子命令仅做文件系统管理类操作。

---

## 关键设计决策

1. **交互与执行分离**：默认 TUI + **子进程 LangGraph Server**，避免在 UI 进程内跑重图；ACP 与 `--acp` 则相反，进程内直连

2. **HITL 双通道**：工具侧用 LangGraph `interrupt` + `interrupt_on`；交互 TUI 用**客户端 `auto_approve`** 覆盖，避免服务端图重编译，支持会话内切换

3. **非交互 shell**：`ShellAllowListMiddleware` 用错误消息替代 interrupt，保持**单次 trace**

4. **MCP 安全**：项目 stdio 需**指纹信任**或 `--trust-project-mcp`；远程 SSE/HTTP 与 stdio 预检分离

5. **UI 与流**：`textual_adapter` 严格过滤 namespace 与 summarization 流；工具与 diff、审批通过 **adapter 回调** 与 widget 注册表组合

6. **Hooks 为外部扩展**：JSON 配置 + stdin 管道，不侵入 LangGraph 节点，便于第三方集成
