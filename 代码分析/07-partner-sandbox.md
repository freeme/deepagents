# 七、Partner Sandbox 集成

`deepagents/libs/partners` 下各子包为 Deep Agents 的 `SandboxBackendProtocol` 提供不同运行时：**云沙箱（Daytona、Modal、Runloop）** 与 **进程内 QuickJS（JS REPL 工具链）**。

## 概述

```
                    create_deep_agent(..., backend=sandbox)
                                    │
        ┌──────────────────┬────────┴──────────┬──────────────────┐
        │                  │                   │                  │
   DaytonaSandbox    ModalSandbox       RunloopSandbox    QuickJSMiddleware
  (session+poll)    (sandbox API)      (devbox API)       (进程内 JS REPL)
        │                  │                   │                  │
   Daytona FS API    Modal open()      Runloop file API   quickjs.Context
   upload/download   read/write        upload/download    eval(js_code)
```

**注意**：QuickJS **不是**容器级沙箱，而是向 Agent 注入 `repl` 工具，与 Daytona/Modal/Runloop 不在同一安全隔离层级。

---

## 7.1 Daytona 沙箱（`partners/daytona/`）

### 执行路径：Session + 轮询

```
execute(command):
  1. process.create_session()
  2. execute_session_command(run_async=True)
  3. 轮询 get_session_command() 直至 exit_code 非空
  4. 单调时钟自管超时 → exit_code=124
```

关键实现：

```python
def _execute_via_session_logs(self, command: str, *, timeout: int) -> ExecuteResponse:
    started_at = time.monotonic()
    while True:
        result = process.get_session_command(...)
        if result.exit_code is not None:
            return ExecuteResponse(output=result.output, exit_code=result.exit_code, ...)
        if timeout != 0 and time.monotonic() - started_at >= timeout:
            return ExecuteResponse(output=f"Command timed out after {timeout}s", exit_code=124, ...)
        time.sleep(POLL_INTERVAL)
```

- 默认超时：`_default_timeout = 30 * 60`（1800s）
- `timeout=0` 语义：**无限等待**（注释明确说明）

### 文件操作

- 路径必须以 `/` 开头，否则返回 `invalid_path`
- **下载**：`FileDownloadRequest` 列表 → `sandbox.fs.download_files`
- **上传**：`FileUpload(source=content, destination=path)` → `fs.upload_files`
- 大文件**不依赖 shell 传参**（避免 ARG_MAX）

---

## 7.2 Modal 沙箱（`partners/modal/`）

### 执行

```python
def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
    proc = sandbox.exec("bash", "-c", command, timeout=effective_timeout)
    proc.wait()
    return ExecuteResponse(
        output=proc.stdout.read() + proc.stderr.read(),
        exit_code=proc.returncode,
        ...
    )
```

- 默认超时：`30 * 60`（1800s）
- `timeout=0`：无限等待

### 文件

- 使用 Modal 虚拟文件系统 **`open(path, "rb"/"wb")`** 读写
- 错误映射：解析 `modal.exception.FilesystemExecutionError` 文本 → `file_not_found` / `is_directory` / `permission_denied`

**与 Daytona 的差异**：Modal 文件 API 为单次 open 读写，无显式"批量下载"抽象。

---

## 7.3 Runloop 沙箱（`partners/runloop/`）

### 执行

```python
def execute(self, command: str, *, timeout: int | None = None) -> ExecuteResponse:
    result = devbox.cmd.execute(command, timeout=effective_timeout)
    return ExecuteResponse(
        output=result.stdout + result.stderr,
        exit_code=result.exit_code,
        ...
    )
```

### 文件

- **下载**：`self._devbox.file.download(path=path)` → bytes
- **上传**：`file.upload(path=path, file=content)`

**与其他沙箱的差异**：Runloop 的 API 最简，路径校验不如 Daytona 严格（依赖服务端报错）。

---

## 7.4 QuickJS（`partners/quickjs/`）— 嵌入式 JS REPL

### 定位说明

```
QuickJSMiddleware ≠ OS 级容器沙箱

而是向 Agent 注入一个 "repl" 工具：
  - 允许 Agent 评估 JavaScript 代码
  - 可与 Python 函数双向交互（ptc：Python-to-QuickJS callable）
  - 可选设置 timeout 和 memory_limit
  - 每次调用从零状态开始（无跨调用状态）
```

### 构造参数

```python
class QuickJSMiddleware(AgentMiddleware):
    def __init__(
        self,
        *,
        ptc: list[Callable | BaseTool] | None = None,  # Python callable → JS 外部函数
        add_ptc_docs: bool = False,                      # 是否把签名写进 system prompt
        timeout: int | None = None,                      # 单次 eval 超时（ms）
        memory_limit: int | None = None,                 # 内存限制（bytes）
    ) -> None:
```

### 典型用途

- **Text-to-SQL**：Agent 在 JS REPL 中测试 SQL 查询
- **数学计算**：Python 不方便的数值计算
- **工具组合**：通过 `ptc` 把 LangChain 工具暴露给 JS 环境

---

## 7.5 各沙箱对比

| 特性 | Daytona | Modal | Runloop | QuickJS |
|------|---------|-------|---------|---------|
| 隔离级别 | 容器 | 容器 | 容器 | 进程内 |
| 默认超时 | 1800s | 1800s | 1800s | 可选（ms） |
| 超时标识 | exit_code=124 | process.returncode | result.exit_code | 自定义 |
| 文件 API | 批量 FS API | open() | file.upload/download | 不适用 |
| 大文件安全 | ✅（专用 API） | ✅（open 写入） | ✅（upload） | 不适用 |
| ARG_MAX 绕过 | ✅ | ✅ | ✅ | 不适用 |
| Python 互操作 | ❌ | ❌ | ❌ | ✅（ptc） |
| 跨调用状态 | ✅ | ✅ | ✅ | ❌ |

---

## 7.6 超时层级说明

Harbor 评测中 `HarborSandbox` 默认超时为 **300s**，而 partner 包默认为 **1800s**。部署时需注意**跨层 timeout 谁生效**：

```
create_deep_agent(backend=DaytonaSandbox(timeout=1800))
                                │
                        HarborSandbox（评测层覆盖，300s）
                                │
                        Daytona session poll（自管超时）
```

建议：评测时在最外层（`HarborSandbox`）统一设置超时，避免内层意外覆盖。

---

## 小结

- **Daytona**：session + 轮询 + 显式 124 超时；文件走专用 FS API，最完整的错误处理
- **Modal**：sandbox API + 虚拟文件系统；错误类型映射完善，API 最接近本地 FS 语义
- **Runloop**：devbox 命令 + file API；接口最简，错误处理最少
- **QuickJS**：补充**可限时的 JS 评估工具链**，定位是让 Agent 获得「可编程计算器」而非真正的隔离沙箱
