# Tool-Call Memory System: Research & Spec

> Research conducted: 2026-05-06  
> Status: DRAFT — spec only, no implementation

---

## Table of Contents

1. [What Exists](#1-what-exists)
2. [Gap Analysis](#2-gap-analysis)
3. [Build vs Borrow Decision](#3-build-vs-borrow-decision)
4. [Proposed PostgreSQL Schema](#4-proposed-postgresql-schema)
5. [MCP Server Interface](#5-mcp-server-interface)
6. [Hook Capture Spec](#6-hook-capture-spec)
7. [REST API Spec for Lovable](#7-rest-api-spec-for-lovable)
8. [Honest Assessment](#8-honest-assessment)

---

## 1. What Exists

### 1.1 mem0 (`mem0ai/mem0` — 55k stars, Apache-2.0)

**What it does:**  
Universal memory layer for LLM agents. Extracts facts from conversations using an LLM, stores them as discrete memories, and retrieves them by semantic search. Multi-level entity model: `user`, `agent`, `app` (≈project), `run` (≈session).

New April 2026 algorithm: single-pass ADD-only extraction, entity linking, hybrid retrieval (semantic + BM25 + entity boosting). 91.6 LoCoMo / 93.4 LongMemEval.

**PostgreSQL support:** Not in the OSS Python library. Self-hosted server (`cd server && make bootstrap`) runs via Docker Compose with an internal vector backend (Qdrant-based, not directly PostgreSQL). No raw SQL access or custom schema.

**MCP server:** Yes. `mem0-plugin` exposes 9 tools via `https://mcp.mem0.ai/mcp` (cloud) or locally via the deprecated `openmemory` stack (being sunset as of April 2026). Tools: `add_memory`, `search_memories`, `get_memories`, `get_memory`, `update_memory`, `delete_memory`, `delete_all_memories`, `delete_entities`, `list_entities`. Lifecycle hooks for Claude Code: `SessionStart` (inject prior memories), `UserPromptSubmit` (inject relevant), `Stop` (remind agent to persist).

**Tool-call indexing:** No. Memories are LLM-extracted facts from conversations — summaries and assertions, not raw tool inputs/outputs. There is no way to query "what did Bash return for project X three weeks ago." The unit of storage is a _memory string_, not a tool invocation.

**Project tagging:** `app` dimension maps to project scope; `run` maps to session. Filters work across these dimensions in search/list calls.

**Verdict:** Solves conversational memory well, but operates at the wrong granularity for tool-call archiving. No raw capture, no chunked tool outputs, no structured I/O storage.

---

### 1.2 Zep (`getzep/zep` — 4.5k stars) + Graphiti (`getzep/graphiti` — 25.8k stars, Apache-2.0)

**What Graphiti does:**  
Temporal knowledge graph engine. Ingests _episodes_ (raw text or **structured JSON**), extracts entities and relationships with validity windows (`valid_at` / `invalid_at`), and stores them across Neo4j/FalkorDB/Kuzu/Neptune backends. Every fact has a bi-temporal window — old facts are invalidated, not deleted.

Data model:
- **Entities** (nodes): people, products, concepts — summaries evolve over time
- **Facts/Relationships** (edges): triplets with `valid_at` / `invalid_at`
- **Episodes** (provenance): raw ingested data, text or JSON, the ground truth

**What Zep does:**  
Managed context engineering platform built on Graphiti. Handles per-user context graphs, cross-session persistence, sub-200ms retrieval. SDKs: Python, TypeScript, Go. MCP server in Go with 13 tools including `search_graph`, `get_node`, `get_edge`, `get_episode`, `get_node_edges`.

**PostgreSQL support:** Only in the **deprecated Community Edition** (Go server, now in `legacy/`, no longer maintained). Current Zep/Graphiti uses Neo4j or FalkorDB — not PostgreSQL.

**Tool-call indexing:** Episodes accept structured JSON, so you *could* ingest tool call outputs as episodes. Graphiti would then extract entities and relationships. However, this requires a Neo4j/FalkorDB instance, and the extraction is still LLM-mediated (facts extracted from the JSON), not a direct archive of raw I/O.

**Cross-session memory:** Graphiti: use a consistent `group_id` across sessions. Zep Cloud manages this automatically.

**Verdict:** Graphiti is philosophically the closest to what we want (temporal, provenance-aware, structured JSON support). But the required graph database backend (Neo4j/FalkorDB) is a heavy operational dependency, there is no PostgreSQL path, and the LLM-mediated extraction loses raw fidelity. For a query like "show me every Bash call in project X sorted by date," Graphiti's abstraction adds friction rather than value.

---

### 1.3 Letta / MemGPT (`letta-ai/letta` — 22.5k stars, Apache-2.0)

**What it does:**  
Stateful agent platform. Agents have persistent `memory_blocks` (named string slots the agent can edit in real-time), plus a full conversation thread stored in PostgreSQL. Tool results are stored as typed message objects in the thread.

**PostgreSQL support:** Yes — primary backend, `pgvector/pgvector:0.8.1-pg18-trixie` Docker image, Alembic migrations, composite indexes.

**MCP interface:** Yes — Letta exposes an MCP-compatible API and can consume external MCP tool servers. Tool results flow into the conversation thread.

**Tool-call memory:** Tool results are stored as `tool_return` message objects in the conversation. You can retrieve them by scrolling the thread, but there is no dedicated tool-call archive or cross-session query interface. No chunking, no embedding of tool outputs, no cosine-similarity graph.

**Project tagging:** Not surfaced in core API. Organizations and agents exist, but no explicit project/folder dimension.

**Verdict:** PostgreSQL + pgvector backend is exactly right, and Letta is a mature production system. But it's an agent platform, not a passive observer. Retrofitting it to be a hook-driven tool-call archive would mean using it completely sideways from its design intent. The query interface ("what did Bash return in project X?") has no natural expression in Letta's model.

---

### 1.4 OpenMemory / Mem0 MCP

**What it does:**  
Self-hosted local MCP server (Docker, port 8765) wrapping the mem0 library. Provides the same 9 MCP tools as the cloud plugin. SSE transport, URL: `http://localhost:8765/mcp/<client-name>/sse/<user-id>`.

**Status:** **Being sunset** as of April 2026. Replaced by the Mem0 self-hosted server (`cd server && make bootstrap`).

**PostgreSQL:** Not documented. Docker Compose stack uses Qdrant for vectors internally.

**Verdict:** Deprecated. Same fundamental limitations as mem0 — no tool-call granularity.

---

### 1.5 nexus-reasoning-graph (`Gonzih/nexus-reasoning-graph`)

**What it does:**  
Claude Code plugin + local Node.js service + React/D3 viewer. Intercepts `UserPromptSubmit` and `PostToolUse` hooks, posts raw content to a local service, chunks with a sliding window (512 tok / 128 tok stride), embeds with local ONNX (`all-MiniLM-L6-v2`, 384d) → OpenAI fallback → TF-IDF stub, computes cosine-similarity influence edges, and renders a live force-directed graph at `localhost:7702`.

Architecture:
```
Claude Code hooks
  └─ UserPromptSubmit  ──POST──► /node  (type: intent)
  └─ PostToolUse       ──POST──► /node  (type: tool_name)

Graph Service  (Node.js, localhost:7702)
  ├─ SQLite (service/data/nexus.db)
  ├─ Sliding-window chunker (512 tok / 128 stride)
  ├─ Embeddings (ONNX → OpenAI → TF-IDF)
  └─ Cosine-similarity influence edges

Viewer  (React + D3, force-directed, 2s polling)
```

POST `/node` payload:
```json
{
  "session_id": "string",
  "type": "intent|web_search|web_fetch|file_read|bash|agent|synthesis|compression_cut",
  "tool_name": "string|null",
  "content": "full raw text",
  "timestamp": "ISO8601 (optional)"
}
```

**PostgreSQL:** No — SQLite only.  
**MCP server:** No — hooks post directly to the local HTTP service.  
**Cross-session memory:** No — session_id scoped, no project tagging, no cross-project queries.  
**Project tagging:** No — `session_id` is the only dimension.  
**REST API for external viz:** Partial — has `GET /graph/:session_id`, `GET /sessions`, `GET /events/:session_id` (SSE), but no search, no date filters, no project filters.

**Verdict:** This is the spiritual ancestor of the system we want. Architecturally, it does exactly the right thing (hooks → raw capture → chunk → embed → graph). What it lacks is the persistence layer upgrade (SQLite → PostgreSQL), project/cohort awareness, MCP query interface, and a clean REST API for external viz. **This is the natural fork base.**

---

### 1.6 Claude Code Hooks — What's Available

Source: https://docs.anthropic.com/en/docs/claude-code/hooks

**Events relevant to tool-call capture:**

| Event | Fires when | Can block? | Key fields |
|---|---|---|---|
| `PreToolUse` | Before a tool executes | Yes | `session_id`, `cwd`, `tool_name`, `tool_input` |
| `PostToolUse` | After a tool succeeds | No | `session_id`, `cwd`, `tool_name`, `tool_input`, `tool_response`, `tool_use_id`, `duration_ms` |
| `PostToolUseFailure` | After a tool fails | No | `session_id`, `cwd`, `tool_name`, `tool_input`, `tool_use_id`, `error`, `is_interrupt`, `duration_ms` |
| `PostToolBatch` | After a parallel batch resolves | Yes (blocks next LLM call) | `session_id`, `tool_calls[]` |
| `UserPromptSubmit` | User submits a prompt | Yes | `session_id`, `cwd`, `prompt` |
| `SessionStart` | Session begins/resumes | No | `session_id`, `cwd`, `source`, `model` |
| `SessionEnd` | Session terminates | No | `session_id`, `cwd` |
| `Stop` | Claude finishes responding | Yes | `session_id`, `cwd`, `last_assistant_message` |
| `CwdChanged` | Working directory changes | No | `session_id`, `cwd` |

**Hook types:** `command` (shell), `http` (POST JSON), `mcp_tool` (call MCP tool), `prompt` (LLM eval), `agent` (spawn subagent)

**Critical insight:** Every hook event includes `cwd` — the current working directory. This is the natural project identifier. Combined with `session_id`, every tool call can be tagged with both project (folder/repo) and session without any additional instrumentation.

**PostToolUse payload (full schema):**
```json
{
  "session_id": "abc123",
  "transcript_path": "/path/.../transcript.jsonl",
  "cwd": "/home/user/my-project",
  "permission_mode": "default",
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test", "description": "..." },
  "tool_response": { "stdout": "...", "stderr": "...", "exit_code": 0 },
  "tool_use_id": "toolu_01ABC123...",
  "duration_ms": 342
}
```

---

### 1.7 Vector DB Options: pgvector vs Qdrant

**pgvector (`pgvector/pgvector` — v0.8.2, Postgres 13+):**
- PostgreSQL extension: adds `vector`, `halfvec`, `sparsevec`, `bit` column types
- Up to 16,000 dimensions per vector
- Distance operators: `<->` (L2), `<=>` (cosine), `<#>` (inner product), `<+>` (L1)
- Indexes: HNSW (better query perf, more memory) and IVFFlat (faster build, less memory)
- ACID compliance, replication, JOINs with regular tables
- Single database for all data + vectors — no separate service
- `CREATE INDEX USING hnsw (embedding vector_cosine_ops)` for approximate nearest neighbor

**Qdrant (`qdrant/qdrant` — v1.17.1, Rust):**
- Dedicated vector database, REST (port 6333) + gRPC
- Points = vectors + JSON payload
- Filtering with `should`/`must`/`must_not` on payload fields
- Sparse + dense vectors (hybrid BM25 + semantic)
- Built for very large scale (sharding, replication, SIMD, `io_uring`)
- Requires separate service to manage
- No JOINs with relational data

**Decision for this system: pgvector.**  
We need JOINs between tool_calls, chunks, projects, and sessions constantly. Running a single PostgreSQL instance with pgvector eliminates operational complexity and keeps queries expressive. Qdrant's advantages (scale, Rust performance) don't matter until we're storing tens of millions of chunks. Start with pgvector; migrate to Qdrant if benchmarks justify it.

---

## 2. Gap Analysis

The system we want to build requires **all** of the following simultaneously. No existing system provides all of them:

| Capability | mem0 | Graphiti | Letta | nexus-reasoning-graph | Our System |
|---|:---:|:---:|:---:|:---:|:---:|
| Hook-based raw tool I/O capture | Partial (hooks exist, but stores summaries) | No | No | **Yes** | **Yes** |
| Raw tool_input + tool_response stored verbatim | No | No | Partial | **Yes** | **Yes** |
| PostgreSQL backend | No | No | **Yes** | No | **Yes** |
| pgvector embeddings | No | No | **Yes** | No | **Yes** |
| Sliding-window chunking of tool outputs | No | No | No | **Yes** | **Yes** |
| Project/folder tagging from CWD | Partial (`app`) | No | No | No | **Yes** |
| Cross-session, cross-project cohort memory | Partial | Partial | No | No | **Yes** |
| MCP server for querying memory | **Yes** | **Yes** | **Yes** | No | **Yes** |
| Query by: project + tool + date range | No | No | No | No | **Yes** |
| REST API for external visualization | No | No | No | Partial | **Yes** |
| Influence edges (cosine similarity) | No | Partial (graph edges) | No | **Yes** | **Yes** |
| Live SSE streaming for active session | No | No | No | **Yes** | **Yes** |
| No LLM required for capture pipeline | No | No | No | **Yes** | **Yes** |

**The core gap no existing system fills:** A passive, hook-driven observer that stores raw tool I/O verbatim, chunks and embeds it without requiring LLM extraction, tags it by project (from CWD), and makes it queryable via MCP with SQL-level precision (project X, tool Y, date range Z).

---

## 3. Build vs Borrow Decision

### Fork nexus-reasoning-graph as the base

nexus-reasoning-graph already has the hardest parts: the hook wiring pattern, the chunking pipeline, the embedding fallback chain, the SSE streaming, and the D3 visualization. The architectural shape is exactly right.

**What to change:**
1. SQLite → PostgreSQL + pgvector
2. Session-scoped → project-tagged + cross-session global
3. Add MCP server layer (FastMCP in Python or MCP SDK in TypeScript/Node)
4. Add REST API for Lovable (versioned, documented)
5. Add `PostToolUseFailure` capture (currently only `PostToolUse`)
6. Add `SessionStart` / `SessionEnd` for session lifecycle tracking

### Borrow patterns from

**mem0-plugin:** Lifecycle hook patterns — `SessionStart` to register the session, `Stop` to finalize. Entity dimension model (user, app, run) maps to our (user, project, session).

**Letta:** PostgreSQL + pgvector schema patterns. Alembic for migrations.

**Graphiti:** Temporal validity concept — store `created_at` on every edge; don't delete, just append. Provenance chain: episode → chunk → influence edge.

### Don't adopt

- **Graphiti's graph backend:** Neo4j/FalkorDB adds operational complexity for no benefit at this scale.
- **mem0's LLM extraction:** We want raw capture, not LLM-summarized memories. Summaries lose the "what did Bash return?" fidelity.
- **Letta's agent platform:** We need a passive observer, not a stateful agent runtime.

---

## 4. Proposed PostgreSQL Schema

All tables use UUIDs for portability. `cwd` is stored as the authoritative project identifier; `projects` normalizes it. pgvector `vector(384)` for local ONNX embeddings (`all-MiniLM-L6-v2`); can be widened to `vector(1536)` for OpenAI.

```sql
-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ─────────────────────────────────────────────────────────────────────────────
-- projects: one row per unique CWD (resolved to canonical folder path)
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    folder_path     TEXT UNIQUE NOT NULL,   -- canonical absolute path, e.g. /home/user/ecoclaw
    repo_remote     TEXT,                   -- git remote URL, extracted at session start
    repo_name       TEXT,                   -- e.g. "ecoclaw", extracted from remote or folder
    display_name    TEXT,                   -- override for UI
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_active_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_projects_folder ON projects (folder_path);

-- ─────────────────────────────────────────────────────────────────────────────
-- sessions: one row per Claude Code session
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      TEXT UNIQUE NOT NULL,   -- from hook: session_id field
    project_id      UUID REFERENCES projects(id),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at        TIMESTAMPTZ,
    model           TEXT,                   -- from SessionStart: model field
    source          TEXT,                   -- startup | resume | clear | compact
    transcript_path TEXT,
    permission_mode TEXT
);

CREATE INDEX idx_sessions_project   ON sessions (project_id);
CREATE INDEX idx_sessions_started   ON sessions (started_at DESC);
CREATE INDEX idx_sessions_session_id ON sessions (session_id);

-- ─────────────────────────────────────────────────────────────────────────────
-- tool_calls: raw tool invocations (PostToolUse + PostToolUseFailure)
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE tool_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES sessions(id) NOT NULL,
    project_id      UUID REFERENCES projects(id) NOT NULL,
    tool_use_id     TEXT UNIQUE,            -- from hook: tool_use_id (deduplication key)
    tool_name       TEXT NOT NULL,          -- Bash, Read, Write, Edit, Glob, Grep, WebFetch, etc.
    tool_input      JSONB NOT NULL,         -- verbatim tool_input from hook
    tool_response   JSONB,                  -- verbatim tool_response (NULL on failure)
    error           TEXT,                   -- from PostToolUseFailure: error field
    is_failure      BOOLEAN NOT NULL DEFAULT FALSE,
    is_interrupt    BOOLEAN NOT NULL DEFAULT FALSE,
    duration_ms     INTEGER,
    cwd             TEXT NOT NULL,          -- exact CWD at call time
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tool_calls_session     ON tool_calls (session_id);
CREATE INDEX idx_tool_calls_project     ON tool_calls (project_id);
CREATE INDEX idx_tool_calls_tool_name   ON tool_calls (tool_name);
CREATE INDEX idx_tool_calls_captured_at ON tool_calls (captured_at DESC);
CREATE INDEX idx_tool_calls_project_tool ON tool_calls (project_id, tool_name);
-- JSONB index for common tool_input fields (e.g. file_path, command)
CREATE INDEX idx_tool_calls_input_gin   ON tool_calls USING gin (tool_input);

-- ─────────────────────────────────────────────────────────────────────────────
-- chunks: sliding-window chunks of tool output content, with embeddings
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tool_call_id    UUID REFERENCES tool_calls(id) NOT NULL,
    session_id      UUID REFERENCES sessions(id) NOT NULL,
    project_id      UUID REFERENCES projects(id) NOT NULL,
    chunk_index     INTEGER NOT NULL,       -- 0-based position within tool output
    content         TEXT NOT NULL,
    token_count     INTEGER,
    embedding       vector(384),            -- local ONNX; set to vector(1536) for OpenAI
    embedding_model TEXT,                   -- "all-MiniLM-L6-v2" | "text-embedding-3-small" | "tfidf"
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tool_call_id, chunk_index)
);

CREATE INDEX idx_chunks_tool_call    ON chunks (tool_call_id);
CREATE INDEX idx_chunks_session      ON chunks (session_id);
CREATE INDEX idx_chunks_project      ON chunks (project_id);
CREATE INDEX idx_chunks_created_at   ON chunks (created_at DESC);
-- HNSW index for approximate nearest neighbor (cosine distance)
CREATE INDEX idx_chunks_embedding    ON chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- ─────────────────────────────────────────────────────────────────────────────
-- influence_edges: cosine similarity edges between chunks (within or across sessions)
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE influence_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_chunk_id UUID REFERENCES chunks(id) NOT NULL,
    target_chunk_id UUID REFERENCES chunks(id) NOT NULL,
    similarity      FLOAT NOT NULL,         -- cosine similarity, 0.0–1.0
    session_id      UUID REFERENCES sessions(id),  -- NULL = cross-session edge
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (source_chunk_id, target_chunk_id)
);

CREATE INDEX idx_influence_source    ON influence_edges (source_chunk_id);
CREATE INDEX idx_influence_target    ON influence_edges (target_chunk_id);
CREATE INDEX idx_influence_session   ON influence_edges (session_id);
CREATE INDEX idx_influence_similarity ON influence_edges (similarity DESC);

-- ─────────────────────────────────────────────────────────────────────────────
-- prompts: user prompts captured at UserPromptSubmit
-- ─────────────────────────────────────────────────────────────────────────────
CREATE TABLE prompts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID REFERENCES sessions(id) NOT NULL,
    project_id      UUID REFERENCES projects(id) NOT NULL,
    content         TEXT NOT NULL,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_prompts_session ON prompts (session_id);
CREATE INDEX idx_prompts_project ON prompts (project_id);
```

**Notes on schema decisions:**

- `project_id` is denormalized onto `tool_calls` and `chunks` to avoid joins in the hot query path.
- `tool_use_id` has a UNIQUE constraint — if the hook fires twice for the same tool call (e.g. retry), the second insert is a no-op.
- `influence_edges.session_id = NULL` means the edge crosses session boundaries — this is how cohort memory manifests structurally.
- `embedding vector(384)` fits local ONNX. Widen to `vector(1536)` when switching to OpenAI embeddings (requires rebuilding the HNSW index).
- The HNSW index on `chunks.embedding` enables sub-second ANN search across millions of chunks.

---

## 5. MCP Server Interface

The MCP server wraps the PostgreSQL backend and exposes it to any MCP-compatible client (Claude Code, Cursor, etc.). Built with **FastMCP** (Python) or **@modelcontextprotocol/sdk** (TypeScript/Node, matching nexus-reasoning-graph's stack).

### Tools to expose

#### `search_memory`
Semantic search across all embedded chunks. Returns chunks with their parent tool call context.

```typescript
search_memory({
  query: string,               // natural language query
  project?: string,            // folder_path or repo_name (partial match)
  tool_name?: string,          // filter by tool: "Bash", "Read", etc.
  date_from?: string,          // ISO8601
  date_to?: string,            // ISO8601
  limit?: number,              // default 10, max 50
  min_similarity?: number,     // default 0.5
}): {
  results: Array<{
    chunk_id: string,
    content: string,
    similarity: number,
    tool_call: {
      id: string,
      tool_name: string,
      tool_input: object,
      captured_at: string,
      duration_ms: number,
    },
    project: { folder_path: string, repo_name: string },
    session: { session_id: string, started_at: string },
  }>
}
```

#### `get_tool_calls`
List tool calls with structured filters. No embedding required — pure relational query.

```typescript
get_tool_calls({
  project?: string,            // folder_path or repo_name
  tool_name?: string,          // exact or partial match
  date_from?: string,
  date_to?: string,
  is_failure?: boolean,
  limit?: number,              // default 20, max 100
  offset?: number,
}): {
  total: number,
  tool_calls: Array<{
    id: string,
    tool_name: string,
    tool_input: object,
    tool_response: object | null,
    error: string | null,
    is_failure: boolean,
    duration_ms: number,
    captured_at: string,
    project: { folder_path: string, repo_name: string },
    session_id: string,
  }>
}
```

#### `get_session_graph`
Full influence graph for a session — nodes (tool calls) + edges (cosine similarity). Used to render the D3 graph.

```typescript
get_session_graph({
  session_id: string,          // Claude Code session_id (from hook)
  include_cross_session_edges?: boolean,  // default false
}): {
  session: { session_id: string, started_at: string, model: string },
  project: { folder_path: string, repo_name: string },
  nodes: Array<{
    id: string,              // chunk.id
    tool_call_id: string,
    tool_name: string,
    chunk_index: number,
    content: string,         // truncated to 500 chars
    captured_at: string,
  }>,
  edges: Array<{
    source: string,          // chunk.id
    target: string,          // chunk.id
    similarity: number,
    is_cross_session: boolean,
  }>
}
```

#### `get_project_summary`
Aggregated stats for a project over a time range.

```typescript
get_project_summary({
  project: string,
  date_from?: string,
  date_to?: string,
}): {
  project: { folder_path: string, repo_name: string },
  total_sessions: number,
  total_tool_calls: number,
  total_failures: number,
  tool_breakdown: Array<{ tool_name: string, count: number, avg_duration_ms: number }>,
  most_active_sessions: Array<{ session_id: string, started_at: string, tool_call_count: number }>,
  date_range: { from: string, to: string },
}
```

#### `list_projects`
Enumerate all projects with activity summaries.

```typescript
list_projects({}): {
  projects: Array<{
    id: string,
    folder_path: string,
    repo_name: string,
    total_sessions: number,
    total_tool_calls: number,
    last_active_at: string,
  }>
}
```

#### `get_similar_calls`
Given a tool_use_id, find semantically similar tool calls across all projects and sessions.

```typescript
get_similar_calls({
  tool_use_id: string,
  limit?: number,              // default 10
  project?: string,            // restrict to a project
}): {
  source: { tool_name: string, tool_input: object, captured_at: string },
  similar: Array<{
    chunk_id: string,
    content: string,
    similarity: number,
    tool_call: { tool_name: string, tool_input: object, captured_at: string },
    project: { folder_path: string, repo_name: string },
  }>
}
```

#### `get_bash_history`
Convenience tool — list Bash commands run in a project.

```typescript
get_bash_history({
  project: string,
  date_from?: string,
  date_to?: string,
  pattern?: string,            // substring match on command
  limit?: number,              // default 50
}): {
  commands: Array<{
    command: string,
    exit_code: number | null,
    stdout_preview: string,    // first 200 chars of response
    duration_ms: number,
    captured_at: string,
    session_id: string,
  }>
}
```

### MCP server configuration example (Claude Code)

```json
{
  "mcpServers": {
    "tool-memory": {
      "command": "node",
      "args": ["/path/to/tool-memory-mcp/server.js"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/tool_memory"
      }
    }
  }
}
```

---

## 6. Hook Capture Spec

### Hook configuration snippet (`settings.json`)

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "http",
          "url": "http://localhost:7703/hook/session_start",
          "async": true
        }]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "http",
          "url": "http://localhost:7703/hook/session_end",
          "async": true
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "http",
          "url": "http://localhost:7703/hook/tool_call",
          "async": true
        }]
      }
    ],
    "PostToolUseFailure": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "http",
          "url": "http://localhost:7703/hook/tool_call_failure",
          "async": true
        }]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "http",
          "url": "http://localhost:7703/hook/prompt",
          "async": true
        }]
      }
    ],
    "CwdChanged": [
      {
        "matcher": "*",
        "hooks": [{
          "type": "http",
          "url": "http://localhost:7703/hook/cwd_changed",
          "async": true
        }]
      }
    ]
  }
}
```

**All hooks use `async: true`** — capture must never block Claude Code. PostToolUse is already non-blocking (tool already ran), but explicit async prevents any latency from network failures.

### Project resolution from CWD

The `cwd` field in every hook event is the authoritative project identifier. Resolution algorithm:

```
1. Receive cwd from hook (e.g. "/home/user/ecoclaw")
2. Try: git -C <cwd> remote get-url origin → extract repo_name from URL
3. If git fails: use basename(cwd) as repo_name
4. SELECT id FROM projects WHERE folder_path = realpath(cwd)
5. If not found: INSERT INTO projects (folder_path, repo_name, ...)
6. Return project_id
```

Run this resolution at `SessionStart` and also on `CwdChanged` (user switched directories mid-session). Cache the `(session_id → project_id)` mapping in memory to avoid per-call DB reads.

### Data to capture per PostToolUse

```
Tool name:        hook.tool_name
Tool input:       hook.tool_input  (JSONB, verbatim)
Tool response:    hook.tool_response  (JSONB, verbatim)
Tool use ID:      hook.tool_use_id  (deduplication key)
Duration:         hook.duration_ms
Session ID:       hook.session_id  → resolve to sessions.id
Project ID:       derived from hook.cwd  → resolve to projects.id
CWD:              hook.cwd  (stored verbatim for audit)
Captured at:      NOW()
```

### Content extraction for chunking

`tool_response` is JSONB. Different tools return different shapes:

| Tool | Response shape | Content to chunk |
|---|---|---|
| `Bash` | `{ stdout, stderr, exit_code }` | `stdout` + `stderr` |
| `Read` | string (file contents with line numbers) | string |
| `Grep` | string (matching lines) | string |
| `WebFetch` | string (rendered markdown) | string |
| `WebSearch` | `{ results: [{title, url, snippet}] }` | join titles + snippets |
| `Edit` / `Write` | `{ success, filePath }` | `filePath` + tool_input.content |
| `Glob` | string (file list) | string |
| `Agent` | string (agent result) | string |

For each tool call, extract the text content, then apply the sliding-window chunker:
- Window size: 512 tokens (≈ 2000 chars)
- Stride: 128 tokens (≈ 512 chars)
- Minimum chunk: 50 tokens (skip trivial responses)
- Token counting: simple whitespace split is fine for chunking (not semantic boundary detection)

### Embedding pipeline (same as nexus-reasoning-graph)

Priority order:
1. `@xenova/transformers` `all-MiniLM-L6-v2` (local ONNX, 384d) — preferred, no API cost
2. OpenAI `text-embedding-3-small` (1536d) — if `OPENAI_API_KEY` set and local fails
3. TF-IDF stub (always available) — fallback, lower quality

Store `embedding_model` on each chunk so queries can filter by model type and avoid comparing embeddings across different embedding spaces.

### Influence edge computation

After inserting all chunks for a tool call:
1. Within-session: compute cosine similarity between new chunks and all prior chunks in the session
2. Store edges where `similarity >= 0.6` (threshold configurable)
3. Cross-session cohort edges: nightly batch job computes edges between chunks in the same project across sessions where `similarity >= 0.75`

---

## 7. REST API Spec for Lovable

Base URL: `http://localhost:7703/api/v1`

All endpoints return JSON. Errors: `{ "error": "message", "code": "ERROR_CODE" }`.

Authentication: none (local service). Add bearer token support if exposed externally.

### Projects

#### `GET /projects`
List all projects with activity summaries.

Response:
```json
{
  "projects": [
    {
      "id": "uuid",
      "folder_path": "/home/user/ecoclaw",
      "repo_name": "ecoclaw",
      "display_name": null,
      "total_sessions": 42,
      "total_tool_calls": 1847,
      "total_failures": 23,
      "last_active_at": "2026-05-06T14:23:00Z",
      "first_seen_at": "2026-04-01T09:00:00Z"
    }
  ]
}
```

#### `GET /projects/:id`
Single project detail with tool breakdown.

Response: project object + `{ tool_breakdown: [{ tool_name, count, avg_duration_ms }] }`

#### `GET /projects/:id/sessions`
Sessions for a project, sorted by `started_at DESC`.

Query params: `limit` (default 20), `offset`, `from`, `to`

#### `GET /projects/:id/tool_calls`
All tool calls for a project.

Query params: `limit`, `offset`, `from`, `to`, `tool_name`, `is_failure`

#### `GET /projects/:id/stats`
Time-series stats for charting.

Query params: `from`, `to`, `granularity` (hour|day|week)

Response:
```json
{
  "series": [
    {
      "timestamp": "2026-05-06T00:00:00Z",
      "tool_calls": 47,
      "failures": 2,
      "sessions": 3,
      "p50_duration_ms": 234,
      "p95_duration_ms": 1820
    }
  ]
}
```

---

### Sessions

#### `GET /sessions/:session_id`
Session detail with tool call count.

Response: session object + project + `{ tool_call_count, prompt_count }`

#### `GET /sessions/:session_id/graph`
Full influence graph for a session. Used by the D3 force-directed viewer.

Response:
```json
{
  "session": { "session_id": "...", "started_at": "...", "model": "..." },
  "project": { "folder_path": "...", "repo_name": "..." },
  "nodes": [
    {
      "id": "chunk-uuid",
      "tool_call_id": "uuid",
      "tool_name": "Bash",
      "chunk_index": 0,
      "content": "truncated text...",
      "captured_at": "...",
      "type": "bash"
    }
  ],
  "edges": [
    { "source": "chunk-uuid-a", "target": "chunk-uuid-b", "similarity": 0.82, "is_cross_session": false }
  ]
}
```

#### `GET /sessions/:session_id/timeline`
Chronological list of tool calls with prompts interleaved.

#### `GET /sessions/:session_id/events` (SSE)
Server-sent events stream for a live session. Events emitted as new tool calls arrive.

```
event: node
data: {"type":"tool_call","tool_name":"Bash","captured_at":"...","chunk_count":3}

event: edge
data: {"source":"uuid-a","target":"uuid-b","similarity":0.79}
```

---

### Tool Calls

#### `GET /tool_calls/:id`
Single tool call with all chunks and influence edges.

Response:
```json
{
  "tool_call": {
    "id": "uuid",
    "tool_name": "Bash",
    "tool_input": { "command": "npm test" },
    "tool_response": { "stdout": "...", "stderr": "...", "exit_code": 0 },
    "duration_ms": 3420,
    "captured_at": "2026-05-06T14:23:00Z"
  },
  "project": { ... },
  "session": { ... },
  "chunks": [
    { "id": "uuid", "chunk_index": 0, "content": "...", "token_count": 412 }
  ],
  "outgoing_edges": [
    { "target_chunk_id": "uuid", "similarity": 0.77 }
  ]
}
```

---

### Search

#### `GET /search`
Semantic search across all chunks.

Query params:
- `q` (required): natural language query
- `project`: folder_path or repo_name substring
- `tool_name`: exact tool name
- `from`, `to`: ISO8601 date range
- `limit` (default 10, max 50)

Response:
```json
{
  "query": "what did the npm test return",
  "results": [
    {
      "chunk_id": "uuid",
      "content": "Test Suites: 3 passed...",
      "similarity": 0.91,
      "tool_call": { "id": "...", "tool_name": "Bash", "tool_input": { "command": "npm test" } },
      "project": { "repo_name": "ecoclaw" },
      "session": { "started_at": "..." },
      "captured_at": "2026-05-06T14:23:00Z"
    }
  ]
}
```

---

### Ingestion (called by hooks, not Lovable)

#### `POST /hook/session_start`
Body: raw `SessionStart` hook JSON.

#### `POST /hook/session_end`
Body: raw `SessionEnd` hook JSON.

#### `POST /hook/tool_call`
Body: raw `PostToolUse` hook JSON.

#### `POST /hook/tool_call_failure`
Body: raw `PostToolUseFailure` hook JSON.

#### `POST /hook/prompt`
Body: raw `UserPromptSubmit` hook JSON.

All ingestion endpoints return `202 Accepted` immediately; processing is async.

---

### Stats

#### `GET /stats`
Global system stats.

Response:
```json
{
  "total_projects": 7,
  "total_sessions": 312,
  "total_tool_calls": 18472,
  "total_chunks": 94830,
  "total_influence_edges": 23104,
  "storage_mb": 847,
  "embedding_coverage_pct": 94.2
}
```

---

## 8. Honest Assessment

### Is this worth building from scratch?

**Yes, but don't start from scratch — fork nexus-reasoning-graph.**

The closest existing system is nexus-reasoning-graph, and it's already ~40% of what we need architecturally (hooks, chunking, embeddings, D3 viz, SSE). The jump from nexus-reasoning-graph to the full system is:

1. **Persistence layer:** SQLite → PostgreSQL + pgvector. This is the most important change and touches most of the service code.
2. **Cross-session/project scope:** Replace session-scoped queries with project+date-range queries. Conceptually simple, but requires schema changes.
3. **MCP server layer:** ~200–400 lines of new code (FastMCP or @modelcontextprotocol/sdk). The SQL queries are already implied by the schema.
4. **REST API for Lovable:** ~400–600 lines. Mostly express route handlers wrapping the same SQL queries as MCP.

Total estimated scope for a functional v1: **2–3 days of focused implementation** (not counting the Lovable frontend).

### What could go wrong

- **Embedding model choice:** `all-MiniLM-L6-v2` (384d) is fast and good for code/tool outputs. But it's not trained on code specifically. Consider `nomic-embed-text` or `text-embedding-3-small` for better retrieval quality on code. The schema supports swapping models (chunks store `embedding_model`), but reindexing existing chunks requires re-running the pipeline.
- **HNSW index rebuild cost:** HNSW indexes on `chunks.embedding` need rebuilding when you change the embedding model (different dimensionality). Plan for this with `REINDEX CONCURRENTLY` on a separate maintenance schedule.
- **Tool response size:** Some tool responses are very large (e.g., `Read` on a 10,000-line file). The `tool_response` JSONB column has no size limit. Consider truncating responses > 100KB before storage (store a `truncated: true` flag).
- **Hook async failures:** Since all hooks are `async: true`, network failures are silent. The capture service should expose a `GET /health` endpoint and a dead letter queue (write to a local log file when Postgres is unavailable).
- **CWD resolution edge cases:** Subagents run with the same `cwd` as the parent. Worktrees have different CWDs. Both are correct behavior — worktree tool calls should be tagged to the worktree's folder, which may be a different project row from the parent.

### Build recommendation

**Phase 1 (v1 — core capture):**
- Fork nexus-reasoning-graph
- Replace SQLite with PostgreSQL + pgvector (schema above)
- Add project resolution from CWD
- Add `PostToolUseFailure` capture
- Add `SessionStart`/`SessionEnd` lifecycle

**Phase 2 (v1.1 — MCP + cross-session):**
- Add MCP server (7 tools above)
- Add nightly cross-session influence edge computation
- Add `GET /search` endpoint

**Phase 3 (v1.2 — Lovable REST API):**
- Add all REST endpoints documented in Section 7
- Add OpenAPI spec generation
- Harden ingestion with local fallback log

**Do not build a Graphiti-style temporal knowledge graph.** For the queries we need ("show me all Bash runs in ecoclaw from last week"), relational SQL with a GIN index on `tool_input` and a vector index on `chunks.embedding` is faster to build, easier to query, and simpler to operate than a Neo4j-backed graph extraction pipeline.

---

*Research sources: mem0ai/mem0, getzep/zep, getzep/graphiti, letta-ai/letta, Gonzih/nexus-reasoning-graph, docs.anthropic.com/en/docs/claude-code/hooks, pgvector/pgvector, qdrant/qdrant — all fetched 2026-05-06.*
