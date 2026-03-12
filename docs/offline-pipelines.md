# Offline Pipelines

> This document explains how AutoSkill processes documents, conversation logs, and agent trajectories in batch mode to extract skills — no live conversation needed.

---

## Why Offline Pipelines?

Not all knowledge comes from live conversations. Organizations have:

- **Research papers** with validated procedures and best practices
- **Documentation** with step-by-step instructions
- **Conversation logs** from previous chat sessions (e.g., OpenAI exports)
- **Agent trajectories** — sequences of actions an agent took to solve problems

Offline pipelines convert all of these into skills that can be retrieved during future conversations.

**Rationale**: A comprehensive skill bank should draw from all available knowledge sources. Live extraction alone would require every procedure to be "rediscovered" in conversation. Offline pipelines pre-populate the skill bank with existing organizational knowledge.

---

## Document Pipeline (5 Stages)

The document pipeline (`autoskill/offline/document/`) converts raw documents into skills through five progressive stages. Each stage refines the data into a more usable form.

### Pipeline Overview

```
Raw Document (Markdown, PDF text, etc.)
        │
        ▼
┌──────────────────────────────────────┐
│ Stage 1: INGEST                      │  "Normalize the document"
│ → DocumentRecord                     │
├──────────────────────────────────────┤
│ Stage 2: EXTRACT EVIDENCE            │  "Find atomic facts"
│ → List[EvidenceUnit]                 │
├──────────────────────────────────────┤
│ Stage 3: INDUCE CAPABILITIES         │  "Cluster facts into procedures"
│ → List[CapabilitySpec]               │
├──────────────────────────────────────┤
│ Stage 4: COMPILE SKILLS              │  "Turn procedures into skills"
│ → List[Skill]                        │
├──────────────────────────────────────┤
│ Stage 5: REGISTER VERSIONS           │  "Persist and deduplicate"
│ → VersionRegistrationResult          │
└──────────────────────────────────────┘
        │
        ▼
SkillBank (filesystem)
```

### Stage 1: Ingest Document

**File**: `autoskill/offline/document/ingest.py`

**Input**: Raw text + metadata (title, authors, domain, year)

**Output**: `DocumentRecord`

**What it does:**
1. Parse raw text into structured sections by detecting headings
2. Assign heading levels (H1, H2, H3, etc.)
3. Calculate byte-offset spans for each section
4. Compute SHA-1 checksum of the raw text
5. Check the `DocumentRegistry` — skip if already ingested

```python
# Example
record = ingest_document(
    raw_text="# Methods\n## Field Collection\n...",
    title="Soil Classification Study",
    authors=["Smith, J.", "Lee, K."],
    domain="geography",
    year=2024
)

# record.sections = [
#     DocumentSection(heading="Methods", level=1, text="..."),
#     DocumentSection(heading="Field Collection", level=2, text="...")
# ]
# record.checksum = "a3f2b4c5..."
```

**Rationale for checksum deduplication**: When processing a document collection, some documents might not have changed. The checksum lets the pipeline skip unchanged documents, making incremental updates efficient.

### Stage 2: Extract Evidence

**File**: `autoskill/offline/document/extractor.py`

**Input**: `DocumentRecord`

**Output**: `List[EvidenceUnit]`

**What it does:**
1. For each section, send the text to an LLM with a structured extraction prompt
2. The LLM identifies atomic claims of specific types:
   - `workflow_step` — A procedure step ("Collect samples at 10m intervals")
   - `constraint` — A rule or limitation ("Must use GPS-tagged photos")
   - `decision_rule` — An if/then decision ("If pH > 8, mark as alkaline")
   - `validation` — A verification check ("Confirm sample count matches plan")
3. Each claim gets a confidence score, byte-offset span, and provenance

```python
# Example extracted evidence units:
EvidenceUnit(
    claim_type="workflow_step",
    normalized_claim="Collect soil samples at 10-meter intervals along the transect",
    verbatim_excerpt="Samples were collected at 10m intervals...",
    method_family="field-study",
    task_family="classification",
    confidence=0.92
)

EvidenceUnit(
    claim_type="constraint", 
    normalized_claim="All photographs must include GPS coordinates",
    verbatim_excerpt="Each sample was photographed with GPS tags...",
    confidence=0.88
)
```

**Rationale for atomic claims**: Breaking documents into individual claims makes each piece independently reusable. A "constraint" from one document can be combined with "workflow steps" from another to create a comprehensive skill.

### Stage 3: Induce Capabilities

**File**: `autoskill/offline/document/inducer.py`

**Input**: `List[EvidenceUnit]`

**Output**: `List[CapabilitySpec]`

**What it does:**
1. Cluster related evidence units (by task family, method family, semantic similarity)
2. Within each cluster, synthesize a coherent capability:
   - Order workflow steps into a logical sequence
   - Extract decision rules
   - Compile constraints
   - Identify failure modes
3. Define an output contract (what the capability produces)

```python
# Example capability:
CapabilitySpec(
    title="Field Soil Classification",
    domain="geography",
    task_family="classification",
    stage="execution",
    workflow_steps=[
        "Collect soil samples at 10-meter intervals",
        "Photograph each sample with GPS coordinates",
        "Run pH test using litmus strips",
        "Record results in field journal with sample ID"
    ],
    decision_rules=[
        "If pH > 8, classify as alkaline",
        "If pH < 6, classify as acidic",
        "If 6 ≤ pH ≤ 8, classify as neutral"
    ],
    constraints=[
        "Must use GPS-tagged photographs",
        "Minimum 3 samples per transect point"
    ],
    failure_modes=[
        "GPS signal loss in dense canopy → use manual coordinates",
        "Contaminated litmus strips → discard and re-test"
    ]
)
```

**Rationale**: Evidence units are atoms — useful facts but not actionable alone. Capabilities are molecules — synthesized procedures that combine multiple facts into something an agent can follow step by step.

### Stage 4: Compile Skills

**File**: `autoskill/offline/document/compiler.py`

**Input**: `List[CapabilitySpec]`

**Output**: `List[Skill]` (plus `SkillSpec` intermediaries)

**What it does:**
1. Convert each `CapabilitySpec` into a `Skill` object:
   - Generate a descriptive name and description
   - Format workflow steps into instructions
   - Create triggers from key terms
   - Generate example input/output pairs
   - Add tags from domain and method family
2. Render a SKILL.md artifact

```python
# Example compiled skill:
Skill(
    name="Field Soil Classification",
    description="Classify soil samples using pH testing in field conditions",
    instructions="""
Follow these steps for field soil classification:

1. Collect soil samples at 10-meter intervals along the transect
2. Photograph each sample with GPS coordinates
3. Run pH test using litmus strips
4. Record results in field journal with sample ID

## Decision Rules
- If pH > 8 → alkaline
- If pH < 6 → acidic  
- If 6 ≤ pH ≤ 8 → neutral

## Constraints
- Must use GPS-tagged photographs
- Minimum 3 samples per transect point
""",
    triggers=["soil classification", "pH test", "field collection"],
    tags=["geography", "field-study", "classification"]
)
```

**Rationale**: The Skill format is what the retrieval system understands. This compilation step bridges the gap between academic/documentary knowledge and actionable agent instructions.

### Stage 5: Register Versions

**File**: `autoskill/offline/document/versioning.py`

**Input**: `List[Skill]`

**Output**: `VersionRegistrationResult`

**What it does:**
1. For each compiled skill, check the store for similar existing skills
2. If similar: merge and bump version (same logic as interactive maintenance)
3. If new: create with version 1.0.0
4. Track lifecycle state: draft → validated → active
5. Update the `DocumentRegistry` with provenance links

**Rationale**: Version registration ensures that re-processing an updated document evolves existing skills rather than creating duplicates. The lifecycle states allow skills to go through a validation step before being used in production.

### Running the Pipeline

**Programmatic:**
```python
from autoskill import AutoSkill, AutoSkillConfig
from autoskill.offline.document.pipeline import build_default_document_pipeline

config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-4"},
    embeddings={"provider": "openai", "model": "text-embedding-3-large"},
    store={"provider": "local", "path": "./SkillBank"}
)
sdk = AutoSkill(config)

pipeline = build_default_document_pipeline(sdk=sdk)

result = pipeline.build(
    user_id="u1",
    data="# Field Methods\n1. Collect samples...",
    title="Soil Classification Study",
    domain="geography"
)

print(f"Documents: {len(result.ingest.documents)}")
print(f"Evidence: {len(result.evidence.evidence_units)}")
print(f"Capabilities: {len(result.capabilities.capability_specs)}")
print(f"Skills: {len(result.skills.skill_specs)}")
```

**CLI:**
```bash
python -m autoskill offline document build \
    --file paper.md \
    --title "Soil Classification" \
    --domain geography \
    --user-id u1 \
    --store-dir ./SkillBank
```

---

## Document Registry

**File**: `autoskill/offline/document/registry.py`

The `DocumentRegistry` tracks everything the pipeline has processed:

```
Registry tracks:
├── Documents: which documents have been ingested (by checksum)
├── Evidence: which evidence units exist (by evidence_id)
├── Capabilities: which capabilities were induced (by capability_id)
├── Skills: which skills were compiled (by skill_id)
└── Provenance: which document → evidence → capability → skill
```

**Features:**
- **Checksum deduplication**: Skip already-ingested documents
- **Incremental builds**: Only process new or changed documents
- **Manifest reporting**: Count entities in each state
- **Provenance tracking**: Trace any skill back to its source document

**Rationale**: The registry is essential for production use where document collections are processed repeatedly. Without it, every run would reprocess everything from scratch.

---

## Conversation Pipeline

**Directory**: `autoskill/offline/conversation/`

Converts conversation logs (JSONL/JSON format) into skills.

### Input Format

```json
{"messages": [
    {"role": "user", "content": "How do I deploy?"},
    {"role": "assistant", "content": "Run tests, canary, rollout."}
]}
```

### How It Works

```
JSONL file → Load conversations → For each conversation:
  1. Parse messages
  2. Send to LLM with extraction prompt
  3. Get SkillCandidates
  4. Run through maintenance (dedupe/merge)
  5. Persist to SkillBank
```

### CLI

```bash
python -m autoskill offline conversation extract \
    --file conversations.jsonl \
    --user-id u1 \
    --store-dir ./SkillBank
```

**Rationale**: Many organizations have historical conversation logs (e.g., exported from ChatGPT, Slack bots, or internal tools). This pipeline lets them mine existing conversations for skills without replaying them.

---

## Trajectory Pipeline

**Directory**: `autoskill/offline/trajectory/`

Converts agent action trajectories into skills.

### What's a Trajectory?

A trajectory is a sequence of actions an agent took to accomplish a task:

```json
{
    "task": "File a bug report for login timeout",
    "steps": [
        {"action": "open_browser", "target": "jira.example.com"},
        {"action": "click", "target": "Create Issue"},
        {"action": "fill", "field": "Summary", "value": "Login timeout..."},
        {"action": "select", "field": "Priority", "value": "High"},
        {"action": "click", "target": "Submit"}
    ]
}
```

### How It Works

Similar to the conversation pipeline but specialized for action sequences:

1. Load trajectory files (JSONL/JSON)
2. Format action sequences as conversation-like text
3. Send to LLM for skill extraction
4. Run through maintenance and persist

**Rationale**: Agent-based automation (like web automation or API orchestration) produces valuable procedural knowledge. Trajectories capture not just what was said, but what was done.

---

## OpenAI Conversation Import

The SDK provides a built-in method for importing OpenAI conversation exports:

```python
sdk.import_openai_conversations(
    user_id="u1",
    file_path="~/Downloads/openai_conversations.json",
    metadata={"channel": "openai_import"},
    hint="Focus on coding patterns"
)
```

This handles the specific format of OpenAI's data export feature and automatically extracts skills from all conversations.

---

## Comparing the Pipelines

| Feature | Document Pipeline | Conversation Pipeline | Trajectory Pipeline |
|---------|------------------|----------------------|---------------------|
| **Input** | Structured documents | Chat logs (JSONL) | Action sequences (JSONL) |
| **Stages** | 5 (ingest → evidence → capability → skill → version) | 3 (load → extract → persist) |  3 (load → extract → persist) |
| **LLM Calls** | Multiple per document | One per conversation | One per trajectory |
| **Registry** | Full provenance tracking | Basic | Basic |
| **Best For** | Research papers, documentation | Historical chats | Agent automation logs |

---

## Next Steps

- **[Data Schemas](data-schemas.md)** — See the data models used in the document pipeline
- **[Storage & SkillBank](storage-and-skillbank.md)** — See where pipeline output is stored
- **[Data Flow](data-flow.md)** — See the full 5-stage pipeline flow diagram
