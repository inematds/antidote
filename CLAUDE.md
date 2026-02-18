# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Antidote** is a personal AI assistant built in Python 3.11+. It runs locally on macOS, communicates via Telegram (long-polling), stores memories in SQLite FTS5, and talks to LLMs through OpenRouter or Ollama. The full specification lives in `doc/antidote.md`.

> Status: v0.1.0 — 35 files, 3,412 lines target. Implementation in progress.

## Commands

```bash
# Setup and run
./start.sh                         # One-command launcher: creates venv, installs, runs
python3 -m venv .venv && source .venv/bin/activate
pip install -e .
antidote                           # Runs wizard if no config, else starts bot
antidote setup                     # Force reconfigure

# Tests (pytest, once test suite exists)
python -m pytest tests/ -v
python -m pytest tests/test_config.py -v   # Single test file
```

Config is generated at `~/.antidote/config.json` (gitignored). Secrets go to `~/.antidote/.secrets` (Fernet-encrypted). Memory DB at `~/.antidote/memory.db`.

## Architecture

```
Telegram (long-polling)
    → channels/telegram.py → IncomingMessage
    → agent/loop.py (tool-calling loop, max 5 rounds)
        → agent/context.py (assembles system prompt from SOUL/AGENTS/USER.md + memory search)
        → providers/openrouter.py or providers/ollama.py → LLMResponse
        → tools/registry.py dispatches tool calls
        → memory/store.py (SQLite FTS5 save/search)
    → channels/telegram.py → send OutgoingMessage
```

Every subsystem is built on abstract base classes — new providers, channels, tools, and memory backends plug in by implementing the interface, without touching existing code.

## Interface Contracts

All modules MUST implement their abstract base. Do not break these signatures.

**Provider** (`antidote/providers/base.py`): `async chat(messages, tools, model, temperature) -> LLMResponse`

**Channel** (`antidote/channels/base.py`): `async start(on_message)`, `async send(OutgoingMessage)`, `async stop()`

**Memory** (`antidote/memory/store.py`): `async save(content, category)`, `async search(query, limit)`, `async forget(id)`, `async recent(limit)`

**Tool** (`antidote/tools/base.py`): class attributes `name`, `description`, `parameters` (JSON Schema) + `async execute(**kwargs) -> ToolResult`

See `doc/antidote.md` for the full dataclass definitions.

## Identity Stack

Personality is defined by markdown files in `workspace/`:
- `SOUL.md` — core personality
- `AGENTS.md` — tool use and behavior rules
- `USER.md` — user profile and preferences
- `MEMORY.md` — bootstrap facts loaded at startup

These are loaded by `agent/context.py` to build the system prompt.

## Security

- API keys stored in `~/.antidote/.secrets` via Fernet encryption, keyed to macOS hardware UUID (machine-tied, useless if copied)
- `security/safety.py` enforces a command blocklist (`rm -rf /`, `mkfs`, `dd if=`, etc.) and path traversal protection
- Max 60s timeout per shell command; audit log at `~/.antidote/audit.log`
- `config.json` and `.secrets` are gitignored

## Design Constraints (Do Not Violate)

- **No Docker** — Mac-only personal use, not needed
- **No web UI** — Telegram is the only interface
- **No vector/embedding search** — SQLite FTS5 keyword search only
- **No multi-user logic** — single owner
- **No cross-platform abstractions** — macOS launchd for auto-restart
- **Modular by file addition** — new features add files, not edit existing modules

## Build Phases

Phase A (parallel): W1 Config/Secrets → W2 Providers, W3 Memory/Identity, W4 Telegram channel
Phase B: Agent loop, main entry point, wizard
Phase C: pyproject.toml, branding, README, packaging

Full workstream breakdown with task-level detail is in `doc/antidote.md`.
