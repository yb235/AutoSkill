# Interactive System

> This document explains how AutoSkill's real-time interaction systems work — the session orchestrator, Web UI, OpenAI proxy, and console chat. These are the components that make AutoSkill usable in live conversations.

---

## Overview

AutoSkill provides three interactive interfaces, all built on the same session orchestration layer:

```
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│    Web UI      │   │  Console Chat  │   │  OpenAI Proxy  │
│  (browser)     │   │  (terminal)    │   │  (HTTP API)    │
└───────┬────────┘   └───────┬────────┘   └───────┬────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Interactive    │
                    │  Session        │
                    │  (orchestrator) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  AutoSkill SDK  │
                    └─────────────────┘
```

**Rationale**: Having a shared orchestration layer means all three interfaces behave identically — same retrieval, same extraction, same skill management. The UI layers are thin wrappers around the session.

---

## InteractiveSession: The Heart of the System

The `InteractiveSession` class (`autoskill/interactive/session.py`) is the headless orchestrator that manages a complete conversation with skill retrieval and extraction.

### What It Does per Turn

```python
result = session.turn(user_message="How do I deploy?")
```

Internally, this triggers the following sequence:

```
1. QUERY REWRITING (optional)
   │ LLMQueryRewriter takes the user message
   │ and expands it for better search recall
   │
   │ "How do I deploy?"
   │ → "production deployment process release canary rollout"
   │
   ▼
2. SKILL RETRIEVAL
   │ retrieve_hits_by_scope() searches per scope:
   │   - User scope: /SkillBank/Users/u1/
   │   - Library scope: /SkillBank/Common/
   │ Uses hybrid search (vector + BM25)
   │ Filters by min_score threshold
   │
   ▼
3. SKILL SELECTION (optional)
   │ LLMSkillSelector judges which hits are truly relevant
   │ Reduces false positives from search
   │
   ▼
4. CONTEXT RENDERING
   │ render_skills_context() formats selected skills
   │ into markdown, respecting max_context_chars
   │
   ▼
5. LLM COMPLETION
   │ Main LLM call with:
   │   System: base prompt + rendered skills
   │   User: conversation history + current message
   │
   ▼
6. RESPONSE DELIVERY
   │ Return to caller: {assistant_reply, retrieval, extraction}
   │
   ▼
7. BACKGROUND EXTRACTION (async)
   │ Check extraction gating (should we extract?)
   │ If yes: spawn background thread
   │   → Extract skill candidates
   │   → Deduplicate and merge
   │   → Persist to store
   │
   ▼
8. USAGE TRACKING (async)
   │ LLMSkillUsageJudge evaluates:
   │   - Were retrieved skills relevant?
   │   - Did the assistant actually use them?
   │   - Update usage counters
```

### Session State

Each session maintains:

```python
session = InteractiveSession(
    runtime=runtime,    # AutoSkillRuntime with SDK + LLM + config
    config=InteractiveConfig(
        user_id="u1",
        skill_scope="all",
        rewrite_mode="always",
        extract_mode="auto",
        extract_turn_limit=3,
        min_score=0.4,
        top_k=5,
        max_context_chars=6000,
        ingest_window=6,
        history_turns=10
    )
)

# Session tracks:
# - Conversation history (messages[])
# - Turn count
# - Background extraction jobs
# - Retrieval traces
# - Usage judgments
```

### Turn Return Value

```python
result = session.turn("How do I deploy?")

# result = {
#     "assistant_reply": "Here's the deployment process: ...",
#     "retrieval": {
#         "original_query": "How do I deploy?",
#         "rewritten_query": "production deployment process...",
#         "search_query": "production deployment process...",
#         "hits_user": [SkillHit(...)],
#         "hits_library": [SkillHit(...)],
#         "selected_for_context": [Skill(...)]
#     },
#     "extraction": {
#         "job_id": "job-abc123",
#         "status": "pending"
#     }
# }
```

**Rationale**: Returning full diagnostic data enables transparency. The Web UI uses this to show what skills were found, why they were selected, and what extraction is doing.

---

## Web UI

The Web UI (`examples/web_ui.py` + `web/`) provides a browser-based interface for interacting with AutoSkill.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Browser (web/index.html + web/static/app.js)                │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Chat Panel   │  │ Session List  │  │ Diagnostics     │  │
│  │             │  │              │  │                  │  │
│  │ Messages    │  │ New/Select/  │  │ • Config        │  │
│  │ Composer    │  │ Delete       │  │ • Retrieval     │  │
│  │             │  │              │  │ • Extraction    │  │
│  │             │  │              │  │ • SKILL.md      │  │
│  │             │  │              │  │   Editor        │  │
│  └──────┬──────┘  └──────────────┘  └──────────────────┘  │
│         │                                                   │
│         │ HTTP requests (fetch API)                          │
└─────────┼───────────────────────────────────────────────────┘
          │
┌─────────▼───────────────────────────────────────────────────┐
│ Web UI Server (examples/web_ui.py)                          │
│                                                             │
│  POST /api/session  → Create/get session                    │
│  POST /api/turn     → Send message, get response            │
│  POST /api/extract  → Trigger extraction                    │
│  GET  /api/extraction/<id> → Poll extraction status         │
│  POST /api/skill/<id>/save → Save edited skill              │
│  DELETE /api/skill/<id>    → Delete skill                   │
│                                                             │
│  Session Manager (keeps InteractiveSession per session_id)  │
└─────────────────────────────────────────────────────────────┘
```

### Key Features

1. **Session Management**: Create, switch, and delete sessions
2. **Chat Interface**: Send messages and see responses with formatting
3. **Retrieval Diagnostics**: See original query, rewritten query, hits per scope, and selected skills
4. **Extraction Diagnostics**: See extraction progress, SKILL.md output
5. **SKILL.md Editor**: Edit extracted skills directly in the browser with Save/Rollback/Delete
6. **Configuration Panel**: Adjust scope, rewrite mode, extract mode, thresholds in real-time

### Frontend Technology

The Web UI is a **single-page application** built with vanilla JavaScript (no framework):

- `web/index.html` — Page structure with CSS Grid layout
- `web/static/app.js` — State management and DOM manipulation (~1200 lines)
- `web/static/styles.css` — Modern CSS with responsive layout (~500 lines)

**Rationale**: No framework dependency means the Web UI works without npm, webpack, or build steps. Just serve the files and open in a browser. This aligns with AutoSkill's zero-dependency philosophy.

---

## OpenAI-Compatible Proxy

The proxy server (`autoskill/interactive/server.py`) makes AutoSkill a transparent drop-in for any application using the OpenAI API.

### How It Works

```
┌──────────────┐        ┌─────────────────────┐        ┌─────────────┐
│ Your App     │        │ AutoSkill Proxy      │        │ Actual LLM  │
│              │        │ (localhost:9000)      │        │ (OpenAI etc)│
│ OpenAI SDK   │───────▶│                     │───────▶│             │
│ base_url=    │        │ 1. Identify user     │        │             │
│ localhost:   │◀───────│ 2. Retrieve skills   │◀───────│             │
│ 9000         │        │ 3. Inject context    │        │             │
│              │        │ 4. Forward to LLM    │        │             │
│              │        │ 5. Return response   │        │             │
│              │        │ 6. Extract (bg)      │        │             │
└──────────────┘        └─────────────────────┘        └─────────────┘
```

### Example Usage

```python
# Your existing code — just change base_url!
from openai import OpenAI

client = OpenAI(
    api_key="anything",                    # Proxy handles auth
    base_url="http://localhost:9000/v1"    # Point to proxy
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "How do I deploy?"}],
    user="u1"    # Optional: per-user skill isolation
)
```

### Proxy Features

| Feature | Description |
|---------|-------------|
| **Streaming** | Full SSE streaming support (compatible with `stream=True`) |
| **User isolation** | Per-user skill scoping via `user` field, header, or JWT |
| **Model multiplexing** | Serve multiple model names (all routed to same LLM) |
| **Background extraction** | Async skill extraction with bounded concurrency |
| **Health check** | `GET /health` for load balancers |
| **CORS** | Configurable cross-origin support |

### User Identification Priority

```
1. request.user field (highest priority)
2. X-AutoSkill-User header
3. JWT Bearer token → decode → id field
4. Default proxy user (fallback)
```

**Rationale**: The proxy is the key to **zero-code integration**. Any application using the OpenAI SDK can benefit from AutoSkill just by changing one URL. No code changes, no SDK imports, no new APIs to learn.

---

## Console Chat

The console chat (`autoskill/interactive/app.py`) provides a terminal-based REPL for direct interaction.

### Usage

```bash
python -m examples.interactive_chat \
    --llm-provider openai \
    --embeddings-provider openai \
    --user-id u1
```

### Commands

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/clear` | Clear conversation history |
| `/extract [hint]` | Manually trigger extraction |
| `/skills` | List current user's skills |
| `/config` | Show current configuration |
| `/quit` | Exit the chat |

### Architecture

```python
class InteractiveChatApp:
    """Console REPL wrapper around InteractiveSession."""
    
    def run(self):
        while True:
            user_input = self.io.read()        # Read from stdin
            
            if user_input.startswith("/"):
                self.handle_command(user_input)  # Process command
            else:
                result = self.session.turn(user_input)  # Chat turn
                self.io.write(result["assistant_reply"])  # Print response
                self.display_diagnostics(result)  # Show retrieval info
```

**Rationale**: The console chat is useful for development and testing — no browser needed, immediate feedback, and you can see retrieval/extraction diagnostics directly in the terminal.

---

## AutoSkillRuntime: The Composition Root

`AutoSkillRuntime` (`autoskill/interactive/unified.py`) wires together all the pieces:

```python
class AutoSkillRuntime:
    """Composition root that creates and holds all components."""
    
    def __init__(self, config, interactive_config):
        # Build providers from config
        self.llm = build_llm(config.llm)
        self.embeddings = build_embeddings(config.embeddings)
        
        # Build SDK
        self.sdk = AutoSkill(config)
        
        # Build interactive components
        self.rewriter = LLMQueryRewriter(self.llm) if needed
        self.selector = LLMSkillSelector(self.llm) if needed
        self.usage_judge = LLMSkillUsageJudge(self.llm) if needed
```

**Rationale**: The composition root pattern centralizes object creation. Instead of scattering `build_llm()` and `build_embeddings()` calls throughout the code, they happen in one place. This makes configuration changes easy and testing straightforward (swap out the runtime).

---

## Background Job Architecture

Skill extraction runs asynchronously to avoid blocking chat responses:

```
┌──────────────────────────────────────────────────────────┐
│ Background Extraction Pipeline                            │
│                                                          │
│  Job Queue (FIFO):                                       │
│  ┌─────┐ ┌─────┐ ┌─────┐                               │
│  │Job 3│→│Job 2│→│Job 1│→ Worker Thread                  │
│  └─────┘ └─────┘ └─────┘    │                           │
│                              ▼                           │
│                     ┌─────────────┐                      │
│                     │ Semaphore   │ (max 2 concurrent)   │
│                     │ Gate        │                      │
│                     └──────┬──────┘                      │
│                            ▼                             │
│                     ┌─────────────┐                      │
│                     │ Extract     │                      │
│                     │ → Dedupe    │                      │
│                     │ → Persist   │                      │
│                     └──────┬──────┘                      │
│                            ▼                             │
│                     Job Status: completed/failed          │
│                     (pollable by UI)                     │
└──────────────────────────────────────────────────────────┘
```

**Rationale**: Bounded concurrency (semaphore with max 2) prevents thread explosion if many extraction jobs are triggered rapidly. The FIFO queue ensures jobs are processed in order. Polling (instead of WebSocket) keeps the implementation simple and stateless.

---

## Event Streaming (SSE)

The OpenAI proxy supports Server-Sent Events for real-time extraction progress:

```
GET /v1/autoskill/extractions
Accept: text/event-stream

event: extraction_started
data: {"job_id": "job-abc123", "user_id": "u1"}

event: extraction_completed  
data: {"job_id": "job-abc123", "skills": [{"name": "Release Process", ...}]}

event: extraction_failed
data: {"job_id": "job-abc123", "error": "LLM returned invalid JSON"}
```

**Rationale**: SSE provides real-time updates without the complexity of WebSockets. The client opens one HTTP connection and receives events as they happen.

---

## Next Steps

- **[API Reference](api-reference.md)** — Detailed endpoint documentation
- **[Data Flow](data-flow.md)** — Step-by-step data flow through the session
- **[Deployment Guide](deployment.md)** — How to start the Web UI and Proxy
