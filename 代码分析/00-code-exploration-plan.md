# Deep Agents 代码探索计划

> **项目定位**：Deep Agents 是 LangChain 团队出品的"batteries-included"（开箱即用）Agent Harness，灵感来自 Claude Code，基于 LangGraph 构建，提供规划、文件系统操作、子代理、上下文管理等开箱即用能力，并附带一个终端 TUI 工具（类 Claude Code 体验）。

---

## 项目总体结构

```
deepagents/
├── libs/
│   ├── deepagents/        # 核心 SDK（create_deep_agent 入口）
│   ├── cli/               # 终端 TUI 工具（Textual 框架）
│   ├── acp/               # Agent Context Protocol 服务端适配
│   ├── evals/             # 评测套件与 LangSmith Harbor 集成
│   └── partners/          # 沙箱合作伙伴集成（Daytona/Modal/Runloop/...）
├── examples/              # 示例项目
└── AGENTS.md              # AI 开发助手指引
```

---

## 分析主题与二级主题

### 一、核心架构与入口点

- [ ] **1.1 `create_deep_agent` 工厂函数**
  - 分析 `libs/deepagents/deepagents/graph.py` 中的 `create_deep_agent` 函数
  - 理解参数体系（model/tools/middleware/subagents/skills/memory/...）
  - 梳理 Middleware 堆栈的构建顺序与优先级
  - 理解 `BASE_AGENT_PROMPT` 的设计哲学与行为约束

- [ ] **1.2 LangGraph 集成与图结构**
  - 深入 `create_agent`（来自 `langchain.agents`）的底层图构建机制
  - 理解 `CompiledStateGraph` 的状态流转
  - 分析 `AgentState` / `_InputAgentState` / `_OutputAgentState` 的类型体系
  - 研究 `recursion_limit=9999` 等默认配置的意义

- [ ] **1.3 模型解析与适配**
  - 分析 `_models.py` 中的 `resolve_model` 函数
  - 理解 `provider:model` 格式的解析逻辑
  - 研究 OpenAI Responses API 与 Chat Completions 的切换策略

---

### 二、Middleware 中间件系统

- [ ] **2.1 Middleware 架构概览**
  - 理解 `AgentMiddleware` 基类与协议定义（来自 `langchain.agents.middleware.types`）
  - 分析 `wrap_model_call` / `awrap_model_call` / `before_agent` 等钩子
  - 梳理中间件的组合顺序与职责边界

- [ ] **2.2 FilesystemMiddleware — 文件系统工具**
  - 分析 `libs/deepagents/deepagents/middleware/filesystem.py`
  - 工具集：`ls`、`read_file`、`write_file`、`edit_file`、`glob`、`grep`
  - 工具参数设计与错误处理模式
  - Backend 委托机制

- [ ] **2.3 SubAgentMiddleware — 同步子代理**
  - 分析 `libs/deepagents/deepagents/middleware/subagents.py`
  - `SubAgent` / `CompiledSubAgent` TypedDict 规范
  - `task` 工具的构建与调用流程（`_build_task_tool`）
  - 状态隔离机制：`_EXCLUDED_STATE_KEYS`
  - 默认 general-purpose 子代理的构建

- [ ] **2.4 AsyncSubAgentMiddleware — 异步/远程子代理**
  - 分析 `libs/deepagents/deepagents/middleware/async_subagents.py`
  - `AsyncSubAgent` TypedDict 规范（`graph_id`, `url`, `headers`）
  - 基于 LangGraph SDK 的后台任务管理
  - 任务启动/检查/更新/取消/列表工具集

- [ ] **2.5 SummarizationMiddleware — 上下文压缩**
  - 分析 `libs/deepagents/deepagents/middleware/summarization.py`
  - 对话历史自动摘要的触发时机与策略
  - 大输出自动保存到文件的机制
  - 与 Backend 的协作

- [ ] **2.6 MemoryMiddleware — 持久记忆**
  - 分析 `libs/deepagents/deepagents/middleware/memory.py`
  - `AGENTS.md` 文件的加载与注入
  - 记忆路径配置与显示名称派生
  - 与系统提示词的集成时机

- [ ] **2.7 SkillsMiddleware — Agent Skills**
  - 分析 `libs/deepagents/deepagents/middleware/skills.py`
  - Progressive Disclosure 模式（先暴露元数据，按需读取全文）
  - `SkillMetadata` TypedDict 与 YAML frontmatter 解析
  - Skill 名称验证规则（Agent Skills 规范）
  - 多 source 叠加加载（last wins）

- [ ] **2.8 PatchToolCallsMiddleware — 工具调用修正**
  - 分析 `libs/deepagents/deepagents/middleware/patch_tool_calls.py`
  - 针对模型输出错误的工具调用进行修正的策略
  - 与 Anthropic 特性的关联

- [ ] **2.9 AnthropicPromptCachingMiddleware**
  - 理解 Prompt Caching 在 Middleware 堆栈中的位置（为何在最后）
  - `unsupported_model_behavior="ignore"` 的作用

---

### 三、Backend 后端系统

- [ ] **3.1 BackendProtocol — 统一文件操作接口**
  - 分析 `libs/deepagents/deepagents/backends/protocol.py`
  - 核心方法：`read/write/edit/ls/glob/grep`（同步 + 异步版本）
  - 数据类型体系：`FileData/ReadResult/WriteResult/EditResult/LsResult/GrepResult/GlobResult`
  - 错误标准化：`FileOperationError` Literal 类型
  - 批量操作：`upload_files/download_files`

- [ ] **3.2 SandboxBackendProtocol — 沙箱执行扩展**
  - `execute` / `aexecute` 方法与 `ExecuteResponse`
  - `execute_accepts_timeout` 的反射检测机制
  - 与合作伙伴 Backend 的继承关系

- [ ] **3.3 StateBackend — 内存/状态存储**
  - 分析 `libs/deepagents/deepagents/backends/state.py`
  - 基于 LangGraph state 的文件存储机制
  - `FileData` v1/v2 格式兼容处理

- [ ] **3.4 FilesystemBackend — 本地文件系统**
  - 分析 `libs/deepagents/deepagents/backends/filesystem.py`
  - `root_dir` 沙箱限制与路径校验
  - 二进制文件的 base64 编解码

- [ ] **3.5 LocalShellBackend — 本地 Shell 执行**
  - 分析 `libs/deepagents/deepagents/backends/local_shell.py`
  - 命令执行与超时控制
  - 输出截断策略

- [ ] **3.6 CompositeBackend — 组合后端**
  - 分析 `libs/deepagents/deepagents/backends/composite.py`
  - 读写分离与路由策略
  - CLI 中 `FilesystemBackend + LocalShellBackend` 的组合模式

- [ ] **3.7 StoreBackend — 持久化存储**
  - 分析 `libs/deepagents/deepagents/backends/store.py`
  - 基于 LangGraph `BaseStore` 的持久存储
  - `BackendContext` 与 `NamespaceFactory` 的设计

- [ ] **3.8 LangSmithSandbox — 云端沙箱**
  - 分析 `libs/deepagents/deepagents/backends/langsmith.py`
  - 与 LangSmith Deployments API 的集成

---

### 四、CLI 工具实现

- [ ] **4.1 CLI 入口与应用框架**
  - 分析 `libs/cli/deepagents_cli/main.py` 和 `app.py`
  - Textual TUI 框架的应用结构
  - 交互式与非交互式（headless）模式切换

- [ ] **4.2 Agent 生命周期管理**
  - 分析 `libs/cli/deepagents_cli/agent.py`
  - CLI 中 `create_deep_agent` 的定制化配置
  - `CompositeBackend`（Filesystem + LocalShell）的组装
  - MCP 工具集成、技能加载、记忆加载

- [ ] **4.3 配置系统**
  - 分析 `libs/cli/deepagents_cli/config.py`
  - `settings` 配置文件的解析与优先级
  - Shell 命令白名单（`_ShellAllowAll`）机制
  - 默认编码指令（`get_default_coding_instructions`）

- [ ] **4.4 Hooks 系统**
  - 分析 `libs/cli/deepagents_cli/hooks.py`
  - 生命周期钩子（PreToolUse / PostToolUse / Stop 等）
  - Hooks 的触发机制与扩展点

- [ ] **4.5 MCP 工具集成**
  - 分析 `libs/cli/deepagents_cli/mcp_tools.py` 和 `mcp_trust.py`
  - MCP Server 配置（stdio / SSE / HTTP）
  - 工具信任策略与权限管理

- [ ] **4.6 Sessions 与线程管理**
  - 分析 `libs/cli/deepagents_cli/sessions.py`
  - 会话持久化与历史浏览
  - `thread_selector` Widget 的实现

- [ ] **4.7 TUI Widgets**
  - 分析 `libs/cli/deepagents_cli/widgets/` 目录
  - 消息渲染：`messages.py` / `tool_renderers.py` / `tool_widgets.py`
  - 用户输入：`chat_input.py` / `autocomplete.py`
  - Diff 展示：`diff.py`
  - 权限审批：`approval.py`
  - MCP 查看器：`mcp_viewer.py`

- [ ] **4.8 流式输出与格式化**
  - 分析 `libs/cli/deepagents_cli/output.py` 和 `formatting.py`
  - `textual_adapter.py` 中 LangGraph streaming 与 Textual 的适配
  - Token 统计与 `_session_stats.py`

- [ ] **4.9 人机协作（HITL）与审批流程**
  - 分析 `libs/cli/deepagents_cli/widgets/approval.py`
  - `ask_user.py` / `_ask_user_types.py` 工具
  - `interrupt_on` 配置与 LangGraph interrupt 机制

- [ ] **4.10 Skills CLI 实现**
  - 分析 `libs/cli/deepagents_cli/skills/` 目录
  - Skill 加载（`load.py`）、调用（`invocation.py`）、命令注册（`commands.py`）

---

### 五、ACP（Agent Context Protocol）集成

- [ ] **5.1 ACP Server 实现**
  - 分析 `libs/acp/deepagents_acp/server.py`
  - ACP 协议的 Session 管理与消息格式转换
  - `PromptResponse` / `ToolCallStart` / `ToolCallUpdate` 等协议消息

- [ ] **5.2 ACP 与 LangGraph 的桥接**
  - LangGraph streaming 结果到 ACP 消息的映射
  - 工具调用结果的协议格式化
  - Checkpoint / 状态恢复机制

---

### 六、评测系统（Evals）

- [ ] **6.1 评测框架架构**
  - 分析 `libs/evals/` 的整体结构
  - `deepagents_evals` 与 `deepagents_harbor` 包的职责划分
  - `pytest` 为基础的评测执行机制

- [ ] **6.2 评测类别与指标**
  - 分析 `deepagents_evals/categories.json`
  - Radar 评测维度可视化（`radar.py`）
  - 评测分类：工具使用、内存、子代理、技能、Follow-up 质量等

- [ ] **6.3 Harbor 集成**
  - 分析 `deepagents_harbor/` 目录
  - LangSmith 集成（`langsmith.py`）
  - 评测结果后端存储（`backend.py`）
  - 失败分析（`failure.py`）与统计（`stats.py`）

- [ ] **6.4 具体评测案例**
  - `test_subagents.py` — 子代理能力评测
  - `test_memory.py` / `test_memory_multiturn.py` — 记忆能力评测
  - `test_skills.py` — Skills 能力评测
  - `test_hitl.py` — 人机协作评测
  - `tau2_airline/` — TAU2 航空领域基准测试

---

### 七、Partner Sandbox 集成

- [ ] **7.1 Daytona 沙箱**
  - 分析 `libs/partners/daytona/` 实现
  - Daytona API 的封装与 `SandboxBackendProtocol` 实现

- [ ] **7.2 Modal/Runloop/QuickJS 沙箱**
  - 各合作方沙箱的 Backend 实现方式
  - 超时处理、文件上传下载的差异化实现

---

### 八、安全模型

- [ ] **8.1 威胁模型分析**
  - 研读 `libs/deepagents/THREAT_MODEL.md` 和 `libs/cli/THREAT_MODEL.md`
  - "trust the LLM" 安全模型的边界与假设

- [ ] **8.2 Unicode 安全**
  - 分析 `libs/cli/deepagents_cli/unicode_security.py`
  - Prompt Injection 防护与 Unicode 欺骗检测

- [ ] **8.3 Shell 命令权限控制**
  - Shell 命令白名单机制
  - 沙箱 Backend 对执行权限的约束

---

### 九、扩展与定制模式

- [ ] **9.1 自定义工具注入**
  - 通过 `tools` 参数添加自定义工具
  - 工具格式：`BaseTool` / `Callable` / `dict`

- [ ] **9.2 自定义子代理设计**
  - `SubAgent` vs `CompiledSubAgent` vs `AsyncSubAgent` 的选型
  - 子代理中间件的继承与覆盖机制

- [ ] **9.3 自定义 Backend 实现**
  - 实现 `BackendProtocol` 的最小接口要求
  - `BackendFactory` 工厂模式与运行时状态注入

- [ ] **9.4 Examples 解析**
  - `deep_research/` — 深度研究 Agent 实现
  - `text-to-sql-agent/` — Text-to-SQL Agent
  - `async-subagent-server/` — 异步子代理服务端
  - `content-builder-agent/` — 内容构建 Agent

---

## 建议探索顺序

```
Phase 1（核心流程）：
  一 → 三 → 二（2.1→2.2→2.3→2.5）

Phase 2（扩展能力）：
  二（2.4→2.6→2.7）→ 八（安全）

Phase 3（CLI 与 ACP）：
  四 → 五

Phase 4（评测与生态）：
  六 → 七 → 九
```

---

## 关键文件速查表

| 模块 | 路径 | 重要程度 |
|------|------|----------|
| 工厂函数入口 | `libs/deepagents/deepagents/graph.py` | ⭐⭐⭐⭐⭐ |
| Backend 协议 | `libs/deepagents/deepagents/backends/protocol.py` | ⭐⭐⭐⭐⭐ |
| 子代理中间件 | `libs/deepagents/deepagents/middleware/subagents.py` | ⭐⭐⭐⭐⭐ |
| 文件系统中间件 | `libs/deepagents/deepagents/middleware/filesystem.py` | ⭐⭐⭐⭐ |
| Skills 中间件 | `libs/deepagents/deepagents/middleware/skills.py` | ⭐⭐⭐⭐ |
| 摘要中间件 | `libs/deepagents/deepagents/middleware/summarization.py` | ⭐⭐⭐⭐ |
| 异步子代理 | `libs/deepagents/deepagents/middleware/async_subagents.py` | ⭐⭐⭐⭐ |
| CLI Agent | `libs/cli/deepagents_cli/agent.py` | ⭐⭐⭐⭐ |
| CLI Hooks | `libs/cli/deepagents_cli/hooks.py` | ⭐⭐⭐ |
| ACP Server | `libs/acp/deepagents_acp/server.py` | ⭐⭐⭐ |
| 威胁模型 | `libs/deepagents/THREAT_MODEL.md` | ⭐⭐⭐ |
| AGENTS.md | `AGENTS.md` | ⭐⭐⭐ |
