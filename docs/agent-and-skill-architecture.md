# Agent & Skill Architecture

> This document explains AutoSkill's core concept: **skills as reusable capability units**. It covers what a skill is, how skills are extracted from conversations, how they evolve over time, and how they're used to improve AI responses.

---

## What Is a Skill?

A **skill** in AutoSkill is NOT an autonomous agent. It's a **reusable, injectable piece of knowledge** — a procedure, a best practice, a decision rule — that gets injected into an LLM's context window when relevant.

### What a Skill IS vs. IS NOT

| A Skill IS ✓ | A Skill IS NOT ✗ |
|---|---|
| A reusable prompt/instruction block | An independent agent |
| Discovered via semantic search | A service with its own state |
| Versioned and evolved through merging | A long-running process |
| Stored as a human-readable SKILL.md file | A code module or function |
| Used to augment an LLM's context | A tool or API endpoint |

**Rationale**: AutoSkill deliberately chose this simpler model over multi-agent architectures. Skills as context injections are:
- **Universally compatible** — any LLM application can use them by prepending text to a prompt
- **Zero-overhead** — no agent framework, no orchestration, no message passing
- **Predictable** — the LLM always sees the skill text, no agent scheduling uncertainty
- **Debuggable** — you can read the SKILL.md file and see exactly what the LLM will get

### Anatomy of a Skill

```
┌─────────────────────────────────────────────┐
│             SKILL: Release Process           │
├─────────────────────────────────────────────┤
│                                             │
│  IDENTITY                                   │
│  ├─ id: a1b2c3d4-...                       │
│  ├─ user_id: u1                            │
│  └─ version: 1.2.0                         │
│                                             │
│  CONTENT (what gets injected into LLM)      │
│  ├─ name: "Release Process"                │
│  ├─ description: "Run regression tests..." │
│  ├─ instructions: (full procedure)          │
│  └─ examples: [{input, output, notes}]      │
│                                             │
│  DISCOVERY (how it gets found)              │
│  ├─ triggers: ["How do I release?", ...]   │
│  └─ tags: ["devops", "deployment"]         │
│                                             │
│  LIFECYCLE                                  │
│  ├─ status: ACTIVE                         │
│  ├─ created_at: 2024-01-15T10:30:00Z      │
│  └─ version_history: [snapshots...]        │
│                                             │
└─────────────────────────────────────────────┘
```

---

## The Skill Lifecycle

A skill goes through four phases during its lifetime:

### Phase 1: Extraction

Skills are born from conversations. When a user and an AI have a productive exchange, AutoSkill analyzes the conversation and extracts reusable knowledge.

```
User: "Our deployment failed last week. What went wrong?"
AI: "Looking at the logs, you skipped the canary phase. 
     Always do: 1) regression tests, 2) canary (1% traffic, 30min),
     3) gradual rollout, 4) monitor for 24h."
User: "Thanks, I'll follow that from now on."

     ↓ AutoSkill extracts: ↓

SkillCandidate:
  name: "Safe Deployment Process"
  instructions: "1. Run regression tests..."
  triggers: ["deployment", "release", "rollout"]
  confidence: 0.87
```

**Two extraction strategies:**

| Strategy | When Used | How It Works | Tradeoff |
|----------|-----------|--------------|----------|
| **LLM-based** | Default | Send conversation to LLM with structured extraction prompt | Accurate but costs tokens |
| **Heuristic** | When `provider="mock"` | Generate a generic "Standard Operating Procedure" skill | Free but low quality |

**Rationale**: The LLM-based strategy is preferred because it understands context and generates meaningful skill names, triggers, and instructions. The heuristic strategy exists for testing and environments without LLM access.

**File**: `autoskill/management/extraction.py`

### Phase 2: Maintenance (Dedup & Merge)

Once a skill candidate is extracted, it's not immediately saved. First, it goes through maintenance to prevent duplicates and improve existing skills.

```
SkillCandidate: "Safe Deployment Process"
                          │
                          ▼
┌─────────────────────────────────────────┐
│ STEP 1: Find Similar Existing Skills    │
│                                         │
│ Search by:                              │
│  • Vector similarity (semantic)         │
│  • Name similarity (fuzzy string match) │
│  • Tag/trigger overlap                  │
│                                         │
│ Found: "Release Process" (score: 0.91)  │
└─────────────────────────┬───────────────┘
                          │
                          ▼
┌─────────────────────────────────────────┐
│ STEP 2: Decision                        │
│                                         │
│ similarity = 0.91 (above threshold)     │
│                                         │
│  ADD:     if no similar skills found    │
│  MERGE:   if similar skill exists       │  ← This case
│  DISCARD: if too similar or low quality │
└─────────────────────────┬───────────────┘
                          │
                          ▼
┌─────────────────────────────────────────┐
│ STEP 3: Merge Execution                 │
│                                         │
│ Existing: "Release Process" v1.1.0      │
│  + New candidate instructions           │
│  + New candidate triggers               │
│  + New examples                         │
│  = Updated "Release Process" v1.2.0     │
│                                         │
│ Version history updated:                │
│  [v1.0.0, v1.0.1, v1.1.0, v1.2.0]     │
└─────────────────────────────────────────┘
```

**Two maintenance strategies:**

| Strategy | Config | How Decisions Are Made | Tradeoff |
|----------|--------|----------------------|----------|
| **Heuristic** | `maintenance_strategy="heuristic"` | Score thresholds + fuzzy matching | Fast, free, may miss nuance |
| **LLM** | `maintenance_strategy="llm"` | LLM judges if skills should merge | Accurate, costs tokens |

**Rationale**: Maintenance is the key to preventing skill bloat. Without it, the skill bank fills up with near-duplicates that dilute search quality. The two strategies let users balance cost vs. quality.

**File**: `autoskill/management/maintenance.py`

### Phase 3: Storage & Retrieval

Once persisted, skills are discoverable via hybrid search.

```
User query: "How do I safely deploy?"
                          │
                          ▼
┌─────────────────────────────────────────┐
│ HYBRID SEARCH                           │
│                                         │
│ Vector search (semantic):               │
│  "safely deploy" ≈ "Release Process"    │
│  Score: 0.85                            │
│                                         │
│ BM25 search (keyword):                  │
│  "deploy" matches trigger "deployment"  │
│  Score: 0.72                            │
│                                         │
│ Hybrid blend:                           │
│  (0.7 × 0.85) + (0.3 × 0.72) = 0.81   │
│                                         │
│ Result: SkillHit("Release Process", 0.81)│
└─────────────────────────────────────────┘
```

**Why hybrid search?**
- Vector search catches semantic similarity ("safely deploy" ≈ "release process")
- BM25 catches exact keywords ("deploy" matches "deployment" trigger)
- Together they cover both fuzzy and exact matching

**File**: `autoskill/management/stores/hybrid_rank.py`

### Phase 4: Evolution

Skills evolve over time as they merge with new candidates. Each merge creates a new version, and version history is preserved for rollback.

```
Timeline:
  v1.0.0  │ Initial: "Run tests, deploy, monitor"
  v1.0.1  │ Added: canary step between deploy and monitor
  v1.1.0  │ Added: specific latency thresholds for rollback
  v1.2.0  │ Merged: new team's deployment checklist
  v1.2.1  │ Added: example for database migration case

History stored in metadata._autoskill_version_history
Maximum 30 snapshots kept (FIFO pruning)
Rollback available via pop_skill_snapshot()
```

**Rationale**: Version history makes skills auditable and recoverable. Teams can see how a skill evolved and rollback if a bad merge happens.

**File**: `autoskill/interactive/skill_versions.py`

---

## Skill Scoping: User vs. Library

Skills live in one of two scopes:

### User Scope

```
/SkillBank/Users/u1/release-process/SKILL.md
/SkillBank/Users/u1/monitoring-setup/SKILL.md
/SkillBank/Users/u2/code-review/SKILL.md
```

- **Private**: Only visible to the owning user
- **Personal**: Learned from that user's conversations
- **user_id**: Set to the actual user ID (e.g., "u1")

### Library Scope

```
/SkillBank/Common/anthropics-skill/create-pdf/SKILL.md
/SkillBank/Common/anthropics-skill/web-automation/SKILL.md
```

- **Shared**: Visible to all users
- **Curated**: Pre-packaged or admin-managed
- **user_id**: Set to `"library:<library_name>"` (e.g., "library:common")

### Retrieval Behavior by Scope Setting

| `skill_scope` Config | What Gets Searched | Use Case |
|---------------------|--------------------|----------|
| `"user"` | Only `/Users/<user_id>/` | Personal assistant — only my skills |
| `"library"` | Only `/Common/` | Shared team knowledge — no personal learning |
| `"all"` | Both user + library, merged results | Full experience — personal + shared (recommended) |

**Rationale**: Scope isolation prevents cross-user data leakage while allowing shared knowledge libraries. A team can maintain a common skill library while each member also builds personal skills.

---

## Context Injection Pattern

When skills are used in a conversation, they're injected as a markdown block in the system prompt:

```
┌─────────────────────────────────────────────────────┐
│ System Prompt (what the LLM sees):                  │
│                                                     │
│ You are a helpful assistant.                        │
│                                                     │
│ ## Available Skills                                 │
│                                                     │
│ ### Release Process (v1.2.0)                        │
│ Run regression tests, canary rollout, monitor,      │
│ then full rollout. Never skip canary.               │
│                                                     │
│ **Steps:**                                          │
│ 1. Regression Testing                               │
│ 2. Canary Rollout (1% traffic, 30min)              │
│ 3. Gradual Rollout (10%, 50%, 100%)                │
│ 4. Post-Deploy Monitoring (24h)                     │
│                                                     │
│ **Example:**                                        │
│ Q: Should I deploy directly?                        │
│ A: No, always canary first.                         │
│                                                     │
│ ---                                                 │
│                                                     │
│ Follow the most relevant skill if applicable.       │
│                                                     │
│ [User message follows...]                           │
└─────────────────────────────────────────────────────┘
```

**Rationale**: Markdown is the universal format that LLMs understand well. It's structured enough to be parsed but readable enough to be useful. The rendering respects a configurable character limit (`max_context_chars`) with CJK-aware truncation to prevent cutting multi-byte characters.

**File**: `autoskill/render.py`

---

## Extraction Gating: When to Extract

Not every conversation turn should trigger extraction. AutoSkill uses **gating logic** to decide when:

| Mode | Behavior | Best For |
|------|----------|----------|
| `"always"` | Extract after every turn | Development, testing |
| `"auto"` | Extract every N turns or on topic change | Production use |
| `"never"` | Only on manual `/extract` command | Cost-sensitive, manual control |

### Topic Change Detection

In `"auto"` mode, AutoSkill can detect when the conversation topic changes, which is a good time to extract skills from the previous topic:

```
Turn 1: "How do I deploy?" → (same topic)
Turn 2: "What about canary?" → (same topic)
Turn 3: "Now, about database backups..." → TOPIC CHANGE → extract!
```

**Rationale**: Extracting on topic boundaries captures complete procedures rather than partial ones. If you extract mid-topic, you might only get half a procedure.

**File**: `autoskill/interactive/gating.py`

---

## Usage Tracking: Was the Skill Helpful?

After a conversation turn, AutoSkill can judge whether retrieved skills were actually used by the assistant:

```
Retrieved skill: "Release Process"
Assistant reply: "Here's the deployment process: 1) Run tests, 2) Canary..."

LLM Judge verdict:
  - Relevance: HIGH (skill was relevant to query)
  - Usage: YES (assistant clearly used the skill's instructions)
  - Quality: GOOD (response was accurate and complete)
```

This tracking data feeds back into skill quality metrics, helping identify which skills are actually useful vs. retrieved but ignored.

**Rationale**: Without usage tracking, the system has no signal about skill quality. A skill might rank high in search but actually be unhelpful. Usage judgments close this feedback loop.

**File**: `autoskill/interactive/usage_tracking.py`

---

## Multi-Source Learning

AutoSkill can learn skills from multiple sources:

| Source | How It Works | File |
|--------|-------------|------|
| **Live conversations** | Real-time extraction during chat | `interactive/session.py` |
| **OpenAI exports** | Batch import and extract from exported chats | `client.py` |
| **Documents** | 5-stage pipeline: ingest → evidence → capability → skill → version | `offline/document/` |
| **Conversation logs** | Offline extraction from JSONL files | `offline/conversation/` |
| **Trajectory logs** | Extract from agent action sequences | `offline/trajectory/` |
| **Manual import** | Import existing SKILL.md directories | `management/importer.py` |

**Rationale**: Knowledge exists in many forms. By supporting multiple input sources, AutoSkill can build a comprehensive skill bank from all available organizational knowledge.

---

## Next Steps

- **[Data Flow](data-flow.md)** — Trace the exact steps in extraction and retrieval
- **[Storage & SkillBank](storage-and-skillbank.md)** — See how skills are stored on disk
- **[Provider System](provider-system.md)** — Understand the LLM/embedding providers that power extraction and search
