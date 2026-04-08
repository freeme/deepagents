# 八、安全模型

## 架构关系概览（ASCII）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        信任边界（Trust Boundaries）                       │
├─────────────────────────────────────────────────────────────────────────┤
│  TB1 用户/应用 ──► 框架 create_deep_agent（模型、工具、backend、prompt）   │
│  TB2 框架 ──► LLM 提供商（消息出站，模型行为不可控）                        │
│  TB3 框架 ◄── LLM 工具调用（task / shell cmd / 用户工具参数）              │
│  TB4 框架 ◄── Backend 存储（memory/skills/state 文件内容）                 │
│  TB5 Backend ──► 宿主机 OS（Filesystem / LocalShell）                   │
│                                                                         │
│  ┌──────────── SDK（libs/deepagents）──────────────────────────────┐   │
│  │ 默认 StateBackend → 无 shell；LocalShell 需显式 opt-in             │   │
│  │ Memory/Skills 原文注入 system prompt；LocalShell 无命令过滤         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  ┌──────────── CLI（libs/cli）────────────────────────────────────┐    │
│  │  HITL + 审批 UI；unicode_security 扫描工具 args；shell 白名单    │    │
│  │  langgraph dev @127.0.0.1 + LANGGRAPH_AUTH_TYPE=noop           │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8.1 威胁模型分析

### 8.1.1 「信任 LLM」的边界与假设

SDK 威胁模型（`libs/deepagents/THREAT_MODEL.md`）明确：

- 框架将**用户提供的模型、工具、system prompt、backend、memory/skills 路径**视为用户控制；**不验证**这些参数的安全性或内容
- LLM 输出的工具名与参数在 **TB3** 处重新进入框架执行
- 框架对 `subagent_type` 做白名单校验，但 **`task` 的 `description`、shell 命令串、异步任务结果**等多为**未校验的 LLM 生成文本**

**默认安全基线**：
- 默认 `StateBackend`，不实现 `SandboxBackendProtocol`，因此 **`execute` 工具在 FilesystemMiddleware 层被过滤**
- 用户必须显式配置 `LocalShellBackend` 等才能开启 shell 执行

### 8.1.2 明确不在范围内的威胁

| 类别 | 说明 |
|------|------|
| `libs/cli/` | 独立威胁模型，SDK 不负责 |
| 用户应用代码与 prompt | 用户/部署者责任 |
| LLM 越狱与模型行为 | 模型提供商责任 |
| 部署基础设施 | 部署者责任 |
| 远程 LangGraph 服务器（AsyncSubAgent） | 各云服务商责任 |
| 用户 AGENTS.md 中自写密钥 | 用户责任 |

### 8.1.3 SDK 与 CLI 威胁模型差异

| 维度 | SDK | CLI |
|------|-----|-----|
| 运行形态 | 库，编译 `CompiledStateGraph` | TUI + `langgraph dev` 子进程 + HTTP/SSE |
| Shell 控制 | 无命令白名单；`LocalShell` 是 opt-in | 白名单 + 危险模式检测；非交互模式强依赖 |
| Unicode/URL 安全 | 威胁模型未作为核心控制 | `detect_dangerous_unicode`、`check_url_safety` 用于审批 UI |
| 认证 | 无 | 本地 dev 服务器 `LANGGRAPH_AUTH_TYPE=noop`，任意本机进程可连（TB6） |
| HITL | 可选配置 `interrupt_on` | 默认为核心安全闸门 |

---

## 8.2 Unicode 安全（`unicode_security.py`）

### 8.2.1 检测的码点与类别

`_DANGEROUS_CODEPOINTS` 为固定 frozenset，包含：

| 类别 | 码点范围 |
|------|---------|
| BiDi 方向控制 | `U+202A`–`U+202E`、`U+2066`–`U+2069` |
| 零宽/不可见 | `U+200B`–`U+200F`、`U+2060`、`U+FEFF`（BOM） |
| 软连字符 | `U+00AD` |
| 组合用连接符 | `U+034F` |
| 韩文填充符 | 其他特殊字符 |

`detect_dangerous_unicode` 逐字符检查是否落在危险字符集中。

### 8.2.2 与 Prompt Injection 防护的关系

**TB3** 定位：对 **`fetch_url` / shell 命令等工具参数**做 Unicode/URL 扫描；**工具返回内容**不扫描 prompt 注入模式。

即：**降低审批 UI 被欺骗批准的风险**，不是对 LLM 语义层注入的通用防护。

### 8.2.3 URL 安全检测（`check_url_safety`）

```
检测流程：
1. 对整段 URL 调用 detect_dangerous_unicode；有则标记 suspicious + 警告
2. urlparse 取 hostname；无 hostname 则提前返回
3. _decode_hostname：对 xn-- 标签做 IDNA/punycode 解码；失败标签标为 suspicious
4. 若主机为 localhost 或合法 IP 字面量 → 跳过脚本混合检测（本地安全场景）
5. 对每个 label：_scripts_in_label 分类脚本（Latin/Cyrillic/Greek...）
   多脚本 → warning
6. _label_has_suspicious_confusable_mix：
   含 CONFUSABLES 中字符 且 多脚本时标 suspicious（单脚本同形字不标）
```

辅助工具：`strip_dangerous_unicode`、`render_with_unicode_markers`、`iter_string_values` 递归检查 dict 中的 URL 形键。

### 8.2.4 触发时机与用户通知

- **审批对话框展示前**，对工具参数格式化时集成 `check_url_safety` 与 `strip_dangerous_unicode`
- 用户看到的是**带警告的说明**，而非静默拦截

---

## 8.3 Shell 命令权限控制

### 8.3.1 白名单与 `_ShellAllowAll`

```python
class _ShellAllowAll(list):
    """哨兵子类，表示无限制 shell 访问（--shell-allow-list=all）"""

SHELL_ALLOW_ALL: list[str] = _ShellAllowAll(["__ALL__"])
```

`is_shell_command_allowed(command, allow_list)` 核心逻辑：

```
若 allow_list 是 _ShellAllowAll 实例：
  → 直接返回 True（跳过所有检测，包括 contains_dangerous_patterns）

否则：
  1. contains_dangerous_patterns（重定向、$(...)、反引号、$VAR、后台 & 等）
  2. 按管道/复合命令分段
  3. shlex.split 取首 token 与白名单集合比对
```

`RECOMMENDED_SAFE_SHELL_COMMANDS`：预置只读类命令（`ls`、`cat`、`grep`、`find` 等）。

### 8.3.2 沙箱 Backend 对执行权限的约束

```
FilesystemMiddleware._supports_execution(backend):
  → 仅当 backend 是 SandboxBackendProtocol 时返回 True
  → 否则 execute 工具根本不暴露给模型

重要警告（SDK 威胁模型）：
  virtual_mode=True 只约束文件路径 API
  LocalShellBackend 仍可用 shell 绕过路径限制（TB6）
  生产环境需容器/VM 级隔离
```

### 8.3.3 `interrupt_on` 与 HITL 层次

| Agent 类型 | HITL 继承规则 |
|-----------|-------------|
| 主 agent | `interrupt_on` 传入 `HumanInTheLoopMiddleware` |
| 声明式 `SubAgent` | `spec.get("interrupt_on", interrupt_on)` — 子级可覆盖 |
| `CompiledSubAgent` | **不继承** 顶层 `interrupt_on`，需在预编译 runnable 内自行配置 |
| `AsyncSubAgent` | **不继承** 顶层 `interrupt_on`，远程行为在远端配置 |

**警告**：CLI 的 `auto_approve` 可绕过全部 HITL（TB4）；`--shell-allow-list=all` 跳过所有 shell 命令检测。

---

## 小结

DeepAgents 的安全模型采用**分层防御**策略：

```
Layer 1（默认安全）：StateBackend 无 shell，不暴露 execute 工具
Layer 2（可选 shell）：LocalShellBackend + ShellAllowListMiddleware
Layer 3（UI 层）：HITL 审批 + Unicode/URL 检测
Layer 4（开发者）：显式文档说明安全边界，诚实标注不在范围内的威胁
```

框架选择**诚实地定义安全边界**而非过度承诺，将真正的隔离交给云沙箱（Daytona/Modal/Runloop）或容器/VM 层。
