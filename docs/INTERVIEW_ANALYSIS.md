# MiniCode 项目深度剖析 —— 面试准备文档

## 目录

1. [项目概述与定位](#1-项目概述与定位)
2. [整体架构](#2-整体架构)
3. [核心引擎：Agent Loop](#3-核心引擎agent-loop)
4. [赛博控制论系统](#4-赛博控制论系统)
5. [记忆系统](#5-记忆系统)
6. [会话、检查点与回退](#6-会话检查点与回退)
7. [上下文管理](#7-上下文管理)
8. [工具系统与 MCP](#8-工具系统与-mcp)
9. [模型适配层](#9-模型适配层)
10. [权限与安全](#10-权限与安全)
11. [配置系统](#11-配置系统)
12. [TUI 与交互](#12-tui-与交互)
13. [双语言实现对比](#13-双语言实现对比)
14. [设计模式与工程实践](#14-设计模式与工程实践)
15. [面试高频问题 & 参考答案](#15-面试高频问题--参考答案)

---

## 1. 项目概述与定位

### 1.1 MiniCode 是什么

MiniCode 是一个**开源的 AI 编码代理（AI Coding Agent）**，灵感源自 Anthropic 的 Claude Code。它的核心能力是在终端中与开发者进行交互式对话编程——理解代码库、执行文件操作、运行 shell 命令、搜索代码，通过 **Tool Use（工具调用）** 完成实际开发任务。

**一行概括**：一个带有运行时透明性、可恢复编辑、持久记忆和赛博控制论的本地终端编码助手。

### 1.2 与 Claude Code 的关系

| 维度 | Claude Code | MiniCode |
|------|-------------|----------|
| 源码 | 闭源 | 开源 |
| 定位 | 商业产品 | 轻量开源参考实现 |
| 运行时 | Node.js | Python + TypeScript 双实现 |
| 记忆系统 | 基础 | 三级记忆（工作/项目/会话） |
| 控制论 | 无 | PID 反馈 + 前馈 + 预测控制 |
| 会话回退 | 无 | 检查点 + 回退预览 + 安全组 |

### 1.3 MiniCode 家族

| 版本 | 语言 | 仓库 | 特点 |
|------|------|------|------|
| TypeScript | TypeScript | LiuMengxuan04/MiniCode | 主线，TUI + MCP + Skills |
| Python | Python | QUSETIONS/MiniCode-Python | 本地优先，更强的运行时控制 |
| Rust | Rust | harkerhand/MiniCode-rs | 系统级实验 |
| Java | Java | hobbescalvin414-tech/minicode4j | Java 实现 |

**本仓库**是 Python 版本（当前活跃包为根目录 `minicode/`），同时包含 TypeScript 源码（`ts-src/`）和中文 Superpowers 插件。

### 1.4 设计原则

1. **运行时透明**：每个阶段、每次 widening、每个停止原因都可被检查
2. **恢复优先**：文件编辑带检查点，支持预览和回退
3. **记忆即运行时子系统**：非事后添加，编译、注入、反思全在运行时路径中
4. **验证即执行**：不是事后报告，verification gate 阻止过早"done"
5. **信号驱动**：优先使用可测量的信号而非 prompt 直觉
6. **文档对齐实现**：只记录已实现的行为

---

## 2. 整体架构

### 2.1 分层架构

```
┌──────────────────────────────────────────────────────┐
│                   表现层 (Presentation)                 │
│  tty_app.py → tui/ (screen, input, markdown, renderer) │
├──────────────────────────────────────────────────────┤
│                   应用层 (Application)                  │
│  agent_loop.py → turn_kernel.py → session.py           │
│  cli_commands.py → auto_mode.py → headless.py         │
├──────────────────────────────────────────────────────┤
│                   服务层 (Service)                      │
│  model_registry.py → tooling.py → permissions.py       │
│  skills.py → hooks.py → mcp.py                        │
├──────────────────────────────────────────────────────┤
│                基础设施层 (Infrastructure)               │
│  config.py → state.py → memory.py → context_manager.py │
└──────────────────────────────────────────────────────┘
```

### 2.2 数据流（一次完整的用户请求）

```
用户输入
  │
  ▼
main.py / headless.py    组装 system prompt + 初始化组件
  │
  ▼
agent_loop.run_agent_turn()   核心循环入口
  │
  ├─► prelude: 任务图构建、分层上下文、控制论控制器初始化
  │
  └─► while step < max_steps:     # 主循环
        │
        ├─► turn_kernel.derive_turn_step_policy()
        │     计算当前 phase (explore/execute/verify)
        │     判断是否需要 widening
        │
        ├─► _call_model(model, messages)
        │     调用 LLM，获取 AgentStep
        │
        ├─► 解析响应:
        │     - assistant text → 累加到最终回答
        │     - tool_calls → 权限检查 → 并行/串行执行
        │     - progress message → 显示进度
        │     - 空响应 → 重试逻辑
        │
        ├─► 控制论信号采集:
        │     上下文压力、成本速率、错误率、进度
        │
        ├─► 控制动作执行:
        │     compaction, checkpoint, rewind, budget adjust
        │
        └─► 停止条件判定:
               done / max_steps / await_user / blocked /
               verification_failed / widen_needed
```

### 2.3 关键组件关系图

```
agent_loop.py (主循环, 2700+ 行)
  ├─► turn_kernel.py (步进策略、阶段转换、widening)
  ├─► cybernetic_orchestrator.py (控制论协调器)
  │     ├─► feedback_controller.py
  │     ├─► feedforward_controller.py
  │     ├─► predictive_controller.py
  │     ├─► decoupling_controller.py
  │     ├─► adaptive_pid_tuner.py
  │     ├─► state_observer.py
  │     └─► stability_monitor.py
  ├─► context_compactor.py (上下文压缩)
  ├─► memory_pipeline.py (记忆检索、注入、反思)
  ├─► model_registry.py → anthropic/openai_adapter
  └─► tooling.py → tools/* (30+ 工具)
```

---

## 3. 核心引擎：Agent Loop

### 3.1 入口函数

`run_agent_turn()` (`agent_loop.py:662`) 是整个 Agent 的入口，接收以下关键参数：

```python
def run_agent_turn(
    model: ModelAdapter,       # LLM 适配器
    tools: ToolRegistry,       # 工具注册表
    messages: list[ChatMessage],  # 初始消息列表
    cwd: str,                  # 工作目录
    permissions: PermissionManager,
    session: Session | None,
    max_steps: int = 50,       # 最大工具调用步数
    on_tool_start/on_tool_result/on_assistant_message/..., # 回调
    context_manager: ContextManager,
    runtime: dict,             # 运行时配置
    enable_work_chain: bool = True,  # 启用控制论子系统
)
```

### 3.2 消息格式 (ChatMessage)

```python
class ChatMessage(TypedDict, total=False):
    role: "system" | "user" | "assistant" |
          "assistant_progress" | "assistant_tool_call" | "tool_result"
    content: str
    toolUseId: str       # 工具调用 ID
    toolName: str        # 工具名称
    input: Any           # 工具参数
    isError: bool        # 工具是否出错
```

### 3.3 模型响应格式 (AgentStep)

```python
class AgentStep:
    type: "assistant" | "tool_calls"
    content: str           # 文本内容
    kind: "final" | "progress" | None  # 是否最终回复
    calls: list[ToolCall]  # 工具调用列表
    diagnostics: StepDiagnostics  # stop_reason, blockTypes
```

### 3.4 Turn Kernel：步进策略

`turn_kernel.py` 实现了 Agent 的**三阶段执行模型**：

| 阶段 | 触发条件 | 行为 |
|------|----------|------|
| **explore**（探索） | step ≤ 1 (single) 或 step ≤ 2 (single-deep) | 检查、分解、锚定任务 |
| **execute**（执行） | 其他大部分步骤 | 具体工具使用、增量编辑 |
| **verify**（验证） | step ≥ max_steps × 70% 或 strict 模式 | 验证变更、测试证据 |

### 3.5 Widening（拓宽）机制

Widening 是 Agent 从"窄路径"（死胡同）中自我解救的机制。触发条件：

1. **工具错误**：`tool_error_count > 0` 且到达 widen 步数阈值
2. **证据后停滞**：已有验证证据但下一步仍 stall
3. **空响应反复**：模型在窄路径上反复返回空响应
4. **可恢复中断反复**：pause_turn / max_tokens 持续发生

Widening 激活后：
- 重置空响应和可恢复中断计数
- 可选增加步数
- 引导模型"比较替代方案，不要重复同一攻击线路"

### 3.6 停止条件 (TurnStopReason)

```python
TurnStopReason = Literal[
    "done",                  # 正常完成
    "max_steps",             # 达到最大步数
    "await_user",            # 需要用户输入
    "blocked",               # 被阻塞
    "verification_failed",   # 验证未通过
    "widen_needed",          # 需要拓宽但无法拓宽
]
```

### 3.7 重试逻辑

- **空响应重试**：默认 2 次 (single) / 3 次 (single-deep)
- **可恢复思维中断重试**：默认 3 次 (single) / 5 次 (single-deep)，处理 `pause_turn` 和 `max_tokens` 中断
- **API 错误重试**：通过 `api_retry.py` 实现指数退避

---

## 4. 赛博控制论系统

这是 MiniCode Python 版独有的**工程控制论子系统**，借鉴经典控制理论（PID、前馈、解耦、状态观测）来动态管理 Agent 运行时。

### 4.1 架构

```
                    ┌──────────────────────────┐
   Agent Loop ─────►│ CyberneticOrchestrator    │
                    │  (统一控制生命周期)        │
                    └──────────┬───────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ FeedbackCtrl    │  │ FeedforwardCtrl │  │ PredictiveCtrl  │
│ (PID 反馈控制)  │  │ (前馈预测控制)  │  │ (预测控制)      │
│                 │  │                 │  │                 │
│ 误差 → 纠正     │  │ 预判 → 预防     │  │ 趋势 → 超前调整 │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ DecouplingCtrl  │  │ StateObserver   │  │ StabilityMonitor│
│ (解耦控制)      │  │ (状态观测估计)  │  │ (稳定性监控)    │
│ 交互变量分离    │  │ 不可测→可估     │  │ 振荡检测        │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 4.2 核心控制器

**FeedbackController（反馈控制器）**——闭环纠正
- 输入：上下文使用率、成本速率、错误率等信号
- 输出：`limit_max_steps`、`adjust_token_budget`、`reduce_parallelism`、`adjust_concurrency`
- 仅当 confidence > 0.6 时才应用控制信号

**FeedforwardController（前馈控制器）**——提前预防
- 在任务开始时预测需要的资源，提前调整
- 不同于反馈的事后纠偏

**PredictiveController（预测控制器）**——趋势预判
- 基于历史趋势预测未来状态，超前调整

**AdaptivePIDTuner（自适应 PID 调谐器）**——参数自整定
- 动态调整 PID 的 Kp、Ki、Kd 参数
- 应对不同工作负载的特征变化

**DecouplingController（解耦控制器）**——变量独立
- 处理上下文压力和成本压力之间的交互影响
- 防止一个维度的调整引发另一维度振荡

### 4.3 成本控制 PID (`cost_control.py`)

实现了完整的 PID 成本控制闭环：

```
Setpoint (target_cost_rate)
    │ (-)
    ▼
┌───────────────────┐
│ BudgetPIDController│  P + I + D
│ Kp×e + Ki×∫e + Kd×e'
└────────┬──────────┘
         │ budget_multiplier [0.3, 2.0]
         ▼
┌───────────────────┐
│ BudgetActuator    │  threshold/base = raw × multiplier
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ ToolResultBudget  │  持久化/裁剪工具结果
└────────┬──────────┘
         │ smaller context → fewer tokens
         ▼
┌───────────────────┐
│ CostTracker       │  cost_rate = $/min
│ (传感器)          │
└───────────────────┘
         │ (+)
         └──────────► 反馈回 Setpoint
```

核心逻辑：
- `cost_rate > setpoint` → `multiplier < 1.0` → 收紧预算 → 节省 tokens
- `cost_rate < setpoint` → `multiplier > 1.0` → 放宽预算 → 更丰富的上下文
- 积分项防止稳态漂移
- 微分项抑制突发消费时的振荡

### 4.4 稳定性监控 (`stability_monitor.py`)

检测 Agent 行为中的振荡模式：
- 工具调用成功率波动过大
- 上下文使用率周期性飙升/下降
- PID 积分项积分饱和 (anti-windup)
- Lyapunov 稳定性分析（见 `docs/memory_theory.md`）

---

## 5. 记忆系统

### 5.1 三级记忆架构

```
┌─────────────────────────────────────────────────────┐
│                   WORKING MEMORY                      │
│  working_memory.py                                   │
│  - 当前任务上下文，受保护，不被 compaction 清除       │
│  - TTL: 1800s (single) / 7200s (single-deep)        │
│  - 包含：任务标题、目标、进度、最新工具结果           │
├─────────────────────────────────────────────────────┤
│                   PROJECT MEMORY                      │
│  memory.py                                           │
│  - 长期项目知识，跨会话持久化                         │
│  - 存储：~/.mini-code/memory/memory.json              │
│  - 三大作用域：user / project / local                 │
│  - 分类：architecture, code-pattern, testing,         │
│          decision, preference, etc.                   │
├─────────────────────────────────────────────────────┤
│                   MEMORY PIPELINE                     │
│  memory_pipeline.py                                  │
│  - 闭合回路：检索 → 排序 → 注入 → 反思 → 写回         │
│  - 注入控制器：memory_injector.py                     │
│  - 记忆策展代理：memory_curator_agent.py              │
│  - 重排序器：memory_reranker.py                       │
└─────────────────────────────────────────────────────┘
```

### 5.2 记忆值函数（Memory Value Function）

来自 `docs/memory_theory.md`：

```
V(m, t, c) = relevance(m, t) × freshness(m) × utility(m, c)

relevance(m, t) = BM25_score(m, t)  ∈ [0, 1]
freshness(m)    = exp(-age_days / τ) with τ = 30 days
utility(m, c)   = 1 + α × ln(1 + usage_count)
```

- V → 0 as age → ∞（指数衰减）
- V 随使用次数亚线性增长（边际递减）
- 自适应 cooldown：τ_cool = τ_base × (1 - context_pressure)

### 5.3 记忆流水线闭合回路

```
                     ┌──────────────┐
   Agent Turn ──────►│ T1: 检索     │ BM25 + 语义检索
                     │   (Retrieve) │
                     └──────┬───────┘
                            │ 候选记忆
                            ▼
                     ┌──────────────┐
                     │ T2: 重排序   │ MemoryReranker (Haiku级)
                     │   (Rerank)   │ ~$0.0005/query
                     └──────┬───────┘
                            │ Top-K 记忆
                            ▼
                     ┌──────────────┐
                     │ T3: 注入     │ MemoryInjector
                     │   (Inject)   │ 自适应 cooldown
                     └──────┬───────┘
                            │ 注入到上下文
                            ▼
                     ┌──────────────┐
                     │ T4: 反思     │ MemoryCuratorAgent
                     │   (Reflect)  │ 每 N 步或显式触发
                     └──────────────┘
                            │ 新记忆 + 更新
                            ▼
                     写回 memory.json
```

### 5.4 重排序器 (`memory_reranker.py`)

- 使用轻量级模型（Haiku 级别）
- 对 BM25 检索结果进行语义重排序
- 保证 `P@3 ≥ max(P@3_bm25, P@3_rerank)` ——管道不会比最佳组件更差
- 域加权降噪：`noise_rate ≤ noise_bm25 × (1 - Jaccard)`

### 5.5 时间线记忆 (`timeline_memory.py`)

- 175KB 的大文件，记录所有 Agent 决策的时间线
- 支持按时间、会话、标签检索历史决策

### 5.6 工作记忆保护机制

`working_memory.py` + `StableTaskPack` 确保关键上下文不被 compaction 清除：
- 任务标题和目标
- 意图类型和动作类型
- 任务图摘要
- 进度摘要
- 验证摘要
- 受保护上下文列表（最多 5 项，每项 ≤ 240 字符）

---

## 6. 会话、检查点与回退

### 6.1 会话系统 (`session.py`)

52KB 的大文件，实现了完整的会话生命周期：

- **会话创建**：每次 `minicode-py` 启动创建新会话，分配 UUID
- **会话快照**：定期保存消息历史、检查点、控制论状态
- **会话持久化**：保存到磁盘，支持进程重启后恢复
- **会话回放**：`/session-replay` 重放历史消息和控制论事件
- **会话检查**：`/session` 显示当前活跃会话状态
- **会话列表**：`/sessions` 浏览所有历史会话

### 6.2 检查点系统

```
文件编辑操作
    │
    ├─► 自动创建检查点（修改前文件内容）
    │
    ├─► /checkpoints    查看检查点历史
    │
    ├─► /rewind-preview 预览回退影响
    │     显示：哪些文件会被修改、回退到什么状态
    │
    └─► /rewind         执行回退
         - /rewind          回退最近的编辑组
         - /rewind --steps N    回退 N 步
         - /rewind --checkpoint ID  回退到指定检查点
```

### 6.3 回退安全组

- 文件编辑被分组为"安全组"
- `/rewind` 默认按组回退，不是单个文件
- 预览模式让用户在实际修改前看到回退效果

---

## 7. 上下文管理

### 7.1 上下文管理器 (`context_manager.py`)

- 跟踪当前上下文窗口使用量
- 支持不同模型的上下文窗口限制（Claude: 200K, Haiku: 100K, GPT-4o: 128K）
- 上下文使用率 = 当前 tokens / 最大窗口

### 7.2 上下文压缩器 (`context_compactor.py`)

压缩策略：
- **基于预算**：`ToolResultBudgetManager` 为每条消息设置 token 预算
- **基于阶段**：verify 阶段和 widening 激活时使用激进压缩
- **记忆感知**：压缩时保留工作记忆条目，只裁剪冗余工具结果
- **回调机制**：通过 CyberneticOrchestrator 协调压缩决策

### 7.3 分层上下文 (`layered_context.py`)

```
System Prompt
├── Agent 角色定义
├── 可用工具列表 (含 MCP 工具)
├── 工作区上下文
├── 激活的技能指令
├── 注入的项目记忆
└── 安全约束

Instruction Layers (按优先级)
├── Managed Policy (机器管理, ~/.mini-code/MANAGED.md)
├── Project Guidance (项目, .mini-code/USER.md)
├── User Profile (用户, ~/.mini-code/USER.md)
└── Auto-injected Memory (记忆管道自动注入)
```

---

## 8. 工具系统与 MCP

### 8.1 工具注册与执行

```python
# 工具定义（TypeScript 用 Zod schema，Python 用类）
ToolRegistry:
    register(name, handler)   # 注册工具
    find(name) → ToolDef      # 查找工具
    execute(name, input, ctx) → ToolResult  # 执行工具
    list() → [ToolDef]        # 列出所有工具
    dispose()                 # 清理资源（关闭 MCP 连接等）
```

### 8.2 内置工具（30+）

**核心工具**：
| 工具 | 功能 |
|------|------|
| `read_file` | 读取文件 |
| `write_file` | 写入文件 |
| `edit_file` | 精确编辑（查找替换） |
| `modify_file` | 修改文件 |
| `patch_file` | 应用 Diff/Patch |
| `list_files` | 列出目录 |
| `grep_files` | 代码搜索 |
| `file_tree` | 目录树 |
| `run_command` | 执行 Shell 命令 |

**开发工具**：
| 工具 | 功能 |
|------|------|
| `git` | Git 操作 |
| `code_nav` | 代码导航 |
| `code_review` | 代码审查 |
| `test_runner` | 运行测试 |
| `todo_write` | 任务列表 |
| `task` | 任务管理 |

**工具函数**：
`http_utils`, `json_utils`, `csv_utils`, `crypto_utils`, `encoding_utils`, `regex_utils`, `text_utils`, `diff_viewer`, `archive_utils`, `batch_ops`

**外部集成**：
| 工具 | 功能 |
|------|------|
| `web_fetch` | 网页抓取 |
| `web_search` | 网络搜索 |
| `ask_user` | 向用户提问 |
| `load_skill` | 加载技能 |

### 8.3 MCP 集成 (`mcp.py`)

支持四种 MCP 传输协议：

| 传输 | 说明 |
|------|------|
| `stdio` | 标准输入输出管道（进程通信） |
| `SSE` | Server-Sent Events |
| `HTTP` | HTTP 请求/响应 |
| `WebSocket` | 双向 WebSocket |

MCP 工具发现流程：
1. 启动时读取 `.mcp.json` / `~/.mini-code/mcp.json`
2. 为每个 MCP Server 启动子进程/建立连接
3. 通过 MCP 协议协商发现工具
4. 将外部 MCP 工具包装成本地 `ToolDef`，受权限系统约束

### 8.4 工具并行调度 (ToolScheduler)

- 支持并行执行多个独立的工具调用
- 通过 FeedbackController 动态调整并发度
- 受 `_force_max_workers` 限制（默认 4，振荡时降为 2）

---

## 9. 模型适配层

### 9.1 统一接口 (ModelAdapter Protocol)

```python
class ModelAdapter(Protocol):
    def next(
        self,
        messages: list[ChatMessage],
        on_stream_chunk: Callable[[str], None] | None = None,
        store: Any | None = None,      # AppState 存储
        on_thinking_delta: ... = None,  # 思考增量回调
    ) -> AgentStep: ...
```

### 9.2 适配器实现

| 适配器 | 文件 | 支持模型 | API 格式 |
|--------|------|----------|----------|
| `AnthropicModelAdapter` | `anthropic_adapter.py` | Claude 3/4 全系列 | Anthropic Messages API |
| `OpenAIModelAdapter` | `openai_adapter.py` | GPT-4o, o1, DeepSeek, 自定义端点 | OpenAI Chat Completions |

### 9.3 提供者检测 (`detect_provider`)

```python
def detect_provider(model: str, runtime: dict) -> Provider:
    # 1. OpenRouter 前缀检测 (anthropic/, openai/, google/, deepseek/, ...)
    # 2. DeepSeek 直接 API 检测
    # 3. OpenAI 前缀检测 (gpt-4, gpt-3.5, o1-, o3-)
    # 4. 自定义端点检测 (CUSTOM_API_BASE_URL)
    # 5. 默认: Anthropic
```

### 9.4 模型切换与回退 (`model_switcher.py`)

- 同步当前活跃模型到所有控制器
- 支持故障转移：主模型失败 → 回退模型
- 回退链配置：`fallbackModels` / `anthropicFallbackModels` / `openaiFallbackModels` / `customFallbackModels`
- 模型选型评分：power × (1-budget_pressure) - latency_penalty

### 9.5 API 重试 (`api_retry.py`)

- 指数退避策略
- 可重试状态码：429, 500, 502, 503, 504
- 最大重试次数可配置（默认 4）

---

## 10. 权限与安全

### 10.1 三级权限 (`permissions.py`)

| 级别 | 行为 |
|------|------|
| `allow` | 自动允许，无需确认 |
| `ask` | 每次执行前向用户确认 |
| `deny` | 始终拒绝 |

### 10.2 权限维度

- **命令白名单/黑名单**：限制可执行 Shell 命令
- **文件路径隔离**：限制文件操作在工作目录内
- **路径遍历防护**：防止 `../` 攻击
- **网络访问控制**：限制 web_fetch/web_search 目标
- **会话级缓存**：用户选择"always allow"后本次会话不再询问

### 10.3 安全执行 (`safe_execution.py`)

- 工作目录限制
- 符号链接安全处理
- 敏感环境变量过滤
- 命令注入防护

---

## 11. 配置系统

### 11.1 配置加载优先级

```
1. CLI 参数（最高优先级）
2. 环境变量 (MINI_CODE_*, ANTHROPIC_MODEL, CUSTOM_API_KEY, ...)
3. ~/.mini-code/settings.json（项目级）
4. ~/.claude/settings.json（全局，MiniCode 读取其中共享字段）
5. 代码默认值（最低优先级）
```

### 11.2 `load_effective_settings()` 合并顺序

```python
claude_settings  (~/.claude/settings.json)
  → merge with global MCP
    → merge with project MCP (.mcp.json)
      → merge with mini_code_settings (~/.mini-code/settings.json)
```

`~/.mini-code/settings.json` 有最高优先级，可覆盖 Claude Code 的 `"model": "haiku"` 等字段。

---

## 12. TUI 与交互

### 12.1 TUI 架构

```
TTY App (tty_app.py)
├── Screen (tui/screen.py)        ← 终端屏幕缓冲管理
├── Input (tui/input.py)          ← 键盘输入处理
├── InputParser (tui/input_parser.py)  ← 斜杠命令解析
├── MarkdownRenderer (tui/markdown.py) ← Markdown → 终端
├── TranscriptPanel (tui/transcript.py) ← 对话历史面板
├── Chrome (tui/chrome.py)        ← 状态栏
├── Theme (tui/theme.py)          ← 颜色主题
└── Renderer (tui/renderer.py)    ← 主渲染器
```

### 12.2 斜杠命令 (`cli_commands.py`)

| 命令 | 功能 |
|------|------|
| `/session` | 查看当前会话快照 |
| `/sessions` | 列出历史会话 |
| `/session-replay [id]` | 回放会话 |
| `/memory` | 查看记忆系统 |
| `/checkpoints` | 查看检查点历史 |
| `/rewind-preview` | 预览回退效果 |
| `/rewind [--steps N\|--checkpoint ID]` | 执行回退 |
| `/readiness` | 检查运行时/提供者健康 |
| `/help` | 帮助 |

### 12.3 运行模式

| 命令 | 模式 | 场景 |
|------|------|------|
| `minicode-py` | 交互式 TUI | 日常开发 |
| `minicode-headless "prompt"` | 无头单次 | CI/CD |
| `minicode-gateway` | HTTP 网关 | 接入 Bot |
| `minicode-cron` | 定时任务 | 自动化 |

### 12.4 运行时档位

| 档位 | 最大步数 | 严格验证 | 适用场景 |
|------|----------|----------|----------|
| `single` | 50 | 否 | 日常快速任务 |
| `single-deep` | 80 | 是 | 复杂深度任务 |

---

## 13. 双语言实现对比

### 13.1 功能覆盖

| 模块 | TypeScript (`ts-src/`) | Python (`minicode/`) |
|------|:---:|:---:|
| Agent Loop | ✓ | ✓（更丰富） |
| Tool System (12 tools) | ✓ | ✓（30+ tools） |
| TUI | ✓ | ✓（更丰富） |
| MCP | ✓ | ✓（4种传输） |
| Permissions | ✓ | ✓ |
| Skills | ✓ | ✓ |
| 控制论系统 | ✗ | ✓（10+ 控制器） |
| 记忆系统 | ✗ | ✓（三级 + 管道） |
| 会话/检查点/回退 | ✗ | ✓ |
| 上下文压缩 | ✗ | ✓ |
| 成本控制 PID | ✗ | ✓ |
| 任务图 | ✗ | ✓ |

### 13.2 技术栈对比

| 维度 | TypeScript | Python |
|------|------------|--------|
| 运行时 | Node.js (ESM) | Python 3.11+ |
| 类型系统 | TypeScript (静态) | 类型注解 + TypedDict |
| 包管理 | npm | pip + pyproject.toml |
| 测试 | vitest | pytest (927 通过) |
| TUI | 自研 ink 风格 | Rich + 自研 |
| 工具验证 | Zod schema | 类内验证 |

---

## 14. 设计模式与工程实践

### 14.1 使用到的设计模式

| 模式 | 应用 | 代码位置 |
|------|------|----------|
| **适配器** | ModelAdapter → Anthropic/OpenAI | `anthropic_adapter.py`, `openai_adapter.py` |
| **注册表** | ToolRegistry, ModelRegistry | `tooling.py`, `model_registry.py` |
| **策略** | PermissionPolicy (allow/ask/deny) | `permissions.py` |
| **观察者** | 回调: onToolStart, onToolResult | `agent_loop.py` |
| **管道** | PromptPipeline, MemoryPipeline | `prompt_pipeline.py`, `memory_pipeline.py` |
| **模板方法** | 工具执行通用流程 | `tooling.py` |
| **工厂** | `create_model_adapter()` | `model_registry.py:517` |
| **责任链** | 多级权限策略链 | `permissions.py` |
| **发布-订阅** | Hooks 系统 | `hooks.py` |
| **PID 控制器** | 成本控制、上下文控制 | `cost_control.py` |
| **状态模式** | TurnRecurrentState 生命周期 | `turn_kernel.py` |

### 14.2 测试策略

- **927 个测试**通过，2 个跳过
- 测试文件按模块对应：`test_agent_loop.py`, `test_memory_e2e.py`, `test_session.py` 等
- 共享 fixtures 定义在根目录 `conftest.py`（`memory_manager`, `temp_workspace` 等）
- 使用 `pytest.mark.benchmark` 进行性能基准测试
- 集成测试覆盖端到端场景（`test_integration.py`, `test_memory_e2e.py`）

---

## 15. 面试高频问题 & 参考答案

### Q1: MiniCode 的 Agent Loop 是如何工作的？

**答**：核心是 `run_agent_turn()` 函数（`agent_loop.py:662`），实现经典的 感知→思考→行动→观察 循环：

1. **Prelude（前奏）**：构建任务图、分层上下文、初始化控制论控制器
2. **主循环**：while step < max_steps
   - `turn_kernel.derive_turn_step_policy()` 计算当前阶段（explore/execute/verify）
   - 调用 LLM → 解析响应（文本/工具调用/进度消息/空响应）
   - 工具调用前经过权限检查，支持并行执行
   - 控制论信号采集 + 动作执行（compaction, checkpoint, budget adjust）
3. **停止条件**：done / max_steps / await_user / blocked / verification_failed / widen_needed

关键是 **Turn Kernel 的三阶段模型**（探索→执行→验证）和 **Widening 机制**（从窄路径自我解救），这让 Agent 不会在死胡同里无限循环。

### Q2: Widening 是什么？为什么需要它？

**答**：Widening 是 Agent 的"拓宽"机制——当模型在窄路径上反复失败（工具错误、空响应、可恢复中断），系统检测到后自动激活 widening 模式，引导模型"比较替代方案、不要重复同一攻击线路"。

触发条件（`_derive_widening_signal`）：
1. 工具错误数量 > 0 且到达 widen 步数阈值
2. 已有验证证据但仍 stall
3. 无工具结果但空响应超过上限
4. 可恢复中断反复发生

这解决了 LLM Agent 的常见问题：模型倾向于在同一个思路上反复尝试而不知道"换条路"。

### Q3: MiniCode 的三级记忆系统是如何设计的？

**答**：三级记忆分别是：

1. **Working Memory（工作记忆）**：当前任务上下文，受 compaction 保护，有 TTL（1800s/7200s）
2. **Project Memory（项目记忆）**：长期知识，跨会话持久化到 `~/.mini-code/memory/memory.json`，分 user/project/local 三个作用域
3. **Memory Pipeline（记忆管道）**：闭合回路 T1(检索) → T2(重排序) → T3(注入) → T4(反思) → 写回

记忆值函数：`V(m) = relevance × freshness × utility`
- relevance: BM25 分数
- freshness: exp(-age/30天)
- utility: 使用次数的亚线性函数

最特别的是**记忆感知的 compaction**——压缩上下文时不会丢弃工作记忆，而是保留任务关键信息。

### Q4: 赛博控制论（Engineering Cybernetics）在这个项目中具体指什么？

**答**：借鉴经典控制理论（PID 控制、前馈、解耦、状态观测）来动态管理 Agent 运行时：

- **PID 反馈控制**：监控上下文使用率/成本速率，计算误差 → 调整 token 预算/并发度
- **前馈控制**：在任务开始前预判资源需求
- **预测控制**：基于历史趋势超前调整
- **解耦控制**：防止上下文和成本两个维度互相干扰
- **稳定性监控**：检测振荡，避免积分饱和(anti-windup)
- **Lyapunov 稳定性分析**：数学上证明了 PID 控制器的渐近稳定性

实际效果：Agent 不会因为突发大量工具调用而耗尽上下文窗口，也不会因为过度保守而给出浅层回答。

### Q5: MiniCode 如何实现文件的回退（Rewind）？

**答**：通过 `session.py` 中的检查点系统：

1. 每次文件编辑操作前，自动创建检查点（保存原始内容）
2. `/checkpoints` 查看检查点历史
3. `/rewind-preview` 预览回退影响（哪些文件会变，变成什么样）
4. `/rewind` 执行回退——支持三种粒度：
   - 回退最近编辑组
   - 回退 N 步
   - 回退到指定检查点 ID
5. 编辑按"安全组"分组，防止部分回退导致不一致状态

### Q6: MiniCode 如何集成外部 MCP 服务？

**答**：通过 `mcp.py` 实现完整的 MCP 协议客户端：

1. 启动时从 `.mcp.json` / `~/.mini-code/mcp.json` 加载 MCP Server 列表
2. 为每个 Server 启动子进程（stdio）或建立网络连接（SSE/HTTP/WebSocket）
3. 通过 MCP 协议握手协商（支持 2024-11-05 和 2025-03-26 版本兼容）
4. 发现远程工具，包装成本地 `ToolDef` 注册到 `ToolRegistry`
5. 外部 MCP 工具**同样受权限系统约束**，不是免检通道
6. 退出时通过 `dispose()` 清理所有 MCP 连接

### Q7: 为什么有 Python 和 TypeScript 两个版本？它们的区别是什么？

**答**：
- **TypeScript 版**（`ts-src/`）是主线版本，更接近 Claude Code 的终端体验，代码更精简，专注于核心循环 + TUI + MCP
- **Python 版**（`minicode/`）是本地优先的增强版本，在核心循环之上增加了完整的控制论系统、三级记忆、会话持久化、检查点回退等

设计意图是：TS 版适合学习 AI Agent 的基本架构，Python 版展示了如何用控制理论方法来增强 Agent 的鲁棒性和可观测性。两者共享相同的核心设计思想（model → tool → model 循环），但 Python 版在运行时工程上走得更远。

### Q8: 上下文压缩（Compaction）是如何触发的？如何保证不丢失关键信息？

**答**：
1. **触发条件**：上下文使用率超过阈值（默认 ~70%）
2. **压缩策略**：
   - 裁剪旧的工具结果，保留最近的
   - 激进的阶段（verify/widening）使用更小的 token 预算
3. **信息保护**：
   - Working Memory 中的条目（任务标题、目标、进度）不被清除
   - StableTaskPack 提供受保护的上下文
   - 记忆注入的内容被标记为 protect，压缩时绕过

### Q9: MiniCode 的模型适配层如何支持多种 LLM 后端？

**答**：通过 Adapter 模式 + Provider 检测：

```python
# 1. Provider 检测：根据模型名和配置推断后端类型
detect_provider("deepseek-chat", runtime) → Provider.CUSTOM

# 2. build_provider_config：构建 provider 专用配置
provider_config = build_provider_config(model, runtime)
# → ProviderConfig(base_url, api_key, model, extra_headers)

# 3. create_model_adapter：工厂方法创建适配器
if provider_config.is_openai_compatible:
    return OpenAIModelAdapter(enriched_runtime, tools)  # OpenAI/Custom/OpenRouter
else:
    return AnthropicModelAdapter(enriched_runtime, tools)  # Anthropic
```

支持的后端：Anthropic（原生）、OpenAI、OpenRouter（200+ 模型代理）、自定义端点（DeepSeek、Ollama、vLLM 等任何 OpenAI 兼容 API）。

### Q10: 这个项目的技术亮点是什么？

**答**：
1. **工程控制论方法**：将 Agent 运行时视为一个可控制的动态系统，使用 PID/前馈/预测控制器管理上下文、成本、并发度，有数学稳定性保证
2. **记忆感知的上下文管理**：三级记忆 + 记忆管道闭合回路，compaction 不会丢弃关键信息
3. **可恢复的编辑系统**：检查点 + 回退预览 + 安全组，编辑错误可以回退
4. **Widening 机制**：Agent 自我检测死胡同并自动切换策略
5. **多提供者热切换**：运行中可切换模型，不影响会话状态
6. **会话可检查/可回放**：不是黑箱，每次运行的决策过程都可审计

### Q11: 这个项目还有哪些不足/改进空间？

**答**（诚实回答，展示批判性思维）：
1. **Windows 兼容性**：1 个测试在 Windows 上因文件权限失败
2. **配置系统耦合**：`~/.claude/settings.json` 中的 `"model": "haiku"` 会干扰 MiniCode，需用 `MINI_CODE_MODEL` 覆盖
3. **TypeScript 版功能滞后**：没有记忆系统、控制论、会话持久化
4. **TUI 复杂度**：`tty_app.py` + `tui/` 目录超过 20 个文件，组件耦合较紧
5. **文档分散**：大量 `.md` 报告散布在 `docs/`、`py-src/`、`ts-src/` 中
6. **无 LSP 集成**：代码导航依赖简单的文本搜索而非语言服务器
7. **单 Agent 架构**：没有真正的多 Agent 协作（`agent_protocol.py` 只定义了协议，未完全集成）

---

## 附录：关键文件速查表

| 文件 | 行数(约) | 职责 |
|------|----------|------|
| `agent_loop.py` | 2700 | 核心 Agent 循环 |
| `turn_kernel.py` | 800 | 步进策略、阶段、widening |
| `session.py` | 1300 | 会话、检查点、回退 |
| `memory.py` | 1900 | 长期记忆管理 |
| `timeline_memory.py` | 4400 | 决策时间线 |
| `memory_pipeline.py` | 600 | 记忆管道 |
| `context_compactor.py` | 1000 | 上下文压缩 |
| `context_manager.py` | 950 | 上下文窗口管理 |
| `cybernetic_orchestrator.py` | 500 | 控制论协调器 |
| `cost_control.py` | 400 | PID 成本控制 |
| `mcp.py` | 850 | MCP 协议集成 |
| `permissions.py` | 570 | 权限系统 |
| `model_registry.py` | 600 | 模型注册与适配器工厂 |
| `tty_app.py` | 290 | TTY 应用 |
| `cli_commands.py` | 1000 | 斜杠命令 |
| `config.py` | 750 | 配置加载与合并 |
