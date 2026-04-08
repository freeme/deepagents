# 九、扩展与定制模式

## 架构关系概览（ASCII）

```
┌─────────────────────────────────────────────────────────────────────────┐
│ create_deep_agent(model, tools, backend, subagents, interrupt_on, ...)  │
├─────────────────────────────────────────────────────────────────────────┤
│  Middleware 栈（顺序见 graph.py）                                         │
│    TodoList → [Skills] → Filesystem → SubAgent → Summarization → Patch  │
│    [AsyncSubAgentMiddleware] → [用户 middleware] → PromptCaching → Memory │
│    [HumanInTheLoopMiddleware]                                            │
├─────────────────────────────────────────────────────────────────────────┤
│  backend: BackendProtocol  │  BackendFactory(runtime) → BackendProtocol  │
│       (FilesystemMiddleware 每次工具调用时 _get_backend(runtime))          │
├─────────────────────────────────────────────────────────────────────────┤
│  subagents: SubAgent (内建中间件栈) │ CompiledSubAgent (自定义 Runnable)  │
│              │ AsyncSubAgent (graph_id → 远程 LangGraph)                │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9.1 自定义工具注入

### 工具格式

```python
create_deep_agent(
    tools=[
        my_tool,           # BaseTool 实例
        my_function,       # 被 @tool 装饰的可调用对象
        {"name": ..., "description": ..., "parameters": ...}  # dict 规范
    ]
)
```

### 与内置工具的关系

```
内置工具（通过中间件注入）      自定义工具（通过 tools 参数）
─────────────────────────     ──────────────────────────────
todo_list                      任意 BaseTool / Callable / dict
read_file, write_file, ...
task（子代理）
compact_conversation
```

两者并存，模型在调用时均可选择。

### 子代理工具继承

- **`general_purpose_spec`**：使用 `"tools": tools or []`（继承主 agent 的 tools）
- **自定义 `SubAgent`**：若未指定 `tools`，默认 `spec.get("tools", tools or [])`（同样继承）
- 若子代理不应访问主 agent 工具，需显式传入 `"tools": []`

---

## 9.2 自定义子代理设计

### 三种子代理规格对比

| 类型 | 定义方式 | 中间件栈 | 适用场景 |
|------|---------|---------|---------|
| **SubAgent** | TypedDict（声明式） | `create_deep_agent` 内自动构建（TodoList + FS + Summarization + Patch + 可选 Skills + AnthropicCache） | 声明式配置；需 `model` 与 `tools` 字段 |
| **CompiledSubAgent** | 含 `runnable` 字段 | 完全由用户控制，**不自动添加任何中间件** | 完全控制图结构；状态必须含 `messages` |
| **AsyncSubAgent** | 含 `graph_id` + `url` | 远程 LangGraph 图，本地只有工具封装 | 后台/并行/长时间运行任务 |

### SubAgent 完整字段

```python
class SubAgent(TypedDict, total=False):
    name: str            # 必填
    description: str     # 必填
    system_prompt: str   # 必填
    tools: list          # 可选，默认继承主 agent tools
    model: str | BaseChatModel  # 可选，默认继承主 agent model
    middleware: list[AgentMiddleware]  # 可选，追加到标准栈之后
    interrupt_on: dict   # 可选，覆盖继承的 interrupt_on
    skills: list[str]    # 可选，追加 SkillsMiddleware
```

### 中间件继承与覆盖规则

```
主 agent interrupt_on
        │
        ├─ SubAgent（声明式）：spec.get("interrupt_on", interrupt_on)
        │   可在 spec 中覆盖，也可继承主 agent 值
        │
        ├─ CompiledSubAgent：不继承，需在 runnable 内自行配置
        │
        └─ AsyncSubAgent：不继承，远程行为在远端配置
```

### 状态隔离细节

```python
_EXCLUDED_STATE_KEYS = {
    "messages",           # 子代理有独立的消息历史
    "todos",              # 避免子代理 TODO 污染主线程
    "structured_response", # 避免子代理结构化输出覆盖主线程
    "skills_metadata",    # 技能元数据不传递
    "memory_contents",    # 记忆不传递给子代理
}
```

- 调用子代理前：从 `runtime.state` **剔除**上述键，再设 `messages=[HumanMessage(content=description)]`
- 返回时：从子代理结果中排除 `_EXCLUDED_STATE_KEYS`，**仅将最后一条 AI 消息文本**作为 `ToolMessage` 回主线程

---

## 9.3 自定义 Backend 实现

### `BackendProtocol` 最小接口

需实现的核心方法：

| 方法 | 说明 |
|------|------|
| `ls(path) -> LsResult` | 目录列表 |
| `read(path, offset, limit) -> ReadResult` | 分页读取 |
| `write(path, content) -> WriteResult` | 创建新文件 |
| `edit(path, old, new) -> EditResult` | 精确字符串替换 |
| `grep(pattern, path) -> GrepResult` | 字面量子串搜索 |
| `glob(pattern, path) -> GlobResult` | glob 匹配 |
| `upload_files(files) -> list[FileUploadResponse]` | 批量上传 |
| `download_files(paths) -> list[FileDownloadResponse]` | 批量下载 |

各方法有默认 `a*` 异步包装（`asyncio.to_thread`），若有原生异步实现可覆盖。

若需 shell 执行，还需实现 `SandboxBackendProtocol`：

```python
class MyBackend(SandboxBackendProtocol):
    @property
    def id(self) -> str:
        return f"my-backend-{self._id}"
    
    def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
        result = run_in_sandbox(command, timeout=timeout)
        return ExecuteResponse(output=result.output, exit_code=result.code, truncated=False)
```

### `BackendFactory` 工厂模式

```python
BackendFactory = Callable[[ToolRuntime], BackendProtocol]

# 用法：
create_deep_agent(
    backend=lambda runtime: MyBackend(user_id=runtime.context.user_id)
)
```

- 允许根据运行时状态（用户 ID、配置等）动态创建 backend
- **注意**：当前代码对 callable 形式发出 `DeprecationWarning`，建议直接传 `BackendProtocol` 实例

### 实现步骤

1. 子类化 `BackendProtocol`（或 `SandboxBackendProtocol`）
2. 实现所有抽象方法，正确返回数据类型（`LsResult`、`ReadResult` 等）和错误码（`FileOperationError`）
3. 在 `create_deep_agent(..., backend=YourBackend())` 中传入
4. 可选：用 `CompositeBackend` 按路径前缀路由多个 backend

---

## 9.4 Examples 解析

### 9.4.1 `deep_research/` — 深度研究 Agent

```python
create_deep_agent(
    model=...,
    tools=[tavily_search, think_tool],
    system_prompt=组合指令,
    subagents=[research_sub_agent]  # SubAgent 规范 dict
)
```

**核心模式**：通过子代理进行**任务隔离**，主 agent 做规划和整合，子 agent 专注单一研究任务。

### 9.4.2 `text-to-sql-agent/` — Text-to-SQL Agent

```python
sql_tools = SQLDatabaseToolkit(...).get_tools()

create_deep_agent(
    tools=sql_tools,              # 注入 SQL 工具集
    memory=["./AGENTS.md"],       # 持久记忆
    skills=["./skills/"],         # 查询/模式探索技能
    backend=FilesystemBackend(root_dir=base_dir),
    subagents=[]                  # 显式关闭额外子代理
)
```

**核心模式**：外部工具集注入（LangChain Toolkit） + Skills 提供专领域知识。

### 9.4.3 `async-subagent-server/` — 异步子代理服务端

```python
# 服务端：FastAPI 实现 LangGraph SDK 期望的 threads/runs HTTP API
agent = create_deep_agent(tools=[web_search])

# 客户端（supervisor）配置：
subagents=[
    AsyncSubAgent(
        name="research-agent",
        description="...",
        graph_id="agent",
        url="http://localhost:8000"  # 异步子代理服务地址
    )
]
```

**核心模式**：`AsyncSubAgent` + 自定义 LangGraph 兼容 HTTP 服务，实现后台并行任务。

### 9.4.4 `content-builder-agent/` — 内容构建 Agent

```python
# 从 YAML 读子代理定义（框架扩展示例）
subagents = load_subagents("subagents.yaml")

create_deep_agent(
    tools=[generate_cover, generate_social_image],  # 图片生成工具
    memory=...,
    skills=...,
    backend=FilesystemBackend(root_dir=EXAMPLE_DIR),
    subagents=subagents  # 来自 YAML 配置
)
```

**核心模式**：从配置文件动态加载子代理（SDK 不原生支持，此为示例扩展）+ 专用生成工具。

---

## 9.5 中间件扩展模式

### 插入自定义中间件

```python
class MyMiddleware(AgentMiddleware):
    async def awrap_model_call(self, request, handler, runtime):
        # 在 LLM 调用前修改请求
        modified_request = add_my_context(request)
        return await handler(modified_request)

create_deep_agent(
    model=...,
    middleware=[MyMiddleware()]  # 插入在 AnthropicPromptCachingMiddleware 之前
)
```

### 自定义中间件的位置

用户 middleware 插入在：**PatchToolCallsMiddleware / AsyncSubAgentMiddleware 之后**，**AnthropicPromptCachingMiddleware 之前**。

---

## 9.6 开发指引（`AGENTS.md` 摘要）

`AGENTS.md` 为 monorepo 级开发规范，扩展时应注意：

| 规范 | 要求 |
|------|------|
| 包管理 | 使用 `uv` / `make` |
| 代码质量 | `ruff` lint + `ty` 类型检查 |
| 安全 | 禁止对用户输入使用 `eval` 等不安全函数 |
| UI | Textual/Rich 约定（CLI 扩展） |
| SDK 版本 | 固定版本 pin |
| 启动路径 | 避免重依赖（保持快速启动） |

---

## 小结

DeepAgents 提供了四个主要扩展点：

```
1. tools      → 注入任意 LangChain 工具（最简单）
2. middleware → 控制每轮 LLM 请求（最灵活）
3. backend    → 自定义文件/执行环境（隔离级别）
4. subagents  → 任务分解与并行（架构级）
```

三种子代理形式从「声明式（最简）」到「预编译图（最灵活）」到「远程异步（最强大）」提供渐进的定制深度：

```
SubAgent（声明式）
  → 适合大多数场景，自动获得完整中间件栈
  → 一行 dict 配置，低门槛

CompiledSubAgent（预编译图）
  → 完全控制图结构和中间件
  → 适合有特殊需求的专用子代理

AsyncSubAgent（远程异步）
  → 后台并行执行，不阻塞主 agent
  → 适合耗时较长的研究/分析任务
```
