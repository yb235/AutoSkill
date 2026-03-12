# Deployment Guide

> This document covers all the ways to run AutoSkill — from a simple Python import to a full Docker deployment.

---

## Deployment Modes

AutoSkill can be deployed in five ways, from simplest to most production-ready:

| Mode | Complexity | Use Case |
|------|-----------|----------|
| **SDK Import** | Minimal | Embed in your own Python app |
| **Console Chat** | Low | Development, testing, exploration |
| **Web UI** | Medium | Interactive exploration with diagnostics |
| **OpenAI Proxy** | Medium | Drop-in enhancement for existing apps |
| **Docker Compose** | Full | Production deployment |

---

## Mode 1: SDK Import (Simplest)

Use AutoSkill as a library in your own Python code.

### Setup

```bash
pip install -e .  # Install AutoSkill in development mode
pip install openai  # Or whichever LLM client you need
```

### Usage

```python
from autoskill import AutoSkill, AutoSkillConfig

config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-4", "api_key": "sk-..."},
    embeddings={"provider": "openai", "model": "text-embedding-3-large"},
    store={"provider": "local", "path": "./SkillBank"}
)

sdk = AutoSkill(config)

# Ingest skills from a conversation
skills = sdk.ingest(
    user_id="u1",
    messages=[
        {"role": "user", "content": "How do I deploy?"},
        {"role": "assistant", "content": "Run tests, canary, rollout."}
    ]
)

# Search for relevant skills
hits = sdk.search("deployment process", user_id="u1")

# Render skills as injectable context
context = sdk.render_context("deployment process", user_id="u1")
```

### When to Use

- You're building your own LLM application
- You want full control over the conversation flow
- You don't need a UI

---

## Mode 2: Console Chat

Interactive terminal chat with skill retrieval and extraction.

### Setup

```bash
pip install -e .
pip install openai  # Or your LLM client

# Set environment variables
export OPENAI_API_KEY=sk-...
```

### Start

```bash
python -m examples.interactive_chat \
    --llm-provider openai \
    --embeddings-provider openai \
    --user-id u1
```

### Commands

```
You: How do I deploy to production?
Assistant: Here's the deployment process: ...

You: /extract deployment steps
[Extracting skills from recent conversation...]

You: /skills
[1] Release Process (v1.0.0) - Run regression tests, canary...

You: /help
Available commands:
  /help     - Show this help
  /clear    - Clear conversation history
  /extract  - Manually trigger extraction
  /skills   - List your skills
  /config   - Show configuration
  /quit     - Exit
```

### When to Use

- Quick testing and exploration
- Development workflows
- No browser needed

---

## Mode 3: Web UI

Browser-based interface with diagnostics panels.

### Setup

```bash
pip install -e .
pip install openai  # Or your LLM client

# Set environment variables (or use .env file)
export OPENAI_API_KEY=sk-...
```

### Start

```bash
python -m examples.web_ui \
    --host 127.0.0.1 \
    --port 8000 \
    --llm-provider openai \
    --embeddings-provider openai \
    --user-id u1 \
    --skill-scope all
```

Then open `http://localhost:8000` in your browser.

### Features

- **Chat panel**: Send messages and see AI responses
- **Session management**: Create, switch, delete sessions
- **Retrieval diagnostics**: See original query, rewritten query, skill hits, selected skills
- **Extraction diagnostics**: See extraction progress, extracted SKILL.md
- **SKILL.md editor**: Edit, save, rollback, delete skills directly
- **Configuration**: Adjust scope, rewrite mode, extract mode, thresholds

### When to Use

- Exploring how AutoSkill works
- Demonstrating the system to stakeholders
- Debugging retrieval and extraction behavior

---

## Mode 4: OpenAI Proxy

Transparent drop-in for any application using the OpenAI API.

### Setup

```bash
pip install -e .
pip install openai  # Or your LLM client

export OPENAI_API_KEY=sk-...
```

### Start

```bash
python -m examples.openai_proxy \
    --host 127.0.0.1 \
    --port 9000 \
    --llm-provider openai \
    --embeddings-provider openai \
    --user-id u1 \
    --served-model gpt-4 \
    --served-model gpt-3.5-turbo
```

### Client Usage

```python
# Your existing code — just change base_url!
from openai import OpenAI

client = OpenAI(
    api_key="anything",                    # Proxy handles auth
    base_url="http://localhost:9000/v1"    # Point to proxy
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "How do I deploy?"}],
    stream=True,
    user="u1"    # Optional: per-user skill isolation
)

for chunk in response:
    print(chunk.choices[0].delta.content, end="")
```

### When to Use

- You have an existing app using the OpenAI SDK
- You want zero-code skill injection
- You want transparent extraction without changing your app

---

## Mode 5: Docker Compose (Production)

Run both Web UI and Proxy in containers.

### Setup

1. Copy `.env.example` to `.env` and fill in your API keys:

```bash
cp .env.example .env
# Edit .env with your preferred editor
```

2. Configure `.env`:

```bash
# Provider selection
AUTOSKILL_LLM_PROVIDER=openai
AUTOSKILL_EMBEDDINGS_PROVIDER=openai

# API keys
OPENAI_API_KEY=sk-...

# User settings
AUTOSKILL_USER_ID=u1
AUTOSKILL_SKILL_SCOPE=all
AUTOSKILL_REWRITE_MODE=always
AUTOSKILL_EXTRACT_MODE=auto
```

### Start

```bash
docker-compose up --build
```

### Services

| Service | URL | Description |
|---------|-----|-------------|
| Web UI | `http://localhost:8000` | Browser-based chat with diagnostics |
| OpenAI Proxy | `http://localhost:9000` | OpenAI-compatible API proxy |

### Docker Architecture

```yaml
# docker-compose.yml
services:
  autoskill-web:
    build: .
    command: python3 -m examples.web_ui --host 0.0.0.0 --port 8000
    ports: ["8000:8000"]
    volumes: ["./SkillBank:/data/SkillBank"]
    env_file: .env

  autoskill-proxy:
    build: .
    command: python3 -m examples.openai_proxy --host 0.0.0.0 --port 9000
    ports: ["9000:9000"]
    volumes: ["./SkillBank:/data/SkillBank"]
    env_file: .env
```

**Key details:**
- Both services share the same `SkillBank` volume — skills created in the Web UI are immediately available via the Proxy
- The Dockerfile uses `python:3.10-slim` for a minimal image
- Environment variables are loaded from `.env`

### When to Use

- Production deployment
- Team-wide shared instance
- Need both Web UI and Proxy running together

---

## Mode 6: OpenClaw Plugin

Integration with the OpenClaw agent framework.

### Setup

```bash
python3 OpenClaw-Plugin/install.py \
    --workspace-dir ~/.openclaw \
    --llm-provider internlm \
    --embeddings-provider qwen \
    --user-id u1
```

### Start

```bash
python3 OpenClaw-Plugin/run_proxy.py \
    --host 127.0.0.1 \
    --port 9000
```

### When to Use

- You're using the OpenClaw agent framework
- You want agents to learn skills from their interactions
- You want automatic skill synchronization with OpenClaw's native skill system

See [OpenClaw Plugin](openclaw-plugin.md) for full details.

---

## Environment Variables Reference

All environment variables that control AutoSkill:

### Provider Configuration

| Variable | Description | Example |
|----------|-------------|---------|
| `AUTOSKILL_LLM_PROVIDER` | LLM provider name | `openai`, `anthropic`, `glm`, `internlm`, `generic` |
| `AUTOSKILL_EMBEDDINGS_PROVIDER` | Embedding provider name | `openai`, `qwen`, `bigmodel`, `hashing` |
| `OPENAI_API_KEY` | OpenAI API key | `sk-...` |
| `ANTHROPIC_API_KEY` | Anthropic API key | `sk-ant-...` |
| `INTERNLM_API_KEY` | InternLM API key | `...` |
| `DASHSCOPE_API_KEY` | DashScope/Qwen API key | `...` |
| `ZHIPUAI_API_KEY` | Zhipu AI API key | `...` |
| `BIGMODEL_API_KEY` | BigModel API key | `...` |

### Generic Provider

| Variable | Description | Example |
|----------|-------------|---------|
| `AUTOSKILL_GENERIC_API_KEY` | API key for generic endpoint | `...` |
| `AUTOSKILL_GENERIC_LLM_URL` | Generic LLM endpoint URL | `https://my-llm.example.com/v1` |
| `AUTOSKILL_GENERIC_LLM_MODEL` | Generic LLM model name | `my-model` |
| `AUTOSKILL_GENERIC_EMBED_URL` | Generic embedding endpoint URL | `https://my-embed.example.com/v1` |
| `AUTOSKILL_GENERIC_EMBED_MODEL` | Generic embedding model name | `my-embed-model` |

### Runtime Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `AUTOSKILL_USER_ID` | Default user ID | `u1` |
| `AUTOSKILL_SKILL_SCOPE` | Search scope | `all` |
| `AUTOSKILL_REWRITE_MODE` | Query rewriting mode | `always` |
| `AUTOSKILL_EXTRACT_MODE` | Extraction mode | `auto` |
| `AUTOSKILL_EXTRACT_TURN_LIMIT` | Extract every N turns | `1` |
| `AUTOSKILL_MIN_SCORE` | Minimum retrieval score | `0.4` |
| `AUTOSKILL_TOP_K` | Number of skills to retrieve | `1` |

### Proxy Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `AUTOSKILL_PROXY_MODELS` | Comma-separated list of served models | (auto-detected) |
| `AUTOSKILL_PROXY_API_KEY` | Optional API key for proxy auth | (none) |

### Bootstrap Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `AUTOSKILL_AUTO_NORMALIZE_IDS` | Fix invalid skill IDs on startup | `1` |
| `AUTOSKILL_AUTO_IMPORT_DIRS` | Directories to auto-import from | (none) |
| `AUTOSKILL_AUTO_IMPORT_SCOPE` | Scope for auto-imported skills | `common` |
| `AUTOSKILL_AUTO_IMPORT_LIBRARY` | Library name for imports | (auto) |
| `AUTOSKILL_AUTO_IMPORT_OVERWRITE` | Overwrite existing on import | `0` |
| `AUTOSKILL_AUTO_IMPORT_INCLUDE_FILES` | Include bundled files on import | `1` |
| `AUTOSKILL_AUTO_IMPORT_MAX_DEPTH` | Max directory depth for imports | `6` |

---

## Next Steps

- **[Configuration Reference](configuration.md)** — Detailed configuration options
- **[API Reference](api-reference.md)** — API endpoints for each deployment mode
- **[Architecture Overview](architecture-overview.md)** — Understand the system you're deploying
