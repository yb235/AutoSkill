# Data Flow

> This document traces the exact path data takes through AutoSkill — from a user typing a message to a skill being stored and retrieved. Follow each flow like a story.

---

## The Two Main Flows

AutoSkill has two primary data flows that happen in every conversation turn:

1. **Retrieve & Respond** — Find relevant skills and use them to help the LLM answer
2. **Extract & Evolve** — Learn new skills from the conversation (happens in background)

Plus one offline flow:

3. **Document → Skills Pipeline** — Convert documents into skills in batch

---

## Flow 1: Retrieve & Respond

This flow runs every time a user sends a message. Its goal: find the most relevant skills and inject them into the LLM's context.

### Step-by-Step Walkthrough

```
┌─────────────────────────────────────────────────────────────────┐
│ USER types: "How should I deploy to production?"                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Query Rewriting (Optional)                              │
│                                                                 │
│ The raw user query might be vague or context-dependent.         │
│ An LLM rewrites it into a better search query.                  │
│                                                                 │
│ Original: "How should I deploy to production?"                  │
│ Rewritten: "production deployment process release canary        │
│             rollout monitoring checklist"                        │
│                                                                 │
│ File: autoskill/interactive/rewriting.py                        │
│ Config: rewrite_mode = "always" | "auto" | "never"              │
│                                                                 │
│ WHY: Search engines work better with expanded queries.          │
│ "deploy to production" might miss a skill called                │
│ "Release Process" but the rewritten query won't.                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: Skill Retrieval (Hybrid Search)                         │
│                                                                 │
│ Search runs separately per scope:                               │
│                                                                 │
│   ┌──────────────┐        ┌──────────────┐                     │
│   │ User Scope   │        │ Library Scope│                     │
│   │ /Users/u1/   │        │ /Common/     │                     │
│   └──────┬───────┘        └──────┬───────┘                     │
│          │                       │                              │
│          ▼                       ▼                              │
│   ┌──────────────────────────────────────┐                     │
│   │ For each scope:                      │                     │
│   │  1. Vector search (semantic)         │                     │
│   │     - Embed query → 1536-dim vector  │                     │
│   │     - Find nearest skill vectors     │                     │
│   │  2. BM25 search (keyword)            │                     │
│   │     - Tokenize query → term freq     │                     │
│   │     - Score skills by term overlap   │                     │
│   │  3. Hybrid blend                     │                     │
│   │     score = (1-w) × vector + w × bm25│                     │
│   │  4. Filter by min_score threshold    │                     │
│   └──────────────┬───────────────────────┘                     │
│                  │                                              │
│                  ▼                                              │
│   Merged results: [SkillHit(skill, score), ...]                │
│                                                                 │
│ Files: autoskill/interactive/retrieval.py                       │
│        autoskill/management/stores/hybrid_rank.py               │
│        autoskill/management/stores/bm25_index.py                │
│                                                                 │
│ WHY: Per-scope search means user skills and library skills      │
│ can have different relevance thresholds. A user's personal      │
│ skill might need a lower threshold to surface.                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: Skill Selection (Optional)                              │
│                                                                 │
│ If enabled, an LLM picks which retrieved skills are actually    │
│ relevant to the query. This is a "reranking" step.              │
│                                                                 │
│ Input: 5 retrieved skills + the user query                      │
│ Output: 2 skills marked as "relevant"                           │
│                                                                 │
│ File: autoskill/interactive/selection.py                        │
│                                                                 │
│ WHY: Hybrid search is fast but imprecise. An LLM can           │
│ understand context better and filter out false positives.       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 4: Context Rendering                                       │
│                                                                 │
│ Selected skills are formatted into a markdown block:            │
│                                                                 │
│ ┌─────────────────────────────────────┐                        │
│ │ ## Available Skills                 │                        │
│ │                                     │                        │
│ │ ### Release Process (v1.2.0)        │                        │
│ │ Run regression tests, canary        │                        │
│ │ rollout, monitor, then full         │                        │
│ │ rollout. Never skip canary.         │                        │
│ │                                     │                        │
│ │ **Example:**                        │                        │
│ │ Q: Should I deploy directly?        │                        │
│ │ A: No, always canary first.         │                        │
│ └─────────────────────────────────────┘                        │
│                                                                 │
│ This block is prepended to the system prompt.                   │
│ CJK-aware truncation ensures it fits within max_context_chars.  │
│                                                                 │
│ File: autoskill/render.py                                       │
│                                                                 │
│ WHY: Markdown is readable by both humans and LLMs.              │
│ CJK-aware truncation prevents cutting Chinese/Japanese          │
│ characters mid-character (which corrupts text).                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 5: LLM Completion                                          │
│                                                                 │
│ System prompt:                                                  │
│   "You are a helpful assistant.                                 │
│    ## Available Skills                                          │
│    [rendered skills from step 4]                                │
│    Follow the most relevant skill if applicable."               │
│                                                                 │
│ User message:                                                   │
│   "How should I deploy to production?"                          │
│                                                                 │
│ → LLM generates response informed by the injected skills        │
│                                                                 │
│ File: autoskill/interactive/session.py (turn method)            │
│                                                                 │
│ WHY: This is standard RAG (Retrieval-Augmented Generation).     │
│ The LLM gets better answers because it has relevant             │
│ procedures in its context window.                               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 6: Response Delivery                                       │
│                                                                 │
│ The assistant's response is returned to the UI along with       │
│ diagnostic metadata:                                            │
│                                                                 │
│ {                                                               │
│   "assistant_reply": "Here's the deployment process: ...",      │
│   "retrieval": {                                                │
│     "original_query": "How should I deploy to production?",     │
│     "rewritten_query": "production deployment process ...",     │
│     "hits_user": [...],                                         │
│     "hits_library": [...],                                      │
│     "selected_for_context": [...]                               │
│   }                                                             │
│ }                                                               │
│                                                                 │
│ WHY: Returning diagnostics lets the Web UI show what skills     │
│ were found and why, making the system transparent.              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Flow 2: Extract & Evolve

This flow runs **in the background** after the assistant responds. Its goal: learn new skills from the conversation.

### Step-by-Step Walkthrough

```
┌─────────────────────────────────────────────────────────────────┐
│ TRIGGER: Extraction Gating                                      │
│                                                                 │
│ Extraction doesn't happen on every turn. The gating logic       │
│ decides when:                                                   │
│                                                                 │
│ extract_mode = "always"  → Extract every turn                   │
│ extract_mode = "auto"    → Extract every N turns or on          │
│                            topic change                         │
│ extract_mode = "never"   → Only on manual /extract command      │
│                                                                 │
│ File: autoskill/interactive/gating.py                           │
│                                                                 │
│ WHY: Extracting on every turn is wasteful (costs LLM tokens).   │
│ Waiting for topic boundaries or periodic intervals balances     │
│ cost vs. completeness.                                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ (background thread)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Skill Extraction                                        │
│                                                                 │
│ The last N messages (ingest_window) are sent to the LLM with    │
│ a structured extraction prompt:                                 │
│                                                                 │
│ "Analyze this conversation and extract reusable skills.         │
│  Return JSON: [{name, description, instructions, triggers,      │
│  examples, tags}]"                                              │
│                                                                 │
│ The LLM response is parsed:                                     │
│   1. Try direct JSON parse                                      │
│   2. Try extracting JSON from markdown code blocks              │
│   3. Try recovering key fields from free text                   │
│   4. Ask LLM to repair into valid JSON                          │
│   5. Skip if all attempts fail                                  │
│                                                                 │
│ Output: List[SkillCandidate]                                    │
│                                                                 │
│ File: autoskill/management/extraction.py                        │
│                                                                 │
│ WHY: LLMs don't always return valid JSON. The multi-step        │
│ parsing strategy handles real-world LLM output gracefully.      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: Deduplication Check                                     │
│                                                                 │
│ For each SkillCandidate:                                        │
│   1. Search existing skills by vector similarity                │
│   2. Compare names (fuzzy string matching)                      │
│   3. Compare tags and triggers (signal overlap)                 │
│   4. Calculate combined similarity score                        │
│                                                                 │
│ If similarity > dedupe_threshold → candidate is a duplicate     │
│                                                                 │
│ File: autoskill/management/maintenance.py                       │
│                                                                 │
│ WHY: Without deduplication, the skill bank fills up with        │
│ slight variations of the same skill ("Release Process",         │
│ "Deployment Steps", "Production Rollout").                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: Maintenance Decision                                    │
│                                                                 │
│ For each candidate, decide:                                     │
│                                                                 │
│   ADD    → New skill, no similar existing skill found           │
│   MERGE  → Similar skill exists, combine knowledge              │
│   DISCARD → Too similar or low quality                          │
│                                                                 │
│ Decision can be made by:                                        │
│   - Heuristic rules (fast, no LLM needed)                      │
│   - LLM judgment (more accurate, costs tokens)                  │
│                                                                 │
│ Config: maintenance_strategy = "heuristic" | "llm"              │
│                                                                 │
│ WHY: Heuristic mode is good for high-volume scenarios           │
│ (saves money). LLM mode is better for quality-sensitive         │
│ deployments where merge decisions matter.                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
┌──────────────────────┐ ┌───────────────────────┐
│ ADD (new skill)      │ │ MERGE (existing skill) │
│                      │ │                        │
│ 1. Generate SKILL.md │ │ 1. Blend instructions  │
│ 2. Assign version    │ │ 2. Merge triggers/tags │
│    1.0.0             │ │ 3. Add new examples    │
│ 3. Persist to store  │ │ 4. Bump version        │
│                      │ │    (1.0.0 → 1.0.1)    │
│                      │ │ 5. Save snapshot       │
│                      │ │ 6. Update store        │
└──────────┬───────────┘ └───────────┬────────────┘
           │                         │
           └────────────┬────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 4: Persistence                                             │
│                                                                 │
│ The skill is saved to the SkillBank:                            │
│                                                                 │
│ /SkillBank/Users/u1/release-process/                            │
│ ├── SKILL.md          # YAML frontmatter + instructions         │
│ └── (optional files)  # Scripts, templates, etc.                │
│                                                                 │
│ The vector index is updated for future searches.                │
│ The BM25 index is updated for keyword matching.                 │
│                                                                 │
│ File: autoskill/management/stores/local.py                      │
│       autoskill/management/formats/agent_skill.py               │
│                                                                 │
│ WHY: Filesystem storage is human-readable and version-control   │
│ friendly. You can browse skills in a file manager, edit them    │
│ by hand, and commit them to git.                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Flow 3: Document → Skills Pipeline (Offline)

This flow converts raw documents (research papers, documentation, guides) into skills in five stages.

```
┌─────────────────────────────────────────────────────────────────┐
│ INPUT: Raw document (Markdown, PDF text, etc.)                  │
│ "# Field Classification Method                                  │
│  1. Collect soil samples at 10m intervals                       │
│  2. Photograph each sample with GPS tag                         │
│  3. Run pH test using litmus strips                             │
│  4. Record results in field journal..."                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 1: Ingest Document → DocumentRecord                       │
│                                                                 │
│ - Parse raw text into structured sections (by headings)         │
│ - Assign section levels (H1, H2, H3)                           │
│ - Calculate checksum for deduplication                          │
│ - Check registry: skip if already ingested                      │
│                                                                 │
│ Output: DocumentRecord with title, authors, sections            │
│                                                                 │
│ File: autoskill/offline/document/ingest.py                      │
│                                                                 │
│ WHY: Structured sections let the next stage focus on specific   │
│ parts of the document rather than processing it all at once.    │
│ Checksum-based dedup prevents re-processing unchanged docs.     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 2: Extract Evidence → List[EvidenceUnit]                  │
│                                                                 │
│ For each section, an LLM identifies atomic claims:              │
│                                                                 │
│ Evidence units extracted:                                       │
│   - "Collect soil samples at 10m intervals" (workflow_step)     │
│   - "Must use GPS-tagged photographs" (constraint)              │
│   - "pH test requires litmus strips" (workflow_step)            │
│   - "If pH > 8, mark as alkaline" (decision_rule)              │
│                                                                 │
│ Each unit includes: claim type, confidence, byte offsets,       │
│ verbatim excerpt, method/task family classification             │
│                                                                 │
│ File: autoskill/offline/document/extractor.py                   │
│                                                                 │
│ WHY: Breaking a document into atomic evidence units makes       │
│ each unit independently reusable and combinable.                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 3: Induce Capabilities → List[CapabilitySpec]             │
│                                                                 │
│ Cluster related evidence units into capabilities:               │
│                                                                 │
│ Capability: "Field Soil Classification"                         │
│   workflow_steps:                                               │
│     1. "Collect soil samples at 10m intervals"                  │
│     2. "Photograph each sample with GPS tag"                    │
│     3. "Run pH test using litmus strips"                        │
│     4. "Record results in field journal"                        │
│   decision_rules:                                               │
│     - "If pH > 8, mark as alkaline"                             │
│   constraints:                                                  │
│     - "Must use GPS-tagged photographs"                         │
│                                                                 │
│ File: autoskill/offline/document/inducer.py                     │
│                                                                 │
│ WHY: Individual evidence units aren't useful alone. Clustering  │
│ them into coherent capabilities creates actionable procedures.  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 4: Compile Skills → List[Skill]                           │
│                                                                 │
│ Each CapabilitySpec is compiled into a Skill object:            │
│   - name: "Field Soil Classification"                           │
│   - description: "Classify soil samples using pH testing..."    │
│   - instructions: (full step-by-step procedure)                 │
│   - triggers: ["soil classification", "pH test", "field work"]  │
│   - examples: (generated from evidence)                         │
│   - tags: ["geography", "field-study"]                          │
│                                                                 │
│ A SKILL.md artifact is also generated.                          │
│                                                                 │
│ File: autoskill/offline/document/compiler.py                    │
│                                                                 │
│ WHY: The Skill format is what the retrieval system              │
│ understands. Compiling turns academic knowledge into            │
│ actionable instructions an LLM can follow.                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ STAGE 5: Register Versions                                      │
│                                                                 │
│ - Deduplicate against existing skills in the store              │
│ - If similar skill exists: merge and bump version               │
│ - If new: create with version 1.0.0                             │
│ - Track lifecycle state: draft → validated → active             │
│ - Update the DocumentRegistry (provenance tracking)             │
│                                                                 │
│ File: autoskill/offline/document/versioning.py                  │
│       autoskill/offline/document/registry.py                    │
│                                                                 │
│ WHY: Versioning ensures that re-processing an updated           │
│ document evolves existing skills rather than creating           │
│ duplicates. The registry maintains full provenance.             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Flow Summary: What Happens in a Single Chat Turn

Here's the complete sequence for one user message:

```
1. User sends message
2. [Foreground] Query rewriting (optional LLM call)
3. [Foreground] Skill retrieval (vector + BM25 search)
4. [Foreground] Skill selection (optional LLM call)
5. [Foreground] Context rendering (format skills as markdown)
6. [Foreground] LLM completion (generate response)
7. [Foreground] Return response + diagnostics to UI
8. [Background] Extraction gating (should we extract?)
9. [Background] Skill extraction (LLM call to identify skills)
10. [Background] Deduplication (search for similar skills)
11. [Background] Maintenance decision (add/merge/discard)
12. [Background] Persistence (save to SkillBank)
13. [Background] Usage tracking (was the skill helpful?)
```

**Steps 1–7** happen in ~2–5 seconds (one LLM call for response).
**Steps 8–13** happen asynchronously over ~5–15 seconds without blocking the user.

---

## Next Steps

- **[Data Schemas](data-schemas.md)** — See the exact data structures used in each flow step
- **[API Reference](api-reference.md)** — See the HTTP endpoints that trigger these flows
- **[Storage & SkillBank](storage-and-skillbank.md)** — See where and how skills are persisted
