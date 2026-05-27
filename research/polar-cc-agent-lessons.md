---
title: Polar Lessons Applied to cc-agent
date: 2026-05-24
tags: [cc-agent, rl, agent-training, polar, architecture, proposal]
type: research
---

# Polar Lessons Applied to cc-agent

**Reference:** [[polar-agent-rl-infrastructure]]
**cc-agent repo:** https://github.com/gonzih/cc-agent
**Architecture files studied:** src/evaluator.ts, src/store.ts, src/swarm.ts, src/seeds.ts, src/index.ts

---

## What cc-agent Already Has That Polar Needs

Before proposing additions, audit what exists.

### Scoring infrastructure (evaluator.ts + store.ts)

`JobRecord` has a `score: number | null` field. The evaluator agent computes scores from 0.0 to 1.0 via three methods:
- `test_pass_rate`: `(passing_tests / total_tests) * 0.7 + (exitCode === 0 ? 0.3 : 0)`
- `pr_merged`: binary `1.0 / 0.5 / 0.0`
- `manual`: LLM-assigned

`set_job_score` writes to `JobRecord.score`. This is already a reward signal. It is:
- Per-episode (per job)
- Scalar (0.0–1.0)
- Computed after completion
- Persisted in Redis with 7-day TTL

This is structurally identical to `calculate_reward()` in NanoForge's ForgeAgent interface.

### Learnings store (store.ts, LearningsStore)

`learningsStore.addLearning(namespace, content)` pushes free-text observations to a Redis list (max 50, 90-day TTL). `getLearnings(namespace, limit)` retrieves them. `clearLearnings(namespace)` resets.

This is a per-namespace memory that persists across job executions. Currently used for ad-hoc observations. Not connected to job outcomes or scores.

### Profile templates (seeds.ts)

8 built-in profiles: `fix-issue`, `implement-feature`, `write-tests`, `security-audit`, `refactor`, `review-pr`, `bump-deps`, `coder`. Each is a task template with a `defaultBudgetUsd` and optional `preamble`.

These are static. They do not update based on job outcomes. But they are the equivalent of "harness definitions" — structured task templates that constrain agent behavior.

### Swarm (swarm.ts)

`runSwarm` decomposes a goal into N sub-tasks via Claude Haiku, fans out agents, waits for completion, then runs a synthesis agent. This is multi-agent, multi-episode. Each sub-job produces a `JobRecord` with score, output, exit code.

---

## The Gap: No Feedback Loop

cc-agent has:
- Episodes (jobs)
- Rewards (scores)
- Memory (learnings)
- Templates (profiles)

What it does not have: any connection between rewards and templates. Job outcomes do not update profiles. High-scoring jobs do not propagate patterns forward. Low-scoring jobs do not trigger prompt revision. The learnings store is written to manually or by agents ad-hoc, not systematically from job outcomes.

This is the gap Polar closes. The loop Polar enables: rollout → reward → update policy → better rollout.

cc-agent has all the pieces. They are not connected.

---

## Three Concrete Proposals

### Proposal 1 — Reward-Conditioned Learning: Score → Learnings Auto-Write

**What:** When a job completes with a score, automatically trigger a "reflection" step that writes a learning entry.

**How it works:**

When `set_job_score` is called (or when a job transitions to `done` with a score), spawn a lightweight reflection agent:

```
Inputs:
  - job.task (the prompt given to the agent)
  - job.score (the reward)
  - job.recentTools (what tools were called)
  - job.exitCode
  - abbreviated job output (last 50 lines)

Task to reflection agent:
  Score: {score}. Analyze this job execution. Write 1-3 specific, actionable learnings
  for the profile "{profile_name}" based on what succeeded or failed. Be concrete —
  name tools, patterns, failure modes. Do not generalize. Format as plain text bullets.
  
Output:
  learningsStore.addLearning(namespace=profile_name, content=...)
```

**Why this matters:** The learnings store is already injected into preambles for subsequent jobs. If learnings reflect actual outcome patterns, future agents on the same profile get contextualized guidance from prior runs.

**Threshold:** Only trigger reflection for jobs with `score !== null`. Optionally: only for `score < 0.5` (failures are more informative) or `score > 0.9` (successes worth propagating).

**Cost:** One Haiku call per scored job. ~$0.001. Negligible.

---

### Proposal 2 — Profile Score Tracking: Turn Profiles Into Learnable Policies

**What:** Track per-profile score distribution over time. Use it to surface which profiles are underperforming and need prompt revision.

**Schema addition to Profile:**

```typescript
// In store.ts, Profile interface
scoreHistory?: Array<{
  jobId: string;
  score: number;
  timestamp: string;
}>;
avgScore?: number;        // rolling average
totalJobsScored?: number;
```

**How it works:**

1. When a job completes with a score AND has a profile origin (add `sourceProfile?: string` to JobRecord), append to `profile.scoreHistory` (cap at 100 entries) and recompute `profile.avgScore`.

2. Add `get_profile_stats` MCP tool: returns avg score, score distribution, low-scoring job IDs for a profile.

3. Optionally: expose `suggest_profile_improvement` — runs a reflection agent over the 5 lowest-scoring jobs for a profile and proposes a preamble revision.

**Why this matters:** This is policy improvement at the template level. A profile is a policy — it defines what the agent does. Score history is the reward signal over that policy. When `avgScore` drops below a threshold, the profile's preamble needs revision.

This is the cc-agent equivalent of Polar's "train → better policy → deploy" loop, except the "policy" is a prompt template, not model weights. Prompt optimization is explicitly in scope for Agent Lightning's algorithm zoo.

---

### Proposal 3 — Episode Replay: Swarm as RL Batch

**What:** cc-agent's swarm already runs N parallel jobs on N tasks. This is structurally identical to running N rollouts in a training batch. Connect the swarm's aggregate score to a synthesis prompt that improves future task decomposition.

**Current swarm flow:**
```
goal → decompose (Haiku) → N sub-jobs → synthesis agent → output
```

**Polar-inspired swarm flow:**
```
goal → decompose → N sub-jobs [scored] → score aggregation → policy reflection → synthesis + profile update
```

**Changes:**
1. After all sub-jobs complete, compute `batchScore = avg(sub-job scores)`.
2. If `batchScore < threshold` (e.g., 0.6), run a "batch retrospective" step:
   - Inputs: decomposed tasks, per-task scores, task outputs
   - Task: "The swarm batch achieved avg score {X}. Identify which task decompositions failed and why. Suggest improved task descriptions for this goal pattern."
   - Output: learnings tagged with `swarm:{swarm_id}`
3. On future swarms with similar goals (detected by embedding similarity or keyword match), inject these swarm-level learnings into the decomposition step.

**Why this matters:** The decomposition quality directly determines sub-job quality. Bad decompositions produce correlated failures. The swarm's batch score is a meta-reward signal for the decomposition strategy, not just for individual jobs.

---

## Minimal "Polar-Inspired" Learning Loop for cc-agent

If only one thing gets implemented, this is it:

```
Job completes with score
    ↓
IF score !== null:
  reflection_agent.run(job, score)  # Haiku, cheap
    ↓
  learningsStore.addLearning(profile_name, reflection)
    ↓
Next job on same profile:
  getLearnings(profile_name, limit=5) → injected into preamble
    ↓
Agent behavior shifts based on prior outcomes
```

This is one closed loop. No model fine-tuning. No gradient computation. No vLLM server. Just: score → text reflection → context injection.

It does not require token-level trajectories (cc-agent runs Claude Code, which is a black box for token capture — the OPENAI_BASE_URL trick does not apply to Claude CLI). This is why the prompt-level loop is the right adaptation: it operates at the layer cc-agent actually controls (text in, text out).

---

## What cc-agent Cannot Do (and Why)

Polar's core mechanism — token-level trajectory capture via proxy interception — requires:
1. A vLLM server (or compatible inference server returning token IDs)
2. Model weight access (to actually run RL on gradients)
3. The agent using the OpenAI Python SDK (so `OPENAI_BASE_URL` override works)

cc-agent runs Claude Code via CLI (`claude --continue ...`). Claude Code:
- Is a black-box binary
- Calls Anthropic's API directly, not a local vLLM server
- Does not expose logprobs or token IDs

Therefore: cc-agent cannot implement gradient-level RL. It cannot capture token trajectories. It cannot fine-tune model weights from job outcomes.

What it CAN do is everything Polar does above the token layer:
- Define episodes (jobs = episodes)
- Compute rewards (scores = rewards)
- Aggregate trajectories across episodes (learnings = trajectory abstractions)
- Update policies (profile preambles = prompts = soft policies)

This is **inference-time policy improvement** (prompt optimization) rather than weight-level RL. It is less powerful but fully achievable with cc-agent's current architecture.

---

## Implementation Priority

| Proposal | Impact | Effort | Priority |
|----------|--------|--------|----------|
| 1 — Score → Learnings Auto-Write | High (closes the loop) | Low (1 function, 1 Haiku call) | Ship first |
| 2 — Profile Score Tracking | Medium (surfaces underperformers) | Medium (schema + new tool) | Ship second |
| 3 — Swarm as RL Batch | Medium (improves decomposition) | Medium-high (swarm refactor) | Third |

Proposal 1 alone closes the feedback loop. Proposals 2 and 3 add instrumentation and optimization on top.

---

## Comparison to Polar's Architecture

| Polar Component | cc-agent Equivalent | Gap |
|-----------------|---------------------|-----|
| Proxy interception | N/A (Claude CLI, black box) | Cannot close |
| Agent lifecycle (preprocess/run/postprocess) | Job spawn + run + exit code | Matches semantically |
| `calculate_reward()` | `set_job_score()` | Exists, not automated |
| Trajectory store | Redis job store + learnings | Learnings not reward-conditioned |
| Prefix grouping / GRPO | N/A | Not applicable without token access |
| Concurrent rollouts | Swarm parallel sub-jobs | Matches structurally |
| Policy update | Proposal 1 (preamble update via reflection) | Does not exist yet |
| Training-Agent Disaggregation | Meta-agent / coordinator separation | Partially exists |

The architectural alignment is strong. The missing piece is the reward → policy feedback path, which is purely software (no GPU required).

---

## Open Questions

1. **Reward signal quality:** The current `test_pass_rate` and `pr_merged` heuristics are coarse. For cc-agent's use cases (general coding, research, job applications), a richer reward model would improve learning signal quality. What rubrics should the manual evaluator use?

2. **Namespace scope:** Learnings are namespaced to the cc-agent instance. Should profile-level learnings cross-pollinate across namespaces (e.g., money-brain jobs improving profiles used in ecoclaw jobs)? Currently they cannot.

3. **Feedback latency:** The reflection agent adds latency after job completion. For rapid iteration, this could block the next job. Should reflection run asynchronously, fire-and-forget?

4. **Learning decay:** The learnings store caps at 50 entries with 90-day TTL. High-frequency profiles may cycle through learnings faster than they compound. Should score-weighted retention be implemented (high-reward learnings persist longer)?

5. **Exploration vs exploitation:** A pure "learn from what worked" strategy risks converging on one style of task execution. Polar handles this via stochastic sampling (`score_prop` selection). Should cc-agent's evaluator default to `score_prop` rather than `best_score` for evolutionary plans?
