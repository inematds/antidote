# Relatório de Análise — Antidote

> Análise completa do arquivo `doc/antidote.md` gerada por Claude Opus 4.6.

---

## 1. O que é o projeto

**Antidote** é um assistente pessoal de IA construído do zero em Python (~3.000 linhas), rodando localmente em um Mac e se comunicando exclusivamente via Telegram. O nome é literal: é o "antídoto" contra frameworks de IA inchados.

- **Tagline**: *"The antidote to bloated AI frameworks."*
- **Público**: uso pessoal de Mark Kashef, single-user, sem pretensão de multi-tenancy
- **Status**: v0.1.0 entregue — 35 arquivos, 3.412 linhas, 6 workstreams completos

---

## 2. Decisões Arquiteturais

| Decisão | Escolha | Justificativa |
|---------|---------|---------------|
| Linguagem | Python 3.11+ | Iteração rápida com Claude Code, ecossistema LiteLLM |
| Deployment | Mac local (always-on) | Uso pessoal, sem necessidade de nuvem |
| LLMs | OpenRouter + Ollama | 100+ modelos via uma chave + fallback local grátis |
| Canal | Telegram long-polling | Sem URL pública, setup simples, mídia rica |
| Memória | SQLite FTS5 + BM25 | 90% das necessidades, custo zero, sem embeddings |
| Identidade | Stack SOUL.md | Arquivos markdown legíveis e editáveis pelo usuário |
| Segurança | Fernet + UUID hardware | Secrets inutilizáveis fora do Mac original |
| Auto-restart | macOS launchd | Sobrevive sleep, crash e reboot |

---

## 3. O que foi Explicitamente Descartado

| Item | Motivo |
|------|--------|
| Docker | Desnecessário para uso pessoal em Mac-only |
| Autenticação multi-usuário | Projeto single-user |
| Web UI / Dashboard | Telegram é a única interface |
| Vector/embedding search | FTS5 keyword search é suficiente |
| Binário único | Python + script é mais simples para iteração |
| Arquitetura gateway/WebSocket | Overkill para um único canal |
| Suporte cross-platform | Mac-only; launchd é específico do macOS |

---

## 4. Estrutura de Arquivos

### Raiz

| Arquivo | Função |
|---------|--------|
| `wizard.py` | Wizard interativo de setup (rich + questionary) |
| `start.sh` | Launcher one-command: cria venv, instala, roda |
| `com.antidote.agent.plist` | Auto-start/restart via macOS launchd |
| `pyproject.toml` | Pacote Python com entry point `antidote` |
| `requirements.txt` | Dependências Python |

### `antidote/` (core)

| Arquivo | Função |
|---------|--------|
| `main.py` | Entry point — router CLI + wiring de todos os componentes |
| `config.py` | Loader de configuração com singleton `Config` |
| `agent/loop.py` | **Core agent loop**: recebe mensagem, usa ferramentas (max 5 rounds), responde |
| `agent/context.py` | Monta system prompt com SOUL/AGENTS/USER.md + memórias relevantes |
| `providers/openrouter.py` | OpenRouter via LiteLLM com retry exponencial |
| `providers/ollama.py` | Ollama local via LiteLLM, fallback gratuito |
| `channels/telegram.py` | Telegram long-polling, split de mensagens, typing indicator |
| `memory/store.py` | SQLite FTS5 com ranking BM25 e deduplicação (>80% overlap) |
| `tools/filesystem.py` | `read_file`, `write_file`, `list_directory` (restrito ao workspace) |
| `tools/shell.py` | `run_command` com safety check + timeout |
| `security/secrets.py` | Store Fernet encriptado, chave derivada do UUID do hardware |
| `security/safety.py` | Blocklist de comandos + audit log |

### `workspace/` (identidade)

| Arquivo | Função |
|---------|--------|
| `SOUL.md` | Personalidade da IA (direto, proativo, honesto) |
| `AGENTS.md` | Regras de comportamento e uso de ferramentas |
| `USER.md` | Perfil e preferências do usuário |
| `MEMORY.md` | Fatos de bootstrap para seed inicial da memória |

---

## 5. Contratos de Interface (Classes Abstratas)

O coração da arquitetura são 4 interfaces que todo módulo DEVE implementar:

### BaseProvider (`antidote/providers/base.py`)
```
async chat(messages, tools, model, temperature) -> LLMResponse
```
Implementações: `OpenRouterProvider`, `OllamaProvider`

### BaseChannel (`antidote/channels/base.py`)
```
async start(on_message) -> None
async send(OutgoingMessage) -> None
async stop() -> None
```
Implementação: `TelegramChannel`

### BaseMemory (`antidote/memory/store.py`)
```
async save(content, category) -> int
async search(query, limit) -> list[MemoryEntry]
async forget(memory_id) -> bool
async recent(limit) -> list[MemoryEntry]
```
Implementação: `MemoryStore` (SQLite FTS5)

### BaseTool (`antidote/tools/base.py`)
```
name: str
description: str
parameters: dict  # JSON Schema
async execute(**kwargs) -> ToolResult
```
Implementações: `ReadFileTool`, `WriteFileTool`, `ListDirTool`, `RunCommandTool`

### Fluxo completo de uma mensagem

```
Telegram (long-polling)
    ↓
AgentLoop.process_message(IncomingMessage)
    ├── ContextBuilder → SOUL.md + AGENTS.md + USER.md + memory.search()
    ├── Provider.chat(messages, tools) → LLMResponse
    │       └── tool_calls? → ToolRegistry.execute() → re-envia (max 5 rounds)
    ├── Memory.save() → salva resumo da conversa
    └── Channel.send(OutgoingMessage) → resposta ao Telegram
```

---

## 6. Os 6 Workstreams

### Grafo de dependências

```
W1 (Config/Secrets) ──→ W2 (Providers)
        │                      │
        ├──→ W3 (Memory) ──────┤
        │                      │
        └──→ W4 (Channel) ─────┴──→ W5 (Main/Wizard/Launcher)
                  │                            │
                  └── W4 (Agent Loop) ─────────┘
                                               │
                                               ↓
                                    W6 (CLI/Branding/Packaging)
```

### Detalhamento por workstream

| WS | Papel | Arquivos | Depende de | Entrega |
|----|-------|----------|------------|---------|
| **W1** Config/Secrets | Infraestrutura/segurança | 5 | Nada | Config, Fernet, blocklist, requirements |
| **W2** Providers | Integração LLM | 4 | W1 | OpenRouter + Ollama via LiteLLM + factory |
| **W3** Memory/Identity | Dados/memória | 7 | W1 | SQLite FTS5 + ContextBuilder + identity files |
| **W4** Channel+Loop | Desenvolvimento core | 6 | W1-W3 | Telegram + tools + agent loop |
| **W5** Main/Wizard | UX/onboarding | 4 | W1-W4 | main.py + wizard 14 passos + start.sh + launchd |
| **W6** CLI/Branding | UX/branding | 6 | W1-W5 | pyproject.toml + CLI router + wizard verde + README |

---

## 7. Fases de Build

| Fase | O que roda | Detalhes |
|------|-----------|----------|
| **A** (paralela) | W1 + início W2/W3/W4 | Config não depende de nada; base classes e identity files podem ser escritos em paralelo |
| **B** | Agent loop + W5 | Loop precisa de W2 e W3 completos; main.py precisa de tudo |
| **C** | W6 — polish final | pyproject.toml, wizard branded, README, .gitignore |
| **D** | Git | Init, commit, push para `earlyaidopters/antidote` (privado) |

---

## 8. Stack Técnica

| Dependência | Versão | Papel |
|-------------|--------|-------|
| `litellm` | ≥1.40.0 | Abstração unificada para LLMs (OpenRouter e Ollama com a mesma interface) |
| `python-telegram-bot` | ≥21.0 | Telegram Bot API async com long-polling |
| `cryptography` | ≥42.0 | Fernet (AES-128-CBC + HMAC-SHA256) para secrets |
| `aiosqlite` | ≥0.20.0 | SQLite async sem bloquear o event loop |
| `python-dotenv` | ≥1.0 | Fallback de configuração via `.env` |
| `rich` | ≥13.0 | Terminal rico: painéis, tabelas, cores (wizard e banner) |
| `questionary` | ≥2.0 | Prompts interativos no wizard de setup |
| `pytest` + `pytest-asyncio` | — | Runner de testes com suporte async |

---

## 9. Segurança

### Camada 1 — Encriptação de Secrets (Fernet)
- Algoritmo: AES-128-CBC + HMAC-SHA256
- Chave derivada do UUID do hardware macOS via PBKDF2
- **Arquivo `.secrets` é inútil em outra máquina**
- Fallback: solicita senha se UUID indisponível
- Local: `~/.antidote/.secrets`

### Camada 2 — Blocklist de Comandos
- Padrões bloqueados: `rm -rf /`, `mkfs`, `dd if=`, `shutdown`, `reboot`, `> /dev/sd`
- Proteção contra path traversal (`../` fora do workspace)
- Timeout: 60 segundos por comando
- API: `is_safe(command) -> (bool, reason)`

### Camada 3 — Audit Log
- Todo comando executado (bloqueado ou não) é registrado em `~/.antidote/audit.log`

### Proteções adicionais
- Ferramentas de filesystem restritas ao diretório workspace
- Saída de comandos truncada em 10KB
- Leitura de arquivos limitada a 100KB
- `config.json`, `.secrets`, `.env`, `*.db`, `*.log` todos no `.gitignore`

---

## 10. Features Futuras

### Phase 2 (após o core funcionar)

| Feature | Módulo | Esforço |
|---------|--------|---------|
| Cron scheduling | `tools/cron.py` | Médio |
| Web search/fetch | `tools/web.py` | Pequeno |
| Skills loader | `skills/loader.py` | Médio |
| Agent profiles | `agent/profiles.py` | Pequeno |
| Solution memory | `memory/solutions.py` | Médio |
| Filesystem tools polish | `tools/filesystem.py` | Pequeno |

> Ollama já foi entregue na v0.1.0 (era item 1 desta fase).

### Phase 3 (lista completa de ingredientes)

| Feature | Módulo | Esforço |
|---------|--------|---------|
| Browser automation (Playwright) | `tools/browser.py` | Grande |
| Webhook triggers | `tools/webhook.py` | Médio |
| Screenshot & vision | `tools/screenshot.py` | Médio |
| Obsidian integration | `integrations/obsidian.py` | Pequeno |
| 1Password integration | `integrations/onepassword.py` | Pequeno |
| SearXNG search | `integrations/searxng.py` | Médio |
| Image generation | `integrations/imagegen.py` | Médio |
| MCP protocol client | `integrations/mcp.py` | Grande |

---

## 11. Estratégia de Testes

| Módulo | O que testar | Tipo |
|--------|-------------|------|
| Config | Carrega defaults, merge de overrides, validação | Unitário |
| Secrets | Round-trip encriptação/decriptação | Unitário |
| Safety | Bloqueia perigosos, permite seguros | Unitário |
| Memory | Save/search/forget/recent, ranking FTS5 | Unitário |
| Provider | Mock LiteLLM, formatação de mensagens + tool calls | Unitário |
| Telegram | Mock Bot API, split de mensagens, typing indicator | Unitário |
| Agent Loop | Mock provider + memory, fluxo de tool calls, max 5 rounds | Integração |
| Full E2E | Mensagem real no Telegram → resposta | Manual |

---

## 12. Riscos e Mitigações

| Risco | Mitigação |
|-------|-----------|
| Mac sleep mata o bot | launchd `KeepAlive` faz auto-restart com throttle de 10s |
| OpenRouter rate limits | Retry exponencial (3x) + fallback Ollama |
| Lock contention no SQLite | Modo WAL + padrão single-writer + acesso async |
| Mensagem Telegram muito longa | Auto-split em 4096 chars nos limites de parágrafos |
| Loop infinito de tool calls | Máximo 5 rodadas, depois força resposta em texto |
| Arquivo de secrets roubado | Fernet vinculado ao UUID do hardware — inútil em outra máquina |
| Comando shell perigoso | Blocklist + restrição ao workspace + timeout de 60s |
| Features futuras não encaixam | Cada módulo tem classe abstrata — troca sem impactar outros |

---

*Relatório gerado por Claude Opus 4.6 com base em `doc/antidote.md`.*
