# Provider System

> This document explains how AutoSkill abstracts away LLM, embedding, and storage providers — letting you swap backends by changing one config line.

---

## The Problem Providers Solve

AutoSkill needs three external capabilities:

1. **LLM** — Generate text (for extraction, maintenance decisions, chat responses)
2. **Embeddings** — Convert text to vectors (for semantic search)
3. **Storage** — Persist and query skills

Different users have different infrastructure. Some use OpenAI, others use Anthropic, others use self-hosted models. AutoSkill should work with all of them without code changes.

**Rationale**: By abstracting providers behind interfaces, AutoSkill achieves **zero external dependencies** (`dependencies = []` in `pyproject.toml`). Users install only the client libraries they need.

---

## Architecture: Interface + Factory + Plugin

Each provider type follows the same three-layer pattern:

```
┌─────────────────────────────────────────────────┐
│ Layer 1: INTERFACE (Abstract Base Class)         │
│                                                  │
│ Defines what a provider must do.                 │
│ Example: LLM must have complete() method.        │
│                                                  │
│ File: autoskill/llm/base.py                      │
│       autoskill/embeddings/base.py               │
│       autoskill/management/stores/base.py        │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│ Layer 2: IMPLEMENTATIONS                         │
│                                                  │
│ Concrete classes for each supported provider.    │
│ Example: OpenAIChatLLM, AnthropicLLM, etc.      │
│                                                  │
│ Files: autoskill/llm/openai.py                   │
│        autoskill/llm/anthropic.py                │
│        autoskill/llm/glm.py                      │
│        ... (one file per provider)               │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│ Layer 3: FACTORY + PLUGIN REGISTRY               │
│                                                  │
│ Maps config dicts to concrete implementations.   │
│ Supports custom provider registration.           │
│                                                  │
│ Files: autoskill/llm/factory.py                  │
│        autoskill/embeddings/factory.py           │
└─────────────────────────────────────────────────┘
```

---

## LLM Providers

### Interface

```python
class LLM:
    """Abstract interface for language model providers."""
    
    def complete(self, system: str, user: str, temperature: float = 0.7) -> str:
        """Generate a text completion."""
        raise NotImplementedError
    
    def stream_complete(self, system: str, user: str, temperature: float = 0.7):
        """Optional: Stream a text completion chunk by chunk."""
        raise NotImplementedError
```

**Rationale**: The interface is intentionally minimal — just `complete()` and optional `stream_complete()`. This makes it trivial to implement a new provider.

### Available Implementations

| Provider | Class | Config `provider` Value | Required Package | What It Connects To |
|----------|-------|------------------------|------------------|---------------------|
| OpenAI | `OpenAIChatLLM` | `"openai"` | `openai` | GPT-4, GPT-3.5, etc. |
| Anthropic | `AnthropicLLM` | `"anthropic"` | `anthropic` | Claude models |
| Zhipu GLM | `GLMChatLLM` | `"glm"`, `"zhipu"`, `"bigmodel"` | HTTP requests | GLM-4, etc. |
| InternLM | `InternLMChatLLM` | `"internlm"` | HTTP requests | InternLM models |
| Generic | `GenericChatLLM` | `"generic"` | HTTP requests | Any OpenAI-compatible endpoint |
| Mock | `MockLLM` | `"mock"` | None | Returns placeholder text (for testing) |

### Configuration Examples

```python
# OpenAI
llm_config = {
    "provider": "openai",
    "model": "gpt-4",
    "api_key": "sk-..."    # or set OPENAI_API_KEY env var
}

# Anthropic
llm_config = {
    "provider": "anthropic",
    "model": "claude-3-sonnet-20240229",
    "api_key": "sk-ant-..."
}

# Zhipu GLM
llm_config = {
    "provider": "glm",
    "model": "glm-4",
    "api_key": "..."       # or set ZHIPUAI_API_KEY env var
}

# InternLM
llm_config = {
    "provider": "internlm",
    "model": "intern-s1-pro",
    "api_key": "..."       # or set INTERNLM_API_KEY env var
}

# Generic (any OpenAI-compatible endpoint)
llm_config = {
    "provider": "generic",
    "model": "my-model",
    "api_key": "...",
    "base_url": "https://my-llm-server.example.com/v1"
}

# Mock (for testing — no API needed)
llm_config = {
    "provider": "mock"
}
```

### Plugin Registration (Custom Providers)

You can register your own LLM provider:

```python
from autoskill.llm.factory import register_llm_connector

def my_custom_builder(config: dict):
    return MyCustomLLM(
        api_key=config.get("api_key"),
        model=config.get("model")
    )

# Register with a name and optional aliases
register_llm_connector("mycustom", my_custom_builder, aliases=["custom"])

# Now use it in config:
config = AutoSkillConfig(
    llm={"provider": "mycustom", "api_key": "..."},
    ...
)
```

**Rationale**: The plugin system means AutoSkill never needs to be forked to add new providers. Third parties can register providers at runtime.

---

## Embedding Providers

### Interface

```python
class EmbeddingModel:
    """Abstract interface for embedding providers."""
    
    def embed(self, texts: list[str]) -> list[list[float]]:
        """Convert texts to vector embeddings."""
        raise NotImplementedError
```

### Available Implementations

| Provider | Class | Config `provider` Value | Required Package | Notes |
|----------|-------|------------------------|------------------|-------|
| OpenAI | `OpenAIEmbedding` | `"openai"` | `openai` | text-embedding-3-large, etc. |
| Zhipu BigModel | `BigModelEmbedding` | `"bigmodel"`, `"zhipu"` | HTTP requests | Zhipu embedding API |
| Generic | `GenericEmbedding` | `"generic"` | HTTP requests | Any embedding HTTP endpoint |
| DashScope | (via generic) | `"qwen"`, `"dashscope"` | HTTP requests | Alibaba Qwen embeddings |
| Hashing | `HashingEmbedding` | `"hashing"` | None | Deterministic hash vectors (no API!) |
| None | `NoneEmbedding` | `"none"` | None | Placeholder (returns zeros) |

### The Hashing Embedding (Clever!)

The `HashingEmbedding` is a unique provider that creates vector embeddings **without any API call**:

```python
# How it works:
# 1. Tokenize text into words
# 2. Hash each word to a position in a fixed-size vector
# 3. Increment that position
# 4. Normalize the vector

"How to deploy" → hash("how")=42, hash("to")=99, hash("deploy")=167
                → [0, ..., 1, ..., 0, ..., 1, ..., 0, ..., 1, ...]
                      (pos 42)     (pos 99)     (pos 167)
```

**Rationale**: Hashing embeddings are deterministic, free, and work offline. They're less accurate than neural embeddings but sufficient for testing and environments without API access. Combined with BM25, they still provide useful search results.

### Configuration Examples

```python
# OpenAI
embeddings_config = {
    "provider": "openai",
    "model": "text-embedding-3-large",
    "api_key": "sk-..."
}

# DashScope (Qwen)
embeddings_config = {
    "provider": "qwen",
    "api_key": "..."    # or set DASHSCOPE_API_KEY env var
}

# Hashing (no API needed — great for testing)
embeddings_config = {
    "provider": "hashing",
    "dims": 256         # vector dimensionality
}
```

---

## Storage Providers

### Interface

```python
class SkillStore:
    """Abstract interface for skill storage backends."""
    
    def upsert(self, skill, raw=None) -> None:
        """Create or update a skill."""
    
    def get(self, skill_id: str):
        """Get a skill by ID."""
    
    def delete(self, skill_id: str) -> bool:
        """Delete a skill."""
    
    def list(self, user_id: str) -> list:
        """List all skills for a user."""
    
    def search(self, user_id: str, query: str, limit: int, filters=None) -> list:
        """Search for skills by query."""
    
    # Optional advanced methods:
    def record_skill_usage_judgments(self, user_id, judgments) -> dict: ...
    def get_skill_usage_stats(self, user_id, skill_id) -> dict: ...
```

### Available Implementations

| Provider | Class | Config `provider` Value | Required Package | Best For |
|----------|-------|------------------------|------------------|----------|
| **Local filesystem** | `LocalSkillStore` | `"local"` | None | **Production (recommended)** |
| In-memory | `InMemorySkillStore` | `"inmemory"` | None | Testing |
| ChromaDB | `ChromaSkillStore` | `"chroma"` | `chromadb` | Persistent vector DB |
| Pinecone | `PineconeSkillStore` | `"pinecone"` | `pinecone-client` | Cloud vector DB |
| Milvus | `MilvusSkillStore` | `"milvus"` | `pymilvus` | Self-hosted vector DB |

### Why Local Filesystem is Recommended

The `LocalSkillStore` stores skills as SKILL.md files in directories:

```
/SkillBank/
├── Users/u1/
│   ├── release-process/
│   │   ├── SKILL.md        # Human-readable skill definition
│   │   └── script.py       # Optional bundled file
│   └── monitoring-setup/
│       └── SKILL.md
├── Common/
│   └── anthropics-skill/
│       ├── create-pdf/SKILL.md
│       └── web-automation/SKILL.md
└── vectors/
    └── <hash>.bin          # Persistent vector index
```

**Rationale**: Filesystem storage has unique advantages:
1. **Human-readable**: Browse skills with a file manager or `cat`
2. **Version-control friendly**: Commit skills to git
3. **No infrastructure**: No database server to maintain
4. **Portable**: Copy the directory to move skills
5. **Debuggable**: Open SKILL.md in a text editor to see exactly what the system stores

### Configuration Examples

```python
# Local filesystem (recommended)
store_config = {
    "provider": "local",
    "path": "./SkillBank"
}

# In-memory (for testing)
store_config = {
    "provider": "inmemory"
}

# ChromaDB
store_config = {
    "provider": "chroma",
    "path": "./chroma_db"
}

# Pinecone
store_config = {
    "provider": "pinecone",
    "api_key": "...",
    "environment": "us-east-1",
    "index_name": "autoskill"
}
```

---

## How Providers Are Resolved

When you create an `AutoSkill` instance, the config dicts are resolved to concrete implementations:

```python
config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-4"},
    embeddings={"provider": "hashing", "dims": 256},
    store={"provider": "local", "path": "./SkillBank"}
)

sdk = AutoSkill(config)

# Internally:
# 1. build_llm({"provider": "openai", ...})
#    → Looks up "openai" in the LLM registry
#    → Returns OpenAIChatLLM(model="gpt-4")
#
# 2. build_embeddings({"provider": "hashing", ...})
#    → Looks up "hashing" in the embeddings registry
#    → Returns HashingEmbedding(dims=256)
#
# 3. build_store({"provider": "local", ...})
#    → Looks up "local" in the store registry
#    → Returns LocalSkillStore(path="./SkillBank")
```

**Rationale**: Dict-based config means you can load settings from environment variables, JSON files, or code. No need to import provider-specific classes in user code.

---

## Provider Fallback and Graceful Degradation

AutoSkill is designed to degrade gracefully when providers fail:

| Scenario | Fallback Behavior |
|----------|-------------------|
| LLM extraction fails | Use heuristic extractor (generate generic skill) |
| LLM query rewriting fails | Use original query unchanged |
| LLM skill selection fails | Use all retrieved skills |
| LLM merge decision fails | Use heuristic scoring rules |
| Embedding API fails | Fall back to BM25-only search |
| Missing SKILL.md file | Generate on-the-fly from stored data |

**Rationale**: In production, external APIs can fail. Graceful degradation ensures AutoSkill keeps working (with reduced quality) rather than crashing.

---

## Next Steps

- **[Configuration Reference](configuration.md)** — See all config options for each provider
- **[Architecture Overview](architecture-overview.md)** — See where providers fit in the layer diagram
- **[Design Patterns](design-patterns.md)** — Deep dive into the factory and plugin patterns
