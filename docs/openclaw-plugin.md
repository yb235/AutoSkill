# OpenClaw Plugin

> This document explains how AutoSkill integrates with the OpenClaw agent framework through a dedicated plugin that provides skill-augmented agent interactions.

---

## What Is OpenClaw?

OpenClaw is a Claude-compatible agent framework that manages AI agent conversations. The AutoSkill OpenClaw Plugin adds automatic skill retrieval and extraction to OpenClaw's agent interactions.

**Location**: `OpenClaw-Plugin/` directory

---

## Plugin Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│ OpenClaw Agent Framework                                         │
│                                                                  │
│  Agent ←──── Main Turn ────▶ LLM                                │
│                                                                  │
└────────────────────┬─────────────────────────────────────────────┘
                     │ intercepted by
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ AutoSkill OpenClaw Plugin                                        │
│                                                                  │
│  ┌─────────────────────┐                                        │
│  │ UpstreamChatProxy   │  Intercepts LLM requests               │
│  │ (HTTP proxy)        │  Injects skills into context           │
│  └─────────┬───────────┘                                        │
│            │                                                     │
│  ┌─────────▼───────────┐                                        │
│  │ MainTurnState       │  Tracks agent conversation state        │
│  │ Manager             │  Decides when to extract                │
│  └─────────┬───────────┘                                        │
│            │                                                     │
│  ┌─────────▼───────────┐                                        │
│  │ OpenClawSkill       │  HTTP server with endpoints for:       │
│  │ Runtime             │  - Chat completions (proxy)             │
│  │                     │  - Extraction management                │
│  │                     │  - Skill sync (mirroring)               │
│  └─────────┬───────────┘                                        │
│            │                                                     │
│  ┌─────────▼───────────┐                                        │
│  │ SkillMirror         │  Syncs skills to OpenClaw's folder     │
│  │                     │  structure for agent access              │
│  └─────────────────────┘                                        │
│                                                                  │
│  ┌─────────────────────┐                                        │
│  │ Agentic Prompt      │  Custom extraction/merge prompts        │
│  │ Profile             │  optimized for agent trajectories       │
│  └─────────────────────┘                                        │
│                                                                  │
│  ┌─────────────────────┐                                        │
│  │ Conversation        │  JSONL conversation archiving           │
│  │ Archive             │  for offline analysis                    │
│  └─────────────────────┘                                        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Plugin Components

### 1. UpstreamChatProxy

**File**: `OpenClaw-Plugin/openclaw_main_turn_proxy.py`

An HTTP proxy that sits between the OpenClaw agent and the upstream LLM:

```
OpenClaw Agent ──▶ UpstreamChatProxy ──▶ Actual LLM (e.g., Claude)
                   │
                   ├─ Intercept request
                   ├─ Retrieve relevant skills
                   ├─ Inject skills into system prompt
                   ├─ Forward modified request to LLM
                   ├─ Return response to agent
                   └─ Trigger background extraction
```

**Key class**: `OpenClawMainTurnStateManager`
- Accumulates messages across agent turns
- Tracks the conversation window for extraction
- Decides when to trigger extraction based on turn count and topic changes

**Rationale**: By acting as a transparent proxy, the plugin doesn't require changes to the OpenClaw agent code. It intercepts and enhances requests silently.

### 2. OpenClawSkillRuntime

**File**: `OpenClaw-Plugin/service_runtime.py`

The main HTTP server that handles all plugin endpoints:

```
Endpoints:

POST /v1/chat/completions
  Standard OpenAI-compatible endpoint (with skill injection)

POST /v1/autoskill/extractions
  SSE stream of extraction events

POST /v1/autoskill/extract_now
  Manually trigger extraction

POST /v1/autoskill/openclaw/skills/sync
  Sync skills to OpenClaw's folder structure

GET /v1/autoskill/status
  Plugin status and diagnostics

GET /health
  Health check
```

### 3. OpenClawSkillMirror

**File**: `OpenClaw-Plugin/openclaw_skill_mirror.py`

Syncs skills from AutoSkill's SkillBank to OpenClaw's native skill folder structure:

```
AutoSkill SkillBank:                    OpenClaw Skills Folder:
/SkillBank/Users/u1/                    /~/.openclaw/skills/
├── release-process/SKILL.md    ──▶     ├── release-process/
├── monitoring-setup/SKILL.md   ──▶     │   ├── SKILL.md
└── code-review/SKILL.md        ──▶     │   └── deploy-script.sh
                                         ├── monitoring-setup/
                                         │   └── SKILL.md
                                         └── code-review/
                                             └── SKILL.md
```

**Sync process:**
1. List active skills for the user
2. Load the OpenClaw skills manifest
3. For each skill:
   - Allocate a folder name (auto-slug with dedup)
   - Export SKILL.md and bundled files
   - Write to OpenClaw folder
   - Update manifest
4. Prune stale skills (no longer active in AutoSkill)

**Rationale**: OpenClaw expects skills in its own folder format. The mirror keeps both representations in sync so AutoSkill can manage the lifecycle while OpenClaw can discover and use the skills.

### 4. Agentic Prompt Profile

**File**: `OpenClaw-Plugin/agentic_prompt_profile.py`

Custom LLM prompts optimized for agent trajectory extraction:

```python
class OpenClawTrajectorySkillExtractor:
    """Extracts skills specifically from agent action sequences."""
    
    # Custom prompts that understand:
    # - Tool usage patterns (the agent used search, then code, then test)
    # - Multi-step workflows (not just single-turn conversations)
    # - Agent-specific vocabulary (tools, actions, observations)
```

**How it integrates**: The plugin uses Python's monkey-patching to replace the default extraction and merge prompts:

```python
import autoskill.management.maintenance as _m

# Replace default LLM-based decision with agentic version
_m._decide_candidate_action_with_llm = _decide_candidate_action_with_llm_agentic
_m._merge_with_llm = _merge_with_llm_agentic
```

**Rationale**: Agent trajectories are structurally different from human conversations. They contain tool calls, observations, and multi-step reasoning chains. Custom prompts understand this structure and extract better skills. Monkey-patching avoids forking the core SDK.

### 5. Conversation Archive

**File**: `OpenClaw-Plugin/openclaw_conversation_archive.py`

Archives all agent conversations as JSONL files for offline analysis:

```
/archive/
├── 2024-01-15_session_abc123.jsonl
├── 2024-01-15_session_def456.jsonl
└── 2024-01-16_session_ghi789.jsonl
```

Each line is a complete conversation turn with messages, tools used, and results.

**Rationale**: Archived conversations can be re-processed by the offline conversation pipeline to extract additional skills, or used for evaluation and debugging.

---

## Installation

```bash
python OpenClaw-Plugin/install.py \
    --workspace-dir ~/.openclaw \
    --llm-provider internlm \
    --embeddings-provider qwen \
    --user-id u1
```

The installer:
1. Copies plugin files to the OpenClaw workspace
2. Configures environment variables
3. Sets up the proxy to intercept agent LLM calls
4. Creates initial skill directories

---

## Data Flow: Agent Turn with Skills

```
1. OpenClaw agent needs to make an LLM call
   │
2. UpstreamChatProxy intercepts the request
   │
3. MainTurnStateManager accumulates the message
   │
4. If skill score threshold met:
   │  ├─ Retrieve skills for the agent's current task
   │  └─ Inject skills into the system prompt
   │
5. Forward modified request to upstream LLM
   │
6. LLM responds
   │
7. Return response to OpenClaw agent
   │
8. If extraction window conditions met (async):
   │  ├─ OpenClawTrajectorySkillExtractor analyzes trajectory
   │  ├─ Extract SkillCandidates with agentic prompts
   │  ├─ Maintenance: dedupe/merge with existing skills
   │  └─ Persist to SkillBank
   │
9. Optionally: SkillMirror syncs to OpenClaw folder
```

---

## Configuration

The OpenClaw plugin uses environment variables (set via `.env` or `install.py`):

```bash
# LLM Configuration
AUTOSKILL_LLM_PROVIDER=internlm
INTERNLM_API_KEY=...

# Embedding Configuration
AUTOSKILL_EMBEDDINGS_PROVIDER=qwen
DASHSCOPE_API_KEY=...

# Plugin Configuration
AUTOSKILL_USER_ID=u1
AUTOSKILL_SKILL_SCOPE=all
AUTOSKILL_EXTRACT_MODE=auto
AUTOSKILL_EXTRACT_TURN_LIMIT=3
AUTOSKILL_MIN_SCORE=0.4

# OpenClaw Integration
OPENCLAW_WORKSPACE_DIR=~/.openclaw
OPENCLAW_SKILLS_SYNC=true
OPENCLAW_SKILLS_PRUNE=true
```

---

## Testing

The plugin includes its own test suite:

```
OpenClaw-Plugin/tests/
├── test_proxy.py           # Proxy request/response tests
├── test_skill_mirror.py    # Skill sync tests
└── test_extraction.py      # Agentic extraction tests
```

---

## Next Steps

- **[Architecture Overview](architecture-overview.md)** — See where the plugin fits in the overall system
- **[Agent & Skill Architecture](agent-and-skill-architecture.md)** — Understand skills as capability units
- **[Deployment Guide](deployment.md)** — See how to deploy the plugin
