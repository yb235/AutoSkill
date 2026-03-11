# Design Patterns

> This document explains the key engineering patterns used in AutoSkill — *what* each pattern is, *where* it's used, and most importantly, *why* it was chosen over alternatives.

---

## Pattern 1: Zero External Dependencies

### What

AutoSkill's `pyproject.toml` has `dependencies = []`. The core package requires nothing beyond Python's standard library.

### Where

```toml
# pyproject.toml
[project]
dependencies = []  # Nothing!
```

### Why

AutoSkill is designed to be **embedded** inside other applications. If it required `openai>=1.0`, it might conflict with a host project that uses `openai==0.28`. By having zero dependencies, AutoSkill works in any Python 3.9+ environment without dependency conflicts.

**Alternative considered**: Listing optional dependencies with `extras_require`. Rejected because even optional dependencies can cause issues in enterprise environments with strict dependency policies.

**Tradeoff**: Users must install LLM/embedding client libraries themselves. But since they're already using those libraries in their own code, this is rarely an extra step.

---

## Pattern 2: Factory + Plugin Registry

### What

Provider implementations (LLM, embeddings, storage) are registered in a global registry and instantiated by factory functions from config dicts.

### Where

```python
# autoskill/llm/factory.py

_registry = {}  # Maps provider name → builder function

def register_llm_connector(provider, builder, aliases=None):
    """Register a custom LLM provider."""
    _registry[provider] = builder
    for alias in (aliases or []):
        _registry[alias] = builder

def build_llm(config: dict) -> LLM:
    """Create an LLM instance from a config dict."""
    provider = config["provider"]
    builder = _registry[provider]
    return builder(config)
```

### Why

This pattern provides **maximum flexibility** with **minimum coupling**:

1. **User code never imports provider classes** — just pass a dict
2. **New providers can be added at runtime** — no code changes to AutoSkill
3. **Config can come from anywhere** — environment variables, JSON files, code
4. **Testing is easy** — register a mock provider

**Alternative considered**: Dependency injection with constructor arguments. Rejected because it would require users to import specific classes, coupling their code to AutoSkill internals.

**Alternative considered**: Configuration files (YAML/TOML). Rejected because it adds a file dependency and makes programmatic configuration harder.

---

## Pattern 3: Strategy Pattern (Extraction & Maintenance)

### What

Extraction and maintenance have interchangeable algorithms selected by configuration.

### Where

```python
# Extraction strategies
if config.llm["provider"] == "mock":
    extractor = HeuristicSkillExtractor()  # Fast, no LLM
else:
    extractor = LLMSkillExtractor(llm)     # Accurate, uses LLM

# Maintenance strategies
if config.maintenance_strategy == "heuristic":
    # Use score thresholds and fuzzy matching
elif config.maintenance_strategy == "llm":
    # Ask LLM to judge merge/discard decisions
```

### Why

Different environments have different requirements:

| Scenario | Extraction | Maintenance | Reason |
|----------|-----------|-------------|--------|
| Development | Mock/Heuristic | Heuristic | Fast iteration, no API costs |
| Production (cost-sensitive) | LLM | Heuristic | Good extraction, cheap maintenance |
| Production (quality-sensitive) | LLM | LLM | Best quality, higher cost |

**Alternative considered**: Always use LLM. Rejected because it makes testing expensive and slow, and some environments don't have LLM access.

---

## Pattern 4: Hybrid Retrieval (Vector + BM25)

### What

Skill search combines semantic vector search with keyword-based BM25 search, then blends scores.

### Where

```python
# autoskill/management/stores/hybrid_rank.py

def blend_scores(vector_hits, bm25_hits, bm25_weight=0.3):
    """Combine vector similarity and BM25 keyword scores."""
    # hybrid = (1 - weight) × vector_score + weight × bm25_score
    combined = {}
    for hit in vector_hits:
        combined[hit.id] = (1 - bm25_weight) * hit.score
    for hit in bm25_hits:
        combined[hit.id] = combined.get(hit.id, 0) + bm25_weight * hit.score
    return sorted(combined.items(), key=lambda x: -x[1])
```

### Why

Neither search method alone is sufficient:

| Query | Vector Search | BM25 Search | Which Wins? |
|-------|--------------|-------------|-------------|
| "How to release software" | ✅ Finds "Release Process" by meaning | ❌ Might miss (no exact word match) | Vector |
| "deployment" | ❌ Too generic for vectors | ✅ Matches "deployment" trigger exactly | BM25 |
| "canary rollout steps" | ✅ Semantic match | ✅ Keywords match triggers | Both |

The hybrid approach catches both fuzzy semantic matches and exact keyword matches.

**Alternative considered**: Vector-only search. Rejected because it misses exact keyword matches and requires good embeddings to work well.

**Alternative considered**: BM25-only search. Rejected because it misses semantic similarity (synonyms, paraphrases).

---

## Pattern 5: Scope-Based Isolation

### What

Skills are organized into scopes (user, library) with independent search and management.

### Where

```
SkillBank/
├── Users/u1/       # User scope: private to u1
├── Users/u2/       # User scope: private to u2
└── Common/          # Library scope: shared across all users
```

```python
# autoskill/interactive/retrieval.py

def retrieve_hits_by_scope(query, config):
    if config.skill_scope in ("user", "all"):
        user_hits = search(query, user_id=config.user_id)
    if config.skill_scope in ("library", "all"):
        library_hits = search(query, user_id="library:common")
    return merge(user_hits, library_hits)
```

### Why

1. **Privacy**: User u1's skills should never leak to user u2
2. **Shared knowledge**: Common procedures should be available to everyone
3. **Independent thresholds**: Library skills might need higher confidence to surface
4. **Clean diagnostics**: The Web UI shows user hits and library hits separately

**Alternative considered**: Single flat namespace. Rejected because it doesn't support multi-user deployments safely.

---

## Pattern 6: Background Async Processing

### What

Skill extraction runs in background threads, not blocking the user's chat response.

### Where

```python
# autoskill/interactive/session.py

def turn(self, user_message):
    # Foreground: retrieve + respond (2-5 seconds)
    response = self._generate_response(user_message)
    
    # Background: extract skills (5-15 seconds)
    job_id = self._schedule_extraction(user_message)
    
    return {"assistant_reply": response, "extraction": {"job_id": job_id}}
```

### Why

Extraction requires an LLM call that takes 5–15 seconds. Running it synchronously would double the response time, making chat feel sluggish.

**Implementation details:**
- Jobs are queued in a FIFO queue
- A semaphore limits concurrent extractions (default: 2)
- Job status is pollable via API
- SSE streaming for real-time updates

**Alternative considered**: Synchronous extraction after each turn. Rejected because it makes chat unusably slow.

**Alternative considered**: WebSocket for real-time updates. Rejected because SSE is simpler and sufficient for one-way event streaming.

---

## Pattern 7: Graceful Degradation

### What

Every external call (LLM, embeddings, storage) has a fallback path so the system keeps working even when components fail.

### Where

| Component | Failure | Fallback |
|-----------|---------|----------|
| LLM extraction | API error | Heuristic extractor (generic skill) |
| Query rewriting | API error | Use original query unchanged |
| Skill selection | API error | Use all retrieved skills |
| Merge decision | API error | Heuristic scoring rules |
| Embedding API | API error | BM25-only search |
| SKILL.md file | Missing | Generate on-the-fly from stored data |

### Why

In production, external APIs fail regularly (rate limits, timeouts, outages). Crashing the entire system because the query rewriter is down is unacceptable.

**Design principle**: The system should always return *something useful*, even if quality is degraded.

**Alternative considered**: Strict error propagation (crash on any failure). Rejected because availability is more important than perfection for a skill augmentation system.

---

## Pattern 8: Version History in Metadata

### What

Skills store a history of previous versions as snapshots in their `metadata` field, rather than in a separate versioning system.

### Where

```python
skill.metadata["_autoskill_version_history"] = [
    {"version": "1.0.0", "snapshot": {full skill state}},
    {"version": "1.0.1", "snapshot": {full skill state}},
    # ... up to 30 entries
]
```

### Why

1. **Self-contained**: The skill file contains its own history — no external version database needed
2. **Portable**: Copy a skill directory and you get its history too
3. **Simple**: No migration, no schema, just a list in the metadata dict
4. **Bounded**: FIFO pruning at 30 entries prevents unbounded growth

**Alternative considered**: Git-based versioning (store history in git). Rejected because it adds a dependency on git and complicates the storage layer.

**Alternative considered**: Separate version database. Rejected because it breaks the "one directory = one skill" principle and adds infrastructure.

---

## Pattern 9: Deterministic IDs for Imports

### What

When importing skills from files, IDs are generated deterministically from (scope, owner, path) rather than randomly.

### Where

```python
# autoskill/management/identity.py

def generate_deterministic_id(scope, owner, path):
    """Generate a UUID that's the same every time for the same inputs."""
    content = f"{scope}:{owner}:{path}"
    return str(uuid.uuid5(uuid.NAMESPACE_URL, content))
```

### Why

**Problem**: If you import a skill library, then import it again later, you'd get duplicates because each import generates new random UUIDs.

**Solution**: Deterministic IDs ensure the same skill from the same source always gets the same ID, so re-imports are upserts (update-or-insert) rather than duplicates.

**Alternative considered**: Track imports in a registry. Rejected because it adds state and breaks the principle of idempotent imports.

---

## Pattern 10: CJK-Aware Text Sizing

### What

When truncating text to fit within `max_context_chars`, AutoSkill counts CJK (Chinese/Japanese/Korean) characters correctly.

### Where

```python
# autoskill/utils/units.py

def count_text_units(text):
    """Count text size, with CJK characters counting as 2 units."""
    count = 0
    for char in text:
        if is_cjk(char):
            count += 2  # CJK characters are wider and take more tokens
        else:
            count += 1
    return count
```

### Why

CJK characters are:
- **Wider** in monospace display (take 2 columns)
- **More token-expensive** in most LLM tokenizers
- **Corruptible** if cut mid-character in UTF-8

Without CJK-aware sizing, a 6000-character limit would allow far fewer CJK characters than Latin characters, leading to unfair truncation for Chinese/Japanese users.

**Rationale**: AutoSkill is used with Chinese LLM providers (Zhipu, DashScope, InternLM) and likely has Chinese-speaking users. Fair text sizing is essential for a good experience.

---

## Pattern 11: Monkey-Patching for Plugin Customization

### What

The OpenClaw plugin replaces default functions in the maintenance module at runtime.

### Where

```python
# OpenClaw-Plugin/agentic_prompt_profile.py

import autoskill.management.maintenance as _m

# Replace default decision function with agentic version
_m._decide_candidate_action_with_llm = _decide_candidate_action_with_llm_agentic
_m._merge_with_llm = _merge_with_llm_agentic
```

### Why

1. **No forking**: The core AutoSkill code stays unchanged
2. **Scoped**: Only loaded when the OpenClaw plugin runs
3. **Flexible**: Can override any function, not just interfaces
4. **Reversible**: Just don't import the plugin to get default behavior

**Alternative considered**: Abstract base classes with plugin overrides. Rejected because the granularity of customization needed (individual functions, not entire classes) makes ABC-based plugins too coarse.

**Tradeoff**: Monkey-patching is fragile if the patched functions change signatures. This is acceptable because the plugin and core are maintained together.

---

## Pattern Summary

| Pattern | Where Used | Primary Benefit |
|---------|-----------|----------------|
| Zero Dependencies | `pyproject.toml` | Embeddable anywhere |
| Factory + Plugin | `llm/factory.py`, `embeddings/factory.py` | Swap providers without code changes |
| Strategy | Extraction, Maintenance | Balance cost vs. quality |
| Hybrid Retrieval | `stores/hybrid_rank.py` | Catch both semantic and keyword matches |
| Scope Isolation | `Users/` vs `Common/` | Multi-user privacy + shared knowledge |
| Background Async | `interactive/session.py` | Non-blocking chat experience |
| Graceful Degradation | Throughout | Keep working when components fail |
| Version History | `metadata._autoskill_version_history` | Self-contained, portable skill history |
| Deterministic IDs | `management/identity.py` | Idempotent imports |
| CJK-Aware Sizing | `utils/units.py` | Fair multi-language support |
| Monkey-Patching | OpenClaw Plugin | Plugin customization without forking |

---

## Next Steps

- **[Architecture Overview](architecture-overview.md)** — See how these patterns form the architecture
- **[Provider System](provider-system.md)** — Deep dive into the factory/plugin pattern
- **[Storage & SkillBank](storage-and-skillbank.md)** — See the version history pattern in action
