# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## Project Overview

MiniCode is an open-source AI coding agent with dual Python and TypeScript implementations. The **active Python package** is `minicode/` at the repository root, configured by `pyproject.toml` as `minicode-py`. The TypeScript implementation lives in `ts-src/`. Both share the same architectural pattern: a `model → tool → model` loop with TUI, permissions, MCP server support, and a skills system.

## Commands

### Install
```bash
python -m pip install -e .[dev]
```

### Test
```bash
# Run all tests (1030+ passed)
pytest -q

# Run a single test file
pytest -q tests/test_agent_loop.py

# Run a specific test
pytest -q tests/test_session.py -k "test_checkpoint"

# Compile check (always run before pytest)
python -m compileall -q minicode tests
```

### Run
```bash
minicode-py          # Interactive TUI agent
minicode-headless "Explain this repo."  # Single-shot headless mode
minicode-gateway      # Start gateway server
minicode-cron         # Run scheduled tasks
```

### TypeScript (ts-src/)
```bash
cd ts-src
npm install
npm run dev           # Run in development
npm run check         # Type-check with tsc
```

## Architecture

```
User input → agent_loop.py → turn_kernel.py (phase policy, widening, verification gate)
                ↕
           Model adapters (anthropic_adapter.py, openai_adapter.py)
                ↕
           Tool registry → tools/* (files, shell, web, git, code review, etc.)
```

The Python runtime wraps the agent loop with a **cybernetic control layer** that monitors signals (context pressure, cost, errors, progress) and triggers runtime actions (compaction, checkpoint, rewind, budget adjustment, recovery). This is orchestrated by `cybernetic_orchestrator.py`.

### Key Python modules

| Module | Role |
|--------|------|
| `minicode/agent_loop.py` | Core `model → tool → model` loop with event flow and product integration |
| `minicode/turn_kernel.py` | Step policy, phase transitions, widening logic, and verification gates |
| `minicode/session.py` | Durable sessions with inspect, replay, checkpoint, and rewind |
| `minicode/cli_commands.py` | Slash commands: /session, /memory, /checkpoints, /rewind, /readiness |
| `minicode/memory.py` + `working_memory.py` + `memory_pipeline.py` | Three-tier memory: long-term storage, protected working entries, retrieval/injection pipeline |
| `minicode/context_manager.py` + `context_compactor.py` | Context window management and memory-aware compaction |
| `minicode/cybernetic_orchestrator.py` | Runtime control lifecycle — reads signals, triggers actions |
| `minicode/model_registry.py` + `model_switcher.py` | Multi-provider model registry with fallback and failover |
| `minicode/prompt.py` + `prompt_pipeline.py` | System prompt construction and pipelined processing |
| `minicode/permissions.py` | Path, command, and tool permission enforcement |
| `minicode/mcp.py` | MCP server lifecycle for stdio/SSE/HTTP/WebSocket transports |
| `minicode/skills.py` | Skill discovery from `.Codex/skills/` directories |
| `minicode/hooks.py` | Lifecycle hooks (session-start, tool-before/after, etc.) |
| `minicode/tty_app.py` + `tui/` | Terminal UI: screen, input, markdown rendering, transcript panel |

### TypeScript architecture (ts-src/)

Mirrors the Python version in a lighter form: `agent-loop.ts`, `tool.ts` + `tools/`, `tty-app.ts` + `tui/`, `mcp.ts`, `permissions.ts`, `skills.ts`, `anthropic-adapter.ts`. See `ts-src/ARCHITECTURE.md` for details.

## Configuration

- `pyproject.toml` — package definition, pytest options (`pythonpath = ["."]`, `testpaths = ["tests"]`)
- `.env.example` — environment variable reference (API keys, model selection, logging, gateway, cron)
- `.mcp.json` — project-level MCP server configuration
- `conftest.py` — shared pytest fixtures (`memory_manager`, `temp_workspace`, etc.) and collection ignore rules
- Config search path (Python): defaults → `~/.mini-code/config.json` → `.mini-code/config.json` → env vars (`MINICODE_*`) → CLI args
- Config search path (TS): `~/.mini-code` directory

## Memory system directories (runtime-generated, not committed)

- `.mini-code-memory/` — durable workspace-level memory
- `.mini-code-memory-local/` — local-only memory
- `.mini-code-session-memory/` — per-session memory
