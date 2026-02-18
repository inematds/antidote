# Antidote — Build Plan

> **The antidote to bloated AI frameworks.**
>
> A purpose-built Python AI assistant for personal use. Runs on Mac, talks via Telegram, remembers everything, has a defined personality. Built from scratch (~3,000 lines), designed around 18 features, ships with 3 on day one.
>
> **Owner**: Mark Kashef
> **Location**: `/Users/marwankashef/Desktop/YouTube/OpenClaw Antidote/antidote/`
> **Repo**: `github.com/earlyaidopters/antidote` (private)
> **Reference repos**: Same parent directory (openclaw, zeroclaw, nanoclaw, nanobot, picoclaw, ironclaw, tinyclaw, agent-zero, awesome-openclaw)
> **Full analysis**: `FRAMEWORK-DEEP-DIVE.md` and `my-framework-ingredients.md` in parent directory
> **Status**: v0.1.0 shipped. All 5 workstreams complete. 35 files, 3,412 lines committed.

---

## Decisions Made

| Decision | Choice | Why |
|----------|--------|-----|
| Language | Python 3.11+ | Claude Code iterates fastest, LiteLLM ecosystem, huge library support |
| Deployment | Mac-local (always on) | Personal use, no cloud needed |
| AI Providers | OpenRouter (cloud) + Ollama (local) | OpenRouter = 100+ models via one key. Ollama = free, private fallback |
| Channel | Telegram (long-polling) | Simplest setup, no public URL needed, rich media support |
| Memory | SQLite FTS5 (keyword search) | 90% of recall needs, zero API cost, no embedding models needed |
| Identity | Soul.md stack (from NanoBot/ZeroClaw) | Markdown personality files, readable, editable |
| Onboarding | Themed setup wizard | OpenClaw-style branded CLI with ASCII art, color palette, dynamic taglines |
| CLI | `antidote` command | pip console_scripts entry point via pyproject.toml, auto-launches wizard if no config |
| Architecture | Modular agent loop | Each feature = a module. Add by creating files, not editing existing ones |
| Security | Encrypted secrets + command blocklist | Fernet encryption for API keys, blocklist for dangerous shell commands |
| Auto-restart | macOS launchd plist | Survives sleep, crash, reboot. Always running |

## What We Explicitly Chose NOT to Do

- No Docker (unnecessary for Mac-only personal use)
- No multi-user auth or pairing codes (single user)
- No web UI dashboard (Telegram IS the interface)
- No vector/embedding search (FTS5 keyword search is enough)
- No single binary (Python + start script is simpler for ongoing iteration)
- No gateway/WebSocket architecture (overkill for single-channel)
- No cross-platform support (Mac only, for now)

---

## Project Structure

```
antidote/
├── wizard.py                   # Themed setup wizard (rich + questionary)
├── start.sh                    # One-command launcher (venv + install + run)
├── com.antidote.agent.plist    # macOS launchd auto-restart
├── pyproject.toml              # Package config — `antidote` CLI entry point
├── requirements.txt            # Python dependencies
├── README.md                   # Legendary README with ASCII banner + badges
├── .gitignore                  # Excludes pycache, .venv, secrets, logs, db
├── antidote/
│   ├── __init__.py             # Version string
│   ├── __main__.py             # `python -m antidote` support
│   ├── main.py                 # Entry point — CLI router + wires everything
│   ├── config.py               # Config loader (config.json + .env)
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── loop.py             # Core agent loop
│   │   ├── context.py          # System prompt builder
│   │   └── profiles.py         # Agent profile switcher [Phase 2]
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── base.py             # Provider interface (abstract)
│   │   ├── openrouter.py       # OpenRouter via LiteLLM
│   │   └── ollama.py           # Local Ollama (built in Phase 1)
│   ├── channels/
│   │   ├── __init__.py
│   │   ├── base.py             # Channel interface (abstract)
│   │   └── telegram.py         # Telegram Bot API long-polling
│   ├── memory/
│   │   ├── __init__.py
│   │   ├── store.py            # SQLite FTS5 memory
│   │   └── solutions.py        # Solution memory [Phase 2]
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── base.py             # Tool interface (abstract)
│   │   ├── registry.py         # Tool discovery and registration
│   │   ├── filesystem.py       # File read/write/list
│   │   ├── shell.py            # Command execution (with safety)
│   │   ├── web.py              # Web search/fetch [Phase 2]
│   │   ├── browser.py          # Browser automation [Phase 3]
│   │   ├── screenshot.py       # Screenshot & vision [Phase 3]
│   │   ├── cron.py             # Cron scheduling [Phase 2]
│   │   └── webhook.py          # Webhook triggers [Phase 3]
│   ├── skills/
│   │   ├── __init__.py
│   │   ├── loader.py           # Skill discovery & loading [Phase 2]
│   │   └── builtin/            # Built-in skills as markdown [Phase 2]
│   ├── integrations/
│   │   ├── __init__.py
│   │   ├── obsidian.py         # Obsidian vault [Phase 3]
│   │   ├── onepassword.py      # 1Password CLI [Phase 3]
│   │   ├── searxng.py          # SearXNG search [Phase 3]
│   │   ├── imagegen.py         # Image generation [Phase 3]
│   │   └── mcp.py              # MCP protocol client [Phase 3]
│   └── security/
│       ├── __init__.py
│       ├── secrets.py          # Encrypted secret store
│       └── safety.py           # Command blocklist
├── workspace/
│   ├── SOUL.md                 # AI personality definition
│   ├── AGENTS.md               # Behavior rules and instructions
│   ├── USER.md                 # User preferences
│   └── MEMORY.md               # Bootstrap long-term memory
└── config.json                 # Generated by wizard (gitignored)
```

---

## Interface Contracts

These are the abstract base classes every module implements. Agents building different modules MUST conform to these interfaces so everything plugs together.

### Provider Interface (`antidote/providers/base.py`)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class Message:
    role: str          # "system", "user", "assistant", "tool"
    content: str
    tool_calls: list | None = None
    tool_call_id: str | None = None

@dataclass
class ToolDefinition:
    name: str
    description: str
    parameters: dict   # JSON Schema

@dataclass
class LLMResponse:
    content: str | None
    tool_calls: list | None    # List of {id, name, arguments}
    usage: dict | None         # {prompt_tokens, completion_tokens}

class BaseProvider(ABC):
    @abstractmethod
    async def chat(
        self,
        messages: list[Message],
        tools: list[ToolDefinition] | None = None,
        model: str | None = None,
        temperature: float = 0.7,
    ) -> LLMResponse:
        """Send messages to LLM, get response. Supports tool calling."""
        ...
```

### Channel Interface (`antidote/channels/base.py`)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class IncomingMessage:
    text: str
    sender_id: str
    sender_name: str
    chat_id: str
    timestamp: float
    media: list | None = None   # List of {type, url, caption}

@dataclass
class OutgoingMessage:
    text: str
    chat_id: str
    media: list | None = None
    reply_to: str | None = None

class BaseChannel(ABC):
    @abstractmethod
    async def start(self, on_message: callable) -> None:
        """Start listening. Call on_message(IncomingMessage) for each incoming."""
        ...

    @abstractmethod
    async def send(self, message: OutgoingMessage) -> None:
        """Send a message to the channel."""
        ...

    @abstractmethod
    async def stop(self) -> None:
        """Gracefully disconnect."""
        ...
```

### Memory Interface (`antidote/memory/store.py`)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class MemoryEntry:
    id: int
    content: str
    category: str       # "fact", "conversation", "solution", "preference"
    created_at: str
    relevance: float    # Search relevance score (0-1)

class BaseMemory(ABC):
    @abstractmethod
    async def save(self, content: str, category: str = "fact") -> int:
        """Store a memory. Returns the ID."""
        ...

    @abstractmethod
    async def search(self, query: str, limit: int = 10) -> list[MemoryEntry]:
        """Search memories by keyword. Returns ranked results."""
        ...

    @abstractmethod
    async def forget(self, memory_id: int) -> bool:
        """Delete a specific memory."""
        ...

    @abstractmethod
    async def recent(self, limit: int = 20) -> list[MemoryEntry]:
        """Get most recent memories."""
        ...
```

### Tool Interface (`antidote/tools/base.py`)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class ToolResult:
    success: bool
    output: str
    error: str | None = None

class BaseTool(ABC):
    name: str               # Unique identifier (e.g., "read_file")
    description: str        # Shown to LLM — what does this tool do?
    parameters: dict        # JSON Schema for inputs

    @abstractmethod
    async def execute(self, **kwargs) -> ToolResult:
        """Run the tool with given parameters. Return result."""
        ...
```

---

## Agent Team Workstreams

The build is split into **5 parallel workstreams** that can execute simultaneously. Dependencies between workstreams are marked explicitly.

```
WORKSTREAM DEPENDENCY MAP
=========================

  [W1: Config & Secrets]
         |
         v
  [W2: Provider Layer] ----+----> [W5: Main + Wizard + Launcher]
         |                 |              ^        ^
         v                 |              |        |
  [W3: Memory & Identity]--+              |        |
         |                                |        |
         v                                |        |
  [W4: Telegram Channel]  ---------------+        |
         |                                         |
  [W4: Agent Loop] -------------------------------+
                                                   |
                                                   v
                                    [W6: CLI, Branding, Packaging]

BUILD ORDER:
  Phase A (parallel): W1, W2-base, W3-identity, W4-channel can all start simultaneously
  Phase B (needs W1-W4): W4-agent-loop, W5
  Phase C (needs W1-W5): W6 — pyproject.toml, branded wizard, README, .gitignore
  Phase D: Git init, commit, push to earlyaidopters/antidote (private)
```

---

### Workstream 1: Config, Secrets, and Safety

**Agent role**: Infrastructure / security specialist
**Files to create**: 5 files
**Depends on**: Nothing (can start immediately)
**Blocks**: Everything else (all modules read config)

#### Task 1.1 — Config Loader (`antidote/config.py`)

Create the configuration system. Loads from `config.json` (generated by wizard) with `.env` override support.

**Config schema** (what config.json looks like):

```json
{
  "name": "Antidote",
  "version": "0.1.0",
  "providers": {
    "default": "openrouter",
    "openrouter": {
      "model": "anthropic/claude-sonnet-4-20250514"
    },
    "ollama": {
      "model": "llama3.2",
      "base_url": "http://localhost:11434"
    }
  },
  "channels": {
    "telegram": {
      "enabled": true
    }
  },
  "memory": {
    "db_path": "~/.antidote/memory.db",
    "max_context_memories": 10
  },
  "workspace": "~/.antidote/workspace",
  "identity": {
    "soul": "workspace/SOUL.md",
    "agents": "workspace/AGENTS.md",
    "user": "workspace/USER.md"
  },
  "safety": {
    "blocked_commands": ["rm -rf /", "mkfs", "dd if=", "shutdown", "reboot", "> /dev/sd"],
    "max_command_timeout": 60
  }
}
```

**Requirements**:
- Load config.json from `~/.antidote/config.json`
- Expand `~` in all paths
- Merge with defaults (so config.json can be sparse)
- Validate required fields exist
- Expose as a singleton `Config` object importable everywhere
- Read secrets from encrypted store (see Task 1.2), falling back to env vars

**Reference**: NanoBot's `nanobot/config/schema.py` (Pydantic-based, clean)

#### Task 1.2 — Encrypted Secret Store (`antidote/security/secrets.py`)

Store API keys encrypted at rest. Uses Python's `cryptography` library (Fernet symmetric encryption). The encryption key is derived from a machine-specific identifier (macOS hardware UUID) so the secrets file is useless if copied to another machine.

**Requirements**:
- `save_secret(name: str, value: str)` — Encrypt and store
- `get_secret(name: str) -> str | None` — Decrypt and return
- `list_secrets() -> list[str]` — Return secret names (not values)
- `delete_secret(name: str)` — Remove a secret
- Storage location: `~/.antidote/.secrets` (JSON, values are Fernet-encrypted strings)
- Key derivation: Use macOS `ioreg` hardware UUID + PBKDF2 to derive Fernet key
- Fallback: If hardware UUID unavailable, prompt user for a password on first run

**Secrets stored**: `OPENROUTER_API_KEY`, `TELEGRAM_BOT_TOKEN`, plus any future API keys

**Reference**: ZeroClaw's `src/security/` (ChaCha20, similar concept but Rust)

#### Task 1.3 — Command Safety (`antidote/security/safety.py`)

Prevent the AI from running dangerous shell commands.

**Requirements**:
- `is_safe(command: str) -> tuple[bool, str | None]` — Returns (safe, reason_if_blocked)
- Check against blocklist from config (substring match)
- Block path traversal patterns (`../` beyond workspace)
- Enforce command timeout (default 60s from config)
- Log all executed commands to `~/.antidote/audit.log`

**Reference**: NanoBot's safety guards in `nanobot/agent/tools/`, PicoClaw's blocklist

#### Task 1.4 — Package Init (`antidote/__init__.py`)

```python
__version__ = "0.1.0"
__name__ = "antidote"
```

#### Task 1.5 — Requirements File (`requirements.txt`)

```
litellm>=1.40.0
python-telegram-bot>=21.0
cryptography>=42.0
aiosqlite>=0.20.0
python-dotenv>=1.0
rich>=13.0
questionary>=2.0
```

**Acceptance criteria**: Another agent can `from antidote.config import Config` and get a fully loaded config object. Secrets are encrypted at rest. Safety checker works standalone.

---

### Workstream 2: Provider Layer (OpenRouter + Ollama)

**Agent role**: AI/LLM integration specialist
**Files to create**: 4 files
**Depends on**: W1 (needs Config for API keys and model selection)
**Blocks**: W5 (agent loop needs a provider to call)

#### Task 2.1 — Provider Base Class (`antidote/providers/base.py`)

Implement the `BaseProvider` abstract class and all dataclasses exactly as defined in the Interface Contracts section above. This is the contract all providers implement.

#### Task 2.2 — OpenRouter Provider (`antidote/providers/openrouter.py`)

Primary cloud provider via LiteLLM.

**Requirements**:
- Implements `BaseProvider`
- Uses LiteLLM's `acompletion()` for async calls
- Model format: `openrouter/anthropic/claude-sonnet-4-20250514` (LiteLLM prefix)
- Reads API key from secrets store (Task 1.2), falls back to `OPENROUTER_API_KEY` env var
- Supports tool calling (function calling) — pass tools as LiteLLM `tools` parameter
- Retry with exponential backoff on 429 (rate limit) and 5xx errors — max 3 retries
- Log token usage from response for cost awareness
- Timeout: 120 seconds per request

**Reference**: NanoBot's `nanobot/providers/litellm_provider.py` (same LiteLLM approach)

#### Task 2.3 — Ollama Provider (`antidote/providers/ollama.py`)

Local LLM fallback. Zero cost, full privacy.

**Requirements**:
- Implements `BaseProvider`
- Uses LiteLLM's `acompletion()` with `ollama/model_name` prefix
- Base URL from config (default `http://localhost:11434`)
- Graceful failure if Ollama isn't running (return clear error, don't crash)
- Tool calling support depends on model — check model capabilities, fall back to prompt-based tool use if native tool calling unavailable

**Reference**: NanoBot and PicoClaw both use LiteLLM for Ollama

#### Task 2.4 — Provider `__init__.py` (`antidote/providers/__init__.py`)

Factory function:

```python
from antidote.config import Config

def get_provider(name: str | None = None) -> BaseProvider:
    """Get provider by name. Defaults to config default."""
    name = name or Config().providers.default
    if name == "openrouter":
        from .openrouter import OpenRouterProvider
        return OpenRouterProvider()
    elif name == "ollama":
        from .ollama import OllamaProvider
        return OllamaProvider()
    else:
        raise ValueError(f"Unknown provider: {name}")
```

**Acceptance criteria**: `provider = get_provider(); response = await provider.chat(messages)` works end-to-end with OpenRouter. Tool calls come back as structured data, not raw text.

---

### Workstream 3: Memory System + Identity Files

**Agent role**: Data/memory specialist
**Files to create**: 7 files
**Depends on**: W1 (needs Config for db_path and workspace path)
**Blocks**: W5 (agent loop needs memory for context, context builder needs identity files)

#### Task 3.1 — SQLite FTS5 Memory Store (`antidote/memory/store.py`)

The brain. Stores facts, conversation summaries, and learned information with full-text search.

**Requirements**:
- Implements `BaseMemory` interface from contracts above
- Uses `aiosqlite` for async SQLite access
- WAL mode enabled (for concurrent reads during writes)
- Schema:

```sql
CREATE TABLE memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    content TEXT NOT NULL,
    category TEXT NOT NULL DEFAULT 'fact',
    created_at TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at TEXT
);

CREATE VIRTUAL TABLE memories_fts USING fts5(
    content,
    category,
    content='memories',
    content_rowid='id'
);

-- Triggers to keep FTS in sync
CREATE TRIGGER memories_ai AFTER INSERT ON memories BEGIN
    INSERT INTO memories_fts(rowid, content, category) VALUES (new.id, new.content, new.category);
END;

CREATE TRIGGER memories_ad AFTER DELETE ON memories BEGIN
    INSERT INTO memories_fts(memories_fts, rowid, content, category) VALUES ('delete', old.id, old.content, old.category);
END;

CREATE TRIGGER memories_au AFTER UPDATE ON memories BEGIN
    INSERT INTO memories_fts(memories_fts, rowid, content, category) VALUES ('delete', old.id, old.content, old.category);
    INSERT INTO memories_fts(rowid, content, category) VALUES (new.id, new.content, new.category);
END;
```

- `search()` uses FTS5 `MATCH` with `bm25()` ranking
- `save()` auto-deduplicates: if content is >80% similar to existing entry (by word overlap), update instead of insert
- DB location from config: `~/.antidote/memory.db`
- Auto-create tables on first access

**Reference**: ZeroClaw's `src/memory/sqlite.rs` (same FTS5 approach, translated to Python)

#### Task 3.2 — Memory `__init__.py` (`antidote/memory/__init__.py`)

Export `MemoryStore` as the default.

#### Task 3.3 — Context Builder (`antidote/agent/context.py`)

Builds the system prompt sent to the LLM on every message. Assembles identity files + relevant memories + conversation history.

**Requirements**:
- `build_system_prompt()` — Reads and concatenates:
  1. `SOUL.md` (personality — always included)
  2. `AGENTS.md` (behavior rules — always included)
  3. `USER.md` (user preferences — always included)
  4. Recent memories (last 10 from memory store, category "fact" or "preference")
  5. Available tools list (names and descriptions)
- `build_conversation_context(messages, query)` — Given conversation history + new user query:
  1. Search memory store for relevant memories matching the query
  2. Prepend relevant memories as a "Relevant memories:" section
  3. Keep conversation history under token budget (truncate oldest if over ~8000 tokens, rough estimate by char count / 4)
- Output format: list of `Message` objects ready for the provider

**Reference**: NanoBot's `nanobot/agent/context.py` (same pattern, Python)

#### Task 3.4 — SOUL.md (`workspace/SOUL.md`)

Default personality template. User can edit later.

```markdown
# Soul

You are Antidote, a personal AI assistant built for Mark.

## Personality
- Direct and concise. No fluff, no filler.
- Proactive: suggest things Mark hasn't asked for when relevant.
- Honest: if you don't know, say so. If an idea is bad, say why.
- Remember context from previous conversations.

## Communication Style
- Use short paragraphs. Bullet points for lists.
- Match Mark's energy — casual when he's casual, detailed when he needs detail.
- No emojis unless Mark uses them first.
- When sharing information, lead with the answer, then context.

## Values
- Privacy first. Never share Mark's data externally.
- Efficiency over thoroughness. A good answer now beats a perfect answer later.
- Be opinionated. Mark wants a sparring partner, not a yes-man.
```

#### Task 3.5 — AGENTS.md (`workspace/AGENTS.md`)

```markdown
# Agent Instructions

## Core Behavior
- Always check memory before answering. If you've discussed this topic before, reference it.
- When you learn something new about Mark or his preferences, save it to memory.
- If a task requires multiple steps, outline them before executing.
- When uncertain, ask for clarification rather than guessing.

## Tool Usage
- Use tools when they'd be more accurate than your knowledge.
- For file operations, always confirm paths before writing.
- Shell commands: prefer safe, read-only commands. Ask before anything destructive.

## Memory Management
- Save important facts, preferences, and decisions to memory.
- Don't save trivial or temporary information.
- When Mark corrects you, update the relevant memory.
```

#### Task 3.6 — USER.md (`workspace/USER.md`)

```markdown
# User Profile

## About Mark
- Non-technical. Explain technical concepts simply.
- Prefers actionable advice over theory.
- Values time — be concise.

## Preferences
- (The setup wizard and ongoing conversations will populate this)
```

#### Task 3.7 — MEMORY.md (`workspace/MEMORY.md`)

```markdown
# Bootstrap Memory

This file is loaded on first run to seed the memory database.

## Facts
- Mark built Antidote as a personal AI assistant.
- Antidote runs on Mark's Mac and communicates via Telegram.
- Mark uses OpenRouter for cloud AI and Ollama for local/free AI.
```

**Acceptance criteria**: `context = ContextBuilder(config, memory); messages = await context.build(conversation, query)` returns a properly formatted message list with personality, memories, and history.

---

### Workstream 4: Telegram Channel + Agent Loop

**Agent role**: Core application developer
**Files to create**: 6 files
**Depends on**: W1 (config), W2 (provider for LLM calls), W3 (memory + context)
**Blocks**: W5 (main.py wires this together)

**NOTE**: The channel (Task 4.1-4.2) can start in parallel with W2 and W3 since it only needs W1. The agent loop (Task 4.3) needs W2 and W3 complete.

#### Task 4.1 — Channel Base Class (`antidote/channels/base.py`)

Implement `BaseChannel` exactly as defined in Interface Contracts above.

#### Task 4.2 — Telegram Channel (`antidote/channels/telegram.py`)

The primary (and for now, only) way to talk to Antidote.

**Requirements**:
- Implements `BaseChannel`
- Uses `python-telegram-bot` library (async, v21+)
- Long-polling mode (no webhook, no public URL needed)
- Bot token from secrets store, falls back to `TELEGRAM_BOT_TOKEN` env var
- Handle text messages, photos (with captions), documents, voice messages
- Voice messages: save to temp file, note in message that transcription is a future feature
- Photos: save to temp file, pass file path in `IncomingMessage.media`
- Long responses (>4096 chars): split into multiple Telegram messages at paragraph boundaries
- Markdown formatting: send with `parse_mode="Markdown"`, fall back to plain text if Telegram rejects the formatting
- Typing indicator: send `chat_action=typing` while waiting for LLM response
- Error handling: if LLM call fails, send user a friendly error message, don't crash
- Only respond in private DMs (not groups) by default. Config option to enable group mode later.

**Reference**: NanoBot's `nanobot/channels/telegram.py`, PicoClaw's `pkg/channels/telegram.go`

#### Task 4.3 — Tool Base + Registry (`antidote/tools/base.py` and `antidote/tools/registry.py`)

**base.py**: Implement `BaseTool` exactly as in Interface Contracts.

**registry.py**: Central tool registry.

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, BaseTool] = {}

    def register(self, tool: BaseTool) -> None:
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool | None:
        return self._tools.get(name)

    def all(self) -> list[BaseTool]:
        return list(self._tools.values())

    def as_definitions(self) -> list[ToolDefinition]:
        """Convert all tools to ToolDefinition for LLM."""
        return [
            ToolDefinition(name=t.name, description=t.description, parameters=t.parameters)
            for t in self._tools.values()
        ]
```

#### Task 4.4 — Core Tools (`antidote/tools/filesystem.py` and `antidote/tools/shell.py`)

**filesystem.py** — Three tools: `read_file`, `write_file`, `list_directory`
- All paths restricted to workspace directory by default
- `read_file`: Read content, max 100KB, return as string
- `write_file`: Write/overwrite, create parent dirs if needed
- `list_directory`: Return file/folder listing with sizes

**shell.py** — One tool: `run_command`
- Execute shell commands via `asyncio.create_subprocess_shell`
- Check `safety.is_safe()` before execution
- Enforce timeout from config
- Capture stdout + stderr
- Return combined output (truncated to 10KB if larger)

#### Task 4.5 — Agent Loop (`antidote/agent/loop.py`)

The brain. Receives a message, thinks, optionally uses tools, responds.

**Requirements**:
- `process_message(incoming: IncomingMessage) -> str` — Main entry point
- Flow:
  1. Load relevant memories via context builder
  2. Build message list (system prompt + history + new message)
  3. Call provider with messages + available tools
  4. If LLM returns tool calls:
     a. Execute each tool via registry
     b. Append tool results to messages
     c. Call LLM again with tool results
     d. Repeat until LLM responds with text (max 5 tool rounds to prevent loops)
  5. Save conversation summary to memory (if substantive — skip "hi"/"thanks")
  6. Return final text response
- Maintain conversation history per chat_id (in-memory dict, last 50 messages)
- Include three built-in memory tools the LLM can call:
  - `save_memory(content, category)` — Store something for later
  - `search_memory(query)` — Look up past information
  - `forget_memory(id)` — Delete outdated info

**Reference**: NanoBot's `nanobot/agent/loop.py` (almost identical flow, Python)

#### Task 4.6 — Agent `__init__.py` + Channels `__init__.py` + Tools `__init__.py`

Simple exports for each package.

**Acceptance criteria**: Send a Telegram message, get an AI response that references your personality files and can use tools. Memory persists across restarts.

---

### Workstream 5: Main Entry Point + Setup Wizard + Launcher

**Agent role**: UX / onboarding specialist
**Files to create**: 4 files
**Depends on**: W1-W4 (wires everything together)
**Blocks**: Nothing (this is the final assembly)

#### Task 5.1 — Main Entry Point (`antidote/main.py`)

Wires all components together and starts the bot.

```python
import asyncio
import signal
from antidote.config import Config
from antidote.providers import get_provider
from antidote.memory.store import MemoryStore
from antidote.channels.telegram import TelegramChannel
from antidote.agent.loop import AgentLoop
from antidote.agent.context import ContextBuilder
from antidote.tools.registry import ToolRegistry
from antidote.tools.filesystem import ReadFileTool, WriteFileTool, ListDirTool
from antidote.tools.shell import RunCommandTool

async def main():
    config = Config()

    # Initialize components
    provider = get_provider()
    memory = MemoryStore(config.memory.db_path)
    await memory.initialize()

    # Register tools
    tools = ToolRegistry()
    tools.register(ReadFileTool(config))
    tools.register(WriteFileTool(config))
    tools.register(ListDirTool(config))
    tools.register(RunCommandTool(config))

    # Build agent
    context = ContextBuilder(config, memory, tools)
    agent = AgentLoop(provider, context, memory, tools)

    # Start Telegram channel
    telegram = TelegramChannel(config)

    # Graceful shutdown
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, lambda: asyncio.create_task(shutdown(telegram, memory)))

    print("Antidote is running. Send a message on Telegram.")
    await telegram.start(on_message=agent.process_message)

async def shutdown(channel, memory):
    print("\nShutting down gracefully...")
    await channel.stop()
    # memory auto-closes via aiosqlite
    asyncio.get_event_loop().stop()

if __name__ == "__main__":
    asyncio.run(main())
```

Adapt as needed — this is the target shape, not rigid.

#### Task 5.2 — Setup Wizard (`setup.py`)

Interactive terminal wizard using `questionary` and `rich` for beautiful prompts.

**Flow**:
1. Welcome banner (rich panel): "Welcome to Antidote setup"
2. Check Python version (3.11+ required)
3. Check if `~/.antidote/` exists. If yes, ask: "Existing installation found. Reconfigure or fresh start?"
4. Ask for **Telegram Bot Token**:
   - Show instructions: "Open Telegram, search @BotFather, send /newbot, follow prompts, paste the token here"
   - Validate: call Telegram API `getMe` to verify token works
   - Save to encrypted secrets store
5. Ask for **OpenRouter API Key**:
   - Show instructions: "Go to openrouter.ai/keys, create a key, paste it here"
   - Validate: make a test API call (list models)
   - Save to encrypted secrets store
6. Ask for **default model** (show top 5 OpenRouter models with prices):
   - Claude Sonnet 4 (recommended)
   - Claude Haiku 4 (cheaper)
   - GPT-4.1 Mini (cheapest smart model)
   - DeepSeek R1 (best value reasoning)
   - Custom (type model ID)
7. Ask: **"What should your AI's name be?"** (default: Antidote)
8. Ask: **"Describe your AI's personality in one sentence"** (default: use SOUL.md template)
9. Create directory structure: `~/.antidote/`, `~/.antidote/workspace/`
10. Write `config.json`
11. Write identity files (SOUL.md, AGENTS.md, USER.md, MEMORY.md)
12. Seed memory database with bootstrap facts
13. Install launchd plist (ask permission first)
14. Print success message + "Run `./start.sh` to begin" or "Antidote will start automatically on boot"

**Reference**: ZeroClaw's `zeroclaw onboard --interactive`, OpenClaw's `openclaw onboard`

#### Task 5.3 — Start Script (`start.sh`)

```bash
#!/bin/bash
# Antidote — Start Script
# Usage: ./start.sh

set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$SCRIPT_DIR"

# Check Python
if ! command -v python3 &> /dev/null; then
    echo "Error: Python 3 is required. Install from python.org"
    exit 1
fi

# Check venv exists, create if not
if [ ! -d ".venv" ]; then
    echo "Creating virtual environment..."
    python3 -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
else
    source .venv/bin/activate
fi

# Check config exists
if [ ! -f "$HOME/.antidote/config.json" ]; then
    echo "No config found. Running setup wizard..."
    python3 setup.py
fi

# Run Antidote
echo "Starting Antidote..."
python3 -m antidote.main
```

#### Task 5.4 — macOS launchd Plist (`com.antidote.agent.plist`)

Auto-start on login, restart on crash.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.antidote.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>ANTIDOTE_PATH/start.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    <key>ThrottleInterval</key>
    <integer>10</integer>
    <key>StandardOutPath</key>
    <string>ANTIDOTE_LOG_PATH/antidote.log</string>
    <key>StandardErrorPath</key>
    <string>ANTIDOTE_LOG_PATH/antidote-error.log</string>
    <key>WorkingDirectory</key>
    <string>ANTIDOTE_PATH</string>
</dict>
</plist>
```

The setup wizard replaces `ANTIDOTE_PATH` and `ANTIDOTE_LOG_PATH` with actual paths and installs to `~/Library/LaunchAgents/`.

**Acceptance criteria**: A non-technical user can run `antidote`, answer 5-6 questions, and have a working Telegram bot that auto-starts on login.

---

### Workstream 6: CLI, Branding, and Packaging (Post-build polish)

**Agent role**: UX / branding specialist
**Files to create/modify**: 6 files
**Depends on**: W1-W5 (everything working first)
**Blocks**: Nothing (final polish)

This workstream was added after the initial 5 workstreams were complete, inspired by OpenClaw's branded startup experience.

#### Task 6.1 — pyproject.toml (Package Configuration)

Create a proper Python package with a console script entry point so the bot can be launched with a single `antidote` command.

**Requirements**:
- `[project.scripts]` defines `antidote = "antidote.main:cli"`
- `[tool.setuptools.packages.find]` includes only `antidote*` (excludes `workspace/` from package discovery)
- All dependencies listed in `[project.dependencies]` (mirrors requirements.txt)
- Install via `pip install -e .` into project venv

**Key decision**: Renamed `setup.py` to `wizard.py` because pip's build system interprets `setup.py` as a package build script, which conflicts with the interactive wizard.

#### Task 6.2 — `__main__.py` (Module Runner)

Create `antidote/__main__.py` so `python -m antidote` works:

```python
from antidote.main import cli
cli()
```

#### Task 6.3 — CLI Router in `main.py`

Update `main.py` with smart CLI behavior:

**Requirements**:
- `antidote` with no args: auto-detect missing config → launch wizard → ask to start bot
- `antidote setup`: run wizard directly (for reconfiguring)
- Show ASCII banner + random tagline on startup
- Catch `ValueError` (missing keys) → suggest `antidote setup` instead of traceback
- Import wizard dynamically via `importlib.util` to avoid circular dependencies

#### Task 6.4 — Branded Wizard Theme (`wizard.py`)

Redesign the setup wizard with OpenClaw-style branding. Reference: OpenClaw's `src/cli/banner.ts`, `src/cli/tagline.ts`, `src/terminal/palette.ts`.

**Antidote Theme — Color Palette**:

| Name | Hex | Usage |
|------|-----|-------|
| Accent | `#00D26A` | Bright "antidote green" — primary brand color |
| Accent Dim | `#00A854` | Darker green — rules, borders |
| Cyan | `#00BCD4` | Clinical cyan — instructions, numbered steps |
| Muted | `#6B7280` | Gray — taglines, dim text |
| Warn | `#FFB020` | Amber — warnings |
| Error | `#EF4444` | Red — validation failures |

**ASCII Art Banner** (Calvin S figlet font):

```
    _   _  _ _____ ___ ___  ___ _____ ___
   /_\ | \| |_   _|_ _|   \/ _ \_   _| __|
  / _ \| .` | | |  | || |) | (_) || | | _|
 /_/ \_\_|\_| |_| |___|___/ \___/ |_| |___|
```

**Dynamic Taglines** (18 taglines, randomly selected on each run):

```python
TAGLINES = [
    "The antidote to bloated AI frameworks.",
    "Less framework. More you.",
    "Your AI. Your Mac. Your rules.",
    "One Telegram message away from useful.",
    "Built from scratch. Runs like it means it.",
    "No Docker. No cloud. No nonsense.",
    "2,989 lines of actual usefulness.",
    "The AI assistant that doesn't need a DevOps team.",
    "Bloated frameworks hate this one trick.",
    "Personal AI without the personal data harvesting.",
    "Because your AI shouldn't need Kubernetes.",
    "Lightweight enough to run on a philosophy.",
    "All the power. None of the YAML.",
    "Your terminal just got an upgrade.",
    "AI that remembers you. Runs on your Mac. Talks on Telegram.",
    "Encrypted at rest. Opinionated in conversation.",
    "Fewer dependencies than your morning coffee order.",
    "Small enough to read. Powerful enough to matter.",
]
```

**Questionary Custom Style** (green-themed prompt styling):

```python
QS = QStyle([
    ("qmark", "fg:#00D26A bold"),
    ("question", "bold"),
    ("answer", "fg:#00D26A bold"),
    ("pointer", "fg:#00D26A bold"),
    ("highlighted", "fg:#00D26A bold"),
    ("selected", "fg:#00D26A"),
])
```

**Wizard UI elements**:
- `_step(number, title)` — styled rule dividers between steps
- `_success_mark(msg)` — green checkmark (✓) for successes
- `_warn_mark(msg)` — amber warning marks for non-critical issues
- Model selection table with `box.ROUNDED` border style, speed/notes columns
- Validation feedback shows model count for OpenRouter, bot name for Telegram
- Final "Done" panel with green border

#### Task 6.5 — `.gitignore`

```
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/
.venv/
.env
*.db
config.json
.secrets
audit.log
*.log
.DS_Store
```

#### Task 6.6 — `README.md`

Legendary README with:
- Centered badge row (Python, Telegram, OpenRouter, SQLite, Encrypted)
- ASCII art banner in code block
- "What is this" / "What it is NOT" sections
- Quickstart with wizard output preview
- Full architecture tree
- Data flow diagram (ASCII)
- Identity stack table
- Memory, Security, Commands, Configuration sections
- Built-with dependency table
- Stats line: `~3,000 lines · 31 files · 7 dependencies · 0 Docker containers`

**Acceptance criteria**: Running `antidote` in a fresh terminal (with venv active) shows the green ASCII banner, a random tagline, then either launches the wizard or starts the bot. The repo README renders beautifully on GitHub.

---

## Phase 2 Features (After Day One Works)

Build these one at a time after the core is solid. Each is a self-contained module.

| # | Feature | Module | Effort | Depends On |
|---|---------|--------|--------|------------|
| ~~1~~ | ~~Ollama local provider~~ | ~~`providers/ollama.py`~~ | ~~Small~~ | **Shipped in v0.1.0** |
| 2 | Cron scheduling | `tools/cron.py` | Medium | Agent loop |
| 3 | Web search/fetch | `tools/web.py` | Small | Core working |
| 4 | Skills loader | `skills/loader.py` | Medium | Tool registry |
| 5 | Agent profiles | `agent/profiles.py` | Small | Context builder |
| 6 | Solution memory | `memory/solutions.py` | Medium | Memory store |
| 7 | Filesystem tools polish | `tools/filesystem.py` | Small | Core working |

## Phase 3 Features (Full Ingredient List)

| # | Feature | Module | Effort | Notes |
|---|---------|--------|--------|-------|
| 1 | Browser automation | `tools/browser.py` | Large | Playwright, headless Chromium |
| 2 | Webhook triggers | `tools/webhook.py` | Medium | FastAPI/aiohttp server |
| 3 | Screenshot & vision | `tools/screenshot.py` | Medium | macOS screencapture + vision model |
| 4 | Obsidian integration | `integrations/obsidian.py` | Small | Read/write vault markdown files |
| 5 | 1Password integration | `integrations/onepassword.py` | Small | 1Password CLI (`op`) wrapper |
| 6 | SearXNG search | `integrations/searxng.py` | Medium | Docker SearXNG + API calls |
| 7 | Image generation | `integrations/imagegen.py` | Medium | OpenAI DALL-E or Gemini via OpenRouter |
| 8 | MCP protocol | `integrations/mcp.py` | Large | MCP client, tool auto-discovery |

---

## Testing Strategy

Each workstream should include basic tests. Minimum:

| Module | Test | Type |
|--------|------|------|
| Config | Loads defaults, merges overrides, validates | Unit |
| Secrets | Encrypt/decrypt round-trip | Unit |
| Safety | Blocks dangerous commands, allows safe ones | Unit |
| Memory | Save/search/forget/recent, FTS5 ranking | Unit |
| Provider | Mock LiteLLM, verify message formatting + tool call parsing | Unit |
| Telegram | Mock bot API, verify message splitting, typing indicator | Unit |
| Agent Loop | Mock provider + memory, verify tool call flow, max rounds | Integration |
| Full E2E | Send real Telegram message, get response | Manual |

Test runner: `pytest` with `pytest-asyncio` for async tests.

---

## Risk Mitigations

| Risk | Mitigation |
|------|-----------|
| Mac sleep kills the bot | launchd plist with KeepAlive auto-restarts |
| OpenRouter rate limits | Exponential backoff retry (3 attempts) + Ollama fallback in Phase 2 |
| SQLite lock contention | WAL mode, single-writer pattern, async access |
| Telegram message too long | Auto-split at 4096 chars on paragraph boundaries |
| LLM infinite tool loop | Max 5 tool rounds per message, then force text response |
| Secrets file stolen | Fernet encryption tied to hardware UUID — useless on other machines |
| Bad shell command | Blocklist + workspace restriction + timeout enforcement |
| Future features don't fit | Every module has abstract base class — swap implementations without touching other code |

---

## Quick Reference Commands

```bash
# Clone the repo
git clone git@github.com:earlyaidopters/antidote.git
cd antidote

# First-time setup + run (creates venv, installs deps, launches wizard)
./start.sh

# Or with the CLI directly (after venv is activated)
source .venv/bin/activate
antidote              # Auto-launches wizard if no config, then runs bot
antidote setup        # Run/re-run the setup wizard

# Alternative: manual venv setup
python3 -m venv .venv && source .venv/bin/activate && pip install -e .

# Install auto-start (wizard does this for you)
cp com.antidote.agent.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.antidote.agent.plist

# Stop auto-start
launchctl unload ~/Library/LaunchAgents/com.antidote.agent.plist

# View logs
tail -f ~/.antidote/logs/antidote.log

# Run tests
python3 -m pytest tests/ -v
```
