# Temporal Database for Agentic Systems
## A Datomic-Inspired Design

*Research document — May 2026*

---

## Table of Contents

1. [Datomic Model Explained](#1-datomic-model-explained)
2. [What to Steal](#2-what-to-steal)
3. [What to Drop](#3-what-to-drop)
4. [Existing Implementations](#4-existing-implementations)
5. [Agent-Specific Simplifications](#5-agent-specific-simplifications)
6. [Proposed Data Model](#6-proposed-data-model)
7. [MCP Tool Spec](#7-mcp-tool-spec)
8. [Entity ID Design](#8-entity-id-design)
9. [Time-Travel Query Examples](#9-time-travel-query-examples)
10. [Build Assessment](#10-build-assessment)

---

## 1. Datomic Model Explained

### The Datom: Atomic Unit of Fact

Datomic stores data as **datoms** — the smallest indivisible unit of fact. A datom is a 5-tuple:

```
[Entity  Attribute  Value  Transaction  Operation]
 E        A          V      Tx           Op (assert/retract)
```

Example datoms:
```
[42   :project/name    "ecoclaw"    tx-101  true ]   ; assertion
[42   :project/status  "planning"   tx-101  true ]   ; assertion
[42   :project/status  "planning"   tx-203  false]   ; retraction
[42   :project/status  "active"     tx-203  true ]   ; new assertion
```

Nothing is ever overwritten. When a fact changes, the old datom is retracted (Op=false) and a new datom is asserted (Op=true). The database only grows; it never loses information.

### The Database as an Append-Only Ledger

A Datomic database is the complete, immutable set of all datoms ever transacted. You can think of it as a ledger where every line is a timestamped fact, and crossing something out doesn't erase it — it adds a new line saying "this was crossed out at time T."

This isn't just philosophical. It has concrete operational consequences:
- Any past state of the database is trivially reconstructable (filter by `Tx ≤ T`)
- Reads never block writes (immutable snapshots can be distributed freely)
- No cache invalidation — a database value at time T is forever valid at time T
- The full audit history is structural, not bolted on

### Four Built-In Indexes

Datomic maintains four indexes, each sorted differently to serve different access patterns:

| Index | Sort Order      | Best For                                      |
|-------|-----------------|-----------------------------------------------|
| EAVT  | Entity, Attr, Val, Tx | "Row" access — all facts about one entity |
| AEVT  | Attr, Entity, Val, Tx | "Column" access — all values of an attribute |
| AVET  | Attr, Val, Entity, Tx | Key-value lookup, range scans                |
| VAET  | Val, Attr, Entity, Tx | Reverse reference / graph traversal          |

Every write updates all four indexes. Every read hits exactly the right index for the query pattern.

### Schema as Data

Schema in Datomic is not DDL — it's regular transaction data. You define an attribute by transacting an entity with specific meta-attributes:

```clojure
{:db/ident       :project/name    ; unique keyword name for this attribute
 :db/valueType   :db.type/string  ; string, long, ref, uuid, instant, boolean, ...
 :db/cardinality :db.cardinality/one  ; one value or many (set)
 :db/doc         "Human-readable project name"}
```

Transacting this schema entity creates the `:project/name` attribute. From then on, any entity can have a `:project/name` datom. There's no table to add a column to — you just assert new attributes.

**This means schema evolution is a write, not a migration.** Adding a new attribute doesn't touch existing data; it just makes that attribute available for future assertions.

### Transactions as First-Class Entities

Every transaction is itself an entity in the database. When a transaction commits, Datomic automatically asserts:

```
[tx-entity-id  :db/txInstant  #inst "2026-05-07T14:23:00.000Z"  tx-entity-id  true]
```

You can add arbitrary datoms to the transaction entity for provenance:

```clojure
{:db/id "datomic.tx"  ; special tempid targeting the transaction entity itself
 :agent/session-id  "session-abc-123"
 :agent/tool-name   "write_file"
 :agent/rationale   "User asked to create project plan"}
```

Now every fact in the database carries a link to the transaction that created it, and that transaction carries full agent provenance.

### Time-Travel Queries

Datomic exposes three time-filtering views of the database:

**`as-of(t)`** — Returns a database value containing only datoms asserted at or before time `t`. All queries run unchanged against this view. You see the world as it was at `t`.

```clojure
(d/q '[:find ?status :where [?e :project/name "ecoclaw"] [?e :project/status ?status]]
     (d/as-of db #inst "2026-03-01"))
;; => #{["planning"]}   -- even though status is now "active"
```

**`since(t)`** — Returns only datoms asserted *after* time `t`. Useful for "what changed since my last sync?"

**`history`** — Returns a special view containing ALL datoms, including retracted ones across all time. The Op component (assertion vs. retraction) is visible in queries. Use this to reconstruct the full arc of how any fact evolved.

```clojure
(d/q '[:find ?status ?tx ?op
       :where [?e :project/name "ecoclaw"] [?e :project/status ?status ?tx ?op]]
     (d/history db))
;; => #{["planning" tx-101 true] ["planning" tx-203 false] ["active" tx-203 true]}
```

These APIs require no schema changes or query syntax extension. They're just database-value filters you pass to the existing query engine.

---

## 2. What to Steal

The following Datomic concepts are worth preserving wholesale.

### Immutable, Append-Only Facts

The core insight: facts should never be destroyed, only superseded. When an agent updates its belief about the world, the old belief doesn't disappear — it becomes part of the historical record. This is exactly what agents need: the ability to say "I believed X at time T, and I was wrong by time T+1."

### EAV(T) Data Model

Entity-Attribute-Value with a transaction timestamp is the right primitive for agent world-state. It's:
- **Flexible**: any entity can have any attribute without schema migration
- **Sparse**: entities only carry attributes that have been asserted — no NULL fields
- **Auditable**: the T component makes every fact traceable to a transaction
- **Composable**: Datalog queries join implicitly by sharing variables

### Transaction Entities with Provenance

Every write being linked to a transaction entity that carries arbitrary metadata is essential for agents. An agent shouldn't just write a fact — it should write a fact *with context*: which session, which tool call, which prompt, what the agent's stated reasoning was. This is free in Datomic's model; it requires bolt-on instrumentation in relational systems.

### `as-of` / `since` / `history` Semantics

These three query modifiers are the temporal superpowers that make the model uniquely valuable for agents:
- `as-of(t)`: replay what the agent knew at decision time
- `since(t)`: diff the world since last sync or last agent run
- `history(entity, attribute)`: full arc of belief about any specific fact

### Schema as Data, Schema Evolution as Writes

Adding new attributes by writing entities rather than running DDL means an agent can extend the schema by calling the same `transact` tool it uses for everything else. No special schema migration path; no "ALTER TABLE" permission issues.

---

## 3. What to Drop

These Datomic features are designed for large distributed systems and have no place in a single-agent or small-team deployment.

### The Peer/Transactor/Storage Architecture

Datomic's production architecture involves multiple processes:
- **Transactor**: a single serializing process that accepts all writes, enforces ACID, writes to storage
- **Peers**: application processes that embed the full query engine and maintain an in-process cache of the database
- **Storage**: a pluggable durable backend (DynamoDB, Cassandra, PostgreSQL, etc.)

This architecture is designed to scale read throughput to thousands of peer processes while maintaining write serialization. For a single agent or a small agent fleet, it's massively over-engineered. We need one process that reads and writes.

### The Peer Library

Peers embed the full Datomic query engine in the application process. This makes queries fast (no network hop for reads) but requires distributing the Datomic JAR and managing the peer cache. For an MCP-accessed database, there's no peer — there's just the MCP server process. Eliminate this entirely.

### Distributed Storage Backends

Datomic supports DynamoDB, Cassandra, SQL, and several others as its durable storage layer. This pluggability is valuable at scale; for our use case, PostgreSQL is the backend and that's the only one we need to support.

### Datalog Query Language (as a user-facing API)

Datomic's Datalog query syntax is powerful but has a steep learning curve and requires Clojure or a compatible runtime. Since our only clients are AI agents communicating via MCP tool calls, we don't need a general-purpose query language. We expose specific, named MCP tools for the query patterns agents actually use. The Datalog engine lives inside the server — agents never write Datalog directly.

### The VAET Index

The reverse-reference index (Value→Attribute→Entity→Tx) is used for graph traversal: "which entities point to entity X?" For our initial use case, forward references and explicit query tools are sufficient. This index can be added later if graph traversal becomes a common pattern.

### Large-Value Storage

Datomic explicitly refuses to store large binaries (images, audio, large files) — it stores references to external storage (S3, etc.). We inherit this constraint: the temporal DB stores facts about things (URLs, paths, identifiers, structured metadata), not the things themselves.

---

## 4. Existing Implementations

### DataScript (`tonsky/datascript`)

**What it is**: An in-memory Datalog database and query engine designed for the browser (ClojureScript) and JVM. "Creating a database is as cheap as creating a HashMap." Used in production by Roam Research, LogSeq, and Athens — the most prominent second-brain tools.

**What it keeps**: EAV triples, EAVT/AEVT/AVET indexes, full Datalog query engine (joins, recursive rules, aggregates, negation, disjunction), Pull API, cardinality, unique constraints, upsert, immutable database-as-value semantics.

**What it drops**: Persistence (ephemeral by design), transaction history and time-travel (no `as-of`/`since`/`history`), `:db/valueType` enforcement (any type valid for any attribute), schema queryability.

**What it proves**: You can strip Datomic to just the query model and immutable data structures and get an extremely useful client-side state manager. The Datomic query surface is the core value; everything else is optional infrastructure.

**Relevance to our design**: DataScript's simplified schema model (optional, not enforced) is worth copying. Its lack of persistence and time-travel is intentional for its use case; for agent memory, we need both.

### Datahike (`replikativ/datahike`)

**What it is**: A durable, versioned Datalog database with full Datomic-compatible APIs, "git-like semantics," and pluggable storage. Positioned as a drop-in Datomic replacement. Used in production by Sweden's Public Employment Service (Arbetsförmedlingen), serving a 40,000-concept taxonomy to thousands of caseworkers.

**Storage backends**: Pluggable via `konserve` abstraction — filesystem, LMDB, S3, JDBC, Redis, IndexedDB, memory.

**What it keeps from Datomic**: Full history and time-travel (`d/history`, point-in-time queries), append-only fact log, immutable database snapshots, full Datalog query engine, GDPR-compliant data excision (with audit trail intact).

**What it adds**: `Proximum` — an HNSW vector index add-on for semantic search / RAG (immutable, persistent). Clojure, ClojureScript, JavaScript, Java, Python APIs. Native CLI via GraalVM.

**Stats**: 1,800 stars, 174 releases, latest release May 6, 2026 (extremely active).

**Relevance**: The only open-source implementation with the full Datomic semantic model *and* pluggable storage, actively maintained. If we were building in Clojure, this would be the foundation. Its pluggable-storage architecture (`konserve`) is the right design pattern for our PostgreSQL backend abstraction.

### Datalevin (`juji-io/datalevin`)

**What it is**: A "simple, fast and versatile" durable Datalog database built at Juji for conversational AI. Takes the opposite philosophical stance from Datomic: **when data is deleted, it is gone.** Backed exclusively by LMDB.

**What it drops (deliberately)**: Temporal history — Datalevin explicitly rejects Datomic's time model as "confusing and unusual." No `as-of`, no `since`, no `history`.

**What it adds beyond Datomic**: Cost-based query optimizer (benchmarks 13-18x faster than DataScript on multi-way joins, ~2.4x faster than PostgreSQL on complex join benchmark), full-text search (T-Wand algorithm), SIMD-accelerated vector search (via `usearch`), in-DB embedding/OCR/generation (via `llama.cpp`), document database mode, networked client/server with Raft HA, Babashka pod.

**Most important for us**: **Built-in MCP server** — AI agents can query Datalevin over MCP. This is the only project among all three that ships MCP integration.

**Relevance**: Datalevin proves that dropping the temporal model and adding a serious query optimizer produces something competitive with SQL on raw performance. Its MCP server proves the demand is real. Our design inverts the trade-off Datalevin made: we keep the temporal model (it's our whole point) and add MCP. We skip the query optimizer (PostgreSQL provides it).

### Prior Art: Temporal Agent Databases

**`theronic/datomic-mcp`** (26 stars, Clojure, Apr 2025) and **`xlisp/datomic-mcp-server`** (9 stars, Clojure, Nov 2025): Two embryonic Datomic MCP servers exist. Neither surfaces temporal tools (`as-of`, `since`, `history`, transaction log) as first-class MCP capabilities. They treat Datomic as a plain query endpoint.

**`aexy-io/graphzep`** (TypeScript): A port of Zep's temporal knowledge graph. Graph edges with timestamps and fact invalidation — closest to Datomic-style thinking but built on a property graph, not EAV. Focuses on conversation/session memory only.

**`caudal-labs/caudal`** (Java/Spring Boot): "Temporal relevance memory for AI agents." Computes attention signals from recency + interaction frequency over PostgreSQL. Closer to a temporal decay index than a true temporal database — no time-travel queries.

**The gap**: No project exposes EAV temporal time-travel as first-class MCP tools. The open-source Datalevin (best open EAV database) has no temporal model. The Datomic MCP servers don't exploit temporal semantics. This is the gap our system fills.

---

## 5. Agent-Specific Simplifications

When the only client is an AI agent communicating via MCP, several things become much simpler.

### No Query Language Required

Agents don't write Datalog. They call named tools: `get_entity`, `find_entities`, `query_history`. The server translates these into SQL against the EAV(T) table. The agent's "query language" is the set of MCP tool signatures. This is strictly simpler for the agent (no Datalog to learn) and for the server (no query parser to write).

### Schema Is Optional and Inferred

Agents shouldn't need to run a schema migration before writing a new fact. If an agent asserts `("project:ecoclaw", "budget_usd", 50000)`, the attribute `budget_usd` is implicitly created. We store all values as JSONB (covering strings, numbers, booleans, arrays, objects) — no `:db/valueType` enforcement needed. Schema remains explicit for agents that want it (for documentation and validation), but is never required.

### Entity IDs Are Human-Readable Strings

A numeric auto-increment entity ID is meaningless to an agent. An agent knows "project:ecoclaw" and "session:abc-123" — it doesn't know internal ID 42. We use namespaced string identifiers (`namespace:local-name`) as primary keys, with optional UUID aliases for programmatic generation. See Section 8 for full design.

### Provenance Is Automatic

Every MCP tool call has context: session ID, timestamp, the tool name. The MCP server can inject this into the transaction entity automatically — the agent doesn't need to pass it explicitly. Every fact written through MCP carries full provenance with zero agent effort.

### Write Model Matches Agent Mental Model

Agents think in terms of "I now know that X" (assertion) and "X is no longer true" (retraction), not INSERT/UPDATE/DELETE. The `assert` and `retract` tools match how agents naturally express belief updates. The database handles the immutability mechanics invisibly.

### Multi-Agent Isolation via Namespaces

Different agents or sessions can use the same entity IDs without conflict by prefixing them with session/agent namespaces: `session:abc-123:entity:foo` vs `global:entity:foo`. The server doesn't enforce this, but the naming convention is a natural fit for the namespaced ID system.

---

## 6. Proposed Data Model

### Core Tables

#### `datoms` — The Fact Ledger

```sql
CREATE TABLE datoms (
  id        BIGSERIAL PRIMARY KEY,
  entity_id TEXT        NOT NULL,           -- namespaced string: "project:ecoclaw"
  attribute TEXT        NOT NULL,           -- attribute name: "status"
  value     JSONB       NOT NULL,           -- any JSON value: "active", 42, true, {...}
  tx_id     BIGINT      NOT NULL REFERENCES transactions(id),
  op        BOOLEAN     NOT NULL DEFAULT TRUE -- TRUE=assert, FALSE=retract
);
```

Every row is one datom. The table only grows. No UPDATE, no DELETE. The `op` column distinguishes assertions from retractions.

#### `transactions` — The Transaction Log

```sql
CREATE TABLE transactions (
  id         BIGSERIAL   PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  session_id TEXT,                  -- agent session ID (auto-injected from MCP context)
  agent_id   TEXT,                  -- agent identifier
  tool_name  TEXT,                  -- which MCP tool initiated this tx
  note       TEXT                   -- optional agent rationale / human-readable note
);
```

Every write to `datoms` references a row in `transactions`. This is the transaction entity — the provenance record for a batch of related facts.

#### `attributes` — Optional Schema Registry

```sql
CREATE TABLE attributes (
  name        TEXT PRIMARY KEY,
  value_type  TEXT,           -- 'string' | 'number' | 'boolean' | 'json' | 'ref' | null (any)
  cardinality TEXT NOT NULL DEFAULT 'one', -- 'one' | 'many'
  description TEXT,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Schema is optional — facts can be written for undeclared attributes. Declared attributes get documentation and optional type validation at write time.

### Indexes

The four Datomic indexes translated to PostgreSQL:

```sql
-- EAVT: "What are all the facts about entity X?" (most common agent query)
CREATE INDEX idx_datoms_eavt ON datoms (entity_id, attribute, value, tx_id);

-- AEVT: "What values does attribute A have across all entities?"
CREATE INDEX idx_datoms_aevt ON datoms (attribute, entity_id, value, tx_id);

-- AVET: "Which entity has attribute A = value V?" (lookup by value)
CREATE INDEX idx_datoms_avet ON datoms (attribute, (value::text), entity_id, tx_id);

-- Time index: "What happened in transaction range [T1, T2]?"
CREATE INDEX idx_datoms_tx ON datoms (tx_id, entity_id, attribute);

-- Transaction timestamp: "What was the state as of wall-clock time T?"
CREATE INDEX idx_transactions_time ON transactions (created_at);

-- Full-text and JSONB containment search
CREATE INDEX idx_datoms_value_gin ON datoms USING GIN (value);
```

### Current-State View

For queries that don't need history, a view of current facts is useful:

```sql
CREATE VIEW current_datoms AS
SELECT DISTINCT ON (entity_id, attribute)
  entity_id, attribute, value, tx_id
FROM datoms
WHERE op = TRUE
ORDER BY entity_id, attribute, tx_id DESC;
```

This view returns the most recently asserted value for each (entity, attribute) pair, filtering out retracted facts. For cardinality-many attributes, this requires a different approach (aggregate all asserted, subtract all retracted).

### pgvector Extension (Semantic Search)

For embedding-based lookup of entities by semantic similarity:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE entity_embeddings (
  entity_id  TEXT PRIMARY KEY,
  embedding  VECTOR(1536),    -- OpenAI text-embedding-3-small dimensions
  summary    TEXT,            -- the text that was embedded
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_entity_embeddings_hnsw
  ON entity_embeddings USING hnsw (embedding vector_cosine_ops);
```

The MCP server generates and updates embeddings from entity attribute values when requested.

---

## 7. MCP Tool Spec

### Write Tools

#### `assert_fact`
Assert a single fact about an entity. If the entity already has a value for this attribute (cardinality-one), the old value is automatically retracted in the same transaction.

```typescript
{
  name: "assert_fact",
  description: "Assert a fact about an entity. Records 'entity has attribute = value' as of now.",
  inputSchema: {
    entity_id: { type: "string", description: "Namespaced entity ID, e.g. 'project:ecoclaw'" },
    attribute:  { type: "string", description: "Attribute name, e.g. 'status'" },
    value:      { description: "Any JSON-compatible value: string, number, boolean, array, or object" },
    note:       { type: "string", description: "Optional rationale for this assertion" }
  }
}
```

#### `assert_facts`
Transact multiple facts atomically. All assertions and retractions in the call succeed together or none do.

```typescript
{
  name: "assert_facts",
  description: "Transact multiple fact assertions and/or retractions atomically.",
  inputSchema: {
    facts: {
      type: "array",
      items: {
        op:         { enum: ["assert", "retract"] },
        entity_id:  { type: "string" },
        attribute:  { type: "string" },
        value:      { description: "Required for assert, optional for retract (retracts all if omitted)" }
      }
    },
    note: { type: "string", description: "Optional note for the entire transaction" }
  }
}
```

#### `retract_fact`
Explicitly retract a fact. The datom is preserved in history; it just stops appearing in current-state queries.

```typescript
{
  name: "retract_fact",
  description: "Mark a fact as no longer true. The historical record is preserved.",
  inputSchema: {
    entity_id:  { type: "string" },
    attribute:  { type: "string" },
    value:      { description: "Specific value to retract. If omitted, retracts all values for this attribute." },
    note:       { type: "string" }
  }
}
```

#### `retract_entity`
Retract all current facts about an entity. Entity remains in history; future queries for its past state still work.

```typescript
{
  name: "retract_entity",
  description: "Retract all current facts about an entity. History is preserved.",
  inputSchema: {
    entity_id: { type: "string" },
    note:      { type: "string" }
  }
}
```

### Read Tools (Current State)

#### `get_entity`
Return all current (non-retracted) facts about an entity.

```typescript
{
  name: "get_entity",
  description: "Get all current facts about an entity as a key-value map.",
  inputSchema: {
    entity_id: { type: "string" }
  },
  // Returns: { "status": "active", "name": "ecoclaw", "created_at": "2026-01-15" }
}
```

#### `get_attribute`
Return the current value of a specific attribute for an entity.

```typescript
{
  name: "get_attribute",
  description: "Get the current value of a specific attribute for an entity.",
  inputSchema: {
    entity_id:  { type: "string" },
    attribute:  { type: "string" }
  }
  // Returns: { "value": "active", "asserted_at": "2026-04-01T10:00:00Z", "tx_id": 203 }
}
```

#### `find_entities`
Find all entities where a given attribute equals a given value.

```typescript
{
  name: "find_entities",
  description: "Find all entities that currently have attribute = value.",
  inputSchema: {
    attribute: { type: "string" },
    value:     { description: "Value to match (exact match)" },
    limit:     { type: "number", default: 50 }
  }
  // Returns: [{ entity_id: "project:ecoclaw", ... }, ...]
}
```

#### `find_entities_with_attribute`
Find all entities that have any value for a given attribute.

```typescript
{
  name: "find_entities_with_attribute",
  description: "Find all entities that currently have the specified attribute (any value).",
  inputSchema: {
    attribute: { type: "string" },
    limit:     { type: "number", default: 50 }
  }
}
```

#### `search_entities`
Semantic search over entity attribute values via pgvector.

```typescript
{
  name: "search_entities",
  description: "Find entities semantically similar to a query string.",
  inputSchema: {
    query:      { type: "string" },
    limit:      { type: "number", default: 10 },
    attributes: { type: "array", items: { type: "string" }, description: "Restrict to these attributes" }
  }
}
```

### Read Tools (Time-Travel)

#### `get_entity_as_of`
Return the state of an entity at a specific point in time.

```typescript
{
  name: "get_entity_as_of",
  description: "Get all facts about an entity as they were at a specific time. Returns the entity's state as of that moment.",
  inputSchema: {
    entity_id:  { type: "string" },
    timestamp:  { type: "string", description: "ISO 8601 timestamp, e.g. '2026-03-01T00:00:00Z'" }
  }
}
```

#### `get_attribute_history`
Return the complete history of values an attribute has had for an entity.

```typescript
{
  name: "get_attribute_history",
  description: "Return the full timeline of values an attribute has held for an entity, including retractions.",
  inputSchema: {
    entity_id:  { type: "string" },
    attribute:  { type: "string" }
  }
  // Returns: [
  //   { op: "assert", value: "planning", timestamp: "2026-01-15T...", tx_id: 101, session_id: "..." },
  //   { op: "retract", value: "planning", timestamp: "2026-04-01T...", tx_id: 203 },
  //   { op: "assert", value: "active", timestamp: "2026-04-01T...", tx_id: 203 }
  // ]
}
```

#### `get_entity_history`
Return all datoms (assertions and retractions) ever written about an entity.

```typescript
{
  name: "get_entity_history",
  description: "Return every fact ever asserted or retracted about an entity, in chronological order.",
  inputSchema: {
    entity_id:   { type: "string" },
    since:       { type: "string", description: "Optional: only return history after this timestamp" },
    until:       { type: "string", description: "Optional: only return history before this timestamp" }
  }
}
```

#### `get_changes_since`
Return all datoms written to the database since a given time (the `since` analog).

```typescript
{
  name: "get_changes_since",
  description: "Return all facts asserted or retracted since a given timestamp. Useful for syncing or diffing agent state.",
  inputSchema: {
    timestamp:  { type: "string" },
    entity_ids: { type: "array", items: { type: "string" }, description: "Optional: restrict to these entities" },
    attributes: { type: "array", items: { type: "string" }, description: "Optional: restrict to these attributes" },
    limit:      { type: "number", default: 500 }
  }
}
```

#### `get_transaction`
Return the details of a specific transaction and all datoms it produced.

```typescript
{
  name: "get_transaction",
  description: "Return a transaction's metadata and all facts it wrote.",
  inputSchema: {
    tx_id: { type: "number" }
  }
  // Returns: { tx_id, timestamp, session_id, agent_id, tool_name, note, datoms: [...] }
}
```

#### `get_recent_transactions`
Return recent transactions with their provenance metadata.

```typescript
{
  name: "get_recent_transactions",
  description: "Return recent transactions, optionally filtered by session or agent.",
  inputSchema: {
    limit:      { type: "number", default: 20 },
    session_id: { type: "string", description: "Filter to a specific session" },
    since:      { type: "string", description: "Only return transactions after this timestamp" }
  }
}
```

### Schema Tools

#### `declare_attribute`
Register an attribute with type and cardinality. Optional — attributes are usable without registration.

```typescript
{
  name: "declare_attribute",
  description: "Register an attribute with its type, cardinality, and documentation. Optional — attributes work without this.",
  inputSchema: {
    name:        { type: "string", description: "Attribute name, e.g. 'status'" },
    value_type:  { enum: ["string", "number", "boolean", "json", "ref"], description: "Expected value type" },
    cardinality: { enum: ["one", "many"], default: "one" },
    description: { type: "string" }
  }
}
```

#### `list_attributes`
Return all declared attributes.

```typescript
{
  name: "list_attributes",
  description: "Return all declared attributes with their types and documentation.",
  inputSchema: {}
}
```

---

## 8. Entity ID Design

### The Problem with UUIDs

UUIDs are the obvious choice for entity IDs in a database — they're collision-resistant and require no coordination. But they're opaque to agents. An agent that stores `project:ecoclaw` data in transaction 42 and gets back entity ID `f47ac10b-58cc-4372-a567-0e02b2c3d479` has lost the semantic connection. To retrieve the entity later, it needs to remember an opaque identifier.

UUIDs also make debugging harder: inspecting the database, you see `f47ac10b-...` instead of `project:ecoclaw`. The human-readable name is gone.

### Namespaced String IDs

The better design for agent use cases: **namespaced string identifiers** of the form `namespace:local-name`.

Examples:
```
project:ecoclaw
project:nexus-research
session:abc-123
session:abc-123:task:plan-phase-1
agent:claude-opus-4
user:gonzih
concept:temporal-database
decision:2026-05-07:use-postgresql
```

**Benefits**:
- An agent can reference the same entity across sessions without any lookup — it knows the name
- Human-readable in the database, in logs, in queries
- Namespaces provide natural grouping (all `project:*` entities, all `session:*` entities)
- Hierarchical structure via nested namespaces: `session:X:task:Y` is a task within session X
- Collision resistance comes from namespace discipline, not cryptography

**Constraints**:
- Must be valid for `LIKE` pattern matching (no SQL-injection characters)
- Server validates: only alphanumeric, `-`, `_`, `:`, `.` characters
- Recommended max length: 256 characters
- Namespaces are user-defined; the server doesn't enforce a namespace registry

### UUID Aliases

For programmatic entity generation where a human-readable name isn't available, agents can use UUID-style IDs with a meaningful prefix:

```
event:f47ac10b-58cc-4372-a567-0e02b2c3d479
observation:3b1a8f2c-1234-5678-abcd-ef0123456789
```

The prefix makes the entity type clear while the UUID provides uniqueness. The server treats these identically to named IDs.

### Entity ID Conventions (Recommended, Not Enforced)

| Namespace       | Description                                | Example                         |
|----------------|--------------------------------------------|---------------------------------|
| `project:`     | Long-lived project entities                | `project:ecoclaw`               |
| `session:`     | Agent session contexts                     | `session:abc-123`               |
| `task:`        | Discrete tasks / work items                | `task:implement-auth`           |
| `decision:`    | Recorded decisions with rationale          | `decision:2026-05-07:db-choice` |
| `concept:`     | Domain concepts / knowledge entries        | `concept:temporal-database`     |
| `user:`        | Human users                                | `user:gonzih`                   |
| `agent:`       | AI agent instances or roles                | `agent:planner`                 |
| `file:`        | File references                            | `file:src/main.ts`              |
| `event:`       | Discrete events (UUID suffix OK)           | `event:f47ac10b-...`            |

### Why Not Lookup Refs?

Datomic supports "lookup refs" — `[:attribute value]` pairs that resolve to entity IDs using unique attributes. This requires declaring attributes as `:db/unique`, which is fine for a full Datomic system but adds complexity. For our design, the entity ID *is* the primary key — no lookup ref indirection needed. Agents address entities by name directly.

---

## 9. Time-Travel Query Examples

These are concrete SQL queries behind the MCP tool implementations, illustrating how the EAV(T) model enables temporal queries with straightforward SQL.

### Current State: Get Entity

```sql
-- "What are all the current facts about project:ecoclaw?"
SELECT DISTINCT ON (attribute)
  attribute, value, t.created_at, d.tx_id
FROM datoms d
JOIN transactions t ON d.tx_id = t.id
WHERE d.entity_id = 'project:ecoclaw'
  AND d.op = TRUE
ORDER BY attribute, d.tx_id DESC;
```

### As-Of: Entity State at a Past Time

```sql
-- "What did we know about project:ecoclaw on March 1, 2026?"
SELECT DISTINCT ON (attribute)
  attribute, value
FROM datoms d
JOIN transactions t ON d.tx_id = t.id
WHERE d.entity_id = 'project:ecoclaw'
  AND t.created_at <= '2026-03-01T00:00:00Z'
ORDER BY attribute, d.tx_id DESC
-- Then filter to only rows where the last op was TRUE (asserted, not retracted)
-- Wrap in outer query:
```

Full as-of query:
```sql
WITH history AS (
  SELECT DISTINCT ON (attribute)
    attribute, value, op
  FROM datoms d
  JOIN transactions t ON d.tx_id = t.id
  WHERE d.entity_id = 'project:ecoclaw'
    AND t.created_at <= '2026-03-01T00:00:00Z'
  ORDER BY attribute, d.tx_id DESC
)
SELECT attribute, value FROM history WHERE op = TRUE;
```

### Attribute History: Full Timeline

```sql
-- "How has project:ecoclaw's status changed over time?"
SELECT
  d.op,
  d.value,
  t.created_at,
  t.tx_id,
  t.session_id,
  t.note
FROM datoms d
JOIN transactions t ON d.tx_id = t.id
WHERE d.entity_id = 'project:ecoclaw'
  AND d.attribute = 'status'
ORDER BY d.tx_id ASC;

-- Returns:
-- (true,  "planning", 2026-01-15, tx-101, session-A, "Project started")
-- (false, "planning", 2026-04-01, tx-203, session-B, "Status updated after review")
-- (true,  "active",   2026-04-01, tx-203, session-B, "Status updated after review")
-- (false, "active",   2026-05-01, tx-301, session-C, "Project on hold")
-- (true,  "paused",   2026-05-01, tx-301, session-C, "Project on hold")
```

### Since: What Changed After Last Sync

```sql
-- "What changed in the database since session-B's last sync at 2026-04-01?"
SELECT d.entity_id, d.attribute, d.value, d.op, t.created_at, t.session_id
FROM datoms d
JOIN transactions t ON d.tx_id = t.id
WHERE t.created_at > '2026-04-01T12:00:00Z'
ORDER BY t.created_at ASC;
```

### Find Entities: Attribute-Value Lookup

```sql
-- "Which projects are currently active?"
SELECT DISTINCT ON (entity_id)
  entity_id
FROM datoms d
JOIN transactions t ON d.tx_id = t.id
WHERE d.attribute = 'status'
ORDER BY entity_id, d.tx_id DESC
-- Outer query filters to op=TRUE and value='active'
```

Full query:
```sql
WITH latest AS (
  SELECT DISTINCT ON (entity_id)
    entity_id, value, op
  FROM datoms
  WHERE attribute = 'status'
  ORDER BY entity_id, tx_id DESC
)
SELECT entity_id FROM latest
WHERE op = TRUE AND value = '"active"';  -- JSONB: string values are quoted
```

### Transaction Replay: What Did Agent See?

```sql
-- "What was session-B's decision context? Replay what it knew at the time of transaction 203."
SELECT t.created_at, t.session_id, t.agent_id, t.note,
       d.entity_id, d.attribute, d.value, d.op
FROM transactions t
JOIN datoms d ON d.tx_id = t.id
WHERE t.id = 203;

-- Then to get the full world-state the agent saw BEFORE that transaction:
-- Use as-of query with timestamp = (SELECT created_at FROM transactions WHERE id = 203) - 1ms
```

### Semantic Search via pgvector

```sql
-- "Find entities related to 'machine learning deployment'"
SELECT e.entity_id, e.summary, 1 - (e.embedding <=> $1::vector) AS similarity
FROM entity_embeddings e
ORDER BY e.embedding <=> $1::vector
LIMIT 10;
-- $1 = embedding vector for the query text
```

---

## 10. Build Assessment

### How Hard Is This?

**Short answer**: Medium complexity. The data model is simple; the implementation surface is the MCP server + SQL query generation. Estimated 2-3 weeks for a solid MVP that an agent can actually use productively.

### MVP Scope (v0.1)

The minimum viable system an agent can use day-one:

1. **PostgreSQL schema**: `datoms` + `transactions` tables, EAVT/AEVT/AVET indexes, `current_datoms` view
2. **MCP server** (Node.js + TypeScript): `@modelcontextprotocol/sdk`, `pg` for PostgreSQL
3. **Write tools**: `assert_fact`, `assert_facts`, `retract_fact`
4. **Read tools**: `get_entity`, `get_attribute`, `find_entities`
5. **Temporal tools**: `get_attribute_history`, `get_entity_as_of`, `get_changes_since`
6. **Schema tools**: `declare_attribute`, `list_attributes`

That's 11 tools, which is enough for an agent to use the system meaningfully. Semantic search and pgvector are nice-to-have for v0.2.

### Technology Stack

| Layer       | Choice          | Rationale                                                   |
|-------------|-----------------|-------------------------------------------------------------|
| Database    | PostgreSQL 16+  | ACID, JSONB, window functions, `DISTINCT ON`, pgvector       |
| MCP server  | Node.js + TypeScript | `@modelcontextprotocol/sdk` is the reference implementation |
| DB driver   | `pg` (node-postgres) | Mature, no ORM overhead, parameterized queries              |
| MCP transport | stdio or HTTP SSE | Stdio for local Claude Code; HTTP for hosted deployment  |
| Embeddings  | OpenAI `text-embedding-3-small` | 1536 dims, fast, cheap; or local via Ollama  |

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| `DISTINCT ON` performance on large tables | EAVT index covers the `ORDER BY entity_id, tx_id DESC` pattern; benchmark at 10M datoms |
| JSONB equality matching for `find_entities` | Cast to `::text` for exact match; GIN index for containment queries |
| cardinality-many attributes tricky in current-state view | Separate query path: aggregate all asserts, subtract all retracts |
| MCP tool call explosion (agent loops) | Rate limit per session; add `get_recent_transactions` so agents can audit their own writes |
| Entity ID conflicts between agents | Namespace convention; consider optional namespace locking per session |

### Build Phases

**Phase 1 — Core (2 weeks)**
- PostgreSQL schema and migration
- MCP server scaffold with stdio transport
- Write tools: `assert_fact`, `assert_facts`, `retract_fact`, `retract_entity`
- Read tools: `get_entity`, `get_attribute`, `find_entities`
- Basic temporal: `get_attribute_history`, `get_entity_as_of`

**Phase 2 — Temporal Completeness (1 week)**
- `get_entity_history`, `get_changes_since`, `get_transaction`, `get_recent_transactions`
- Schema tools: `declare_attribute`, `list_attributes`
- Provenance auto-injection from MCP session context

**Phase 3 — Semantic Search (1 week)**
- pgvector extension setup
- Embedding generation on entity write (background job or sync)
- `search_entities` MCP tool
- HTTP SSE transport for hosted deployment

**Phase 4 — Production Hardening**
- Connection pooling (PgBouncer or `pg-pool`)
- Transaction size limits (max datoms per transaction)
- Query timeouts
- `VACUUM` / `ANALYZE` scheduling (the table only grows)
- Monitoring: datom count, tx rate, query latency

### Why This Beats Alternatives

| Alternative | Problem for Agents |
|-------------|-------------------|
| Plain PostgreSQL with JSONB | No temporal model — UPDATE destroys history; time-travel requires custom audit tables and complex queries |
| Redis / key-value store | No history, no queries, no schema |
| Graph DB (Neo4j, FalkorDB) | Temporal edges are bolt-on, not structural; no EAV — entity shape is fixed by the graph schema |
| Actual Datomic | Proprietary, JVM-only, peer/transactor architecture, expensive; no MCP server |
| Datalevin | Explicitly drops temporal model; LMDB backend (not PostgreSQL); no TypeScript/Node bindings |
| Datahike | Clojure-only; no MCP server; operational complexity of running JVM service |

A bespoke TypeScript MCP server over PostgreSQL is the right choice because:
1. **TypeScript**: native language of the MCP SDK; largest pool of contributors
2. **PostgreSQL**: ACID, window functions, JSONB, `DISTINCT ON`, pgvector — everything needed in one battle-tested database
3. **MCP-only interface**: agents call tools, not SQL; the temporal complexity is hidden inside the server
4. **No proprietary dependencies**: fully open-source stack, self-hostable

### Verdict

This is a well-bounded, clearly motivated system. The data model is simple (two tables, four indexes). The query patterns are known (EAV temporal queries are well-understood SQL). The interface is narrow (11-15 MCP tools). The gap in existing tools is real — no project currently exposes temporal time-travel as first-class MCP tools for agents.

The highest-risk part is not the database or the queries — it's the ergonomics of the entity ID conventions and whether agents naturally use the system correctly. Plan to test with real agent workflows in Phase 1 before investing in Phase 3 features.
