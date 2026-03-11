# Storage & SkillBank

> This document explains how skills are stored on disk, the SkillBank directory layout, the SKILL.md file format, vector indexes, and version history management.

---

## SkillBank Directory Layout

The default storage location is `./SkillBank/`. Here's the full structure:

```
SkillBank/
├── Users/                           # Per-user skill directories
│   ├── u1/                          # User "u1"
│   │   ├── release-process/         # One directory per skill
│   │   │   ├── SKILL.md            # Skill definition (YAML + Markdown)
│   │   │   └── deploy-script.sh    # Optional bundled file
│   │   ├── monitoring-setup/
│   │   │   └── SKILL.md
│   │   └── code-review-checklist/
│   │       └── SKILL.md
│   └── u2/                          # User "u2"
│       └── data-migration/
│           └── SKILL.md
│
├── Common/                          # Shared library skills
│   └── anthropics-skill/            # Library name
│       ├── create-pdf/
│       │   ├── SKILL.md
│       │   └── pdf_template.py
│       ├── create-docx/
│       │   ├── SKILL.md
│       │   └── docx_template.py
│       ├── create-pptx/
│       │   └── SKILL.md
│       ├── web-automation/
│       │   └── SKILL.md
│       └── data-analysis/
│           └── SKILL.md
│
├── vectors/                         # Persistent vector indexes
│   ├── <embeddings_hash>.bin       # Vector data (numpy arrays)
│   └── <embeddings_hash>.meta      # Vector metadata
│
└── web_sessions/                    # Web UI session data
    ├── session-abc123.json
    └── session-def456.json
```

**Rationale for this layout:**

1. **Users/**: Per-user directories enforce isolation. User u1 cannot access u2's skills even if there's a bug in the search code — they're in separate directories.

2. **Common/**: A shared library that all users can access. This is where pre-packaged skills live (e.g., the built-in `anthropics-skill` collection).

3. **One directory per skill**: Skills are self-contained — the SKILL.md plus any bundled files live together. You can zip up a skill directory and share it.

4. **Human-readable names**: Directory names are derived from skill names (slugified), making it easy to browse with a file manager.

---

## The SKILL.md Format

Every skill is stored as a `SKILL.md` file — a combination of YAML frontmatter and Markdown content.

### Full Example

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
  - production
triggers:
  - "How do I release?"
  - "What's the release process?"
  - "deployment steps"
  - "production rollout"
examples:
  - input: "Should I deploy directly to production?"
    output: "No, always do a canary deployment first with at least 1% of traffic."
    notes: "Safety-first approach"
  - input: "How long should the canary run?"
    output: "Minimum 30 minutes, ideally 2 hours with active monitoring."
    notes: "Based on post-mortem data from previous incidents"
files:
  deploy-script.sh: |
    #!/bin/bash
    echo "Running regression tests..."
    npm test
    echo "Starting canary deployment..."
---

# Release Process

Follow these steps before releasing any service to production.

## Steps

1. **Regression Testing**: Run the full test suite locally and in CI. All tests must pass.
2. **Canary Rollout**: Deploy to 1% of traffic. Monitor error rates and latency for 30+ minutes.
3. **Gradual Rollout**: Increase to 10%, then 50%, then 100% over 2 hours.
4. **Post-Deploy Monitoring**: Watch error rates, latency, and user reports for 24 hours.

## Constraints

- Never skip regression tests, even for "small" changes
- Always do canary first — minimum 30 minutes observation
- Have a rollback plan ready before starting deployment

## Failure Modes

- **Test suite fails** → Fix issues and retest before proceeding
- **Canary error rate > 2%** → Immediate rollback
- **Latency spike > 200ms** → Pause rollout and investigate
- **User reports increase** → Investigate before continuing rollout
```

### Format Anatomy

```
┌───────────────────────────────────────┐
│ YAML Frontmatter (between --- lines)  │
│                                       │
│ Machine-readable metadata:            │
│ • id, name, description, version      │
│ • status (active/archived)            │
│ • tags, triggers (arrays)             │
│ • examples (array of objects)         │
│ • files (map of filename → content)   │
└───────────────────────────────────────┘
┌───────────────────────────────────────┐
│ Markdown Body                         │
│                                       │
│ Human-readable instructions:          │
│ • Heading with skill name             │
│ • Step-by-step procedure              │
│ • Constraints and rules               │
│ • Failure modes and edge cases        │
└───────────────────────────────────────┘
```

**Rationale for YAML frontmatter + Markdown:**

1. **YAML is parseable** — AutoSkill reads the frontmatter to populate Skill objects
2. **Markdown is readable** — Humans can open SKILL.md in any text editor and understand it
3. **Industry standard** — This is the same format used by Jekyll, Hugo, Anthropic Skills, and many documentation tools
4. **Git-friendly** — SKILL.md files diff cleanly in version control
5. **Editable** — Users can edit skills in the Web UI's SKILL.md editor or directly in a text editor

**File**: `autoskill/management/formats/agent_skill.py`

---

## How Skills Are Parsed and Rendered

### Rendering (Skill → SKILL.md)

```python
# autoskill/management/formats/agent_skill.py

def render_agent_skill_md(skill: Skill) -> str:
    """Convert a Skill object to a SKILL.md string."""
    
    # 1. Build YAML frontmatter dict
    frontmatter = {
        "id": skill.id,
        "name": skill.name,
        "description": skill.description,
        "version": skill.version,
        "status": skill.status.value,
        "tags": skill.tags,
        "triggers": skill.triggers,
        "examples": [asdict(e) for e in skill.examples],
    }
    if skill.files:
        frontmatter["files"] = skill.files
    
    # 2. Serialize to YAML + Markdown
    return f"---\n{yaml_dump(frontmatter)}---\n\n{skill.instructions}"
```

### Parsing (SKILL.md → Skill)

```python
def parse_agent_skill_md(content: str) -> Skill:
    """Parse a SKILL.md string into a Skill object."""
    
    # 1. Split frontmatter from body
    frontmatter, body = split_yaml_frontmatter(content)
    
    # 2. Parse YAML frontmatter
    meta = yaml_load(frontmatter)
    
    # 3. Build Skill object
    return Skill(
        id=meta["id"],
        name=meta["name"],
        description=meta["description"],
        instructions=body.strip(),
        version=meta.get("version", "1.0.0"),
        status=SkillStatus(meta.get("status", "active")),
        tags=meta.get("tags", []),
        triggers=meta.get("triggers", []),
        examples=[SkillExample(**e) for e in meta.get("examples", [])],
        files=meta.get("files", {}),
        ...
    )
```

---

## Vector Index Storage

Skills are searchable by vector similarity. The vector index is stored alongside the SkillBank:

```
SkillBank/vectors/
├── abc123def456.bin      # Serialized vector data (numpy array)
└── abc123def456.meta     # Metadata: skill_id → vector index mapping
```

The filename hash is derived from the embedding configuration, so different embedding models get separate indexes.

### How the Index Works

```
                ┌──────────────────────┐
User query ──▶ │ Embedding Model      │ ──▶ Query vector [1536 floats]
                └──────────────────────┘
                                              │
                                              ▼
                ┌──────────────────────────────────────┐
                │ Vector Index                          │
                │                                      │
                │ Skill "Release Process"  → [0.1, ...] │
                │ Skill "Monitoring Setup" → [0.3, ...] │
                │ Skill "Code Review"      → [0.2, ...] │
                │                                      │
                │ Cosine similarity search:             │
                │  1. Release Process    (score: 0.92)  │
                │  2. Monitoring Setup   (score: 0.71)  │
                │  3. Code Review        (score: 0.34)  │
                └──────────────────────────────────────┘
```

**Rationale for file-based vector storage**: Keeps the zero-dependency promise. No database server needed. Vector indexes are small (a few MB for thousands of skills) and load fast from disk.

---

## BM25 Index

In addition to vector search, AutoSkill maintains a BM25 keyword index:

```
SkillBank/
└── (BM25 index is stored in-memory and rebuilt from SKILL.md files on startup)
```

The BM25 index tokenizes skill names, descriptions, triggers, and tags into a term-frequency index. When a user searches, both the BM25 score and the vector score are blended:

```
hybrid_score = (1 - bm25_weight) × vector_score + bm25_weight × bm25_score
```

**Rationale**: See [Design Patterns → Hybrid Retrieval](design-patterns.md) for why both search methods are needed.

**File**: `autoskill/management/stores/bm25_index.py`

---

## Version History

Each skill can maintain up to 30 historical snapshots in its metadata:

```python
skill.metadata["_autoskill_version_history"] = [
    {
        "version": "1.0.0",
        "snapshot": {
            "name": "Release Process",
            "description": "Run tests and deploy",
            "instructions": "1. Run tests\n2. Deploy",
            "triggers": ["deploy", "release"],
            "tags": ["devops"],
            "examples": []
        }
    },
    {
        "version": "1.0.1",
        "snapshot": {
            "name": "Release Process",
            "description": "Run tests, canary, then deploy",
            "instructions": "1. Run tests\n2. Canary\n3. Deploy",
            "triggers": ["deploy", "release", "canary"],
            "tags": ["devops"],
            "examples": [{"input": "...", "output": "..."}]
        }
    }
]
```

### Operations

| Operation | Method | Description |
|-----------|--------|-------------|
| **Save snapshot** | `push_skill_snapshot(skill)` | Save current state before modifying |
| **Rollback** | `pop_skill_snapshot(skill)` | Restore to previous version |
| **History pruning** | Automatic | FIFO — oldest snapshots removed when limit (30) exceeded |

**Rationale**: Version history makes skills auditable and recoverable. If a bad merge corrupts a skill, you can rollback to the previous version without losing the original knowledge.

**File**: `autoskill/interactive/skill_versions.py`

---

## Skill Identity and Deduplication

Each skill has multiple identity layers:

```
┌─────────────────────────────────────────────────────┐
│ Identity Layer 1: UUID                               │
│                                                     │
│ Primary key: "a1b2c3d4-e5f6-7890-abcd-ef1234567890"│
│ Used in: file paths, API references                 │
│                                                     │
│ Generated: UUID4 for new skills, deterministic hash │
│ for imported skills                                 │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Identity Layer 2: Description Norm                   │
│                                                     │
│ Normalized description for dedup comparison:         │
│ "release process run regression tests canary..."     │
│                                                     │
│ Generated by: normalize and lowercase the description│
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Identity Layer 3: Identity Hash                      │
│                                                     │
│ SHA-1 of the normalized description:                 │
│ "7a3f2b4c5d6e..."                                   │
│                                                     │
│ Used for: fast dedup lookup without full text compare│
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Identity Layer 4: User Scope                         │
│                                                     │
│ user_id determines ownership:                        │
│ "u1" → personal, "library:common" → shared          │
│                                                     │
│ Dedup only happens within the same scope             │
└─────────────────────────────────────────────────────┘
```

**Rationale**: Multiple identity layers serve different purposes — UUIDs for unique references, description norms for semantic dedup, hashes for efficient comparison, and scope for isolation.

**File**: `autoskill/management/identity.py`

---

## Bootstrap Process

When AutoSkill starts up, a bootstrap process runs to ensure the SkillBank is in a consistent state:

```
Bootstrap on startup:
  1. Normalize skill IDs (ensure all skills have valid UUIDs)
  2. Auto-import skill directories (if configured)
  3. Rebuild BM25 index from SKILL.md files
  4. Load vector indexes from disk
```

Configuration:
```bash
AUTOSKILL_AUTO_NORMALIZE_IDS=1          # Fix any invalid IDs
AUTOSKILL_AUTO_IMPORT_DIRS=./skills     # Import from directory
AUTOSKILL_AUTO_IMPORT_SCOPE=common      # Import to library scope
AUTOSKILL_AUTO_IMPORT_OVERWRITE=0       # Don't overwrite existing
```

**Rationale**: Bootstrap ensures a clean state on every startup. Auto-import lets teams distribute skill libraries through file syncing or git, and have them automatically loaded.

**File**: `autoskill/management/bootstrap.py`

---

## Importing External Skills

AutoSkill can import skill directories from external sources:

```python
# Import a directory of SKILL.md files
from autoskill.management.importer import import_skills_from_directory

import_skills_from_directory(
    store=sdk.store,
    directory="./shared-skills/",
    user_id="library:common",       # Import as library skills
    overwrite=False,                # Don't overwrite existing
    include_files=True,             # Include bundled files
    max_depth=6                     # Search up to 6 levels deep
)
```

The importer:
1. Recursively scans the directory for SKILL.md files
2. Parses each SKILL.md into a Skill object
3. Generates a deterministic ID from (scope, owner, path) for reproducibility
4. Upserts into the store (respecting overwrite flag)

**Rationale**: Importing makes it easy to share skills between teams or environments. Export from one AutoSkill instance, commit to git, and import into another.

**File**: `autoskill/management/importer.py`

---

## Next Steps

- **[Data Schemas](data-schemas.md)** — See the Skill data model in detail
- **[Agent & Skill Architecture](agent-and-skill-architecture.md)** — Understand skill lifecycle and evolution
- **[Configuration Reference](configuration.md)** — See storage configuration options
