# DeepAgents 源码深度探索计划

> 生成时间：2026-04-07  
> 基于对 deepagents monorepo 的全局扫描，提供一条由浅入深、有层次的源码分析路线。

---

## 总体策略

分析分为五个层次，层层递进：

```
Layer 1: 契约层   → 理解对外承诺（接口/协议）
    │
    ▼
Layer 2: 运行机制 → 理解一次请求如何流动
    │
    ▼
Layer 3: 扩展机制 → 理解如何"插入"新能力
    │
    ▼
Layer 4: 工程质量 → 理解正确性如何保障
    │
    ▼
Layer 5: 生态系统 → 理解与外部世界的连接
```

---

## 项目整体结构

```
deepagents/
├── libs/
│   ├── deepagents/          ← 核心 SDK（最先分析）
│   │   └── deepagents/
│   │       ├── __init__.py        对外导出符号
│   │       ├── graph.py           create_deep_agent 主入口
│   │       ├── _models.py         多 provider 模型解析
│   │       ├── _version.py
│   │       ├── backends/          存储/执行后端抽象层
│   │       │   ├── protocol.py    BackendProtocol 接口
│   │       │   ├── state.py
│   │       │   ├── store.py
│   │       │   ├── filesystem.py
│   │       │   ├── composite.py
│   │       │   ├── sandbox.py
│   │       │   ├── local_shell.py
│   │       │   ├── langsmith.py
│   │       │   └── utils.py
│   │       └── middleware/        能力插件层
│   │           ├── __init__.py    中间件架构说明文档
│   │           ├── filesystem.py
│   │           ├── subagents.py
│   │           ├── async_subagents.py
│   │           ├── summarization.py
│   │           ├── memory.py
│   │           ├── skills.py
│   │           ├── patch_tool_calls.py
│   │           └── _utils.py
│   ├── cli/                 ← TUI 终端界面（类 Claude Code）
│   ├── acp/                 ← Agent Client Protocol
│   ├── evals/               ← 评测框架（Harbor）
│   └── partners/            ← Sandbox 集成：runloop/daytona/modal/quickjs
└── examples/                ← 使用示例
    ├── deep_research/
    ├── async-subagent-server/
    ├── text-to-sql-agent/
    └── ...
```

---

## Layer 1：契约层 — 理解对外承诺

**目标**：不看实现，只看接口。搞清楚这个框架对使用者承诺了什么。

### 关键文件

| 文件 | 分析重点 |
|------|---------|
| `deepagents/__init__.py` | 对外暴露了哪些符号？导出策略是什么？ |
| `deepagents/graph.py`（仅签名） | `create_deep_agent()` 有哪些参数？默认值意味着什么设计决策？ |
| `deepagents/backends/protocol.py` | `BackendProtocol` 的最小抽象是什么？文件格式 v1/v2 有什么区别？ |
| `deepagents/middleware/__init__.py` | 中间件的接口契约是什么？是否有基类？ |
| `deepagents/_models.py` | 支持哪些模型 provider？如何路由到不同 provider？ |

### 探索问题

- `create_deep_agent` 的必填参数 vs 可选参数——哪些是设计核心，哪些是便利设施？
- `BackendProtocol` 的接口设计——文件读写抽象和执行抽象是统一的还是分离的？
- 中间件是否有统一的基类或 Protocol？还是基于 duck typing？

---

## Layer 2：运行机制 — 一次请求的完整流动

**目标**：跟踪一次 `agent.invoke({"messages": [...]})` 调用，从入口到输出的完整路径。

### 中间件装配顺序（`graph.py` 中）

```
create_deep_agent() 调用时：

  resolve_model()
       │
       ▼
  StateBackend()（默认 backend）
       │
       ▼
  ┌─────────────────────────────────────┐
  │           中间件链装配               │
  │                                     │
  │  1. TodoListMiddleware              │  ← 注入 write_todos 工具
  │  2. FilesystemMiddleware            │  ← 绑定文件读写工具
  │  3. SubAgentMiddleware              │  ← 注入 task() 工具
  │  4. SummarizationMiddleware         │  ← 长上下文处理
  │  5. PatchToolCallsMiddleware        │  ← Anthropic 格式修复
  │  6. SkillsMiddleware (optional)     │  ← 技能文件加载
  │  7. AsyncSubAgentMiddleware (opt.)  │  ← 异步子智能体
  │  8. user middleware (optional)      │  ← 用户自定义
  │  9. AnthropicPromptCachingMW        │  ← Token 缓存优化
  │  10. MemoryMiddleware               │  ← AGENTS.md 记忆注入
  │  11. HumanInTheLoopMiddleware (opt.)│  ← 中断确认
  └─────────────────────────────────────┘
       │
       ▼
  langchain.create_agent()
       │
       ▼
  CompiledStateGraph（返回值）
```

### 关键文件（按执行顺序）

1. `graph.py` — **全文精读**，理解整个装配逻辑
2. `middleware/filesystem.py` — 文件工具如何绑定到 backend
3. `middleware/subagents.py` — task() 工具的实现
4. `middleware/summarization.py` — 何时触发摘要？摘要策略是什么？
5. `middleware/memory.py` — AGENTS.md 如何被发现和注入？
6. `middleware/patch_tool_calls.py` — 修复了什么格式问题？为什么需要？

### 探索问题

- 中间件的执行顺序是固定的还是可配置的？顺序重要吗？
- `SummarizationMiddleware` 的触发条件是 token 数量还是轮次？
- `PatchToolCallsMiddleware` 是临时 hack 还是长期设计？

---

## Layer 3：扩展机制 — 如何"插入"新能力

**目标**：理解 deepagents 的三个扩展点，以及它们的设计权衡。

### 扩展点光谱

```
最轻量                                        最重量
   │                                              │
   ▼                                              ▼
tool                middleware               backend
─────────────────────────────────────────────────────
直接增加工具          注入到调用链              替换存储/执行层
无副作用              可修改上下文              改变数据持久化
对话粒度              请求粒度                  会话粒度
对用户可见            对用户透明                对用户完全透明
```

### 子智能体的两种模式

```
同步模式（SubAgentMiddleware）        异步模式（AsyncSubAgentMiddleware）

主 Agent                             主 Agent
  │                                     │
  │ task(...)                           │ task(...) × N
  ▼                                     │
子 Agent                             ┌──┴──┬──────┐
  │                                  ▼     ▼      ▼
  │ 完成后返回                      子A   子B    子C
  ▼                                  │     │      │
主 Agent 继续                        └──┬──┴──────┘
                                        ▼
                                   全部完成后汇总
```

### 关键文件

| 文件 | 分析重点 |
|------|---------|
| `middleware/__init__.py` | 中间件架构的设计文档，务必精读 |
| `middleware/async_subagents.py` | 异步分叉模型的实现细节 |
| `middleware/skills.py` | 技能文件格式和加载策略 |
| `backends/composite.py` | 如何组合多个 backend？ |
| `backends/sandbox.py` | sandbox 执行环境的抽象 |
| `backends/local_shell.py` | 本地 shell 执行的安全边界 |

### 探索问题

- 中间件链是如何串联的？是责任链模式还是 Pipeline？能否中断？
- 异步子智能体如何处理状态共享和结果收集？
- `CompositeBackend` 的组合策略——读写是否分离到不同 backend？

---

## Layer 4：工程质量 — 如何保证正确性

**目标**：理解测试策略、安全设计和 CI 流程。

### 质量保障体系

```
单元测试    libs/deepagents/tests/
集成测试    libs/cli/tests/
评测框架    libs/evals/deepagents_harbor/    ← 对 Agent 能力打分
CI 流程     .github/workflows/              ← 多个 workflow
代码规范    .pre-commit-config.yaml (ruff, ty)
威胁模型    libs/cli/THREAT_MODEL.md        ← 安全思维文档
```

### 关键文件

| 文件 | 分析重点 |
|------|---------|
| `libs/cli/THREAT_MODEL.md` | CLI 作为代码执行工具，安全边界如何划定？ |
| `libs/deepagents/tests/` | 测试策略：是否有中间件链的集成测试？ |
| `libs/evals/deepagents_harbor/` | Harbor 如何对 Agent 能力定量评测？ |
| `.github/workflows/` | CI 触发条件、评测何时运行？ |

### 探索问题

- 测试里有没有 end-to-end 的 agent 执行测试？还是只有单元测试？
- `THREAT_MODEL.md` 识别了哪些威胁？沙箱逃逸、prompt injection？
- Harbor 评测是离线的还是需要真实 LLM 调用？

---

## Layer 5：生态系统 — 与外部世界的连接

**目标**：理解 deepagents 如何与 sandbox、MCP、ACP 等外部系统集成。

### 外部连接图

```
                  ┌─────────────────────────┐
                  │     DeepAgents Core     │
                  └──────┬──────────┬───────┘
                         │          │
               ┌─────────┘          └──────────┐
               ▼                               ▼
    ┌─────────────────────┐        ┌──────────────────────┐
    │   Sandbox Partners  │        │   Protocol Layer     │
    │                     │        │                      │
    │  langchain_runloop  │        │  MCP（工具调用协议）  │
    │  langchain_daytona  │        │  ACP（Agent 通信协议）│
    │  langchain_modal    │        │                      │
    │  langchain_quickjs  │        │  CLI ↔ ACP ↔ Agent  │
    └──────────┬──────────┘        └──────────────────────┘
               │
               ▼
    隔离的代码执行环境（云端/本地）
```

### 关键文件

| 文件 | 分析重点 |
|------|---------|
| `libs/acp/` | ACP 是什么？自创协议还是行业标准？ |
| `libs/partners/runloop/` | sandbox 如何通过 langchain 插件集成？ |
| `libs/cli/deepagents_cli/` | CLI 如何与 SDK 交互？MCP 集成在哪层？ |
| `examples/async-subagent-server/` | 子智能体作为独立服务运行的模式 |

### 探索问题

- ACP（Agent Client Protocol）和 MCP（Model Context Protocol）的职责如何划分？
- `libs/partners/` 里的各 sandbox 集成方式是否一致？有无统一的 adapter 模式？
- CLI 中的「远程图」模式（LangGraph SDK）是什么使用场景？

---

## 推荐阅读顺序

```
Day 1（契约）
  __init__.py
  → graph.py（仅签名）
  → backends/protocol.py
  → middleware/__init__.py

Day 2（主流程）
  graph.py（全文）
  → middleware/ 按装配顺序逐个读

Day 3（边缘情况）
  middleware/patch_tool_calls.py
  → middleware/async_subagents.py
  → backends/composite.py

Day 4（安全与质量）
  libs/cli/THREAT_MODEL.md
  → libs/deepagents/tests/
  → .github/workflows/

Day 5（生态与示例）
  libs/acp/
  → libs/partners/runloop/
  → examples/deep_research/
```

---

## 待回答的开放问题清单

在分析过程中，以下问题值得重点关注：

- [ ] 中间件的接口契约是什么？是否有基类或 Protocol？
- [ ] `SummarizationMiddleware` 的触发条件和摘要策略
- [ ] `PatchToolCallsMiddleware` 修复了什么问题？是否有 issue 记录？
- [ ] 异步子智能体如何处理状态共享？是否有竞态风险？
- [ ] `BackendProtocol` 文件格式 v1 vs v2 的区别
- [ ] `CompositeBackend` 的读写路由策略
- [ ] ACP 协议的来源（自创 or 行业标准）
- [ ] Harbor 评测框架的工作原理
- [ ] `THREAT_MODEL.md` 识别了哪些具体威胁

---

*此文档为探索起点，随分析深入可持续更新。*
