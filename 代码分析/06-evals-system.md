# 六、评测系统（Evals）

本文说明 `deepagents/libs/evals` 中**行为评测（pytest + LangSmith）** 与 **Harbor 基准集成（deepagents_harbor）** 的分工、数据流与关键模块。

## 概述：双轨架构

```
+---------------------------+          +------------------------------+
| deepagents_evals          |          | deepagents_harbor             |
| - categories.json         |          | - HarborSandbox（异步沙箱）  |
| - radar.py（雷达图）      |          | - DeepAgentsWrapper（Harbor  |
|   可视化与汇总输入         |          |   Agent）                    |
+---------------------------+          | - langsmith.py 数据集/实验   |
        ^                              | - failure.py 失败归因        |
        | 类别标签、雷达轴              | - stats.py 置信区间/MDE      |
        |                              +---------------+--------------+
+---------------------------+                          |
| tests/evals/              |                          | Harbor CLI / 任务
| pytest + TrajectoryScorer |                          v
| LangSmith @test 装饰      |                  +---------+----------+
+---------------------------+                  | Harbor Environment |
                                               +--------------------+
```

- **`deepagents_evals`**：轻量库，提供**类别定义**（`categories.json`）与**雷达图生成**（`radar.py`），供脚本与 CI 聚合使用
- **`tests/evals/` + `conftest.py`**：以 **pytest** 运行端到端评测，**强制依赖 LangSmith 追踪**
- **`deepagents_harbor`**：在 Harbor 任务里跑 Deep Agent，实现 **ATIF 轨迹**、**沙箱后端**、**LangSmith 数据集/实验/奖励回写** 与 **基础设施元数据**

---

## 6.1 评测框架架构

### Pytest 执行机制

**强制 LangSmith**（`conftest.py`）：

```python
def pytest_configure(config: pytest.Config) -> None:
    if not tracing_enabled:
        pytest.exit(
            "Aborting: LangSmith tracing is not enabled. "
            "Please set LANGSMITH_API_KEY and LANGSMITH_TRACING=true"
        )
```

**CLI 参数**：

| 参数 | 作用 |
|------|------|
| `--model` | 参数化 `model_name`，默认 `get_default_model().model` |
| `--eval-category` | 按 `@pytest.mark.eval_category("...")` 过滤用例 |
| `--openrouter-provider` | 指定 OpenRouter 提供商 |

### 核心评测框架（`tests/evals/utils.py`）

```
AgentTrajectory          ← 从消息列表整理步骤与工具观测
└── AgentStep[]

TrajectoryScorer         ← 双层断言
├── .success(...)       ← 硬失败（pytest assert）
└── .expect(...)        ← 仅记录效率（步数、工具调用数），不失败

run_agent()              ← 驱动 CompiledStateGraph + LangSmith testing 集成
```

---

## 6.2 评测类别与指标

### `categories.json` 结构

```json
{
  "categories": [
    "file_operations",
    "retrieval",
    "tool_use",
    "memory",
    "conversation",
    "summarization",
    "unit_test"
  ],
  "radar_categories": [
    "file_operations",
    "retrieval",
    "tool_use",
    "memory",
    "conversation",
    "summarization"
  ]
}
```

**注意**：`unit_test` 存在于总类别中，但**默认不在 `radar_categories`**，用于 SDK/HITL/子代理等「管线正确性」测试而非模型能力对比。

### 评测指标汇总

| 指标 | 说明 |
|------|------|
| `correctness` | 硬失败率（success 断言通过率） |
| `step_ratio` | 实际步数 / 期望步数（效率） |
| `tool_call_ratio` | 实际工具调用数 / 期望数 |
| `solve_rate` | Harbor 场景下的解决率 |
| `median_duration_s` | 中位执行时长 |

---

## 6.3 Radar 雷达图（`deepagents_evals/radar.py`）

```python
def generate_radar(
    results: list[ModelResult],
    categories: list[str] = EVAL_CATEGORIES,
    ...
) -> Figure:
    """生成极坐标雷达图，比较多模型在各评测维度的正确率（0-1）"""
```

- 启动时读取同目录 **`categories.json`**（数据一致性保证）
- **`EVAL_CATEGORIES`**：雷达轴顺序（从顶部顺时针），不含 `unit_test`
- **`load_results_from_summary`**：读取 `evals_summary.json`（含 `category_scores`）
- 支持多模型叠加对比

---

## 6.4 Harbor 集成

### `HarborSandbox`（`deepagents_harbor/backend.py`）

实现 `SandboxBackendProtocol` 的**异步**方法；同步方法一律 `NotImplementedError`。

**文件读写策略**（避免 ARG_MAX 问题）：

```
awrite / aedit:  Harbor upload_file / download_file（原生文件传输）
aread / als / agrep / aglob:  通过 shell 命令运行
```

默认命令超时 **300s**；超时返回 exit_code 124（与 GNU timeout 约定一致）。

### `DeepAgentsWrapper`（`deepagents_harbor/deepagents_wrapper.py`）

继承 Harbor `BaseAgent`，`run(instruction, environment, context)` 中：

1. 创建 `HarborSandbox(environment)` 作为 backend
2. 可选 `create_cli_agent`（`auto_approve=True`）或 `create_deep_agent`
3. 使用 `collect_sandbox_metadata` 把 CPU/内存等写入轨迹 `agent.extra.infrastructure`
4. 把消息转为 Harbor **ATIF** `Trajectory`（`schema_version="ATIF-v1.2"`），写入 `trajectory.json`

### LangSmith 集成（`deepagents_harbor/langsmith.py`）

| 功能 | 实现 |
|------|------|
| 数据集创建 | 从 Harbor registry 下载任务，读 `instruction.md` + `task.toml`，生成确定性 example id |
| 实验创建 | `create_experiment_async`，返回 LangSmith UI 对比 URL |
| 奖励回写 | 扫描 trial `result.json`，提取 `verifier_result.rewards.reward`，写入 `harbor_reward` |

### 失败分类（`deepagents_harbor/failure.py`）

```
FailureCategory
├── 能力失败（agent 未能完成任务）
├── OOM（exit_code 137）
├── 超时（exit_code 124）
└── 沙箱错误

注意：从 ATIF 轨迹的 observation 中提取 exit code，
而非从模型文本中解析（避免幻觉）
```

### 统计（`deepagents_harbor/stats.py`）

- **Wilson 置信区间**：用于成功率的统计置信
- **最小可检测效应（MDE）**：判断两次 run 的差异是否超出噪声范围

---

## 6.5 具体评测案例

| 测试文件 | 类别 | 内容 |
|----------|------|------|
| `test_file_operations.py` | file_operations, retrieval | 文件工具、并行读写、grep/glob |
| `test_tool_selection.py` | tool_use | 工具选择与多步链路 |
| `test_tool_usage_relational.py` | tool_use | 关系型多步工具链 |
| `test_memory.py` | memory | AGENTS.md、复合后端、记忆读写 |
| `test_memory_multiturn.py` | memory | 多轮对话中的记忆持久化 |
| `test_external_benchmarks.py` | retrieval, tool_use | FRAMES、Nexus、BFCL v3 基准 |
| `test_summarization.py` | summarization | 摘要中间件与任务延续 |
| `test_hitl.py` | unit_test | 人机协同中断审批流程 |
| `test_subagents.py` | unit_test | 子代理调度与状态隔离 |
| `test_skills.py` | unit_test | Skills 加载与调用 |
| `tau2_airline/` | conversation | 航司多轮对话 + DB 评测（TAU2 基准） |

---

## 小结

- **本地/CI 行为评测**以 **pytest + LangSmith + TrajectoryScorer** 为中心，`deepagents_evals` 提供**类别与雷达可视化**的数据契约
- **Harbor** 路径通过 **`HarborSandbox` + `DeepAgentsWrapper`** 把 Deep Agents 嵌进标准基准流水线，并配合 **LangSmith 数据集/实验/harbor_reward 反馈** 与 **失败归因/统计** 形成闭环
- 两条路径共享 `deepagents_evals` 的类别定义，保证评测维度一致性
