# Architecture Overview

> This document explains AutoSkill's overall system design — what each layer does, why it exists, and how the pieces connect. Think of it as a map of the entire system.

---

## What Is AutoSkill?

AutoSkill is a **skill management SDK** for AI agents. It solves a specific problem: when an AI agent has a good conversation with a user and discovers useful knowledge or procedures, that knowledge is lost after the conversation ends. AutoSkill captures those discoveries as reusable **skills** and makes them available in future conversations.

**In plain English**: AutoSkill gives AI agents a "long-term memory" for procedures and best practices.

---

## The Five-Layer Architecture

AutoSkill is organized into five distinct layers, each with a clear responsibility. Data flows downward through these layers during operation.

```
┌──────────────────────────────────────────────────────────────┐
│                   1. USER INTERFACE LAYER                     │
│                                                              │
│    Web UI        Console Chat       OpenAI Proxy             │
│   (browser)      (terminal)        (HTTP server)             │
│                                                              │
│  Rationale: Multiple interfaces so AutoSkill can be used     │
│  as a standalone app, embedded tool, or drop-in proxy.       │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│              2. INTERACTIVE SESSION LAYER                     │
│                                                              │
│  Query Rewriting → Skill Retrieval → Context Rendering       │
│  → LLM Completion → Background Extraction → Usage Tracking   │
│                                                              │
│  Rationale: Orchestrates the full turn cycle so that UI      │
│  layers stay thin and don't duplicate logic.                 │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│                  3. SDK CORE LAYER                            │
│                                                              │
│  AutoSkill class: ingest() / search() / render_context()     │
│  + import/export utilities                                   │
│                                                              │
│  Rationale: Provides a clean, simple API for any Python      │
│  code to use — no HTTP servers needed.                       │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│                 4. MANAGEMENT LAYER                           │
│                                                              │
│  ┌────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│  │ Extraction  │   │ Maintenance  │   │ Storage Backend  │   │
│  │             │   │              │   │                  │   │
│  │ Heuristic   │   │ Deduplicate  │   │ Local filesystem │   │
│  │ or LLM      │   │ Merge        │   │ ChromaDB         │   │
│  │             │   │ Version      │   │ Pinecone         │   │
│  └────────────┘   └──────────────┘   │ Milvus           │   │
│                                      │ In-memory         │   │
│  Rationale: Separates "what to       └──────────────────┘   │
│  extract" from "how to store it"                             │
│  from "how to merge duplicates".                             │
└──────────────────────────┬───────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────┐
│             5. PROVIDER ABSTRACTION LAYER                     │
│                                                              │
│  ┌─────────────────────┐   ┌──────────────────────────┐     │
│  │   LLM Providers     │   │  Embedding Providers     │     │
│  │                     │   │                          │     │
│  │  OpenAI (GPT-4)     │   │  OpenAI (text-embedding) │     │
│  │  Anthropic (Claude) │   │  DashScope (Qwen)        │     │
│  │  Zhipu (GLM)        │   │  Zhipu (BigModel)        │     │
│  │  InternLM           │   │  Generic HTTP             │     │
│  │  Generic HTTP       │   │  Hashing (no API needed)  │     │
│  │  Mock (testing)     │   │  None (placeholder)       │     │
│  └─────────────────────┘   └──────────────────────────┘     │
│                                                              │
│  Rationale: AutoSkill doesn't care which LLM or embedding    │
│  service you use. Swap providers by changing one config line. │
└──────────────────────────────────────────────────────────────┘
```

---

## Why Five Layers?

Each layer exists to solve a specific problem:

| Layer | Problem It Solves | Why It's Separate |
|-------|------------------|-------------------|
| **User Interface** | "How do users interact?" | Different users need different interfaces (browser, terminal, API) |
| **Interactive Session** | "How does a full conversation turn work?" | Orchestration logic is complex and shouldn't be in UI code |
| **SDK Core** | "What's the simplest way to use AutoSkill?" | Other Python projects should be able to `import autoskill` and call 3 methods |
| **Management** | "How are skills extracted, merged, and stored?" | These are independent concerns that change at different rates |
| **Provider Abstraction** | "Which LLM/embedding service do we use?" | Users should be able to swap providers without touching business logic |

---

## Component Map

Here's every major component and its source file:

### User Interface Layer

| Component | File(s) | Description |
|-----------|---------|-------------|
| **Web UI** | `examples/web_ui.py`, `web/` | Browser-based chat with skill diagnostics panel |
| **Console Chat** | `autoskill/interactive/app.py` | Terminal-based REPL with commands like `/help`, `/extract` |
| **OpenAI Proxy** | `autoskill/interactive/server.py`, `examples/openai_proxy.py` | Drop-in replacement for OpenAI API that adds skill injection |

### Interactive Session Layer

| Component | File | Description |
|-----------|------|-------------|
| **Session Orchestrator** | `autoskill/interactive/session.py` | Manages the full turn cycle (retrieve → respond → extract) |
| **Runtime Compositor** | `autoskill/interactive/unified.py` | Wires together session, SDK, LLM, and config |
| **Query Rewriter** | `autoskill/interactive/rewriting.py` | Uses LLM to expand/clarify user queries before search |
| **Skill Selector** | `autoskill/interactive/selection.py` | Uses LLM to pick the most relevant skills from search results |
| **Extraction Gate** | `autoskill/interactive/gating.py` | Decides *when* to run extraction (every N turns, on topic change) |
| **Usage Tracker** | `autoskill/interactive/usage_tracking.py` | Judges whether retrieved skills were actually useful |
| **Version Manager** | `autoskill/interactive/skill_versions.py` | Manages skill snapshots and rollback |

### SDK Core Layer

| Component | File | Description |
|-----------|------|-------------|
| **AutoSkill** | `autoskill/client.py` | Main class: `ingest()`, `search()`, `render_context()`, import/export |
| **Config** | `autoskill/config.py` | `AutoSkillConfig` dataclass with all settings |
| **Models** | `autoskill/models.py` | `Skill`, `SkillHit`, `SkillStatus` dataclasses |
| **Renderer** | `autoskill/render.py` | Formats skills into injectable markdown context |

### Management Layer

| Component | File | Description |
|-----------|------|-------------|
| **Skill Extractor** | `autoskill/management/extraction.py` | Extracts skill candidates from conversations (heuristic or LLM) |
| **Skill Maintainer** | `autoskill/management/maintenance.py` | Deduplicates, merges, and versions skills |
| **Skill Identity** | `autoskill/management/identity.py` | Generates deterministic IDs and dedup hashes |
| **Skill Stores** | `autoskill/management/stores/` | Storage backends (local, in-memory, Chroma, Pinecone, Milvus) |
| **Vector Indexes** | `autoskill/management/vectors/` | Vector search implementations |
| **BM25 Index** | `autoskill/management/stores/bm25_index.py` | Keyword-based search index |
| **Hybrid Ranker** | `autoskill/management/stores/hybrid_rank.py` | Blends vector + BM25 scores |
| **SKILL.md Format** | `autoskill/management/formats/agent_skill.py` | Renders and parses the SKILL.md artifact format |
| **Artifact Export** | `autoskill/management/artifacts.py` | Writes skill directories to disk |
| **Skill Importer** | `autoskill/management/importer.py` | Imports skills from existing directories |
| **Bootstrap** | `autoskill/management/bootstrap.py` | Startup tasks (normalize IDs, auto-import skills) |

### Provider Abstraction Layer

| Component | File | Description |
|-----------|------|-------------|
| **LLM Interface** | `autoskill/llm/base.py` | Abstract base class for LLM providers |
| **LLM Factory** | `autoskill/llm/factory.py` | `build_llm()` + plugin registration |
| **Embedding Interface** | `autoskill/embeddings/base.py` | Abstract base class for embedding providers |
| **Embedding Factory** | `autoskill/embeddings/factory.py` | `build_embeddings()` + plugin registration |

---

## How the Layers Talk to Each Other

Each layer only talks to the layer directly below it:

```
Web UI  ──calls──▶  InteractiveSession  ──calls──▶  AutoSkill (SDK)
                                                         │
                                                    ┌────┴────┐
                                                    ▼         ▼
                                              Extraction  Storage
                                                    │         │
                                                    ▼         ▼
                                              LLM Provider  Embedding Provider
```

**Rationale**: This one-way dependency makes the code easier to test and reason about. You can use the SDK layer without any UI. You can use the management layer without the SDK. You can swap LLM providers without touching anything above them.

---

## Offline Pipeline (Separate Path)

In addition to the interactive path above, AutoSkill has an **offline pipeline** for batch-processing documents, conversations, and trajectories into skills:

```
┌──────────────────┐     ┌────────────────────┐     ┌───────────────┐
│  Raw Documents   │     │  Conversation Logs  │     │  Trajectories │
│  (PDF, Markdown) │     │  (JSONL exports)    │     │  (Action logs)│
└────────┬─────────┘     └────────┬────────────┘     └───────┬───────┘
         │                        │                          │
         ▼                        ▼                          ▼
┌────────────────────────────────────────────────────────────────────┐
│                    OFFLINE PIPELINE LAYER                          │
│                                                                    │
│  Document Pipeline (5 stages):                                     │
│  Ingest → Extract Evidence → Induce Capabilities → Compile Skills  │
│  → Register Versions                                               │
│                                                                    │
│  Conversation Pipeline: Load → Extract → Persist                   │
│  Trajectory Pipeline: Load → Extract → Persist                     │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                         ▼
               ┌──────────────────┐
               │  SkillBank       │
               │  (filesystem)    │
               └──────────────────┘
```

**Rationale**: Offline pipelines exist because not all learning happens in real-time conversations. Research papers, documentation, and historical logs contain valuable knowledge that should be converted into skills too.

---

## Key Architectural Decisions

### 1. Zero External Dependencies

```toml
# pyproject.toml
dependencies = []  # Nothing!
```

**Rationale**: AutoSkill is designed to be **embedded** in other projects. If it required specific versions of `openai`, `anthropic`, etc., it would conflict with the host project's dependencies. By having zero dependencies, it works in any Python 3.9+ environment.

### 2. Configuration-Driven Provider Selection

```python
config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-4"},          # Just a dict
    embeddings={"provider": "openai", "model": "text-embedding-3-large"},
    store={"provider": "local", "path": "./SkillBank"}
)
```

**Rationale**: Dict-based config means you can load settings from environment variables, config files, or code. No need to import provider-specific classes.

### 3. Skills as Stateless Context Injections

AutoSkill does **not** create autonomous agents. A skill is just a piece of text (instructions + examples) that gets injected into the LLM's system prompt.

**Rationale**: This is the simplest possible integration pattern. Any LLM application can add skill injection by prepending text to the system prompt. No agent framework required.

### 4. Hybrid Retrieval (Vector + BM25)

Skills are found using both semantic (vector) search and keyword (BM25) search, then scores are blended.

**Rationale**: Vector search alone misses exact keyword matches (e.g., a skill named "Release Process" might not rank high for the query "release"). BM25 alone misses semantic similarity. The hybrid approach catches both.

### 5. Background Extraction

Skill extraction runs in a background thread after the assistant responds, not during the conversation turn.

**Rationale**: Extraction requires an LLM call which takes 2–10 seconds. Running it in the foreground would make the chat feel slow. Background processing keeps the user experience snappy.

---

## Next Steps

- **[Data Flow](data-flow.md)** — See how data moves through these layers step by step
- **[Agent & Skill Architecture](agent-and-skill-architecture.md)** — Understand what skills are and how they're managed
- **[Design Patterns](design-patterns.md)** — Deep dive into the engineering patterns used
