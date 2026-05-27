---
title: Polar — Agent RL Rollout Infrastructure
date: 2026-05-24
tags: [rl, agent-training, infrastructure, nvidia, nanoforge, agent-lightning]
type: research
---

# Polar — Agent RL Rollout Infrastructure

**Author:** Binfeng Xu (NVIDIA / @billxbf)
**Announced:** May 26, 2026
**Source tweet:** https://x.com/billxbf/status/2059323616009838703
**Primary repo:** billxbf/NanoForge (the open reference implementation)
**Related:** billxbf/agent-lightning (Microsoft fork), billxbf/Griffins (autonomous loop)

---

## What Polar Is

Polar is agent RL rollout infrastructure — not an RL algorithm, not a training framework. It sits between the agent and the trainer.

The core insight: real-world agent harnesses (Codex, Claude Code, OpenClaw, Hermes) are already defined. The training problem is not "how do we build agents" but "how do we extract token-level trajectories from arbitrary agent execution and feed them to an RL optimizer without touching the agent code."

Polar's answer: transparent API interception. An agent thinks it's calling OpenAI at `http://localhost:8080/v1`. Polar's proxy sits at that URL, intercepts every chat completion call, logs `prompt_token_ids + response_token_ids + logprobs`, and forwards the call to the real vLLM backend. The agent never knows. Zero code change.

This is a reproduction of the architecture described in Minimax's "Forge" system (published Jan 2026 in their M2.1 post-training report). NanoForge is the open reference implementation. Binfeng Xu's "Polar" tweet refers to this class of infrastructure at scale.

---

## Architecture

### Layer 1 — The Transparent Proxy

```
Agent Process
    ↓  (OPENAI_BASE_URL = http://localhost:9000)
NanoForge Proxy  ←→  FastAPI, per-rollout routing
    ↓  (forwards with logprobs=True, return_token_ids=True injected)
vLLM Server
    ↑  (returns response + token IDs + logprobs)
NanoForge Proxy
    ↓  stores in _completion_store[rollout_id]
In-Memory Store
```

Route structure: `/rollout/{rollout_id}/v1/chat/completions`

The `rollout_id` is embedded in the base URL. This is how NanoForge routes completions to the right rollout without any agent-side awareness.

### Layer 2 — The Agent Interface

Four-method lifecycle inherited from Minimax Forge:

```python
class ForgeAgent(ABC):
    def agent_preprocess(self, task: dict) -> dict:   # init, set env
    def agent_run(self, context: dict) -> dict:        # execute (LLM calls intercepted)
    def agent_postprocess(self, result: dict) -> dict: # cleanup
    def calculate_reward(self, task, result, trajectories) -> float:  # scalar reward
```

The agent calls `openai.ChatCompletion.create()` inside `agent_run`. That call hits the proxy, not vLLM. The agent sees a normal response. The proxy sees everything.

### Layer 3 — Trajectory Construction

After `agent_run` completes:
1. All completions for that `rollout_id` are retrieved from the store
2. Sorted by `sequence_id` (turn order)
3. Converted to trajectory dicts: `{prompt_token_ids, response_token_ids, logprobs, reward, sequence_id, prefix_group}`
4. Reward assigned to the **final trajectory only** — all intermediate turns get `reward=None`
5. Prefix merging: turns sharing a common prompt prefix get the same `prefix_group` ID (required for GRPO advantage computation)

### Layer 4 — Parallelism

`run_rollouts_batch()` runs `N` rollouts concurrently via `asyncio`. Each rollout gets a unique `rollout_id`. The proxy is shared; the in-memory store is namespace-partitioned by `rollout_id`. Thread-safe via `asyncio.Lock`. Max concurrent defaults to 10.

### Layer 5 — Training Connection

NanoForge outputs trajectory dicts ready for GRPO training. The `prefix_group` field is the key: GRPO computes advantages relative to other trajectories in the same group. Trajectories sharing a prefix are in the same group — this makes multi-turn conversations tractable because earlier turns inform advantage estimation for later turns.

The downstream is typically veRL, OpenRLHF, or a custom GRPO loop. NanoForge is unopinionated about the trainer.

---

## Why "Without Code Change" Works

The mechanism is `OPENAI_BASE_URL` environment variable injection. The proxy runner:

1. Sets `OPENAI_BASE_URL = http://localhost:{proxy_port}/rollout/{rollout_id}` before calling `agent_run`
2. The OpenAI Python SDK reads this env var at call time
3. Every `client.chat.completions.create()` inside the agent hits the proxy URL, not the real endpoint
4. Model name in the request is overridden to the actual vLLM-loaded model

This works for any agent that uses the OpenAI Python SDK or compatible client. It does not work for agents using raw HTTP or non-OpenAI APIs. Those would require a shim layer, but the shim stays outside the agent codebase.

---

## The Minimax Forge Origin

Polar implements the same architecture as Minimax's internal Forge system, described in their M2.1 blog post (Jan 2026). Forge powered the RL training for M2.1, which achieved #1 on open-source model AppDev leaderboards and near-SOTA on BrowsComp.

Minimax's Forge uses CISPO as the RL algorithm (Clipped Importance Sampling Policy Optimization) — a REINFORCE-style objective with importance-sampling clipping on the scalar weights rather than on the policy ratio. Key property: all tokens receive gradients (unlike PPO, where clipping silently zeroes gradients for "discourse" tokens like "wait").

Binfeng Xu also has `billxbf/miles` — "an enterprise-facing RL framework for LLM and VLM post-training, forked from and co-evolving with slime." Miles is the training-side component; Polar/NanoForge is the rollout-side.

---

## Related Systems

### Agent Lightning (Microsoft)

Independently developed, published August 2025. arxiv:2508.03680. Same core insight: Training-Agent Disaggregation. Differences:
- Uses span-based observability (agl.emit_xxx() decorators) rather than pure proxy interception
- Supports prompt optimization and SFT in addition to RL
- Formulates agent execution as MDP explicitly, defines unified data interface
- Supports multi-agent systems with selective optimization of individual agents

Agent Lightning is the more complete academic framework. NanoForge/Polar is the minimal production-focused version.

### Griffins (billxbf)

An autonomous multi-round build loop: Host → Planner → Developer → Reviewer per round. Not directly an RL system — uses Claude/Cursor agent loop with a QUESTIONNAIRE.md and exit criteria. Relevant because it shows the trajectory of thinking: agent loops → formalized episodes → RL on those episodes.

---

## Key Design Principles

### 1. Infrastructure Layer Separation

Rollout infrastructure is separate from:
- The training algorithm (GRPO, PPO, CISPO)
- The training framework (veRL, OpenRLHF)
- The agent implementation (any framework)
- The inference backend (vLLM, SGLang, local)

Each layer plugs in independently. This is why harnesses work without code change — the interface is the OpenAI API spec, which everything already implements.

### 2. Token-Level Trajectory Capture

The training signal requires `prompt_token_ids + response_token_ids + logprobs`. These are only available if the inference server returns them — standard vLLM does with `return_token_ids=True`. No tokenizer re-run, no retokenization drift (a known failure mode documented in the vLLM blog post on Agent Lightning).

### 3. Sparse Reward, Dense Gradient

Reward is assigned to the final turn only. But every token in every turn gets a gradient signal through the trajectory structure and GRPO prefix grouping. The challenge: credit assignment across multi-turn sequences. Polar defers this to the training algorithm — GRPO handles it via group-relative advantages.

### 4. Concurrent Rollout as the Scaling Primitive

RL for agents requires many rollouts — you need variance in outcomes to compute meaningful advantages. NanoForge runs rollouts concurrently, not sequentially. The hard limit is the vLLM server's KV cache capacity. At production scale (Minimax's Forge), this is parallelized across many inference servers.

### 5. Harness = Training Environment

This is the conceptual flip that Polar enables. A "harness" (an agent scaffold for a specific task) becomes an RL environment in the OpenAI Gym sense: step() → state, action → LLM call, reward → harness-defined scalar. The environment implementation is the agent code. The RL infrastructure wraps it transparently.

---

## Supported Harnesses

From the tweet: Codex, Claude Code, OpenClaw, Hermes. The "without code change" claim covers any harness using the OpenAI Python SDK with `OPENAI_BASE_URL` support.

For black-box binaries or agents using different APIs, Minimax's Forge handles this by redirecting the agent's base URL at the OS/network level. NanoForge implements the basic case; production Polar likely adds routing at the network layer.

---

## Scalability Design

Single-machine NanoForge: async rollouts, in-memory store, asyncio lock.

Production scale inference:
- Multiple vLLM servers behind a load balancer
- `rollout_id` in the URL enables sticky routing if needed
- Prefix grouping for KV cache reuse across parallel rollouts
- Training batches assembled from N completed rollouts before optimizer step

The Minimax Forge blog notes "intelligent prefix merging" as a training efficiency optimization — identical prompt prefixes across rollouts share KV cache entries, reducing inference cost. This is why `prefix_group` is computed: the trainer can batch trajectories that share prefixes for efficient forward passes.

---

## What Makes This Different

Prior RL-for-agents approaches:
- Simulated environments (ALFWorld, WebArena) — trainable but not the real agent
- Tight coupling — RL training loop wraps agent execution code directly
- Post-hoc trajectory logging — trajectories reconstructed from logs, not captured live
- Single-harness training — each training run tied to one agent type

Polar's differentiator: **harness portability**. The same rollout infrastructure works for any agent that uses the OpenAI API. This means:
- You can train on production Claude Code behavior patterns
- You can compare training trajectories across harnesses
- You can mix harnesses in the same training batch
- The infrastructure investment amortizes across all harnesses

---

## Implications for Agent Training at Scale

1. **Data flywheel:** Every production agent run is a potential training episode. The infrastructure to capture it (proxy interception) is lightweight. The missing piece is usually reward signal design.

2. **Harness design is the new prompt engineering:** The tweet says "Find a problem, design the harness." The agent architecture (tool set, loop structure, error handling) directly shapes the trajectory distribution. Better harnesses = better training signal.

3. **Reward sparsity is the core problem:** Most agent tasks have binary outcomes (did it work?). Polar defers the credit assignment problem to GRPO. But sparse rewards still require many rollouts for meaningful gradient estimates. This is the scaling cost.

4. **Model serving cost at training time:** Each rollout requires the full model. At 128 GPU scale (Youtu-Agent's validated configuration), the training loop becomes expensive. The infrastructure investment in prefix caching and efficient batch construction directly affects per-training-step cost.

5. **Multi-scaffold training is critical:** Minimax explicitly calls this out — training on a single harness (one ReAct loop variant) limits generalization. Production training mixes multiple harness architectures. Polar's harness-agnostic design enables this.
