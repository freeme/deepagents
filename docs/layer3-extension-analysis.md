# Layer 3：扩展机制分析 — 如何"插入"新能力

> 分析时间：2026-04-07  
> 分析范围：`middleware/__init__.py`、`middleware/async_subagents.py`、`middleware/skills.py`、`backends/composite.py`、`backends/state.py`、`backends/sandbox.py`、`backends/local_shell.py`、`backends/langsmith.py`  
> 核心问题：deepagents 的三个扩展维度（tool / middleware / backend）各自的实现边界在哪里？

---

## 一、中间件设计哲学 — middleware/__init__.py

`middleware/__init__.py` 是整个包的「设计文档」，它回答了最核心的设计问题：

### 1.1 为什么用中间件而不是 plain tool？

```
plain tool = 只在 LLM 决定调用它时运行
middleware  = 在每次 LLM 调用前后都运行
```

| 能力 | plain tool | middleware |
|------|-----------|-----------|
| 动态过滤工具列表 | ❌ | ✅ |
| 注入 system prompt | ❌ | ✅ |
| 变换消息历史 | ❌ | ✅ |
| 跨轮次维护状态 | ❌ | ✅ |

### 1.2 选择规则（官方文档明确给出）

**使用 middleware**，当工具需要：
- 按调用粒度修改 system prompt 或工具列表
- 跨轮次追踪状态
- 对所有 SDK 使用者可用（不只是 CLI）

**使用 plain tool**，当：
- 函数是无状态且自包含的
- 无需修改 system prompt 或请求
- 工具只属于特定消费者（如 CLI 专用）

---

## 二、AsyncSubAgentMiddleware — 异步子智能体

### 2.1 架构：任务即 LangGraph Thread

异步子智能体**不是本地 Python 对象**，而是运行在远端 Agent Protocol 服务器上的独立 LangGraph graph。

```
主 Agent
  │
  │ start_async_task(description, subagent_type)
  ▼
LangGraph SDK ──────→ 远端 Agent Protocol 服务器
                            │
                            ├── client.threads.create()   → thread_id（即 task_id）
                            └── client.runs.create()      → run_id
  │
  ← 立即返回 task_id（非阻塞）
  │
主 Agent 继续其他工作...

（稍后）
  │ check_async_task(task_id)
  ▼
client.runs.get(thread_id, run_id) → status
（如果 success）client.threads.get(thread_id) → values["messages"][-1]
```

**关键设计**：`task_id` == `thread_id`，任务状态存在 LangGraph state 的 `async_tasks` 字典中，跨摘要事件持久化不丢失。

### 2.2 _ClientCache — 懒加载 SDK 客户端

```
_ClientCache(agents: dict[str, AsyncSubAgent])
  │
  ├── _sync: dict[(url, frozenset(headers)), SyncLangGraphClient]
  └── _async: dict[(url, frozenset(headers)), LangGraphClient]

缓存键：(url, frozenset(headers.items()))
```

同一 (url, headers) 的多个 async subagent 共享一个 client 实例，避免连接重复创建。  
`x-auth-scheme: langsmith` 默认注入，除非用户已在 headers 中提供。

### 2.3 五个工具的完整语义

| 工具 | 行为 | 关键参数 |
|------|------|---------|
| `start_async_task` | 创建 thread + run，写入 state | `description`, `subagent_type` |
| `check_async_task` | 获取 run status，成功时拿 thread values | `task_id` |
| `update_async_task` | 在同一 thread 上创建新 run（中断旧 run） | `task_id`, `message`；用 `multitask_strategy="interrupt"` |
| `cancel_async_task` | 取消当前 run | `task_id` |
| `list_async_tasks` | 批量获取所有 task 的实时状态 | `status_filter`；async 版本用 `asyncio.gather` 并发 |

### 2.4 状态机与终态优化

```python
_TERMINAL_STATUSES = frozenset({"cancelled", "success", "error", "timeout", "interrupted"})
```

处于终态的任务，`list_async_tasks` 直接返回缓存状态，跳过远端 API 调用——批量 list 时的重要优化。

### 2.5 系统提示中的关键行为约束

`ASYNC_TASK_SYSTEM_PROMPT` 通过提示工程约束 LLM 行为：

```
- 启动后立即返回用户，永不自动 check
- check 每次用户请求只调用一次，不轮询
- 历史状态永远是过期的——必须调工具获取实时状态
- 任务 ID 永远完整显示，不截断
```

---

## 三、SkillsMiddleware — 技能文件加载

### 3.1 SKILL.md 文件格式

```
/skills/user/web-research/
├── SKILL.md          # 必须：YAML frontmatter + markdown 说明
└── helper.py         # 可选：辅助脚本
```

SKILL.md 头部 YAML frontmatter（遵循 Agent Skills 规范 https://agentskills.io/specification）：

```yaml
---
name: web-research            # 必须，1-64 字符，小写字母数字和连字符
description: "..."            # 必须，1-1024 字符
license: MIT                  # 可选
compatibility: "Python 3.10+" # 可选，1-500 字符
allowed-tools: read_file grep # 可选，空格分隔（也兼容逗号分隔）
metadata:                     # 可选，任意 key-value
  version: "1.0"
---

# Web Research Skill
...（markdown 说明正文）...
```

### 3.2 Progressive Disclosure（渐进式暴露）

技能使用**两阶段暴露**模式：

```
第一阶段（before_agent）：
  扫描所有 source 目录 → 解析 SKILL.md frontmatter → 写入 state["skills_metadata"]
  LLM 收到：名称 + 描述 + 路径（约 100 tokens/技能）

第二阶段（LLM 决定时）：
  LLM 主动 read_file("/skills/user/web-research/SKILL.md") 读完整指令
  → 按指令执行
```

**优势**：即使注册了 100 个技能，系统提示只增加约 10,000 tokens；全文只在需要时加载。

### 3.3 多 Sources 优先级（后者覆盖前者）

```python
sources = [
    "/skills/base/",      # 基础技能（最低优先级）
    "/skills/user/",      # 用户技能
    "/skills/project/",   # 项目技能（最高优先级）
]
# 同名技能：project > user > base（last wins）
```

来源名从路径最后一段派生，展示在系统提示中（如 `Project Skills (higher priority)`）。

### 3.4 加载时机与幂等性

```
before_agent(state, runtime, config):
  if "skills_metadata" in state:
    return None  ← 幂等：只加载一次（同 MemoryMiddleware 的策略）

  for source_path in self.sources:
    backend.ls(source_path) → 找所有子目录
    backend.download_files([...SKILL.md paths]) → 批量下载
    → 解析 frontmatter → 收集 SkillMetadata
```

### 3.5 安全限制

- `MAX_SKILL_FILE_SIZE = 10MB`：防止超大 SKILL.md DoS
- 名称不合规只警告，不拒绝（向后兼容）
- 描述超过 1024 字符自动截断

---

## 四、CompositeBackend — 路径前缀路由

### 4.1 路由策略：最长前缀优先

```python
sorted_routes = sorted(routes.items(), key=lambda x: len(x[0]), reverse=True)
# 示例：routes = {"/memories/": StoreBackend(), "/cache/": StoreBackend()}
# 排序后：[("/memories/", ...), ("/cache/", ...)]
# 路径 "/memories/notes.txt" → 匹配 "/memories/" → StoreBackend
# 路径 "/tmp/file.txt"       → 无匹配 → default（StateBackend）
```

路径归一化规则：
- `/memories` (无尾斜杠) → 路由到对应 backend，backend 收到 `/`
- `/memories/notes.txt` → 路由到对应 backend，backend 收到 `/notes.txt`

### 4.2 根目录 ls("/") — 聚合视图

```
ls("/") 的结果：
  ┌─ default.ls("/")       → 默认后端的文件
  ├─ FileInfo("/memories/", is_dir=True)  ← 虚拟目录项
  └─ FileInfo("/cache/", is_dir=True)     ← 虚拟目录项
（结果按路径排序）
```

各路由后端的真实内容只在 `ls("/memories/")` 时才展开。

### 4.3 grep 和 glob 的广播模式

```
grep(pattern, path=None) 或 grep(pattern, path="/"):
  ├─ default.grep(pattern, path, glob) → 合并结果
  └─ for route, backend in routes:
       backend.grep(pattern, "/", glob) → 路径前缀修复后合并

glob(pattern, path="/")（非精确路由时同上）：
  ├─ default.glob(pattern, path) → 合并结果
  └─ for route, backend in routes:
       route_pattern = 去掉 pattern 中 route_prefix 的部分
       backend.glob(route_pattern, "/") → 修复路径后合并
（结果按路径排序，保证确定性）
```

### 4.4 upload_files / download_files — 批量路由

```
download_files(["/memories/a.md", "/tmp/b.txt", "/memories/c.md"])
  │
  ├─ 按后端分组：
  │   StoreBackend  → [("/memories/a.md", 0), ("/memories/c.md", 2)]
  │   StateBackend  → [("/tmp/b.txt", 1)]
  │
  ├─ 每个后端只调用一次（批量），结果放回原始位置
  └─ 返回 [response[0], response[1], response[2]]（原始顺序）
```

**性能设计**：N 个后端、M 个文件 → 只发出 N 次网络请求（而非 M 次）。

### 4.5 execute — 永远走默认后端

```python
def execute(self, command: str, ...) -> ExecuteResponse:
    if isinstance(self.default, SandboxBackendProtocol):
        return self.default.execute(command, ...)
    raise NotImplementedError(...)
```

Shell 命令**不可路由**——执行环境由 default backend 唯一决定，routed backend 不参与执行。

---

## 五、StateBackend — LangGraph State 存储

### 5.1 文件存储在 LangGraph 的哪个位置

```
LangGraph state schema：
  messages: Annotated[list[AnyMessage], add_messages]
  files:    Annotated[dict[str, FileData], dict_merge_reducer]  ← StateBackend 写入这里
  todos:    ...
  ...
```

`files` channel 使用 dict-merge reducer：每次 update 只需提供**变更的文件**，未变更的文件由 reducer 自动保留。

### 5.2 CONFIG_KEY_READ / CONFIG_KEY_SEND 内部机制

StateBackend 通过 LangGraph 内部 Pregel 机制读写 state：

```python
# 读：从当前 superstep 开始时的 checkpoint 读（fresh=False）
config["configurable"][CONFIG_KEY_READ]("files", fresh=False)
# 写：队列化写入（写入在 node boundary 才生效）
config["configurable"][CONFIG_KEY_SEND]([("files", {file_path: new_data})])
```

**重要约束**：同一 step 内的写入不会立即对 read 可见——每个 step 看到的是**一致的 snapshot**。

### 5.3 v1 vs v2 格式

| | v1（旧格式）| v2（默认）|
|--|-----------|---------|
| content 存储 | `list[str]`（按 `\n` 分割的行列表）| `str`（原始字符串）|
| encoding 字段 | 无 | 有（"utf-8" 或 "base64"）|
| 向后兼容 | 读操作自动处理 legacy 格式 | - |

### 5.4 设计约束

- `upload_files` 尚未实现（抛出 NotImplementedError）
- 只能在 LangGraph graph 执行上下文中使用（否则 `get_config()` 失败）
- `ls(path)` 手动模拟目录树（从 flat dict 中推导子目录）

---

## 六、BaseSandbox — 沙箱后端基类

### 6.1 核心设计：一个 execute，所有操作

```
BaseSandbox（抽象类）
  │
  ├── execute()        ← 唯一必须实现的方法（+ upload_files + download_files）
  ├── ls()             → python3 -c "os.scandir..."  via execute()
  ├── read()           → python3 -c "paginated read" via execute()
  ├── write()          → execute(存在性检查) + upload_files(文件内容)
  ├── edit()           → _edit_inline or _edit_via_upload
  ├── glob()           → python3 -c "glob.glob..."   via execute()
  └── grep()           → grep -rHnF ...              via execute()
```

子类只需实现 3 个方法：`execute()`、`upload_files()`、`download_files()`，所有高层操作自动可用。

### 6.2 base64 编码——避免 shell 注入

所有脚本中的参数（路径、glob 模式等）都通过 base64 编码传入：

```python
path_b64 = base64.b64encode(path.encode("utf-8")).decode("ascii")
cmd = f"""python3 -c "
path = base64.b64decode('{path_b64}').decode('utf-8')
..."
```

**原因**：直接插值路径（如包含引号、特殊字符的路径）会导致 shell 命令出错或注入风险。

### 6.3 edit 操作的双路径策略

```
edit(file_path, old_string, new_string):
  payload_size = len(old_string.encode()) + len(new_string.encode())
  │
  ├── payload_size ≤ 50,000 bytes → _edit_inline
  │     单次 execute()：heredoc 传 base64 payload → 服务端 Python 完成替换
  │
  └── payload_size > 50,000 bytes → _edit_via_upload
        upload_files([old_tmp, new_tmp]) → 2 次上传
        execute(_EDIT_TMPFILE_TEMPLATE) → 服务端读 tmp 文件、替换、删除 tmp
        （源文件永远不离开沙箱）
```

大文件编辑的安全保证：**源文件内容不跨越网络边界**，只有 old/new strings 上传。

---

## 七、LocalShellBackend — 无隔离本地执行

### 7.1 继承层次

```
FilesystemBackend          ← 文件读写（本地磁盘）
       +
SandboxBackendProtocol     ← execute() 接口
       ↓
LocalShellBackend          ← subprocess.run(shell=True)
```

### 7.2 execute() 实现的安全边界

```python
subprocess.run(
    command,
    shell=True,           # 意图明确：LLM 控制的 shell 执行
    capture_output=True,
    timeout=effective_timeout,
    env=self._env,        # 默认空环境（除非 inherit_env=True）
    cwd=str(self.cwd),
)
```

**无任何限制**：
- `virtual_mode=True` 对 shell 无效（只限制文件系统路径 API）
- 可访问系统上任何路径
- 可安装包、修改系统配置、建立网络连接

**推荐的唯一防护**：配合 `HumanInTheLoopMiddleware`，让用户审批所有操作。

### 7.3 输出处理

```
stdout + stderr（每行前缀 [stderr]）→ 合并字符串
超过 max_output_bytes（默认 100,000 字节）→ 截断 + 提示
超时（默认 120s）→ exit_code 124（标准超时码）
所有异常 → 返回 ExecuteResponse（不向上抛出）
```

---

## 八、LangSmithSandbox — 云端沙箱

### 8.1 最简实现：只覆盖三个方法

```python
class LangSmithSandbox(BaseSandbox):
    def execute(self, command, *, timeout=None) -> ExecuteResponse:
        result = self._sandbox.run(command, timeout=...)
        return ExecuteResponse(output=..., exit_code=..., truncated=False)

    def write(self, file_path, content) -> WriteResult:
        # 覆盖 BaseSandbox.write（BaseSandbox 用 execute 传内容，会触发 ARG_MAX）
        self._sandbox.write(file_path, content.encode("utf-8"))

    def upload_files/download_files → SDK 原生方法
```

`write()` 覆盖的原因：BaseSandbox 的 `write()` 通过 heredoc 传文件内容（`execute()` 请求体），Linux `ARG_MAX` 约为 128KB，大文件会失败；LangSmith SDK 使用 HTTP 传文件内容，没有此限制。

---

## 九、SummarizationToolMiddleware — 手动压缩工具

### 9.1 compact_conversation 工具

与 `SummarizationMiddleware` 的**自动压缩**相对，`SummarizationToolMiddleware` 暴露一个 LLM 可主动调用的工具：

```
compact_conversation（无参数）

│ 检查是否满足 50% 阈值
├── 未满足 → 返回 "nothing to compact"
│
└── 满足 →
    ├── 计算 cutoff_index
    ├── _create_summary(to_summarize)   ← LLM 调用
    ├── _offload_to_backend(...)        ← 写 /conversation_history/{thread_id}.md
    └── 写入 _summarization_event（与自动压缩共享同一 state key）
```

### 9.2 触发门槛：自动压缩阈值的 50%

```python
# 自动压缩触发：fraction=0.85（占满 85% 上下文）
# compact_conversation 可用：fraction=0.85 × 0.5 = 0.425（42.5%）

for kind, value in trigger_conditions:
    if kind == "fraction":
        threshold = int(max_input_tokens * value * 0.5)
        if should_summarize_based_on_reported_tokens(messages, threshold):
            return True  # 允许手动压缩
```

**意图**：让 agent 可以在自动压缩之前主动整理，避免等到 85% 才被动压缩。CLI 也可通过这个工具提供「Compact」按钮。

---

## 十、AnthropicPromptCachingMiddleware — 来自 langchain_anthropic

这个中间件**不在 deepagents 包中**，而是直接从 `langchain_anthropic.middleware` 导入：

```python
from langchain_anthropic.middleware import AnthropicPromptCachingMiddleware

AnthropicPromptCachingMiddleware(unsupported_model_behavior="ignore")
# unsupported_model_behavior="ignore" → 非 Anthropic 模型时静默跳过
```

### 10.1 装配位置的设计含义

```
主 agent middleware 栈：
  ... 所有工具中间件 ...
  ← 用户自定义 middleware ←
  AnthropicPromptCachingMiddleware   ← 这里！（倒数第 3）
  MemoryMiddleware                   ← Memory 在 Caching 之后
  HumanInTheLoopMiddleware           ← HITL 在最后
```

**Caching 要在 Memory 之前**：Memory 内容每轮可能变化（用户更新了 AGENTS.md），放在 Caching 之后会导致 Anthropic 无法命中 prompt cache prefix（前缀变化则 cache miss）。

---

## 十一、扩展点全景

```
扩展维度 1：Tool（最轻量）
─────────────────────────────────────────────────────────────
tools=[my_function]               ← 传给 create_deep_agent()
  适用：无状态、无 prompt 修改、消费者特定
  粒度：LLM 决定调用时

扩展维度 2：Middleware（中量）
─────────────────────────────────────────────────────────────
middleware=[MyMiddleware()]        ← 插入在 Caching 之前
  适用：需要跨轮次状态、需要修改 system prompt 或工具列表
  粒度：每次 LLM 调用前后

  可用的三个钩子：
  ├── before_agent(state, runtime, config) → 初始化（每 invoke 一次）
  ├── wrap_model_call(request, handler)    → LLM 调用前后
  └── wrap_tool_call(tool_name, args, handler) → 工具调用前后（可选）

  State 扩展：
  └── class MyState(AgentState): ...
      state_schema = MyState        ← 声明在 AgentMiddleware 子类上

扩展维度 3：Backend（最重量）
─────────────────────────────────────────────────────────────
backend=MyBackend()               ← 传给 create_deep_agent()
  适用：替换文件存储策略、改变执行环境
  粒度：会话级（整个 agent 生命周期）

  组合策略：
  └── CompositeBackend(
          default=StateBackend(),
          routes={"/memories/": StoreBackend(), "/code/": LangSmithSandbox(...)}
      )

扩展维度 4：AsyncSubAgent（云端异步）
─────────────────────────────────────────────────────────────
AsyncSubAgent(name, description, graph_id, url)  ← 运行在远端
  适用：长时间运行的任务、专业化远端部署
  粒度：任务级（跨越多个 agent 调用）
```

---

## 十二、遗留问题（转交 Layer 4）

- [ ] `libs/deepagents/tests/` — 有没有覆盖中间件链的集成测试？
- [ ] `FilesystemBackend` — 本地文件系统后端的路径限制策略（`virtual_mode` 详情）
- [ ] `StoreBackend` — 持久化存储后端（相比 StateBackend 的跨 thread 持久化？）
- [ ] `THREAT_MODEL.md` — CLI 识别了哪些具体威胁？LocalShellBackend 的风险如何被文档化？
- [ ] 测试策略 — `SummarizationToolMiddleware` 的触发阈值测试？
- [ ] 并发安全 — `CompositeBackend.upload_files` 中 `backend_batches` 的序列 for 循环（非并发），是否有必要改为并发？

---

## 十三、Layer 3 总结：扩展机制的关键设计决策

```
✅ 扩展维度正交清晰      tool/middleware/backend 三个维度互不依赖
✅ 异步子智能体非本地化   基于远端 Agent Protocol，真正意义上的分布式 agent
✅ Skills 渐进式暴露     元数据 + 按需加载，1000 个技能也不爆 context
✅ CompositeBackend 路径路由  同一 agent 可用不同存储策略，按路径分发
✅ BaseSandbox 最小接口  只需实现 3 个方法，其余 7 个操作自动派生
✅ base64 防注入         所有沙箱脚本参数 base64 编码，无 shell 注入风险
✅ edit 双路径           50KB 阈值内 inline，超过则 upload+服务端替换（源文件不出沙箱）
✅ compact_conversation  50% 阈值让 agent 主动管理上下文，不等被动触发
✅ Caching 边界精确      AnthropicPromptCachingMiddleware 在 Memory 之前，保证 prefix 稳定
```

---

*上一步：[Layer 2 - 运行机制分析](./layer2-runtime-analysis.md)*  
*下一步：[Layer 4 - 工程质量分析](./layer4-quality-analysis.md)（待创建）*
