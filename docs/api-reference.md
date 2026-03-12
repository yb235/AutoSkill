# API Reference

> This document covers every API endpoint and SDK method in AutoSkill — what it does, what it expects, and what it returns. Use this as a reference when integrating with AutoSkill.

---

## SDK API (Python)

The SDK is the simplest way to use AutoSkill. Import it, configure it, and call methods.

### Setup

```python
from autoskill import AutoSkill, AutoSkillConfig

config = AutoSkillConfig(
    llm={"provider": "openai", "model": "gpt-4", "api_key": "sk-..."},
    embeddings={"provider": "openai", "model": "text-embedding-3-large"},
    store={"provider": "local", "path": "./SkillBank"}
)

sdk = AutoSkill(config)
```

### `sdk.ingest()`

Extract skills from a conversation and persist them.

```python
skills = sdk.ingest(
    user_id="u1",                    # Required: who owns the skill
    messages=[                       # Required: conversation messages
        {"role": "user", "content": "How do I deploy?"},
        {"role": "assistant", "content": "Run tests, canary, rollout."}
    ],
    events=None,                     # Optional: structured events
    metadata={"channel": "slack"},   # Optional: extra metadata
    hint="Focus on deployment steps" # Optional: extraction hint
)
# Returns: List[Skill] — the skills that were created or updated
```

**What happens internally:**
1. Extract skill candidates using LLM (or heuristic)
2. Check each candidate against existing skills (deduplication)
3. Decide: ADD new, MERGE with existing, or DISCARD
4. Persist to store
5. Return the resulting skills

### `sdk.extract_candidates()`

Extract skill candidates *without* persisting them. Useful for preview/dry-run.

```python
candidates = sdk.extract_candidates(
    user_id="u1",
    messages=[...],
    max_candidates=3,
    hint="Focus on testing patterns"
)
# Returns: List[SkillCandidate] — raw candidates before dedup
```

### `sdk.search()`

Find relevant skills using hybrid search (vector + BM25).

```python
hits = sdk.search(
    query="How to deploy to production?",   # Required: search query
    user_id="u1",                           # Required: scope user
    limit=5,                                # Optional: max results (default: 5)
    filters={                               # Optional: search filters
        "scope": "all",                     #   "user" | "library" | "all"
        "allow_partial_vectors": False
    }
)
# Returns: List[SkillHit] — skills sorted by relevance score (descending)

for hit in hits:
    print(f"  {hit.score:.3f}  {hit.skill.name}")
    # Output: 0.923  Release Process
    #         0.871  Monitoring Setup
```

### `sdk.render_context()`

Search for skills and format them into an injectable text block.

```python
context = sdk.render_context(
    query="How to deploy?",
    user_id="u1",
    max_chars=6000     # Optional: CJK-aware character limit
)
# Returns: str — formatted markdown block ready for system prompt injection

# Use it like:
system_prompt = f"""You are a helpful assistant.

## Available Skills
{context}

Follow the most relevant skill if applicable."""
```

### `sdk.import_openai_conversations()`

Batch import conversations from an OpenAI export and extract skills.

```python
result = sdk.import_openai_conversations(
    user_id="u1",
    file_path="~/Downloads/openai_export.json",  # Or pass data= directly
    metadata={"channel": "openai_import"},
    hint="Focus on problem-solving patterns"
)
# Returns: Dict with import statistics
```

### `sdk.export_skill_md()`

Render a skill as a SKILL.md string.

```python
skill_md = sdk.export_skill_md(skill_id="a1b2c3d4-...")
# Returns: str — the full SKILL.md content (YAML frontmatter + Markdown)
```

### `sdk.write_skill_dir()`

Export a skill as a directory on disk.

```python
sdk.write_skill_dir(
    skill_id="a1b2c3d4-...",
    output_dir="./exported-skills/release-process"
)
# Creates:
#   ./exported-skills/release-process/
#   ├── SKILL.md
#   └── (any bundled files)
```

---

## Web UI REST API

The Web UI server (`examples/web_ui.py`) exposes these REST endpoints on the configured port (default: 8000).

### Create/Get Session

```
POST /api/session
Content-Type: application/json

{
    "action": "create"               // or "get"
    "session_id": "abc123"           // required for "get"
}

Response:
{
    "session_id": "abc123",
    "trace": {
        "turns": [],
        "retrievalEvents": [],
        "extractionEvents": [],
        "usageEvents": [],
        "configEvents": []
    }
}
```

**Rationale**: Sessions group conversation turns together. The trace object provides full diagnostic history.

### Send Message (Chat Turn)

```
POST /api/turn
Content-Type: application/json

{
    "session_id": "abc123",
    "user_message": "How should I deploy to production?",
    "extract_hint": "deployment steps"   // optional
}

Response:
{
    "assistant_reply": "Here's the deployment process: ...",
    "retrieval": {
        "original_query": "How should I deploy to production?",
        "rewritten_query": "production deployment canary rollout ...",
        "search_query": "production deployment canary rollout ...",
        "hits_user": [
            {
                "skill": { "id": "...", "name": "Release Process", ... },
                "score": 0.923
            }
        ],
        "hits_library": [...],
        "selected_for_context": [
            { "id": "...", "name": "Release Process", ... }
        ]
    },
    "extraction": {
        "job_id": "job-456",
        "status": "pending"
    }
}
```

**Rationale**: The response includes both the assistant's reply and full diagnostic data about retrieval and extraction, enabling the Web UI to show transparency panels.

### Poll Extraction Status

```
GET /api/extraction/<job_id>

Response (while running):
{
    "status": "running"
}

Response (when complete):
{
    "status": "completed",
    "upserted": [
        { "id": "...", "name": "Deployment Process", "version": "1.0.0", ... }
    ],
    "skill_mds": [
        "---\nid: ...\nname: Deployment Process\n---\n..."
    ]
}

Response (on failure):
{
    "status": "failed",
    "error": "LLM returned invalid JSON after 3 repair attempts"
}
```

**Rationale**: Extraction runs in the background. Polling lets the UI show progress without blocking the chat.

### Save Edited Skill

```
POST /api/skill/<skill_id>/save
Content-Type: application/json

{
    "skill_md": "---\nid: ...\nname: Release Process\n---\n# Release Process\n..."
}

Response:
{
    "success": true,
    "message": "Skill saved successfully"
}
```

**Rationale**: Users can edit skills directly in the Web UI's SKILL.md editor. This endpoint persists those edits.

### Delete Skill

```
DELETE /api/skill/<skill_id>

Response:
{
    "success": true,
    "deleted_count": 1
}
```

### Serve Web UI

```
GET /           → serves web/index.html
GET /static/*   → serves web/static/* (CSS, JS)
```

---

## OpenAI-Compatible Proxy API

The proxy server (`autoskill/interactive/server.py`, launched via `examples/openai_proxy.py`) exposes an OpenAI-compatible API on the configured port (default: 9000). Any application using the OpenAI SDK can point to this proxy and get automatic skill injection.

### Chat Completions

```
POST /v1/chat/completions
Content-Type: application/json

{
    "model": "gpt-4",
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "How should I deploy to production?"}
    ],
    "stream": false,             // or true for SSE streaming
    "temperature": 0.7,
    "user": "u1"                 // optional: per-user skill isolation
}

Response (non-streaming):
{
    "id": "chatcmpl-abc123",
    "object": "chat.completion",
    "model": "gpt-4",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Here's the deployment process: ..."
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 150,
        "completion_tokens": 200,
        "total_tokens": 350
    }
}

Response (streaming):
data: {"id":"chatcmpl-abc123","choices":[{"delta":{"content":"Here's"},...}]}
data: {"id":"chatcmpl-abc123","choices":[{"delta":{"content":" the"},...}]}
...
data: [DONE]
```

**What the proxy does transparently:**
1. Receives the standard OpenAI request
2. Retrieves relevant skills for the user query
3. Injects skills into the system prompt
4. Forwards to the actual LLM provider
5. Returns the response in OpenAI format
6. Triggers background skill extraction

### Embeddings

```
POST /v1/embeddings
Content-Type: application/json

{
    "input": "How to deploy to production?",    // or ["text1", "text2"]
    "model": "text-embedding-3-large"
}

Response:
{
    "object": "list",
    "data": [
        {
            "index": 0,
            "embedding": [0.0123, -0.0456, ...],  // 1536 floats
            "object": "embedding"
        }
    ],
    "model": "text-embedding-3-large",
    "usage": {
        "prompt_tokens": 8,
        "total_tokens": 8
    }
}
```

### Model List

```
GET /v1/models

Response:
{
    "object": "list",
    "data": [
        {
            "id": "gpt-4",
            "object": "model",
            "created": 1677610602,
            "owned_by": "autoskill-proxy"
        }
    ]
}
```

### Health Check

```
GET /health

Response:
{
    "status": "ok",
    "version": "0.1.0"
}
```

### User Identification (Priority Order)

The proxy identifies users for skill scoping in this priority order:

| Priority | Method | Example |
|----------|--------|---------|
| 1 (highest) | `user` field in request body | `"user": "u1"` |
| 2 | `X-AutoSkill-User` header | `X-AutoSkill-User: u1` |
| 3 | JWT Bearer token (`id` field) | `Authorization: Bearer eyJ...` |
| 4 (lowest) | Default proxy user | Configured via `AUTOSKILL_USER_ID` |

**Rationale**: Multiple identification methods support different integration scenarios — direct API calls, proxy headers, and OAuth/JWT flows.

---

## CLI Commands

AutoSkill provides a CLI interface via the `autoskill` command (or `python -m autoskill`).

### Offline Document Processing

```bash
# Build skills from a document
python -m autoskill offline document build \
    --file paper.md \
    --title "Research Methods" \
    --domain psychology \
    --user-id u1 \
    --store-dir ./SkillBank

# Build from stdin
cat paper.md | python -m autoskill offline document build \
    --title "Research Methods" \
    --domain psychology \
    --user-id u1
```

### Offline Conversation Extraction

```bash
# Extract skills from conversation logs
python -m autoskill offline conversation extract \
    --file conversations.jsonl \
    --user-id u1 \
    --store-dir ./SkillBank
```

### Offline Trajectory Extraction

```bash
# Extract skills from agent trajectory logs
python -m autoskill offline trajectory extract \
    --file trajectories.jsonl \
    --user-id u1 \
    --store-dir ./SkillBank
```

---

## Next Steps

- **[Data Schemas](data-schemas.md)** — Understand the data structures in API responses
- **[Interactive System](interactive-system.md)** — Understand the session/proxy architecture behind these APIs
- **[Deployment Guide](deployment.md)** — Learn how to run these servers
