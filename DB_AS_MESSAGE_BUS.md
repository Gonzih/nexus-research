# The Database as Message Bus
## How Paperclip Eliminated the Entire Coordination Layer

> Ground truth from direct schema + service source reads.
> This is the insight, extracted mechanically.

---

## The Core Observation

No Kafka. No Redis. No RabbitMQ. No WebSockets for coordination.
Just Postgres with the right table shapes and query patterns.

Three primitives handle everything:

```
1. Atomic UPDATE with WHERE clause     →  claim / checkout / mutex
2. Row status enum + polling           →  message passing / state machine
3. coalescedCount + deferred_*status   →  backpressure / flood control
```

That's it. Every multi-agent synchronization problem Paperclip solves maps to one of these three.

---

## Primitive 1: Atomic UPDATE WHERE = The Distributed Mutex

**The checkout (from `issues.ts` line 780):**

```sql
UPDATE issues
SET
  assignee_agent_id  = :agentId,
  checkout_run_id    = :runId,
  execution_run_id   = :runId,
  status             = 'in_progress',
  started_at         = now(),
  updated_at         = now()
WHERE
  id                 = :issueId
  AND status         IN ('todo', 'backlog', 'blocked')
  AND (assignee_agent_id IS NULL
       OR (assignee_agent_id = :agentId
           AND (checkout_run_id IS NULL
                OR checkout_run_id = :runId)))
  AND (execution_run_id IS NULL
       OR execution_run_id = :runId)
RETURNING *
```

If `RETURNING` is empty → zero rows updated → someone else got it → return 409 conflict.

**What this replaces:**
- No distributed lock service (Redlock, Zookeeper)
- No mutex over the network
- No "check then act" race condition
- No separate coordination table

Postgres guarantees: one UPDATE transaction wins. All others get zero rows back. The database IS the lock.

**The deeper insight:** Two fields, not one.
- `checkout_run_id` = claimed by THIS specific run instance
- `execution_run_id` = being actively executed

Separation allows: if a run crashes mid-execution, `execution_run_id` is stale. A new run by the same agent can *adopt* the stale checkout (`adoptStaleCheckoutRun`). No manual cleanup, no dead-letter queue — the next live run inherits and continues.

---

## Primitive 2: Status Enum + Poll = Async Message Passing

**`agent_wakeup_requests` table status enum:**
```
queued                   ←  "please wake this agent"
claimed                  ←  "scheduler picked this up"
coalesced                ←  "merged into existing run, discarded"
deferred_issue_execution ←  "agent busy, queue this for when it's free"
completed / failed       ←  terminal
```

**`heartbeat_runs` table status enum:**
```
queued    →  running  →  succeeded
                      →  failed
                      →  cancelled
                      →  timed_out
```

**`issues` table status enum:**
```
backlog → todo → in_progress → in_review → done
                             → blocked   → todo (re-enter)
                             → cancelled
```

Each of these is a message queue implemented as a table column.

The "message" is the row. The "send" is an INSERT or UPDATE. The "receive" is a SELECT WHERE status = 'queued'. The "ack" is an UPDATE SET status = 'claimed'. The "dead letter" is UPDATE SET status = 'failed'.

**The CLI polling loop (from `heartbeat-run.ts`):**
```typescript
while (true) {
  const events = await api.get(`/heartbeat-runs/${runId}/events?afterSeq=${lastSeq}&limit=100`);
  // process events...
  const run = await api.get(`/heartbeat-runs/${runId}`);
  if (TERMINAL_STATUSES.has(run.status)) break;
  await delay(200ms);  // 5 polls/sec
}
```

200ms poll interval. No WebSocket needed for the consumer. The DB row IS the event bus.

**`heartbeat_run_events` table** — append-only log with `seq` counter. Every log chunk, status change, adapter invocation → append a row. Consumer polls `afterSeq=N`. This is Kafka's offset model, implemented in Postgres with a sequence integer.

---

## Primitive 3: coalescedCount + deferred = Backpressure Without a Queue

**The problem:** Agent is busy running an experiment (heartbeat run in progress). Three new wakeup requests arrive for the same agent. What happens?

**The naive solution:** Queue them all, run them sequentially. Result: agent falls behind, latency spikes, duplicate work.

**The Paperclip solution (from `heartbeat.ts` line 1860-1870):**

```typescript
// If agent already has an active run AND a deferred request exists for this issue:
await tx.update(agentWakeupRequests)
  .set({
    payload: mergedDeferredPayload,  // MERGE context snapshots
    coalescedCount: existingDeferred.coalescedCount + 1,  // COUNT merges
    updatedAt: new Date(),
  })
  .where(eq(agentWakeupRequests.id, existingDeferred.id));
// Return {kind: "deferred"} — only ONE deferred row exists per agent+issue
```

**The mechanism:**
- First request while agent busy → INSERT deferred row
- Second request while agent busy → UPDATE existing deferred row (merge context, increment count)
- Third, fourth, fifth → same UPDATE
- When agent finishes → picks up the ONE deferred row (which contains merged context from ALL N requests)

Result: N requests → 1 run. No queue buildup. No wasted work. The `coalescedCount` tells you how many got merged.

**Why this matters for Loop architecture:**
If a sending platform fires 5 reply notifications in rapid succession after a campaign, the Loop Worker gets ONE wakeup with merged context, not 5 sequential runs. Automatic debounce built into the data model.

---

## The Full Pattern: DB as Coordination Layer

```
WHAT YOU NEED              WHAT PAPERCLIP USES
──────────────────────────────────────────────────────────
Distributed mutex       →  atomic UPDATE WHERE + RETURNING
Message queue           →  table rows with status enum
Message ack             →  UPDATE SET status = 'claimed'
Dead letter queue       →  UPDATE SET status = 'failed'
Event stream            →  append-only table + seq integer
Consumer offset         →  afterSeq poll parameter
Backpressure            →  coalesce into single deferred row
Pub/sub                 →  poll WHERE status = 'queued'
Job scheduler           →  INSERT wakeup_request WHERE not exists active run
Claim expiry / timeout  →  execution_locked_at + scheduler stuck-run detection
State machine           →  enum transitions with DB-enforced valid states
Audit log               →  append-only activity_log, never mutated
```

Zero external services. One Postgres.

---

## The `startLocksByAgent` In-Memory Layer

There's one more mechanism — a JavaScript-level mutex:

```typescript
// heartbeat.ts line 31
const startLocksByAgent = new Map<string, Promise<void>>();

// Usage: before touching DB for a given agentId
const previous = startLocksByAgent.get(agentId) ?? Promise.resolve();
const marker = previous.then(async () => {
  // do the DB work
});
startLocksByAgent.set(agentId, marker);
marker.finally(() => {
  if (startLocksByAgent.get(agentId) === marker)
    startLocksByAgent.delete(agentId);
});
```

This is a process-local promise chain. It serializes concurrent heartbeat invocations for the same agent *within a single server process* before they even hit the DB.

**Two-layer locking:**
1. In-process: `startLocksByAgent` (Map of Promise chains) — eliminates DB contention for rapid concurrent requests to same agent
2. In-DB: atomic UPDATE WHERE — handles multi-process / multi-server race conditions

The in-process lock is the fast path. The DB lock is the guarantee. Neither alone is sufficient for a production system. Both together are correct and fast.

---

## The Separation of Concerns Principle

From Lobster (OpenClaw's workflow engine):
> "LLMs do what LLMs are good at. Lobster does what code is good at: sequencing, counting, routing, retrying."

From autoresearch:
> `prepare.py` = fixed harness. `train.py` = what the agent touches. Never mix.

From Paperclip:
> Agent touches: one issue at a time, via API. Infrastructure handles: locking, routing, scheduling, coalescing, logging.

**The principle across all three:**

```
AGENT TOUCHES ONE THING         INFRASTRUCTURE HANDLES EVERYTHING ELSE
───────────────────────────────────────────────────────────────────────
train.py (one file)             prepare.py, git, 5-min timer, val_bpb eval
template.json (one asset)       baseline.md, scoring, deploy pipeline, windows
one issue row (via API)         checkout atomics, scheduling, coalescing, budget
```

The agent's job is narrow and well-defined. The harness's job is everything that would cause bugs if the agent touched it.

**This is why the pattern works at scale.** You can have 17 agents running simultaneously, all touching the same DB, with no coordination logic in any agent's code — because all the coordination is in the infrastructure layer.

---

## Practical Implication for the Loop System

When building the Flow/Loop architecture on Paperclip, you don't need to implement:

- ❌ Message broker
- ❌ Job queue (Bull, BullMQ, Celery)
- ❌ Distributed lock (Redlock)
- ❌ Event bus (EventBridge, Kafka)
- ❌ Scheduler (cron daemon, Temporal)
- ❌ Backpressure mechanism

You get all of these from Postgres row semantics + Paperclip's service layer.

What you DO need to design carefully:
- ✅ The issue schema (what fields carry experiment context to the agent)
- ✅ The status transitions (what "done" means for each loop type)
- ✅ The wakeup trigger (assignment vs timer vs callback from scoring)
- ✅ The `contextSnapshot` shape (what the agent reads on wakeup)

Everything else is already solved.

---

## The Meta-Insight

The invention of message queues, pub/sub systems, and distributed coordination layers was largely a response to databases being slow and not supporting the right primitives.

Postgres in 2026 is fast. It supports:
- Row-level locking with `SELECT FOR UPDATE SKIP LOCKED` (not used by Paperclip but available)
- Atomic UPDATE with RETURNING
- `LISTEN/NOTIFY` for push-based events
- Append-only patterns with sequence integers
- JSONB for flexible payloads without schema migrations

The "Postgres as everything" pattern isn't new. It's been rediscovered repeatedly (Faktory, Oban, River, pgqueue). Paperclip is a clean, production-quality implementation of it in the specific context of multi-agent coordination.

**The lesson:** Before adding infrastructure, ask "can Postgres express this?" Usually: yes.
