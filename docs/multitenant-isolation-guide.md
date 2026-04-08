# 多租户模式下的文件系统隔离指南

> 分析时间：2026-04-07  
> 基于源码：`backends/state.py`、`backends/store.py`、`backends/filesystem.py`、`backends/local_shell.py`、`backends/composite.py`、`backends/sandbox.py`、`libs/deepagents/THREAT_MODEL.md`

---

## 一、各 Backend 隔离能力速查

```
StateBackend      ✅ 天然隔离（per thread_id，自动）
StoreBackend      ⚠️ 默认共享，必须手动配置 namespace
FilesystemBackend ⚠️ virtual_mode=True 阻路径遍历，但共享 root_dir
LocalShellBackend ❌ 完全无隔离，多租户下禁止使用
BaseSandbox 实现  ✅ 一个 sandbox 实例 = 一个隔离容器
CompositeBackend  🔧 组合工具，路由设计决定隔离效果
```

---

## 二、StateBackend — 免费获得 per-thread 隔离

`StateBackend` 把文件存在 LangGraph state 的 `files` channel 里，而 LangGraph state 本身**按 `thread_id` 完全隔离**——不同租户用不同 `thread_id` 即可，无需任何额外配置：

```python
# 租户 A
agent.invoke(
    {"messages": [...]},
    config={"configurable": {"thread_id": "tenant-A-session-1"}}
)

# 租户 B（完全看不到 A 写入的文件）
agent.invoke(
    {"messages": [...]},
    config={"configurable": {"thread_id": "tenant-B-session-1"}}
)
```

底层读写通过 LangGraph 内部的 Pregel channel 机制实现，`thread_id` 键控制隔离边界：

```python
# StateBackend 内部（state.py）
config["configurable"][CONFIG_KEY_READ]("files", fresh=False)
# 每个 thread_id 对应独立的 checkpoint，files 不跨越边界
```

### 注意：子 agent 的文件继承

子 agent（通过 `task()` 工具调用）会继承父 agent 的 `files` 状态，这是有意设计，用于同一任务内部协作。该继承**只发生在同一 `thread_id` 内部**，不跨租户。

---

## 三、StoreBackend — 最容易踩坑的地方

StoreBackend 存在**跨租户污染的真实风险**。不设置 `namespace` 时，所有租户写入同一个 `("filesystem",)` 命名空间：

```python
# ❌ 危险：所有租户共享同一 namespace
backend = StoreBackend()
# → namespace = ("filesystem",)
# 租户 A 写的文件，租户 B 可以通过 read/grep/ls 读到！
```

### 正确做法：namespace 工厂函数绑定租户 ID

```python
# ✅ 正确：每个 user_id 对应独立 namespace
backend = StoreBackend(
    namespace=lambda ctx: ("filesystem", ctx.runtime.context.user_id)
)

# ✅ 多层隔离：组织 → 用户 → 文件系统
backend = StoreBackend(
    namespace=lambda ctx: (
        "org", ctx.runtime.context.org_id,
        "user", ctx.runtime.context.user_id,
        "files"
    )
)
```

### namespace 注入防护

`_validate_namespace` 对每个分量做字符白名单校验，防止通配符注入：

```python
# store.py：允许字符集
_NAMESPACE_COMPONENT_RE = re.compile(r"^[A-Za-z0-9\-_.@+:~]+$")
# 拒绝 * ? [ ] { } 等通配符 —— 防止 glob 注入攻击
```

### 版本迁移注意

`namespace` 参数将在 v0.5.0 变为**必填**，提前指定可避免破坏性变更。

---

## 四、FilesystemBackend — virtual_mode 是路由工具，不是安全边界

`THREAT_MODEL.md` 明确声明（威胁 T9）：**`virtual_mode` 的首要用途是配合 `CompositeBackend` 的路径路由语义，不提供多租户安全隔离。**

### virtual_mode 能做到什么

```python
# virtual_mode=True 提供：
# ✅ 阻断 .. 和 ~ 的路径遍历
# ✅ 验证解析后路径仍在 root_dir 内

# 代码路径（filesystem.py：_resolve_path）
if ".." in vpath or vpath.startswith("~"):
    raise ValueError("Path traversal not allowed")
full = (self.cwd / vpath.lstrip("/")).resolve()
full.relative_to(self.cwd)  # 逃逸则抛出 ValueError
```

### virtual_mode 做不到什么

```python
# ❌ 不能阻止同一 root_dir 下不同租户文件的相互访问
# ❌ 不能阻止 LocalShellBackend 的 shell 命令访问任意路径
# ❌ virtual_mode=False（默认）完全不限制，绝对路径可访问系统任意文件
```

### 多租户下的正确配置：每租户独立 root_dir

```python
def get_backend_for_tenant(tenant_id: str) -> FilesystemBackend:
    return FilesystemBackend(
        root_dir=f"/data/tenants/{tenant_id}",
        virtual_mode=True,  # 防止路径遍历到上层目录（其他租户）
    )
```

---

## 五、LocalShellBackend — 多租户下禁止使用

`THREAT_MODEL.md` 将此列为 **T3（High severity，已验证）** 和 **T6（Medium，已验证）**。

### 为什么 virtual_mode 无效

```python
# ❌ 即使设置了 virtual_mode=True，shell 命令完全不受限制
result = backend.execute("cat /etc/passwd")
result = backend.execute(f"cat /data/tenant-other/{secret_file}")  # 读其他租户的文件

# 底层实现（local_shell.py）
subprocess.run(
    command,
    shell=True,    # shell=True 使任何路径限制失效
    ...
    cwd=str(self.cwd),  # cwd 只影响相对路径起点，不限制绝对路径
)
```

文档中明确写道：

> `virtual_mode=True` and path-based restrictions provide NO security with shell access enabled, since commands can access any path on the system.

### 安全替代方案

多租户下需要代码执行能力时，必须使用沙箱——继承 `BaseSandbox` 实现 Docker/VM 后端，每个租户会话一个独立沙箱实例：

```python
# ✅ 正确：每个租户会话独立沙箱容器
class DockerSandbox(BaseSandbox):
    def __init__(self, container_id: str): ...
    def execute(self, command: str, *, timeout=None) -> ExecuteResponse: ...
    def upload_files(self, files): ...
    def download_files(self, paths): ...

sandbox = DockerSandbox(container_id=f"tenant-{tenant_id}-{session_id}")
```

`BaseSandbox` 已提供所有文件操作（`ls`/`read`/`write`/`edit`/`glob`/`grep`）的默认实现，子类只需实现 `execute()`、`upload_files()`、`download_files()` 三个方法。

---

## 六、CompositeBackend — 构建多层隔离的路由

`CompositeBackend` 按路径前缀将操作路由到不同 backend，可以把不同隔离级别的存储组合到统一接口：

```python
composite = CompositeBackend(
    default=StateBackend(),             # 临时文件，per thread_id 自动隔离
    routes={
        "/memories/": StoreBackend(     # 跨 thread 持久化，namespace 隔离
            namespace=lambda ctx: ("tenant", tenant_id, "memories")
        ),
        "/code/": DockerSandbox(        # 代码执行，独立容器隔离
            container_id=f"session-{session_id}"
        ),
    }
)
```

### 路由设计原则

| 路径前缀 | 推荐 Backend | 隔离机制 |
|---------|------------|---------|
| `/tmp/`、`/workspace/` | `StateBackend` | `thread_id` |
| `/memories/`、`/notes/` | `StoreBackend` + namespace | namespace tuple |
| `/code/`、`/output/` | `BaseSandbox` 实现 | 独立容器 |
| `/shared/` | `StoreBackend`（只读 namespace）| 共享但只读 |

---

## 七、推荐的多租户生产架构

```python
def create_tenant_agent(tenant_id: str, user_id: str, session_id: str):
    """为每个租户会话创建完全隔离的 Agent。"""

    # 1. 临时文件：StateBackend（per thread_id 自动隔离，无需额外配置）
    ephemeral = StateBackend()

    # 2. 持久文件：StoreBackend，namespace 绑定 tenant + user
    persistent = StoreBackend(
        namespace=lambda ctx: ("tenant", tenant_id, "user", user_id, "files")
    )

    # 3. 代码执行：独立沙箱（每个会话独立容器）
    sandbox = DockerSandbox(container_id=f"tenant-{tenant_id}-{session_id}")

    # 4. 用 CompositeBackend 按路径分发
    backend = CompositeBackend(
        default=ephemeral,
        routes={
            "/memories/": persistent,   # 跨会话持久化，按 namespace 隔离
            "/code/": sandbox,          # 代码执行在独立容器
        }
    )

    return create_deep_agent(
        model="claude-sonnet-4-5",
        backend=backend,
        memory=["/memories/AGENTS.md"],        # 租户专属记忆
        skills=["/memories/skills/"],           # 租户专属技能
        interrupt_on={"execute": True},         # 代码执行需人工审批
    )

# 调用方式：每个租户会话使用独立 thread_id
agent = create_tenant_agent("org-123", "user-456", "sess-789")
agent.invoke(
    {"messages": [HumanMessage("帮我分析数据")]},
    config={"configurable": {"thread_id": "tenant-org-123-user-456-sess-789"}}
)
```

---

## 八、污染风险汇总

| 污染场景 | 根本原因 | 严重程度 | 解决方案 |
|---------|---------|---------|---------|
| StoreBackend 跨租户可见 | 未设置 `namespace`，默认 `("filesystem",)` 全局共享 | 高 | `namespace=lambda ctx: ("tenant", tenant_id, ...)` |
| FilesystemBackend 路径逃逸 | `virtual_mode=False` 或多租户共用 `root_dir` | 高 | 每租户独立 `root_dir` + `virtual_mode=True` |
| LocalShellBackend 任意路径读写 | `shell=True`，路径限制完全失效 | 高 | 禁止在多租户下使用，替换为 `BaseSandbox` 实现 |
| 子 agent 跨租户继承文件 | 同 `thread_id` 内 `files` 状态传递 | 中 | 不同租户始终使用不同 `thread_id` |
| 记忆/技能内容注入（T1） | `memory`/`skills` 路径内容未校验直接注入 system prompt | 高 | 各租户指向独立的 backend 路径，限制写权限 |
| namespace 通配符注入 | 未验证 namespace 分量 | 低 | 使用 `StoreBackend` 内置的 `_validate_namespace`（已自动生效） |

---

## 九、不同场景的最简选型建议

### 场景 A：SaaS Web 服务（推荐组合）

```python
# 完全无本地磁盘接触，适合多租户 API 服务
backend = CompositeBackend(
    default=StateBackend(),            # 临时文件，per thread 隔离
    routes={
        "/memories/": StoreBackend(
            namespace=lambda ctx: ("tenant", tenant_id, "files")
        ),
    }
)
# 不开放 execute 工具（StateBackend 不实现 SandboxBackendProtocol，FilesystemMiddleware 自动过滤）
```

### 场景 B：代码执行平台（需要沙箱）

```python
# 每个用户会话一个独立容器
backend = CompositeBackend(
    default=StateBackend(),
    routes={
        "/workspace/": DockerSandbox(container_id=session_id),
        "/memories/": StoreBackend(namespace=lambda ctx: ("user", user_id)),
    }
)
```

### 场景 C：本地开发 CLI（单用户，允许磁盘访问）

```python
# 单用户开发工具，不涉及多租户
backend = LocalShellBackend(
    root_dir="/home/user/project",
    virtual_mode=True,
    inherit_env=False,  # 不继承环境变量，防止 API key 泄漏
)
# 强制配合 HITL 使用
create_deep_agent(..., backend=backend, interrupt_on={"execute": True})
```

---

*参考文档：*  
*- [Layer 3 扩展机制分析](./layer3-extension-analysis.md)*  
*- [deepagents SDK THREAT_MODEL](../libs/deepagents/THREAT_MODEL.md)*
