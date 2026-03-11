# AutoSkill Documentation

> **AutoSkill** is an experience-driven lifelong learning (ELL) system that automatically extracts, maintains, and retrieves reusable AI agent skills from conversations and behavior logs.

This documentation breaks down every aspect of AutoSkill's design and implementation into beginner-friendly, standalone files. Each file covers one topic in depth, explains the "why" behind each design decision, and includes diagrams and code examples.

---

## 📖 Documentation Index

| Document | What You'll Learn |
|----------|-------------------|
| [Architecture Overview](architecture-overview.md) | The big picture — how all the layers and components fit together |
| [Data Flow](data-flow.md) | Step-by-step walkthrough of how data moves through the system |
| [Data Schemas](data-schemas.md) | Every data model, what each field means, and how they relate |
| [API Reference](api-reference.md) | All HTTP endpoints and SDK methods with request/response examples |
| [Agent & Skill Architecture](agent-and-skill-architecture.md) | What a "skill" really is, how skills are extracted, maintained, and used |
| [Provider System](provider-system.md) | How LLM, embedding, and storage backends are plugged in |
| [Interactive System](interactive-system.md) | How the chat session, Web UI, and proxy server work |
| [Offline Pipelines](offline-pipelines.md) | How documents, conversations, and trajectories become skills |
| [Storage & SkillBank](storage-and-skillbank.md) | The file system layout, SKILL.md format, and version history |
| [OpenClaw Plugin](openclaw-plugin.md) | How AutoSkill integrates with the OpenClaw agent framework |
| [Design Patterns](design-patterns.md) | The key patterns used and *why* they were chosen |
| [Deployment Guide](deployment.md) | How to run AutoSkill — SDK, Web UI, Proxy, Docker |
| [Configuration Reference](configuration.md) | Every config option explained with defaults and examples |

---

## 🗂 Repository Structure (Quick Reference)

```
AutoSkill/
├── autoskill/               # Core SDK (Python package)
│   ├── client.py            # Main SDK entrypoint (ingest, search, render)
│   ├── models.py            # Core data models (Skill, SkillHit)
│   ├── config.py            # Configuration dataclass
│   ├── render.py            # Context rendering for LLM injection
│   ├── llm/                 # LLM provider abstraction layer
│   ├── embeddings/          # Embedding provider abstraction layer
│   ├── management/          # Skill extraction, maintenance, storage
│   ├── interactive/         # Session orchestration, HTTP servers
│   ├── offline/             # Batch processing pipelines
│   └── utils/               # Shared utilities
├── examples/                # Runnable example scripts
├── OpenClaw-Plugin/         # Integration with OpenClaw agent framework
├── SkillBank/               # Default skill storage directory
├── web/                     # Web UI frontend (HTML/JS/CSS)
├── tests/                   # Test suite
├── docs/                    # ← You are here
├── pyproject.toml           # Package configuration
├── docker-compose.yml       # Container deployment
└── Dockerfile               # Container image definition
```

---

## 🚀 Getting Started

If you're new to AutoSkill, we recommend reading the docs in this order:

1. **[Architecture Overview](architecture-overview.md)** — Understand the big picture first
2. **[Agent & Skill Architecture](agent-and-skill-architecture.md)** — Understand what skills are and how they work
3. **[Data Flow](data-flow.md)** — See how everything connects
4. **[Data Schemas](data-schemas.md)** — Understand the data structures
5. **[Configuration Reference](configuration.md)** — Set up your environment
6. **[Deployment Guide](deployment.md)** — Run AutoSkill

Then dive into specific topics as needed:
- Building integrations? → [API Reference](api-reference.md) + [Provider System](provider-system.md)
- Understanding internals? → [Design Patterns](design-patterns.md) + [Storage & SkillBank](storage-and-skillbank.md)
- Processing documents? → [Offline Pipelines](offline-pipelines.md)
- Using with OpenClaw? → [OpenClaw Plugin](openclaw-plugin.md)
