# Layer 5：生态系统分析 — 与外部世界的连接

> 生成时间：2026-04-07  
> 本文档是 [code-exploration-plan.md](./code-exploration-plan.md) Layer 5 的深度分析产物。

---

## 核心结论速览

| 维度 | 结论 |
|------|------|
| ACP 协议来源 | **行业标准**（agentclientprotocol.com），非自研 |
| MCP 集成层 | CLI 层，通过 `langchain-mcp-adapters` 桥接 |
| Sandbox 适配模式 | 统一的 `BaseSandbox` adapter，五个 provider |
| 远程图模式 | 直接包装 LangGraph SDK 的 `RemoteGraph` |
| 异步子智能体服务 | 独立的 FastAPI 服务，实现 LangGraph 兼容 REST API |

---

## 一、ACP（Agent Client Protocol）

### 1.1 协议定位

ACP 是 **agentclientprotocol.com** 发布的行业标准协议，不是 deepagents 自研的。它的定位是：

```
文本编辑器（如 Zed）  <──── ACP ────>  AI Agent（DeepAgents）
```

deepagents 提供的是一个适配层（`libs/acp/`），让任何 DeepAgent 都能以 ACP 服务端的形式运行，从而接入支持 ACP 的编辑器。

### 1.2 核心类：AgentServerACP

`AgentServerACP` 继承自 ACP SDK 的 `ACPAgent`，构造函数接受两种 agent 形态：

- 静态的 `CompiledStateGraph`（固定 agent）
- 工厂函数 `Callable[[AgentSessionContext], CompiledStateGraph]`（按会话动态创建）

工厂函数形态使得模型切换成为可能——切换时通过 `_reset_agent()` 重建 agent，会话历史通过共享的 `MemorySaver`（checkpointer）保留。

### 1.3 协议握手流程

```
client                         AgentServerACP
  |                                  |
  |-- initialize() ----------------> 返回 AgentCapabilities（支持图像）
  |                                  |
  |-- new_session(cwd) ------------> 分配 session_id，返回 modes/config_options
  |                                  |
  |-- prompt(content_blocks) ------> 转换格式 -> agent.astream() -> 流式回传
  |                                  |
  |-- set_config_option(model=...) -> _reset_agent() 热切换模型
  |                                  |
  |-- cancel() -------------------> 设置 _cancelled 标志，下次循环退出
```

### 1.4 多模态内容转换

ACP 的内容块（ContentBlock）和 LangChain 格式之间需要转换：

| ACP 类型 | LangChain 格式 |
|---------|---------------|
| `TextContentBlock` | `{"type": "text", "text": ...}` |
| `ImageContentBlock` | `{"type": "image_url", "image_url": {"url": "data:..."}}` |
| `ResourceContentBlock` | `{"type": "text", "text": "[Resource: ...]"}` 文件引用转文本描述 |
| `AudioContentBlock` | `NotImplementedError`（尚未支持） |

### 1.5 Human-in-the-Loop 与 ACP 的配合

ACP 对 LangGraph interrupt 有明确的格式要求。当 agent 触发中断时：

1. agent 暂停，ACP 层捕获 `__interrupt__` 事件
2. 将中断转换为 `PermissionOption` 列表（approve / reject / approve_always）
3. 通过 `conn.request_permission()` 向编辑器弹出权限确认对话框
4. 用户决策后，以 `Command(resume={"decisions": [...]})` 恢复执行

**关键安全逻辑**（`approve_always` 的粒度）：
- 普通工具：按工具名 `(tool_name, None)` 记录永久允许
- `execute` 工具：按命令签名 `(execute, "npm install")` 记录（而非整个命令）
- 危险 shell 模式（`$()`、backtick、重定向等）：**永远不自动批准**

---

## 二、MCP（Model Context Protocol）集成

### 2.1 集成层位置

MCP 集成在 **CLI 层**（`libs/cli/deepagents_cli/mcp_tools.py`），不在核心 SDK 中。这意味着：
- 通过代码使用 SDK 时，MCP 工具需用户自行加载并传入 `tools=`
- 通过 CLI 启动时，MCP 自动发现和加载

### 2.2 配置发现优先级

```
优先级（低 -> 高）：
  1. ~/.deepagents/.mcp.json           <- 用户级，始终信任
  2. <project>/.deepagents/.mcp.json   <- 项目级（需 stdio 信任审查）
  3. <project>/.mcp.json               <- 项目级（Claude Code 兼容路径，需审查）
  4. --mcp-config <path>               <- 显式指定，最高优先级，错误致命
```

多配置合并策略：**后者覆盖前者的同名 server**（`dict.update`）。

### 2.3 Transport 类型支持

| Transport | 配置字段 | 特点 |
|-----------|---------|------|
| `stdio` | `command`, `args`, `env` | 启动子进程，持久化 session（不重启） |
| `sse` | `url`, `headers` | Server-Sent Events，适合远程服务 |
| `http` | `url`, `headers` | Streamable HTTP，新型传输方式 |

### 2.4 安全：stdio 信任模型

项目级 stdio 服务器（可执行任意本地命令）需要信任审核：

```
首次加载项目 stdio MCP
        |
        v
计算配置文件指纹（SHA 哈希）
        |
        v
检查 ~/.deepagents/trust.json 是否有对应项目的记录
    |-- 已信任且指纹一致 -> 直接加载
    +-- 未信任或指纹变更 -> 过滤 stdio 服务器 + 发出警告
```

用户显式通过 `--trust-project-mcp` 或确认对话框后，指纹写入信任存储。这防止了供应链攻击（恶意项目在 `.mcp.json` 中注入任意命令）。

### 2.5 预飞检查（Pre-flight）

在实际建立 MCP 会话前：
- **stdio 服务器**：检查 `command` 是否在 `$PATH` 上（`shutil.which`）
- **远程服务器**：发送 HEAD 请求验证连通性（2 秒超时）

### 2.6 会话持久化

`MCPSessionManager` 通过 `AsyncExitStack` 维持长连接，stdio 服务器的进程在整个 CLI 会话中保持运行，避免了每次工具调用重启服务器的开销。

---

## 三、Sandbox Partners

### 3.1 统一适配器模式

所有 sandbox 实现都遵循相同结构：

```
BaseSandbox（抽象基类）
    |-- execute(command) -> ExecuteResponse
    |-- download_files(paths) -> list[FileDownloadResponse]
    +-- upload_files(files) -> list[FileUploadResponse]

具体实现：
    |-- RunloopSandbox（langchain_runloop）
    |-- DaytonaSandbox（langchain_daytona）
    |-- ModalSandbox（langchain_modal）
    |-- LangSmithSandbox（deepagents.backends.langsmith）
    +-- AgentCoreSandbox（langchain_agentcore_codeinterpreter）
```

### 3.2 各 Provider 对比

| Provider | 默认工作目录 | sandbox_id 重连 | 特殊说明 |
|----------|-------------|----------------|---------|
| runloop | `/home/user` | 支持 | REST API，按 devbox id 管理 |
| daytona | `/home/daytona` | 不支持 | 需创建新实例 |
| modal | `/workspace` | 支持 | 通过 `App.lookup` 管理应用 |
| langsmith | `/tmp` | 支持（按名称） | 支持 Docker 模板 |
| agentcore | `/tmp` | 不支持 | AWS Bedrock Code Interpreter，会话级别 |

### 3.3 Provider 工厂模式

CLI 层（`sandbox_factory.py`）实现了工厂模式：

```python
def create_sandbox(provider: str, ...) -> Generator[SandboxBackendProtocol]:
    provider_obj = _get_provider(provider)   # 按名称实例化
    backend = provider_obj.get_or_create(...)
    yield backend                            # 上下文管理器
    if should_cleanup:
        provider_obj.delete(sandbox_id=backend.id)
```

**懒导入策略**：每个 provider 的 SDK 包在实际使用时才 import，不安装时得到清晰的错误提示（`pip install 'deepagents-cli[runloop]'`）。

### 3.4 Sandbox 启动的就绪等待

所有 provider 都使用轮询策略等待 sandbox 就绪，每 2 秒检测一次，默认超时 180 秒，超时后清理并抛出 `RuntimeError`。

---

## 四、CLI 与 SDK 的交互模式

### 4.1 两种工作模式

```
本地模式（默认）                   远程模式（--server-url）
    |                                   |
    v                                   v
create_deep_agent()              RemoteAgent(url=...)
    |                                   |
    v                                   v
CompiledStateGraph               RemoteGraph（LangGraph SDK）
（在本地进程执行）                 （通过 HTTP+SSE 连接远程 server）
```

两种模式对 CLI 其余部分（Textual TUI、流式处理逻辑）**完全透明**——`RemoteAgent` 暴露与本地 `CompiledStateGraph` 相同的接口（`astream`, `aget_state`, `aupdate_state`）。

### 4.2 RemoteAgent：透明代理

`RemoteAgent` 是对 `langgraph.pregel.remote.RemoteGraph` 的薄包装，主要增加了：

1. **消息对象转换**：服务端发来的 JSON dict -> LangChain message 对象
2. **中断对象转换**：`__interrupt__` dict -> `Interrupt` 对象
3. **线程 ID 验证**：确保 config 中携带 `thread_id`

消息转换使用 dispatch table 模式，支持短名（`"ai"`, `"human"`, `"tool"`）和全名（`"AIMessage"` 等）两种格式，以兼容不同版本服务端的序列化方式。

### 4.3 execute 工具的精细权限控制

对于 `execute` 工具的 `approve_always` 决策，系统做了精细的命令签名提取：

```
"python -m pytest tests/" -> "python -m pytest"  # 包含模块名
"npm run build"           -> "npm run build"      # 包含脚本名
"python -c 'code'"        -> "python -c"          # 只记录标志（代码变化频繁）
"ls -la"                  -> "ls"                 # 普通命令只记 base cmd
```

永不批准包含以下 shell 危险模式的命令：`$(`、反引号、`>>`、`${VAR`、独立 `&`（非 `&&`）。

---

## 五、异步子智能体服务器模式

### 5.1 架构定位

`examples/async-subagent-server/` 展示了子智能体作为独立 HTTP 服务运行的完整模式：

```
Supervisor Agent（主智能体）
    |
    |-- POST /threads              <- 创建会话
    |-- POST /threads/{id}/runs    <- 提交任务（fire-and-forget）
    |-- GET  /threads/{id}/runs/{run_id}  <- 轮询状态
    +-- GET  /threads/{id}         <- 获取最终结果（values.messages）
```

### 5.2 协议兼容性

服务器实现的是 **LangGraph SDK 兼容的 REST API** 子集，而非完整的 LangGraph 服务器，这意味着：

- 任何语言（Python/Node/Go）都可以实现这个接口
- 不需要依赖 LangGraph 的完整基础设施
- 状态持久化可以选用任何存储（示例用了内存 SQLite）

### 5.3 执行模型

```python
# 提交任务 -> 立即返回 run_id（fire-and-forget）
asyncio.ensure_future(_execute_run(run_id, thread_id, user_message))
return {"run_id": run_id, "status": "pending"}

# 主智能体通过 AsyncSubAgentMiddleware 轮询
GET /threads/{id}/runs/{run_id}  -> {"status": "running" | "success" | "error"}

# 成功后读取结果（LangGraph SDK 约定的字段路径）
GET /threads/{id}  -> {"values": {"messages": [...]}}
```

---

## 六、生态连接图（完整版）

```
                 +-------------------------------+
                 |     DeepAgents Core SDK       |
                 |  create_deep_agent/middleware  |
                 +----------+----------+---------+
                            |          |
          +-----------------+          +------------------+
          v                                               v
+---------------------+                    +---------------------------+
|  ACP 层（libs/acp）  |                    |   CLI 层（libs/cli）       |
|                     |                    |                           |
|  AgentServerACP     |                    |  本地模式                  |
|  |-- 协议握手        |                    |  |-- MCP 加载/信任          |
|  |-- 多模态转换      |                    |  |-- Sandbox 工厂           |
|  |-- HITL 权限弹窗   |                    |  +-- Textual TUI          |
|  +-- 模型热切换      |                    |                           |
+--------+------------+                    |  远程模式                  |
         |                                 |  +-- RemoteAgent          |
         v                                 |      -> LangGraph Server  |
Zed / 支持 ACP 的编辑器                    +--------+-----------------+
                                                    |
                    +-------------------------------+
                    |
       +------------+------------------------+
       |                                     |
       v                                     v
+-----------------+              +--------------------+
| MCP 服务器      |              | Sandbox Partners    |
|                 |              |                     |
| stdio: 子进程   |              | runloop / daytona   |
| sse: 远程流式   |              | modal / langsmith   |
| http: HTTP 传输  |              | agentcore（AWS）    |
+-----------------+              +--------------------+
                                              |
                                              v
                                   隔离的云端/本地执行环境

+--------------------------------------------------------+
|  async-subagent-server（examples/）                    |
|  任何语言实现的 LangGraph 兼容 REST API                 |
|  主智能体通过 AsyncSubAgentMiddleware 调度              |
+--------------------------------------------------------+
```

---

## 七、探索问题解答

### Q: ACP 和 MCP 的职责如何划分？

| | ACP | MCP |
|--|-----|-----|
| 全称 | Agent Client Protocol | Model Context Protocol |
| 来源 | agentclientprotocol.com 行业标准 | Anthropic 发起，行业标准 |
| 连接对象 | 编辑器 <-> Agent | Agent <-> 外部工具服务 |
| 方向 | 用户输入 -> Agent 输出 | Agent -> 外部服务调用 |
| 功能 | 会话管理、权限弹窗、流式输出 | 工具注册、工具调用 |
| 集成层 | `libs/acp/`（独立包） | `libs/cli/`（CLI 内） |

一句话：ACP 是用户如何与 Agent 对话的协议；MCP 是 Agent 如何调用外部工具的协议。

### Q: libs/partners/ 里的 sandbox 集成是否一致？

**高度一致**，所有 partner 都：
1. 实现 `BaseSandbox` 接口（`execute`, `download_files`, `upload_files`）
2. 通过 `_ProviderXxx` 类包装 SDK 的 `get_or_create` / `delete` 生命周期
3. 通过懒导入 + 清晰错误提示处理可选依赖

**差异点**：
- AgentCore 不支持会话重连（会话级生命周期）
- Daytona 不支持按 ID 重连（已标注 `NotImplementedError`）
- 各 Provider 的默认工作目录不同（需通过 `get_default_working_dir()` 查询）

### Q: CLI 中「远程图」模式是什么使用场景？

适用于：
1. **团队共享 Agent**：Agent 部署在 LangGraph 服务器，多个 CLI 客户端共用
2. **长期任务**：Agent 在服务端运行，不受 CLI 进程生命周期影响
3. **LangSmith 集成**：直接连接 LangSmith 托管的 Agent 进行调试

---

## 八、关键设计决策总结

1. **ACP 选用行业标准而非自研**：降低了编辑器适配的门槛，Zed 等工具开箱即用。

2. **MCP 放在 CLI 层而非 SDK 层**：保持 SDK 轻量，高级用户可自行组合；CLI 用户自动获得 MCP 能力。

3. **Sandbox 统一接口 + 懒导入**：零依赖核心 + 按需安装 provider 包，实现了模块化的渐进增强。

4. **AsyncSubAgent 使用 LangGraph 兼容 REST API**：任何语言都可实现子智能体服务，不绑定 LangGraph 基础设施。

5. **安全防线在两个层次**：
   - ACP 层：`dangerous_patterns` 检查 + 按命令签名（而非整条命令）的 `approve_always`
   - MCP 层：stdio 服务器的指纹信任体系，防止项目级恶意命令注入

---

*本文档为 Layer 5 分析完成产物，五个层次的分析至此全部完成。*
