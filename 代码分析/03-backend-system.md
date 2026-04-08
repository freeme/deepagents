# 三、Backend 后端系统

## 概述

DeepAgents 的 **Backend** 层把「虚拟文件路径上的读写、列表、搜索」以及（可选的）**Shell 执行**抽象成统一协议 `BackendProtocol` / `SandboxBackendProtocol`。具体实现包括：会话内 **LangGraph state**（`StateBackend`）、**本地磁盘**（`FilesystemBackend`）、**本地无隔离 Shell**（`LocalShellBackend`）、**LangGraph Store 持久化**（`StoreBackend`）、**按前缀路由的组合后端**（`CompositeBackend`），以及 **远程沙箱**（`sandbox.BaseSandbox` 及 `LangSmithSandbox`）。共享逻辑集中在 `utils.py`（`FileData` 构造、切片读取、字面量 grep、glob 等）。

### Backend 继承体系（ASCII）

```
                    BackendProtocol (abc.ABC)
                    - 文件: ls/read/write/edit/grep/glob
                    - 批量: upload_files/download_files
                    - 异步: als/aread/awrite/aedit/agrep/aglob/...
                              │
        ┌─────────────────────┼─────────────────────┬──────────────────┐
        │                     │                     │                  │
  StateBackend          FilesystemBackend    StoreBackend      CompositeBackend
  (LangGraph state)     (本地 FS)            (BaseStore)       (前缀路由)
        │                     │
        │                     └── LocalShellBackend
        │                         (FilesystemBackend + SandboxBackendProtocol)
        │
        └─ 仅 BackendProtocol，无 execute

                    SandboxBackendProtocol(BackendProtocol)
                    - id, execute, aexecute
                    - execute_accepts_timeout() 反射
                              │
                         BaseSandbox (ABC)
                         - ls/read/write/edit/grep/glob 默认用 execute 实现
                              │
                         LangSmithSandbox
                         - 包装 langsmith Sandbox
```

**说明**：`CompositeBackend` 在源码中只继承 `BackendProtocol`，但实现了 `execute` / `aexecute`，在默认后端为 `SandboxBackendProtocol` 时可委派执行。`LocalShellBackend` 通过多重继承同时是文件系统后端与沙箱后端。

---

## 3.1 BackendProtocol — 统一接口

定义位置：`protocol.py`

### 核心方法与同步/异步对

| 同步 | 异步 | 说明 |
|------|------|------|
| `ls` | `als` | 目录列表；默认 `als` 用 `asyncio.to_thread(self.ls, path)` |
| `read` | `aread` | 分页读；`aread` 同理 to_thread |
| `write` | `awrite` | 新建文件（已存在则失败） |
| `edit` | `aedit` | 精确字符串替换 |
| `grep` | `agrep` | **字面量**子串搜索（非正则） |
| `glob` | `aglob` | glob 匹配文件 |
| `upload_files` | `aupload_files` | 批量上传 `(path, bytes)` |
| `download_files` | `adownload_files` | 批量下载路径列表 |

### 数据类型体系

| 名称 | 类型 | 说明 |
|------|------|------|
| `FileFormat` | `Literal["v1","v2"]` | v1：按行 `list[str]`；v2：`str` + `encoding` |
| `FileOperationError` | `Literal` | `file_not_found` / `permission_denied` / `is_directory` / `invalid_path` |
| `FileData` | `TypedDict` | `content`, `encoding`（`utf-8`/`base64`），可选时间戳 |
| `FileInfo` | `TypedDict` | `path` 必填；`is_dir`/`size`/`modified_at` 可选 |
| `GrepMatch` | `TypedDict` | `path`, `line`, `text` |
| `ReadResult` | `dataclass` | `error` / `file_data` |
| `WriteResult` | `dataclass` | `error` / `path` / `files_update`（废弃） |
| `EditResult` | `dataclass` | `error` / `path` / `files_update` / `occurrences` |
| `LsResult` | `dataclass` | `error` / `entries` |
| `GrepResult` | `dataclass` | `error` / `matches` |
| `GlobResult` | `dataclass` | `error` / `matches` |
| `FileDownloadResponse` | `dataclass` | `path`, `content`, `error` |
| `FileUploadResponse` | `dataclass` | `path`, `error` |
| `ExecuteResponse` | `dataclass` | `output`, `exit_code`, `truncated` |
| `BackendFactory` | `TypeAlias` | `Callable[[ToolRuntime], BackendProtocol]` |

### 批量操作

`upload_files` / `download_files` 设计为**按输入顺序**返回列表，支持**部分成功**（逐条带 `error`）。

---

## 3.2 SandboxBackendProtocol — 沙箱执行扩展

`SandboxBackendProtocol` 继承 `BackendProtocol`，增加：

- **`id: str`**：实例唯一标识（子类实现）
- **`execute(command, *, timeout=None) -> ExecuteResponse`**
- **`aexecute`**：异步包装；若 `execute_accepts_timeout(type(self))` 为真，则 `to_thread(self.execute, command, timeout=timeout)`，否则不传 `timeout`（避免旧实现不支持 `timeout` 形参时出错）

### `execute_accepts_timeout` 反射检测机制

```python
@lru_cache(maxsize=128)
def execute_accepts_timeout(cls) -> bool:
    try:
        sig = inspect.signature(cls.execute)
        return "timeout" in sig.parameters
    except Exception:
        logger.debug(...)
        return False  # 无法 introspect 时假定不支持
```

---

## 3.3 StateBackend — 内存/状态存储

文件：`state.py`

### 基于 LangGraph state 的读写机制

```
LangGraph Pregel 运行时
        │
        ├─ CONFIG_KEY_READ  → _read_files() → 读取当前 superstep 快照
        └─ CONFIG_KEY_SEND  → _send_files_update() → dict-merge reducer 追加写
```

- **`_read_files`**：`read("files", fresh=False)`，同一 superstep 内看到的是**一致快照**，当前步内排队写入在节点边界才可见
- **`_send_files_update`**：`send([("files", update)])`，利用 dict-merge reducer 做部分更新
- 若在图外调用，`_get_config` 会抛出明确 `RuntimeError`，提示需在 `create_deep_agent` 上下文中使用

### FileData v1/v2 格式兼容

- 初始化参数 `file_format: FileFormat = "v2"`
- `_prepare_for_storage` 在 v1 时调用 `utils._to_legacy_file_data`
- 读取、列表、glob 等处对 **legacy `list[str]` content** 做兼容处理

### 限制

- **`upload_files`**：未实现，抛 `NotImplementedError` 并提示通过 `invoke` 传入内存文件
- **`download_files`**：从 state 读出，按 `encoding` 转 `bytes`（utf-8 或 base64 解码）

---

## 3.4 FilesystemBackend — 本地文件系统

文件：`filesystem.py`

### `root_dir` 与 `virtual_mode`

```
virtual_mode=True（推荐）:
  路径视为以 cwd 为根的虚拟绝对路径
  禁止 ".." 和 "~"
  解析后必须落在 cwd 下（路径逃逸检测）

virtual_mode=False（默认，已废弃）:
  绝对路径按原样使用，相对路径相对 cwd
  不能防止代理访问 root_dir 之外
  未指定时会发 DeprecationWarning
```

### 二进制与 base64

```python
if _get_file_type(file_path) != "text":
    content = base64.standard_b64encode(raw_bytes).decode()
    return FileData(content=content, encoding="base64")
```

### grep / glob 实现

| 场景 | 实现方式 |
|------|----------|
| grep（优先） | `rg --json -F`，超时 30s |
| grep（fallback） | Python 递归搜索，受 `max_file_size_bytes` 限制 |
| glob | `search_path.rglob(pattern)`；virtual_mode 下路径映射 |

---

## 3.5 LocalShellBackend — 本地 Shell 执行

文件：`local_shell.py`，继承 `FilesystemBackend` + `SandboxBackendProtocol`

### 执行机制

```python
subprocess.run(
    command,
    shell=True,
    cwd=str(self.cwd),
    env=self._env,
    timeout=effective_timeout,
    capture_output=True
)
```

### 输出截断策略

```
max_output_bytes = 100_000（默认）
超过时：截断 + 追加说明文本 + truncated=True

stderr 行加 "[stderr]" 前缀，与 stdout 合并
非零退出码：末尾附加 "Exit code: ..."
TimeoutExpired：exit_code=124（与 GNU timeout 约定一致）
```

### 安全说明

文档**明确声明**：LocalShellBackend **不是沙箱**，Shell 命令可以访问整个系统文件。生产环境应使用 StateBackend 或远程沙箱（Daytona/Modal/Runloop）。

---

## 3.6 CompositeBackend — 组合后端

文件：`composite.py`

### 路由策略（最长前缀优先）

```
CompositeBackend
├── default: FilesystemBackend（主文件系统）
└── routes:
    ├── "/memories/" → StoreBackend（持久记忆）
    ├── "/uploads/"  → StateBackend（上传文件）
    └── ...（用户自定义路由）

_route_for_path 按 sorted_routes 最长前缀匹配
路径等于前缀无斜杠时可映射到子后端 "/"
```

### 各操作的聚合行为

| 操作 | 行为 |
|------|------|
| `ls("/")` | 合并 default 列表 + 各路由前缀作为虚拟目录 |
| `grep` | 未命中具体路由时搜索 default + 所有路由后端并聚合 |
| `glob` | 未命中具体路由时 default + 各路由均搜索 |
| `execute` | **始终委托 default**；若 default 非 SandboxBackendProtocol 则 NotImplementedError |
| `upload/download` | 按目标后端分组，保持原始输入顺序填回结果 |

---

## 3.7 StoreBackend — 持久化存储

文件：`store.py`

### 基于 LangGraph BaseStore 的持久存储

```python
def _get_store() -> BaseStore:
    return (
        self._store                    # 显式注入
        or get_store()                 # LangGraph 运行时注入
        or raise RuntimeError(...)     # 明确报错
    )
```

- `ls` / `grep` / `glob` 通过 **`_search_store_paginated`** 拉全量再本地过滤
- 异步方法使用 store 原生 `aget`/`aput` 避免阻塞事件循环

### BackendContext 与 NamespaceFactory

```python
@dataclass
class BackendContext:
    state: AgentState
    runtime: Runtime[ContextT]

NamespaceFactory = Callable[[BackendContext], tuple[str, ...]]
```

- **`_validate_namespace`**：每段为非空字符串，且仅允许安全字符集（防 glob 注入）
- 未提供 `namespace` 时走 **legacy 路径**（`assistant_id` + `"filesystem"`），并发 DeprecationWarning

---

## 3.8 LangSmithSandbox — 云端沙箱

文件：`langsmith.py`，继承 `BaseSandbox`

### 执行

```python
def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
    result = self._sandbox.run(command, timeout=effective_timeout)
    return ExecuteResponse(
        output=result.stdout + result.stderr,
        exit_code=result.exit_code,
        truncated=False,
    )
```

- 默认超时 `30 * 60` 秒
- **`write` 覆盖**：用 SDK `write` 避免通过 shell 传内容触发 **ARG_MAX** 限制

### BaseSandbox 基类（`sandbox.py`）

```
BaseSandbox (ABC)
├── execute, upload_files, download_files, id（抽象方法）
├── read：服务端 Python 脚本分页，文本 500KiB 上限
├── write：先检查存在性，再 upload_files 传内容
└── edit：
    ├── 小负载（<50000B）：内联 JSON
    └── 大负载：上传临时文件再服务端替换
```

---

## 数据类型体系（汇总）

```
BackendProtocol
├── BackendFactory = Callable[[ToolRuntime], BackendProtocol]
├── FileFormat = Literal["v1", "v2"]
├── FileOperationError = Literal["file_not_found", "permission_denied", "is_directory", "invalid_path"]
├── FileData = TypedDict(content, encoding, created_at?, modified_at?)
├── FileInfo = TypedDict(path, is_dir?, size?, modified_at?)
├── GrepMatch = TypedDict(path, line, text)
│
├── ReadResult(error?, file_data?)
├── WriteResult(error?, path?, files_update?)         ← files_update 已废弃
├── EditResult(error?, path?, files_update?, occurrences?)
├── LsResult(error?, entries?)
├── GrepResult(error?, matches?)
└── GlobResult(error?, matches?)

SandboxBackendProtocol
└── ExecuteResponse(output, exit_code, truncated)

Batch Operations
├── FileUploadResponse(path, error?)
└── FileDownloadResponse(path, content?, error?)
```

---

## 关键设计决策

1. **统一虚拟路径 + 多种存储**：所有后端对代理暴露以 `/` 为主的逻辑路径；`FilesystemBackend` 的 `virtual_mode` 与 `CompositeBackend` 的路由剥离配合，得到稳定路径语义

2. **状态更新不通过 `files_update` 回传**：`StateBackend` 强制走 `CONFIG_KEY_SEND`，废弃在结果里塞 `files_update`

3. **异步默认线程卸载**：`BackendProtocol` 的 `a*` 方法大量 `asyncio.to_thread`，避免阻塞事件循环；`StoreBackend` 另提供原生 async API

4. **沙箱与超时兼容**：`execute_accepts_timeout` 解决旧后端未实现 `timeout` 形参的兼容问题

5. **安全边界诚实说明**：`FilesystemBackend` / `LocalShellBackend` 文档强调**非沙箱**、Shell 可绕过路径限制；生产推荐 state/store/真沙箱

6. **字面量 grep**：协议与工具使用子串匹配（非正则），避免用户传入特殊正则字符导致意外行为

7. **批量 API 面向 LLM/工具**：`FileOperationError` 与逐文件 `error` 支持模型理解与重试，而非整体抛异常
