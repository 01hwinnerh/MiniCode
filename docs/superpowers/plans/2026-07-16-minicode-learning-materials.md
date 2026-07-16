# MiniCode 深度学习资料 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 基于仓库中实际实现，产出一份项目全景学习手册和一份可用于技术面试的深度问答题库。

**Architecture:** 文档以根包 `minicode/` 为唯一事实来源，README 与测试作为能力边界的辅助证据。项目手册负责建立整体心智模型；题库复用该模型，针对每个问题给出口述答案、源码锚点、取舍与优化方向。

**Tech Stack:** Markdown、Python 3.11、pytest、MiniCode 源码与仓库文档。

---

### Task 1: 核验架构事实

**Files:**
- Read: `README.md`
- Read: `pyproject.toml`
- Read: `minicode/agent_loop.py`, `minicode/turn_kernel.py`, `minicode/session.py`
- Read: `minicode/memory_pipeline.py`, `minicode/mcp.py`, `minicode/permissions.py`

- [ ] **Step 1: 建立主包边界**

确认 `pyproject.toml` 中包名为 `minicode-py`，命令入口指向根目录 `minicode/`，将 `ts-src/` 作为并行实现而非当前 Python 主运行时。

- [ ] **Step 2: 列出每个核心模块的职责与交互**

以 `agent_loop.py` 为中心，标注 turn kernel、tools、memory、session、cybernetic orchestrator、permissions、MCP、model switcher 和 TUI 的输入输出关系。

### Task 2: 编写项目全景学习手册

**Files:**
- Create: `docs/MiniCode-项目全景学习手册.md`

- [ ] **Step 1: 写入零基础摘要、术语表和架构图**

说明 Coding Agent、模型、工具、上下文、会话、checkpoint、MCP 的关系，并提供一次请求的端到端时序。

- [ ] **Step 2: 写入模块级源码导读**

按 `agent_loop → turn_kernel → tools → memory → session → orchestrator` 展开，注明职责、关键方法、数据流及失败路径。

- [ ] **Step 3: 写入设计评估**

分别列出亮点、局限、可落地优化和面试表达边界，不把 README 愿景误称为已经实现的能力。

### Task 3: 编写面试题库与详解

**Files:**
- Create: `docs/MiniCode-面试题库与详解.md`

- [ ] **Step 1: 收录 46 道问题并标记 15 道重点题**

重点题使用 `🔥 **重点题**` 及加粗结论；问题按项目总览、运行时、模型、安全、会话、记忆、MCP、质量八组整理。

- [ ] **Step 2: 为每题写出口述答案与源码锚点**

每题至少包括：可直接口述的回答、代码位置、面试官追问、优化方向。将“当前实现”和“建议实现”显式分开。

### Task 4: 校对与交付

**Files:**
- Verify: `docs/MiniCode-项目全景学习手册.md`
- Verify: `docs/MiniCode-面试题库与详解.md`

- [ ] **Step 1: 扫描未完成占位符**

Run: `rg -n "TODO|TBD|待补充|实现以后" docs/MiniCode-项目全景学习手册.md docs/MiniCode-面试题库与详解.md`

Expected: 没有命中。

- [ ] **Step 2: 校验重点题数量和文档结构**

Run: `rg -n "🔥 \*\*重点题\*\*" docs/MiniCode-面试题库与详解.md`

Expected: 恰好 15 条命中。

