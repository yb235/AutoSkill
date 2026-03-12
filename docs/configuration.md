# Configuration Reference

> This document describes every configuration option in AutoSkill — what it does, what values are valid, and when to change it from the default.

---

## AutoSkillConfig

The master configuration object, passed when creating an `AutoSkill` instance.

```python
from autoskill import AutoSkill, AutoSkillConfig

config = AutoSkillConfig(
    llm={...},
    embeddings={...},
    store={...},
    # ... other options
)

sdk = AutoSkill(config)
```

---

### Provider Configuration

#### `llm` (Dict)

Configuration for the LLM (Large Language Model) provider. Used for skill extraction, maintenance decisions, query rewriting, and chat responses.

```python
# Minimal
llm={"provider": "openai", "model": "gpt-4"}

# Full
llm={
    "provider": "openai",       # Required: provider name
    "model": "gpt-4",          # Required: model identifier
    "api_key": "sk-...",        # Optional: API key (or use env var)
    "base_url": "https://...",  # Optional: custom endpoint
    "temperature": 0.7          # Optional: default temperature
}
```

| Provider | Config Value | Required Fields | Environment Variable |
|----------|-------------|----------------|---------------------|
| OpenAI | `"openai"` | `model` | `OPENAI_API_KEY` |
| Anthropic | `"anthropic"` | `model` | `ANTHROPIC_API_KEY` |
| Zhipu GLM | `"glm"`, `"zhipu"`, `"bigmodel"` | `model` | `ZHIPUAI_API_KEY` |
| InternLM | `"internlm"` | `model` | `INTERNLM_API_KEY` |
| Generic | `"generic"` | `model`, `base_url` | `AUTOSKILL_GENERIC_API_KEY` |
| Mock | `"mock"` | (none) | (none) |

**When to change**: Always set this — there's no default. Use `"mock"` for testing without an LLM.

#### `embeddings` (Dict)

Configuration for the embedding provider. Used for vector search (converting text to numeric vectors).

```python
# Minimal
embeddings={"provider": "openai", "model": "text-embedding-3-large"}

# No-API option
embeddings={"provider": "hashing", "dims": 256}
```

| Provider | Config Value | Required Fields | Notes |
|----------|-------------|----------------|-------|
| OpenAI | `"openai"` | `model` | Best quality, requires API |
| DashScope | `"qwen"`, `"dashscope"` | (none) | Alibaba Qwen embeddings |
| BigModel | `"bigmodel"`, `"zhipu"` | (none) | Zhipu embeddings |
| Generic | `"generic"` | `base_url`, `model` | Any HTTP embedding endpoint |
| Hashing | `"hashing"` | `dims` | Deterministic, no API needed |
| None | `"none"` | (none) | Placeholder (zeros) |

**When to change**: Always set this. Use `"hashing"` for testing or offline environments.

#### `store` (Dict)

Configuration for the skill storage backend.

```python
# Filesystem (recommended)
store={"provider": "local", "path": "./SkillBank"}

# In-memory (testing only)
store={"provider": "inmemory"}

# ChromaDB
store={"provider": "chroma", "path": "./chroma_db"}

# Pinecone
store={"provider": "pinecone", "api_key": "...", "index_name": "autoskill"}

# Milvus
store={"provider": "milvus", "host": "localhost", "port": 19530}
```

| Provider | Config Value | Required Fields | Best For |
|----------|-------------|----------------|----------|
| Local filesystem | `"local"` | `path` | **Production (recommended)** |
| In-memory | `"inmemory"` | (none) | Testing |
| ChromaDB | `"chroma"` | `path` | Persistent vector DB |
| Pinecone | `"pinecone"` | `api_key`, `index_name` | Cloud vector DB |
| Milvus | `"milvus"` | `host`, `port` | Self-hosted vector DB |

**When to change**: Default to `"local"` for most cases. Only change if you have a specific need for a vector database.

---

### Skill Management Options

#### `namespace` (str)

Default: `"default"`

Namespace for skill isolation. Skills in different namespaces are completely independent.

```python
namespace="production"   # Production skills
namespace="staging"      # Staging skills (separate bank)
```

**When to change**: Only if you need multiple independent skill banks in the same storage location.

#### `maintenance_strategy` (str)

Default: `"llm"`

How deduplication and merge decisions are made.

| Value | Description | Cost | Quality |
|-------|-------------|------|---------|
| `"llm"` | LLM judges whether to add, merge, or discard | Higher (extra LLM call) | Higher |
| `"heuristic"` | Score thresholds and fuzzy matching | Free | Lower |

**When to change**: Use `"heuristic"` if you're processing high volumes and want to minimize LLM costs.

#### `dedupe_similarity_threshold` (float)

Default: ~`0.85`

Range: 0.0 to 1.0

Similarity score above which two skills are considered duplicates.

```python
dedupe_similarity_threshold=0.90  # Strict: only very similar skills merge
dedupe_similarity_threshold=0.75  # Loose: more aggressive merging
```

**When to change**: Lower it if you're getting too many near-duplicate skills. Raise it if skills are being incorrectly merged.

#### `max_candidates_per_ingest` (int)

Default: `1`

Maximum skill candidates to extract per `ingest()` call.

**When to change**: Usually leave at 1. Higher values may create noise. Increase for very long conversations where multiple skills are clearly present.

#### `max_similar_skills_to_consider` (int)

Default: (varies)

When checking for duplicates, how many existing skills to compare against.

**When to change**: Increase if you have a very large skill bank and want more thorough dedup.

---

### Search Configuration

#### `default_search_limit` (int)

Default: `5`

Number of skills returned by `search()` when no `limit` parameter is passed.

**When to change**: Increase for broader retrieval, decrease for more focused results.

#### `max_context_chars` (int)

Default: (varies)

Maximum characters in rendered skill context (CJK-aware).

```python
max_context_chars=6000   # ~1500 tokens for GPT-4
max_context_chars=12000  # ~3000 tokens (for models with large context)
```

**When to change**: Increase if using models with large context windows. Decrease if using models with small context windows or if you want to leave more room for conversation history.

#### `bm25_weight` (float)

Default: `0.3`

Range: 0.0 to 1.0

Weight of BM25 keyword search in hybrid ranking.

```python
bm25_weight=0.0   # Pure vector search
bm25_weight=0.3   # 70% vector + 30% keyword (default)
bm25_weight=0.5   # Equal mix
bm25_weight=1.0   # Pure keyword search
```

**When to change**: Increase if your skills have very specific trigger keywords. Decrease if semantic similarity is more important than exact keyword matching.

---

### Security Options

#### `redact_sources_before_llm` (bool)

Default: `False`

If true, strip source/provenance metadata before sending to the LLM during extraction.

**When to change**: Enable in environments where conversation content is sensitive and shouldn't be sent to external LLMs.

#### `store_sources` (bool)

Default: `True`

If true, store source provenance metadata with skills.

**When to change**: Disable if you don't need to track where skills came from, or if provenance data is sensitive.

---

## InteractiveConfig

Configuration for interactive sessions (Web UI, Console Chat, Proxy).

```python
from autoskill.interactive.config import InteractiveConfig

interactive_config = InteractiveConfig(
    user_id="u1",
    skill_scope="all",
    rewrite_mode="always",
    extract_mode="auto",
    extract_turn_limit=3,
    min_score=0.4,
    top_k=5,
    max_context_chars=6000,
    ingest_window=6,
    history_turns=10
)
```

#### `user_id` (str)

Required. The user ID for this session. Skills are scoped to this user.

#### `skill_scope` (str)

Default: `"all"`

| Value | What Gets Searched |
|-------|-------------------|
| `"user"` | Only this user's skills (`/Users/<user_id>/`) |
| `"library"` | Only shared library skills (`/Common/`) |
| `"all"` | Both user and library skills (recommended) |

#### `rewrite_mode` (str)

Default: `"always"`

Controls whether user queries are rewritten by an LLM before searching.

| Value | Behavior | Cost |
|-------|----------|------|
| `"always"` | Rewrite every query | 1 extra LLM call per turn |
| `"auto"` | Rewrite only when heuristic suggests it would help | Variable |
| `"never"` | Use original query as-is | Free |

#### `extract_mode` (str)

Default: `"auto"`

Controls when skill extraction runs.

| Value | Behavior | Cost |
|-------|----------|------|
| `"always"` | Extract after every turn | 1 extra LLM call per turn |
| `"auto"` | Extract every N turns or on topic change | Variable |
| `"never"` | Only on manual `/extract` command | Free |

#### `extract_turn_limit` (int)

Default: `1`

In `"auto"` mode, extract every N turns. Set to 3 to extract every 3 turns.

#### `min_score` (float)

Default: `0.4`

Range: 0.0 to 1.0

Minimum relevance score for a skill to be included in results.

```python
min_score=0.3  # Loose: include more skills, possibly less relevant
min_score=0.5  # Moderate: balanced recall/precision
min_score=0.7  # Strict: only highly relevant skills
```

#### `top_k` (int)

Default: `5`

Number of skills to retrieve per scope.

#### `max_context_chars` (int)

Default: `6000`

Maximum characters in rendered skill context (CJK-aware).

#### `ingest_window` (int)

Default: `6`

Number of recent messages to include when extracting skills. Larger windows capture more context but cost more tokens.

#### `history_turns` (int)

Default: `10`

Number of conversation turns to keep in session history. Affects context sent to the LLM.

---

## Proxy Configuration

Additional settings when running the OpenAI-compatible proxy.

```python
from autoskill.interactive.server import AutoSkillProxyConfig

proxy_config = AutoSkillProxyConfig(
    served_models=["gpt-4", "gpt-3.5-turbo"],
    extraction_enabled=True,
    max_bg_extract_jobs=2
)
```

#### `served_models` (List[str])

List of model names the proxy responds to. Client requests for these models are handled by the configured LLM provider.

#### `extraction_enabled` (bool)

Default: `True`

Whether background extraction runs after proxy requests.

#### `max_bg_extract_jobs` (int)

Default: `2`

Maximum concurrent background extraction jobs. Higher values use more memory and CPU.

---

## Quick Configuration Recipes

### Development (Fast, Free)

```python
config = AutoSkillConfig(
    llm={"provider": "mock"},
    embeddings={"provider": "hashing", "dims": 256},
    store={"provider": "inmemory"},
    maintenance_strategy="heuristic"
)
```

### Production (OpenAI)

```python
config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-4"},
    embeddings={"provider": "openai", "model": "text-embedding-3-large"},
    store={"provider": "local", "path": "./SkillBank"},
    maintenance_strategy="llm"
)
```

### Production (Chinese Providers)

```python
config = AutoSkillConfig(
    llm={"provider": "internlm", "model": "intern-s1-pro"},
    embeddings={"provider": "qwen"},
    store={"provider": "local", "path": "./SkillBank"},
    maintenance_strategy="llm"
)
```

### Cost-Optimized

```python
config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-3.5-turbo"},
    embeddings={"provider": "hashing", "dims": 256},  # Free embeddings
    store={"provider": "local", "path": "./SkillBank"},
    maintenance_strategy="heuristic"  # No LLM for maintenance
)
```

---

## Next Steps

- **[Provider System](provider-system.md)** — Understand provider options in depth
- **[Deployment Guide](deployment.md)** — Start running AutoSkill with your config
- **[Design Patterns](design-patterns.md)** — Understand why configuration works this way
