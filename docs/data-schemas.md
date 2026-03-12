# Data Schemas

> This document describes every data model used in AutoSkill — what each field means, why it exists, and how models relate to each other. If you're building on or integrating with AutoSkill, this is your reference.

---

## Core Models

These models are defined in `autoskill/models.py` and used throughout the entire system.

### Skill

The **Skill** is the central data model of AutoSkill. It represents a reusable piece of knowledge — a procedure, a best practice, a decision rule — that an AI agent can use.

```python
@dataclass
class Skill:
    # ── Identity ──────────────────────────────────────────────
    id: str
    # A UUID or deterministic hash that uniquely identifies this skill.
    # Deterministic IDs are used when importing from files so that
    # re-importing doesn't create duplicates.

    user_id: str
    # Who owns this skill.
    # Format: "u1" for user skills, "library:common" for shared library skills.
    # WHY: Skills are scoped per user. Your personal skills don't
    # interfere with mine. Library skills are shared across users.

    # ── Core Content ──────────────────────────────────────────
    name: str
    # Short human-readable name like "Release Process" or "PDF Form Filler".
    # Used for display and name-based deduplication.

    description: str
    # ~100 character summary. This is the primary text used for
    # vector search — it's what gets embedded.

    instructions: str
    # The full procedure or prompt. This is what gets injected into
    # the LLM's context. Can be multiple paragraphs with steps,
    # constraints, and rules.

    # ── Discovery Metadata ────────────────────────────────────
    triggers: List[str]
    # Keywords or phrases that should activate this skill.
    # Example: ["How do I release?", "deployment steps", "rollout"]
    # WHY: Triggers help the BM25 keyword search find this skill
    # even when the user's query doesn't match the description.

    examples: List[SkillExample]
    # Few-shot examples showing how this skill is used.
    # WHY: LLMs perform better with examples. Including them in
    # the skill makes the injected context more effective.

    tags: List[str]
    # Categorical labels like ["devops", "deployment"].
    # Used for filtering and deduplication signal overlap.

    # ── Versioning ────────────────────────────────────────────
    version: str
    # Semantic version string like "1.0.0" or "1.2.3".
    # Bumped automatically when skills are merged.
    # Patch (1.0.X): Minor improvements or additional examples
    # Minor (1.X.0): New instructions or triggers added
    # Major (X.0.0): Significant rewrite

    status: SkillStatus
    # ACTIVE or ARCHIVED. Archived skills are not returned in search.

    # ── Resources ─────────────────────────────────────────────
    files: Dict[str, str]
    # Optional bundled resources: {"script.py": "print('hello')", ...}
    # WHY: Some skills include helper scripts or templates that
    # the agent should use when executing the skill.

    source: Optional[Dict[str, Any]]
    # Provenance metadata: where this skill came from.
    # Example: {"type": "conversation", "turn": 5, "session": "abc123"}
    # WHY: Tracking provenance helps with debugging and auditing.

    metadata: Dict[str, Any]
    # Extensible metadata bucket. AutoSkill uses this for internal
    # bookkeeping like version history:
    #   metadata["_autoskill_version_history"] = [
    #     {"version": "1.0.0", "snapshot": {...}},
    #     {"version": "1.0.1", "snapshot": {...}}
    #   ]

    # ── Timestamps ────────────────────────────────────────────
    created_at: Optional[str]   # ISO 8601 timestamp
    updated_at: Optional[str]   # ISO 8601 timestamp
```

### SkillExample

A single input/output example showing how the skill is used.

```python
@dataclass
class SkillExample:
    input: str      # Example user query or scenario
    output: str     # Expected response or action
    notes: str      # Optional notes explaining the example
```

**Example**:
```python
SkillExample(
    input="Should I deploy directly to production?",
    output="No, always do a canary deployment first with at least 1% of traffic.",
    notes="Safety-first approach — canary catches issues before they affect all users."
)
```

### SkillHit

A search result — a skill paired with a relevance score.

```python
@dataclass(frozen=True)
class SkillHit:
    skill: Skill     # The matched skill
    score: float     # Relevance score from 0.0 to 1.0 (higher = more relevant)
```

**Why frozen?** Search results should not be modified after creation. Making the dataclass frozen prevents accidental mutation.

### SkillStatus

An enum tracking whether a skill is active or archived.

```python
class SkillStatus(str, Enum):
    ACTIVE = "active"
    ARCHIVED = "archived"
```

---

## Extraction Models

These models are used during the skill extraction process (defined in `autoskill/management/extraction.py`).

### SkillCandidate

A raw skill extracted from a conversation, before deduplication or persistence.

```python
@dataclass
class SkillCandidate:
    name: str              # Proposed skill name
    description: str       # Proposed description
    instructions: str      # Proposed instructions
    triggers: List[str]    # Proposed triggers
    examples: List[SkillExample]  # Proposed examples
    tags: List[str]        # Proposed tags
    files: Dict[str, str]  # Proposed bundled files
    confidence: float      # Extraction confidence (0.0–1.0)
    source: Dict[str, Any] # Where this candidate came from
```

**Why separate from Skill?** A candidate doesn't have an ID, version, or status yet. It's a *proposal* that the maintenance system will either accept (ADD), merge with an existing skill (MERGE), or reject (DISCARD).

---

## Configuration Models

### AutoSkillConfig

The master configuration object (defined in `autoskill/config.py`).

```python
@dataclass
class AutoSkillConfig:
    # ── Provider Configuration ─────────────────────────────
    llm: Dict[str, Any]
    # LLM provider config. Example:
    # {"provider": "openai", "model": "gpt-4", "api_key": "sk-..."}

    embeddings: Dict[str, Any]
    # Embedding provider config. Example:
    # {"provider": "openai", "model": "text-embedding-3-large"}

    store: Dict[str, Any]
    # Storage backend config. Example:
    # {"provider": "local", "path": "./SkillBank"}

    # ── Skill Management ───────────────────────────────────
    namespace: str
    # Skill namespace for isolation. Default: "default"

    maintenance_strategy: str
    # "llm" or "heuristic". Controls whether an LLM is used
    # to make merge/discard decisions (more accurate but costly)
    # or simple rules are used (faster and free).

    dedupe_similarity_threshold: float
    # Similarity score [0–1] above which two skills are
    # considered duplicates. Default: ~0.85

    max_candidates_per_ingest: int
    # Maximum skills to extract per ingest call. Default: 1
    # WHY: Extracting too many skills per turn creates noise.
    # Usually one good skill per conversation segment is enough.

    max_similar_skills_to_consider: int
    # When checking for duplicates, how many existing skills to compare.

    # ── Search Configuration ───────────────────────────────
    default_search_limit: int
    # Top-K results to return. Default: 5

    max_context_chars: int
    # Maximum characters in rendered context. CJK-aware.

    bm25_weight: float
    # Weight for BM25 in hybrid ranking [0–1].
    # 0.0 = pure vector search
    # 1.0 = pure keyword search
    # 0.3 = 70% vector + 30% keyword (typical default)

    # ── Security ───────────────────────────────────────────
    redact_sources_before_llm: bool
    # If true, strip source material before sending to LLM.
    # WHY: Prevents leaking sensitive conversation data
    # through the extraction LLM call.

    store_sources: bool
    # If true, store source/provenance in the skill.
```

### InteractiveConfig

Configuration for interactive sessions (defined in `autoskill/interactive/config.py`).

```python
@dataclass
class InteractiveConfig:
    user_id: str              # Session user ID
    skill_scope: str          # "user" | "library" | "all"
    rewrite_mode: str         # "always" | "auto" | "never"
    extract_mode: str         # "always" | "auto" | "never"
    extract_turn_limit: int   # Extract every N turns (for "auto" mode)
    min_score: float          # Minimum retrieval score threshold
    top_k: int                # Number of skills to retrieve
    max_context_chars: int    # Maximum characters in skill context
    ingest_window: int        # Number of recent messages to use for extraction
    history_turns: int        # Number of turns to keep in session history
```

---

## Offline Document Pipeline Models

These models are used in the document-to-skills pipeline (defined in `autoskill/offline/document/models.py`).

### DocumentRecord

A normalized representation of an ingested document.

```python
@dataclass
class DocumentRecord:
    doc_id: str              # Unique document identifier
    source_type: str         # "journal_article", "technical_report", etc.
    title: str               # Document title
    authors: List[str]       # Author names
    year: int                # Publication year
    domain: str              # Subject domain ("geography", "chemistry", etc.)
    raw_text: str            # Full document text
    sections: List[DocumentSection]  # Parsed structure
    metadata: Dict[str, Any] # Additional metadata
    checksum: str            # SHA-1 hash of raw_text for deduplication
```

**Why checksum?** When re-processing a document collection, unchanged documents can be skipped. The checksum lets the system quickly identify what's new.

### DocumentSection

A section within a document, identified by its heading.

```python
@dataclass
class DocumentSection:
    heading: str     # Section title (e.g., "Methods", "Results")
    text: str        # Section content
    level: int       # Heading level (1=H1, 2=H2, etc.)
    span: TextSpan   # Byte offsets in the original document
```

### TextSpan

Byte offsets locating text within a document.

```python
@dataclass
class TextSpan:
    start: int   # Start byte offset
    end: int     # End byte offset
```

### EvidenceUnit

An atomic piece of evidence extracted from a document section.

```python
@dataclass
class EvidenceUnit:
    evidence_id: str         # Unique identifier
    doc_id: str              # Which document this came from
    claim_type: str          # Type of claim:
                             #   "workflow_step" — a procedure step
                             #   "constraint" — a rule or limitation
                             #   "validation" — a verification check
                             #   "decision_rule" — an if/then decision

    section: str             # Section heading where this was found
    span: TextSpan           # Byte offsets within the section
    normalized_claim: str    # Cleaned, normalized version of the claim
    verbatim_excerpt: str    # Original text as it appears in the document
    method_family: str       # Research method ("field-study", "experiment")
    task_family: str         # Task type ("classification", "synthesis")
    confidence: float        # Extraction confidence [0–1]
    provenance: ProvenanceRecord  # Full source tracking
```

### CapabilitySpec

A reusable executable capability synthesized from multiple evidence units.

```python
@dataclass
class CapabilitySpec:
    capability_id: str           # Unique identifier
    title: str                   # Capability name
    domain: str                  # Subject domain
    task_family: str             # Task classification
    method_family: str           # Method classification
    stage: str                   # Workflow stage:
                                 #   "planning", "analysis", "execution"
    workflow_steps: List[str]    # Ordered procedural steps
    decision_rules: List[str]   # Decision logic
    constraints: List[str]      # Rules and limitations
    failure_modes: List[str]    # What can go wrong
    output_contract: Dict[str, Any]  # Expected output schema
```

**Why separate from EvidenceUnit?** Evidence units are raw observations. Capabilities are synthesized procedures that combine multiple evidence units into a coherent workflow. Think of it as: evidence units are atoms, capabilities are molecules.

### SkillSpec

A compiled skill ready for persistence, bridging CapabilitySpec and the Skill model.

```python
@dataclass
class SkillSpec:
    skill_id: str
    # Links to CapabilitySpec
    # Contains compiled Skill fields ready for storage
```

### SkillLifecycle

Tracks the lifecycle state of a skill through state transitions.

```python
@dataclass
class SkillLifecycle:
    skill_id: str
    lifecycle_id: str
    state: str               # Current state:
                             #   "draft" — just created, not validated
                             #   "validated" — checked and approved
                             #   "active" — in use
                             #   "superseded" — replaced by newer version
    transitions: List[Dict]  # History of state changes with timestamps
```

---

## The SKILL.md Format

Skills are persisted as SKILL.md files — a combination of YAML frontmatter and Markdown content. This is the on-disk format used by `LocalSkillStore`.

```markdown
---
id: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
name: "Release Process"
description: "Run regression tests, canary rollout, monitor, then full rollout"
version: "1.2.0"
status: "active"
tags:
  - deployment
  - devops
triggers:
  - "How do I release?"
  - "What's the release process?"
  - "deployment steps"
examples:
  - input: "Should I deploy directly to production?"
    output: "No, always do a canary deployment first."
    notes: "Safety-first approach"
  - input: "How long should canary run?"
    output: "Minimum 30 minutes, ideally 2 hours."
    notes: "Based on team postmortem data"
---

# Release Process

Follow these steps before releasing any service:

## Steps

1. **Regression Testing**: Run the full test suite locally and in CI
2. **Canary Rollout**: Deploy to 1% of traffic and monitor for 30+ minutes
3. **Gradual Rollout**: Increase to 10%, 50%, then 100% over 2 hours
4. **Post-Deploy Monitoring**: Watch error rates and latency for 24 hours

## Constraints

- Never skip regression tests, even for "small" changes
- Always do canary first — minimum 30 minutes
- Have a rollback plan ready before starting

## Failure Modes

- Test suite fails → Fix and retest before proceeding
- Canary error rate > 2% → Immediate rollback
- Latency spike > 200ms → Pause rollout and investigate
```

**Why YAML frontmatter + Markdown?**
- YAML frontmatter is machine-readable (parsed by `agent_skill.py`)
- Markdown body is human-readable (editable in any text editor)
- This format is compatible with GitHub Pages, Jekyll, and other tools
- It's the same format used by Anthropic's Skills feature

---

## Relationships Between Models

```
DocumentRecord
    │
    │ has many
    ▼
EvidenceUnit ──────────── cluster into ──────────▶ CapabilitySpec
    │                                                    │
    │ tracks provenance                                  │ compiles to
    ▼                                                    ▼
ProvenanceRecord                                   SkillSpec
                                                        │
                                                        │ becomes
                                                        ▼
                                               Skill (core model)
                                                        │
                                          ┌─────────────┼─────────────┐
                                          │             │             │
                                          ▼             ▼             ▼
                                    SkillExample   SkillHit      SkillLifecycle
                                   (embedded)    (search result) (state tracking)
```

---

## Serialization Formats

| Context | Format | File |
|---------|--------|------|
| On-disk storage | YAML frontmatter + Markdown (SKILL.md) | `management/formats/agent_skill.py` |
| API responses | JSON | `interactive/server.py`, `examples/web_ui.py` |
| LLM extraction output | JSON (with repair fallbacks) | `management/extraction.py` |
| Document pipeline registry | JSON | `offline/document/registry.py` |
| Vector indexes | Binary (pickled numpy arrays) | `management/vectors/flat.py` |
| BM25 index | JSON | `management/stores/bm25_index.py` |

---

## Next Steps

- **[API Reference](api-reference.md)** — See how these models appear in HTTP responses
- **[Storage & SkillBank](storage-and-skillbank.md)** — See how SKILL.md files are organized on disk
- **[Data Flow](data-flow.md)** — See how data moves between these models
