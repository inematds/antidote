<div align="center">

```
    _   _  _ _____ ___ ___  ___ _____ ___
   /_\ | \| |_   _|_ _|   \/ _ \_   _| __|
  / _ \| .` | | |  | || |) | (_) || | | _|
 /_/ \_\_|\_| |_| |___|___/ \___/ |_| |___|
```

**The antidote to bloated AI frameworks.**

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![Telegram](https://img.shields.io/badge/Telegram-Bot-26A5E4?style=flat-square&logo=telegram&logoColor=white)](https://core.telegram.org/bots)
[![OpenRouter](https://img.shields.io/badge/OpenRouter-100%2B%20Models-6c63ff?style=flat-square)](https://openrouter.ai/)
[![SQLite](https://img.shields.io/badge/Memory-SQLite%20FTS5-003B57?style=flat-square&logo=sqlite&logoColor=white)](https://www.sqlite.org/fts5.html)
[![Ollama](https://img.shields.io/badge/Local%20AI-Ollama-black?style=flat-square)](https://ollama.ai/)
[![Encrypted](https://img.shields.io/badge/Secrets-Fernet%20Encrypted-00D26A?style=flat-square)](https://cryptography.io/)
[![macOS](https://img.shields.io/badge/Platform-macOS-000000?style=flat-square&logo=apple&logoColor=white)](https://www.apple.com/macos/)

*~3,000 lines · 35 files · 7 dependencies · 0 Docker containers*

[Framework Comparison →](https://inematds.github.io/antidote/)

</div>

---

## O que é

**Antidote** é um assistente de IA pessoal construído do zero em Python — sem frameworks de agente pesados, sem cloud obrigatória, sem Docker, sem dashboard web. Roda no seu Mac, fala via Telegram, lembra de tudo, e tem personalidade definida por você.

É exatamente o que o nome diz: o antídoto contra frameworks de IA inchados.

```
Você no Telegram
     ↓
Antidote recebe via long-polling
     ↓
Agent loop pensa, busca memória, usa ferramentas
     ↓
OpenRouter (cloud) ou Ollama (local) gera a resposta
     ↓
Você recebe a resposta no Telegram
```

---

## O que este projeto NÃO é

| ❌ O que não é | ✅ O que é |
|---------------|-----------|
| Multi-usuário | Single-owner, para você |
| Dependente de cloud | Roda 100% local com Ollama |
| Cheio de YAML | 7 dependências pip, ponto |
| Precisa de Docker | `./start.sh` e acabou |
| Interface web | Telegram é a interface |
| Black-box | ~3.000 linhas legíveis |

---

## Quickstart

```bash
# Clone
git clone https://github.com/inematds/antidote.git
cd antidote

# Inicia tudo: cria venv, instala dependências, roda o wizard
./start.sh
```

O wizard guia você por tudo:

```
  ┌─────────────────────────────────────────────────────┐
  │   Bem-vindo ao Antidote                              │
  │   The antidote to bloated AI frameworks.             │
  └─────────────────────────────────────────────────────┘

  [1/6] Telegram Bot Token
        Abra o Telegram → @BotFather → /newbot → cole o token
        ✓ Token válido — bot: @MeuAntidoteBot

  [2/6] OpenRouter API Key
        openrouter.ai/keys → crie uma chave → cole aqui
        ✓ Chave válida — 147 modelos disponíveis

  [3/6] Modelo padrão
        ❯ Claude Sonnet 4 (recomendado)
          Claude Haiku 4 (mais barato)
          GPT-4.1 Mini (mais econômico)
          DeepSeek R1 (melhor custo-raciocínio)

  [4/6] Nome do seu assistente
        > Antidote

  [5/6] Personalidade em uma frase
        > Direto, honesto, sem enrolação.

  ✓ Configuração salva
  ✓ Banco de memória criado
  ✓ Auto-start instalado (launchd)

  Pronto. Mande uma mensagem no Telegram.
```

---

## Arquitetura

```
antidote/
├── wizard.py                    # Wizard de setup (rich + questionary)
├── start.sh                     # Launcher: venv + install + run
├── com.antidote.agent.plist     # macOS launchd — auto-restart
├── pyproject.toml               # Pacote Python, entry point `antidote`
├── requirements.txt             # 7 dependências
│
├── antidote/
│   ├── main.py                  # Entry point + CLI router
│   ├── config.py                # Loader de config (singleton)
│   │
│   ├── agent/
│   │   ├── loop.py              # ★ Core: recebe → pensa → responde (max 5 rounds)
│   │   └── context.py          # Monta system prompt + memórias relevantes
│   │
│   ├── providers/
│   │   ├── base.py              # Interface abstrata
│   │   ├── openrouter.py        # OpenRouter via LiteLLM (100+ modelos)
│   │   └── ollama.py            # Ollama local (zero custo, zero privacidade comprometida)
│   │
│   ├── channels/
│   │   ├── base.py              # Interface abstrata
│   │   └── telegram.py         # Long-polling, split de msg, typing indicator
│   │
│   ├── memory/
│   │   └── store.py             # SQLite FTS5 + BM25 ranking + deduplicação
│   │
│   ├── tools/
│   │   ├── base.py              # Interface abstrata
│   │   ├── registry.py          # Descoberta e dispatch de ferramentas
│   │   ├── filesystem.py        # read_file, write_file, list_directory
│   │   └── shell.py             # run_command (com safety check)
│   │
│   └── security/
│       ├── secrets.py           # Store Fernet (chave = UUID do hardware)
│       └── safety.py            # Blocklist de comandos + audit log
│
└── workspace/
    ├── SOUL.md                  # Personalidade da IA
    ├── AGENTS.md                # Regras de comportamento
    ├── USER.md                  # Perfil e preferências do usuário
    └── MEMORY.md                # Fatos de bootstrap (seed inicial)
```

### Fluxo de uma mensagem

```
Telegram long-polling
    ↓  IncomingMessage
AgentLoop.process_message()
    ├── ContextBuilder
    │       ├── SOUL.md + AGENTS.md + USER.md
    │       └── memory.search(query) → memórias relevantes
    │
    ├── Provider.chat(messages, tools)
    │       ↓  LLMResponse
    │       └── tool_calls? ──→ ToolRegistry.execute()
    │                               └── re-envia ao LLM (max 5 rounds)
    │
    ├── memory.save(resumo da conversa)
    │
    └── Channel.send(OutgoingMessage)
            ↓
        Telegram (split automático em >4096 chars)
```

---

## Stack de Identidade

A personalidade do Antidote vive em arquivos Markdown editáveis — sem banco de dados especial, sem interface separada. Edite no editor de texto.

| Arquivo | O que define |
|---------|-------------|
| `workspace/SOUL.md` | Personalidade: tom, valores, estilo de resposta |
| `workspace/AGENTS.md` | Regras: quando usar ferramentas, como usar memória |
| `workspace/USER.md` | Perfil do usuário, preferências, contexto de vida |
| `workspace/MEMORY.md` | Fatos de bootstrap carregados na primeira vez |

Esses arquivos são carregados por `agent/context.py` a cada mensagem e montam o system prompt enviado ao LLM.

---

## Memória

Usa **SQLite FTS5** com ranking **BM25** — sem embeddings, sem chamadas de API para busca, sem custo extra.

```sql
-- Busca full-text com ranking de relevância
SELECT * FROM memories_fts
WHERE memories_fts MATCH 'projeto python semana passada'
ORDER BY bm25(memories_fts)
LIMIT 10;
```

| Categoria | O que armazena |
|-----------|---------------|
| `fact` | Fatos sobre você, preferências, contexto |
| `conversation` | Resumos de conversas importantes |
| `solution` | Soluções para problemas recorrentes |
| `preference` | Como você gosta das coisas |

- **Deduplicação automática**: se o conteúdo novo tem >80% de sobreposição com uma entrada existente, atualiza em vez de duplicar
- **Persistência**: `~/.antidote/memory.db` — sobrevive a reinicializações
- **Ferramentas do LLM**: `save_memory`, `search_memory`, `forget_memory` disponíveis diretamente na conversa

---

## Segurança

### Secrets encriptados (Fernet)
```
API Key → AES-128-CBC + HMAC-SHA256 → ~/.antidote/.secrets
                ↑
    Chave derivada do UUID do hardware macOS via PBKDF2
    (arquivo .secrets é inútil em outra máquina)
```

### Blocklist de comandos
```python
BLOCKED = ["rm -rf /", "mkfs", "dd if=", "shutdown", "reboot", "> /dev/sd"]
# Bloqueio por substring — qualquer variação é pega
```

### Outros controles
- Path traversal: ferramentas de filesystem restritas ao `workspace/`
- Timeout: 60s máximo por comando shell
- Truncamento: output de comandos limitado a 10KB, leitura de arquivo a 100KB
- Audit log: todo comando executado em `~/.antidote/audit.log`
- Gitignore: `config.json`, `.secrets`, `.env`, `*.db`, `*.log` nunca sobem

---

## Provedores de LLM

### OpenRouter (padrão)
- Acesso a 100+ modelos com uma única chave de API
- Claude, GPT, Gemini, DeepSeek, Llama, Mistral — você escolhe
- Retry automático com backoff exponencial (rate limits e 5xx)
- Log de tokens consumidos para controle de custo

### Ollama (local/gratuito)
- Zero custo, zero dados enviados para fora
- Modelos: Llama 3.2, Mistral, Gemma, Phi-4, e outros
- Fallback automático se OpenRouter falhar
- Requer Ollama instalado: `brew install ollama`

---

## Auto-restart (macOS launchd)

O Antidote não morre com sleep, crash ou reboot.

```bash
# O wizard instala automaticamente. Para instalar manualmente:
cp com.antidote.agent.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.antidote.agent.plist

# Para parar:
launchctl unload ~/Library/LaunchAgents/com.antidote.agent.plist
```

Logs ficam em `~/.antidote/logs/` para diagnóstico.

---

## Comandos

```bash
# Inicialização
./start.sh              # Tudo: venv + install + wizard (se necessário) + bot
antidote                # Com venv ativo: auto-detecta config, inicia bot
antidote setup          # Força re-execução do wizard

# Desenvolvimento
source .venv/bin/activate
pip install -e .
python -m pytest tests/ -v

# Logs
tail -f ~/.antidote/logs/antidote.log
tail -f ~/.antidote/audit.log

# launchd
launchctl list | grep antidote
launchctl stop com.antidote.agent
launchctl start com.antidote.agent
```

---

## Configuração

O wizard gera `~/.antidote/config.json` (gitignored). Estrutura:

```json
{
  "name": "Antidote",
  "providers": {
    "default": "openrouter",
    "openrouter": { "model": "anthropic/claude-sonnet-4-20250514" },
    "ollama": { "model": "llama3.2", "base_url": "http://localhost:11434" }
  },
  "channels": {
    "telegram": { "enabled": true }
  },
  "memory": {
    "db_path": "~/.antidote/memory.db",
    "max_context_memories": 10
  },
  "safety": {
    "blocked_commands": ["rm -rf /", "mkfs", "dd if=", "shutdown"],
    "max_command_timeout": 60
  }
}
```

---

## Dependências

| Pacote | Versão | Para que serve |
|--------|--------|---------------|
| `litellm` | ≥1.40.0 | Interface unificada para OpenRouter e Ollama |
| `python-telegram-bot` | ≥21.0 | Bot API async com long-polling |
| `cryptography` | ≥42.0 | Fernet — encriptação dos secrets |
| `aiosqlite` | ≥0.20.0 | SQLite async sem bloquear o event loop |
| `python-dotenv` | ≥1.0 | Fallback de config via `.env` |
| `rich` | ≥13.0 | Terminal: painéis, tabelas, cores |
| `questionary` | ≥2.0 | Prompts interativos no wizard |

---

## Roadmap

### Phase 2
- [ ] Cron scheduling (`tools/cron.py`)
- [ ] Web search/fetch (`tools/web.py`)
- [ ] Skills loader (`skills/loader.py`)
- [ ] Agent profiles (`agent/profiles.py`)

### Phase 3
- [ ] Browser automation — Playwright (`tools/browser.py`)
- [ ] Webhook triggers (`tools/webhook.py`)
- [ ] Screenshot + vision (`tools/screenshot.py`)
- [ ] Obsidian vault integration (`integrations/obsidian.py`)
- [ ] MCP protocol client (`integrations/mcp.py`)

---

## Interfaces (contratos)

Cada subsistema implementa uma classe abstrata. Adicionar um novo provedor, canal ou ferramenta = criar um arquivo novo, sem tocar no código existente.

```python
# Novo provedor: implemente BaseProvider
class MyProvider(BaseProvider):
    async def chat(self, messages, tools, model, temperature) -> LLMResponse: ...

# Nova ferramenta: implemente BaseTool
class MyTool(BaseTool):
    name = "my_tool"
    description = "..."
    parameters = {...}  # JSON Schema
    async def execute(self, **kwargs) -> ToolResult: ...

# Novo canal: implemente BaseChannel
class MyChannel(BaseChannel):
    async def start(self, on_message): ...
    async def send(self, message: OutgoingMessage): ...
    async def stop(self): ...
```

---

<div align="center">

**~3,000 linhas · 35 arquivos · 7 dependências · 0 containers Docker**

[Ver análise completa de frameworks →](https://inematds.github.io/antidote/)

</div>
