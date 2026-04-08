# DeepAgents Layer 4：工程质量分析

> 生成时间：2026-04-07  
> 分析范围：测试策略、安全威胁模型、CI 流程、代码规范  
> 参考文件：`libs/deepagents/tests/`、`libs/evals/`、`.github/workflows/`、`THREAT_MODEL.md`、`.pre-commit-config.yaml`

---

## 概要

DeepAgents 的工程质量体系由四个相互配合的子系统构成：

```
┌──────────────────────────────────────────────────────────────┐
│                      工程质量保障体系                          │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  代码规范     │ │  测试体系     │  │    CI 流水线        │  │
│  │  (pre-commit)│ │  (3层金字塔) │  │   (GitHub Actions) │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              安全威胁模型 (THREAT_MODEL.md × 2)       │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 一、测试体系：三层金字塔

DeepAgents 的测试按正确性保证强度分为三层：

```
                     ┌─────────────────┐
                     │  Evals 行为评测  │  ← 真实 LLM + LangSmith 追踪
                     │  (libs/evals/)  │
                    /└─────────────────┘\
                   /                     \
          ┌────────────────────────────────┐
          │     Integration Tests          │  ← 真实 API Key 必需
          │ (tests/integration_tests/)     │
         /└────────────────────────────────┘\
        /                                    \
┌───────────────────────────────────────────────┐
│              Unit Tests                        │  ← Fake LLM，完全确定性
│         (tests/unit_tests/)                    │
└───────────────────────────────────────────────┘
```

### 1.1 单元测试层（Unit Tests）

**目录结构与测试组织**

```
tests/unit_tests/
├── test_end_to_end.py          ← 核心：用 FakeLLM 驱动完整 agent 执行（2374+ 行）
├── test_graph.py               ← create_deep_agent 装配逻辑
├── test_middleware.py          ← 中间件链测试（同步）
├── test_middleware_async.py    ← 中间件链测试（异步）
├── test_subagents.py           ← 子智能体委派行为
├── test_async_subagents.py     ← 异步子智能体
├── test_local_shell.py         ← LocalShellBackend 安全边界
├── backends/                   ← 每个 Backend 独立测试套
│   ├── test_filesystem_backend.py
│   ├── test_state_backend.py
│   ├── test_composite_backend.py
│   ├── test_sandbox_backend.py
│   ├── test_protocol.py
│   ├── test_file_format.py
│   ├── test_backwards_compat.py  ← 向后兼容性保护
│   └── ...
├── middleware/                 ← 每个中间件独立测试套
│   ├── test_summarization_middleware.py
│   ├── test_memory_middleware.py
│   ├── test_skills_middleware.py
│   └── ...
└── smoke_tests/               ← 系统提示词快照测试
    ├── test_system_prompt.py
    └── snapshots/             ← JSON + Markdown 快照文件
        ├── system_prompt_with_execute.md
        ├── system_prompt_with_memory_and_skills.md
        └── ...（5组场景快照）
```

**核心技术手段**


| 手段                               | 用途                | 实现                                                                 |
| -------------------------------- | ----------------- | ------------------------------------------------------------------ |
| `GenericFakeChatModel`           | 不依赖真实 LLM 的确定性测试  | `langchain_core.language_models.fake_chat_models`                  |
| 参数化 Backend                      | 同一测试跑 3 种 Backend | `@pytest.fixture(params=["filesystem_virtual", "state", "store"])` |
| 快照测试                             | 防止系统提示词被意外修改      | `--update-snapshots` CLI 参数控制更新                                    |
| `assert_all_deepagent_qualities` | 统一的 Agent 质量断言    | `tests/utils.py` 中的复合断言帮助函数                                        |


`**test_end_to_end.py` 的覆盖广度**

这是最重要的测试文件（2374+ 行），使用假 LLM 驱动真实的 agent 图执行，测试：

- 文件读写操作（跨 3 种 Backend）
- 中间件 `wrap_model_call` 调用链
- 系统提示词注入正确性（`SystemMessageCapturingMiddleware`）
- Token 限制和摘要触发
- 子智能体委派和状态隔离
- 向后兼容性（`test_backwards_compat.py`）

**关键设计决策**：单元测试刻意使用 `GenericFakeChatModel` 而非真实模型，保证：

1. 零 API 成本
2. 100% 确定性（CI 不会因模型不稳定而失败）
3. 可在本地离线运行

### 1.2 集成测试层（Integration Tests）

```
tests/integration_tests/
├── test_deepagents.py            ← 真实 LLM 调用的 agent 执行
├── test_filesystem_middleware.py ← 真实文件系统操作
├── test_subagent_middleware.py   ← 真实子智能体调用链
└── test_langsmith_sandbox.py     ← LangSmith sandbox 集成
```

**特点**：

- 需要 `ANTHROPIC_API_KEY`（必填）和 `LANGSMITH_API_KEY`（可选）
- CI **不自动运行**集成测试（需要 secrets，只有特殊触发时执行）
- 集成测试主要作为开发者本地验证工具

### 1.3 行为评测层（Evals）

这是最重要的差异化设计，详见第二节。

### 1.4 性能基准测试（Benchmarks）

```
tests/benchmarks/
└── test_benchmark_create_deep_agent.py  ← 测量 create_deep_agent() 初始化耗时
```

- 通过 **CodSpeed** 平台追踪性能回归
- 在每次 PR 中自动运行（针对 `deepagents` 和 `cli` 的变更）

---

## 二、行为评测框架（Evals）

这是整个质量保障体系中最有创意的部分，属于"Agent 能力的定量测量"而非传统测试。

### 2.1 两层断言模型（Two-Tier Assertion Model）

```
TrajectoryScorer
    │
    ├── .success(assertion)   ← 硬断言：失败则 pytest.fail()
    │   ├── final_text_contains(text)     文本内容正确性
    │   ├── file_equals(path, content)    文件内容精确匹配
    │   ├── file_contains(path, sub)      文件内容包含检查
    │   └── llm_judge(criteria...)        LLM 评判（语义级别）
    │
    └── .expect(...)          ← 软断言：仅记录，永不失败
        ├── agent_steps=N     期望步数
        └── tool_call_requests=N  期望工具调用次数
```

**核心哲学**：

- **成功断言**：测试正确性，必须通过
- **效率断言**：测试是否高效，仅记录到 LangSmith，不卡 CI

这个设计解决了 Agent 测试的核心矛盾：LLM 的输出不是完全确定的，但正确性（产生了正确结果）是可以断言的，而效率（用了多少步）只能期望。

**典型用例**：

```python
scorer = (
    TrajectoryScorer()
    .expect(agent_steps=2, tool_call_requests=1)
    .success(
        final_text_contains("三", case_insensitive=True),
        file_equals("output.txt", "expected content"),
    )
)
```

### 2.2 AgentTrajectory 数据结构

```
AgentTrajectory
├── steps: list[AgentStep]
│   └── AgentStep
│       ├── index: int (1-indexed)
│       ├── action: AIMessage
│       └── observations: list[ToolMessage]
└── files: dict[str, str]   ← 执行后的文件状态
```

`run_agent()` 是统一入口点，负责：

1. 构造输入（支持字符串或消息列表）
2. 注入初始文件（`initial_files`）
3. 调用 agent `invoke()`
4. 解析结果为 `AgentTrajectory`
5. 将输入/输出记录到 LangSmith
6. 运行断言

### 2.3 评测套件覆盖


| 测试文件                            | 类别                             | 评测内容                               |
| ------------------------------- | ------------------------------ | ---------------------------------- |
| `test_file_operations.py`       | `file_operations`, `retrieval` | 文件读写/编辑/搜索，并行操作                    |
| `test_tool_selection.py`        | `tool_use`                     | 工具选择正确性（直接/间接/多步）                  |
| `test_tool_usage_relational.py` | `tool_use`                     | 多步工具链（user→location→weather 依赖查找）  |
| `test_todos.py`                 | `tool_use`                     | Todo 工具使用                          |
| `test_memory.py`                | `memory`                       | AGENTS.md 记忆召回和行为引导                |
| `test_memory_multiturn.py`      | `memory`                       | 多轮记忆：隐式偏好提取、显式记忆指令                 |
| `memory_agent_bench/`           | `memory`                       | MemoryAgentBench（ICLR 2026）：长上下文记忆 |
| `test_summarization.py`         | `summarization`                | 摘要触发、摘要后任务继续、历史卸载                  |
| `test_subagents.py`             | `unit_test`                    | 子智能体委派                             |
| `test_hitl.py`                  | `unit_test`                    | Human-in-the-Loop 审批流程             |
| `test_skills.py`                | `unit_test`                    | SKILL.md 发现和应用                     |
| `test_followup_quality.py`      | `conversation`                 | 追问质量（LLM judge）                    |
| `tau2_airline/`                 | `conversation`                 | tau2-bench 航空域：多轮对话 DB 状态准确率       |
| `test_external_benchmarks.py`   | `retrieval`, `tool_use`        | FRAMES、Nexus、BFCL v3               |


### 2.4 报告指标体系

```
correctness    = 通过所有成功断言的测试比例
step_ratio     = 实际步数 / 期望步数（micro 平均）
tool_call_ratio = 实际工具调用 / 期望调用
solve_rate     = mean(期望步数 / 执行时长) for 通过的测试
median_duration_s = 执行时长中位数
```

加上**雷达图**（per-category 得分可视化），可以直观对比不同模型在各能力维度的表现。

### 2.5 LLM-as-Judge

使用 [openevals](https://github.com/langchain-ai/openevals) 包，允许用自然语言标准评判输出：

```python
scorer = TrajectoryScorer().success(
    llm_judge(
        "答案提到法国首都是巴黎。",
        "语调口语化，不机械。",
    )
)
```

### 2.6 Harbor / Terminal Bench 2.0

Harbor 集成是另一条评测路径：在**真实沙箱环境**（Docker、Daytona、Modal、Runloop）中运行 90+ 个真实终端任务，评分为 0.0-1.0 的测试通过率。

`DeepAgentsWrapper` 实现 Harbor 的 `BaseAgent` 接口：

- 支持 CLI agent（`create_cli_agent`）和 SDK agent（`create_deep_agent`）两种模式
- 自动采集基础设施元数据（用于噪声分析）
- 轨迹以 ATIF-v1.2 格式写入文件
- 与 LangSmith 实验追踪集成，支持跨运行对比

---

## 三、安全威胁模型

DeepAgents 有两份 `THREAT_MODEL.md`（均由自动化系统 `langster-threat-model` 生成）：

- `libs/cli/THREAT_MODEL.md`：CLI 产品的威胁模型
- `libs/deepagents/THREAT_MODEL.md`：SDK 库的威胁模型

### 3.1 CLI 威胁模型（10 个威胁）


| ID  | 威胁                                                | 严重性    | 状态               |
| --- | ------------------------------------------------- | ------ | ---------------- |
| T1  | 通过抓取网页内容进行提示注入，LLM 发起有害操作                         | Medium | Likely           |
| T2  | `--shell-allow-list all` 时，非交互模式任意 shell 命令无需审批执行 | Medium | Verified         |
| T3  | LLM 生成 Unicode 同形字符 URL 欺骗用户审批                    | Low    | Disproven（已有检测）  |
| T4  | Auto-approve 绕过所有 HITL 门控                         | Low    | Verified（用户主动开启） |
| T5  | 本地 SQLite 检查点被篡改，注入恶意历史                           | Low    | Unverified       |
| T6  | **本地无认证 LangGraph dev server**，同机其他进程可访问          | Medium | Verified         |
| T7  | LocalContextMiddleware 将 Makefile 内容注入系统提示        | Low    | Verified         |
| T8  | 自定义子智能体 AGENTS.md 内容未经验证直接作为系统提示                  | Low    | Verified         |
| T9  | `class_path` 配置触发任意 Python 代码执行                   | Low    | Verified         |
| T10 | MCP stdio 子进程 env 字典未过滤（可设 PATH/LD_PRELOAD）       | Low    | Verified         |


**最关注的威胁（T6）**：

`LANGGRAPH_AUTH_TYPE=noop` 是刻意设计（本地开发服务器不需要认证），但意味着：

- 任何能在 localhost 扫到端口的进程，都能向 Agent 注入消息
- 读取 Agent 的完整对话状态（可能包含文件内容）
- 端口默认 2024，可通过 `/proc/{pid}/cmdline` 发现

这是文档化的**接受风险**（Accepted Risk），而非漏洞。

**安全设计亮点**：

- `unicode_security.py`：检测 Unicode 同形字符和 BiDi 混淆，在 HITL 审批界面显示警告
- UUID7 线程 ID：防止路径注入（`sessions.generate_thread_id`）
- MCP 配置 SHA-256 指纹信任机制（`mcp_trust.compute_config_fingerprint`）
- `markdownify` 转换：HTML → Markdown 降低注入面

### 3.2 SDK 威胁模型（9 个威胁）


| ID  | 威胁                                            | 严重性      | 状态                   |
| --- | --------------------------------------------- | -------- | -------------------- |
| T1  | 污染的 memory/skill 文件内容注入系统提示（无任何 sanitization） | **High** | Verified             |
| T2  | 提示注入通过 `task` 工具传播到子智能体 `description`         | Medium   | Likely               |
| T3  | LocalShellBackend 的 `shell=True` 任意命令执行       | **High** | Verified             |
| T4  | LangGraph checkpoint msgpack 不安全反序列化          | Medium   | **Disproven**（上游已修复） |
| T5  | 超大 skill 文件 DoS（内存耗尽）                         | Low      | Verified（有 10MB 限制）  |
| T6  | LocalShellBackend 绕过 virtual_mode 的路径限制       | Medium   | Verified             |
| T7  | OpenAI Responses API 默认保留对话数据                 | Medium   | Verified             |
| T8  | 异步子智能体输出未经消毒注入主 agent 上下文                     | Medium   | Likely               |
| T9  | FilesystemBackend `virtual_mode=None` 默认不限制路径 | Info     | Verified（v0.5.0 会改）  |


**最关键的安全边界**：`StateBackend` 是默认 backend，不支持 shell 执行。只有用户**显式**提供 `LocalShellBackend` 时才开放 `execute` 工具（`FilesystemMiddleware` 会在 backend 不实现 `SandboxBackendProtocol` 时过滤掉该工具）。

**T1 的深层问题**：
memory 文件内容通过 `str.format()` 插值到系统提示，且 `MEMORY_SYSTEM_PROMPT` 指示 LLM 把 memory 内容视为权威——这放大了被污染 AGENTS.md 的攻击效果。比较最危险的场景：克隆包含恶意 `.deepagents/AGENTS.md` 的 git 仓库。

---

## 四、CI 流水线

### 4.1 主 CI（`.github/workflows/ci.yml`）

```
每次 PR 触发：

  ① 变更检测（dorny/paths-filter）
     检测 7 个包：deepagents / cli / evals / daytona / modal / runloop / quickjs

  ② 并行 Lint（仅变更包）
     工具：make lint（ruff format + ruff check + ty 类型检查）
     Python：3.11（大多数包），3.14（evals）

  ③ 并行 Unit Tests（仅变更包 + cli 受 deepagents 变更影响）
     Python 矩阵：3.11 / 3.12 / 3.13 / 3.14（fail-fast=false）
     Coverage 报告：仅 Python 3.12 生成

  ④ CodSpeed 基准测试
     仅针对 deepagents 和 cli 的变更

  ⑤ ci_success 汇总门控
     任何 failure 或 cancellation 都阻断 merge
```

**智能变更检测**：SDK (`deepagents`) 的变更同时触发 `cli` 的测试（因为 CLI 依赖 SDK），避免因依赖链导致漏测。

**注意**：评测（Evals）在 CI 中运行的是**单元测试**（`test-evals`），而非真实 LLM 评测。真实 LLM 评测通过独立的 `evals.yml` 手动触发。

### 4.2 评测 CI（`.github/workflows/evals.yml`）

```
手动触发（workflow_dispatch）：

  ① 输入：模型选择（下拉 / 自定义字符串）+ 评测类别过滤
     支持模型：Anthropic、OpenAI、Google、xAI、Groq、NVIDIA、
              Baseten、Fireworks、OpenRouter、Ollama

  ② 计算矩阵（.github/scripts/models.py eval）

  ③ 并行评测 Job（per-provider 并发控制）
     超时：360 分钟
     同一 provider 的多个 job 串行（避免 rate limit）
     不同 provider 的 job 完全并行

  ④ 汇总 + 雷达图生成
     发布到 eval-assets 分支（raw GitHub URL 可访问）
     写入 GitHub Actions step summary

  ⑤ 打印 per-category 得分表格（step summary）
```

### 4.3 Harbor CI（`.github/workflows/harbor.yml`）

独立的 Harbor/Terminal Bench 工作流，在真实沙箱环境执行 90+ 任务，是评测体系中对"真实能力"最有说服力的测量。

### 4.4 其他 CI 工作流


| 工作流                       | 职责                                        |
| ------------------------- | ----------------------------------------- |
| `release.yml`             | 发布到 PyPI                                  |
| `release-please.yml`      | 自动生成 CHANGELOG                            |
| `check_versions.yml`      | 校验 `_version.py` 与 `pyproject.toml` 版本一致性 |
| `check_lockfiles.yml`     | 校验 `uv.lock` 是否最新                         |
| `check_extras_sync.yml`   | 校验 CLI 的 extras 与依赖同步                     |
| `check_sdk_pin.yml`       | 校验 SDK 版本固定                               |
| `pr_lint.yml`             | PR 标题/链接规范检查                              |
| `require_issue_link.yml`  | PR 必须关联 issue                             |
| `tag-external-issues.yml` | 自动打标签                                     |


---

## 五、代码规范体系（`.pre-commit-config.yaml`）

```
pre-commit 钩子执行顺序：

  通用检查（pre-commit-hooks）
    ├── no-commit-to-branch      → 禁止直接向 main 提交
    ├── check-yaml               → YAML 语法验证
    ├── check-toml               → TOML 语法验证
    ├── end-of-file-fixer        → 文件末尾换行
    └── trailing-whitespace      → 行尾空格清理

  文本修复（texthooks）
    ├── fix-smartquotes          → 智能引号 → 标准引号
    └── fix-spaces               → 非标准空格替换

  包级别格式化和 Lint（local hooks）
    ├── deepagents               → make -C libs/deepagents format lint
    ├── deepagents-cli           → make -C libs/cli format lint
    ├── evals                    → make -C libs/evals format eval-catalog lint
    └── acp                      → make -C libs/acp format lint

  基础设施完整性检查（local hooks）
    ├── lock-check               → uv.lock 与 pyproject.toml 同步检查
    ├── extras-sync              → CLI extras 与必须依赖同步
    └── version-equality         → pyproject.toml ↔ _version.py 版本一致
```

**工具链**：

- **ruff**：格式化（代替 Black）+ lint（代替 flake8/isort）
- **ty**：类型检查（Ruff 团队新开发的高速类型检查器）
- **uv**：包管理和 lockfile 管理

---

## 六、回答探索计划中的开放问题

### Q: 测试里有没有 end-to-end 的 agent 执行测试？

**有，并且很完整**。`test_end_to_end.py`（2374+ 行）使用 `GenericFakeChatModel` 驱动真实的 agent 图执行，覆盖了文件操作、中间件链、子智能体、摘要触发等所有主要场景，且对 3 种 backend 参数化测试。这是不依赖真实 LLM 的"端到端"测试，覆盖面很广。

此外，`libs/evals/` 的评测套件是依赖真实 LLM 的端到端测试，但不在常规 CI 中运行。

### Q: `THREAT_MODEL.md` 识别了哪些具体威胁？

共两份文档：

- **CLI**：10 个威胁（T1-T10），4 个调查后关闭（D1-D4）。最高严重性是 Medium，包括提示注入（T1）、shell 旁路（T2）、本地 dev server 无认证（T6）。
- **SDK**：9 个威胁（T1-T9），3 个调查后关闭（D1-D3）。最高严重性是 **High**（T1 内存文件内容注入、T3 LocalShellBackend 任意命令执行）。

**沙箱逃逸**：`LocalShellBackend` 使用 `shell=True`，`virtual_mode=True` 只限制文件路径但无法限制 shell 命令（T6/SDK）——这是文档化的已知行为，不是 bug。

**Prompt Injection**：T1（CLI）和 T1/T2/T8（SDK）都识别了提示注入路径，但在交互模式下 HITL 是最终防线；HITL 绕过（auto_approve）是用户主动选择。

### Q: Harbor 评测是离线的还是需要真实 LLM 调用？

**需要真实 LLM 调用**。Harbor 运行 Agent 在真实沙箱（Docker/Daytona/Modal/Runloop）执行真实任务，需要：

- `ANTHROPIC_API_KEY`（或其他 provider key）
- `LANGSMITH_API_KEY`（用于追踪）
- 沙箱环境访问权限（如 `DAYTONA_API_KEY`）

评测结果通过 LangSmith 追踪，支持跨运行、跨模型的对比分析。

---

## 七、架构洞察与设计决策总结


| 观察                        | 设计决策                   | 权衡                       |
| ------------------------- | ---------------------- | ------------------------ |
| 使用 FakeChatModel 而非 Mock  | 允许 agent 图完整运行，只替换 LLM | 测试速度快，但 LLM 行为不真实        |
| 两层断言模型（success/expect）    | 正确性硬断言，效率软记录           | 避免 LLM 不确定性卡 CI，同时保留效率信息 |
| LangSmith 深度集成            | 每次 eval 都有完整追踪，支持失败归因  | 需要 API Key，离线环境无法运行      |
| THREAT_MODEL 自动生成         | 低成本持续维护安全文档            | 自动生成的分析可能有遗漏或误判          |
| Evals 不在 PR CI 中运行        | 避免 CI 因模型不稳定或费用过高而失败   | 回归测试覆盖有延迟（依赖手动触发）        |
| 三份 Python 版本矩阵（3.11-3.14） | 提前发现 Python 版本兼容问题     | CI 时间增加 4×               |
| pre-commit extras-sync 检查 | 防止 CLI 依赖配置不一致         | 多一个必须保持同步的文件             |


---

*此文档为 Layer 4 分析产出，与 Layer 1-3 文档共同构成 DeepAgents 源码深度探索的完整记录。*