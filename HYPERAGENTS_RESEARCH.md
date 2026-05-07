# Hyperagents — Research Notes
**Paper:** arxiv.org/abs/2603.19461
**Authors:** Jenny Zhang, Bingchen Zhao, Wannan Yang, Jakob Foerster, Jeff Clune, Minqi Jiang, Sam Devlin, Tatiana Shavrina (Meta AI / Facebook Research / UBC)
**Code:** github.com/facebookresearch/Hyperagents (CC BY-NC-SA 4.0)
**Date read:** 2026-03-23

---

## The Core Bet

Darwin Gödel Machine (DGM) showed self-improving AI in coding works because task (coding) and self-modification mechanism (also coding) are aligned. Outside coding, this alignment breaks — the meta-improvement process stays fixed and handcrafted, capping how fast the system can improve.

**Hyperagents** collapse the task agent and meta agent into a single editable program. The meta-level improvement procedure is itself modifiable. The system can improve how it improves — metacognitive self-modification.

---

## Architecture

**Two-agent system (unified as one editable codebase):**

- `task_agent.py` — solves the target domain. Generic interface: `forward(inputs) → (prediction, message_history)`. LLM-backed, JSON outputs.
- `meta_agent.py` — takes `repo_path`, `eval_path`, `iterations_left`. Instruction: "Modify any part of the codebase at `{repo_path}`". Can edit itself.

Key: meta agent modifying itself is the *same operation* as modifying the task agent. Not a fixed controller layer — same editable substrate.

**Generation loop (`generate_loop.py`):**
1. Initialize Docker environments + archive of agent variants
2. Parent selection: score-proportional or recency heuristics
3. Each generation: isolated Docker → apply lineage patches → meta agent proposes edits → evaluate
4. Staged evaluation: small → medium → full (threshold >0.4 score to advance)
5. Timeouts: 3h ensemble eval, 5h individual eval
6. Checkpoint resumption supported

**Ensemble layer:** multi-agent coordination aggregating scores across variants.

---

## Domains Tested

balrog (RL/game), genesis (physics sim), imo (International Math Olympiad), paper_review, polyglot (multi-language coding), search_arena

All domains — no coding alignment assumption needed.

---

## Striking Results

- DGM-H improves over time across all domains
- Outperforms: (a) no self-improvement baseline, (b) self-improvement without open-ended exploration, (c) prior DGM
- **Spontaneously evolved** persistent memory and performance tracking — improvements to the improvement process itself, not handcrafted
- These meta-level improvements **transfer across domains** and **accumulate across runs**

---

## What We Steal Into Our Stack

### 1. Evolutionary branching in `create_plan` (HIGH VALUE)
Current: linear `depends_on` chains — job A → B → C.
Steal: spawn N variants in parallel, evaluate all, spawn next generation from best performer.
Application: skills refinement workflow. Instead of one agent per repo, spawn 3 variants with different approaches, score by test pass rate / code quality, chain from winner.

### 2. Persistent agent memory across generations
Current: agents start cold per job, no cross-run learning.
Steal: after each job completes, store structured "what worked / what failed" in Redis. Next agent in same namespace reads prior run learnings before starting.
Implementation: cc-agent adds `learnings` field to job hash. Agents write `AGENT_LEARNINGS.md` to repo after completion. Next agent in same namespace reads it first.

### 3. Staged evaluation with score thresholds
Current: agents run until done or timeout.
Steal: gate full runs behind cheap pre-checks. `npm test` on 10% of test suite first — if fails, abort before the 2h full run.
Implementation: preamble injection — prepend "run quick smoke test first, only continue if passes" to all tasks.

### 4. Score-proportional parent selection
Current: when we spawn follow-on agents, we pick by recency.
Steal: track job scores (test pass rate, PR merged/failed, output quality). Use scores to weight which completed jobs get follow-on work spawned.
Implementation: `score` field in cc-agent job hash. `create_plan` supports `parent_selection: "score_prop" | "latest"`.

### 5. Docker isolation per agent generation
Current: agents run in /tmp dirs, shared filesystem.
Steal: each agent job gets a fresh Docker container via colima. Stronger isolation, reproducible, no cross-job pollution.
Already have: colima running system-wide. cc-agent just needs to `docker run` instead of temp-dir clone.

### 6. The editable meta-agent insight for AMAI
Most directly relevant to AMAI Labs: the paper proves that making the meta-level mechanism editable is the key unlock. AMAI's soul-core runtime is currently a fixed meta-layer above task agents. Making soul-core itself an editable, observable program that agents can propose changes to — via agent-jail's eBPF visibility + append-only behavioral ledger — is the AMAI version of this architecture.
The accountability infrastructure becomes the substrate for self-improvement, not just the safety guard around it.

---

## AMAI Positioning Steal

The paper's safety appendix notes: "involves executing untrusted, model-generated code" with prominent safety warnings. This is exactly the gap AMAI fills:

- eBPF observable sandbox (agent-jail) = the Docker isolation layer they use, but with syscall-level observability
- Append-only behavioral ledger = the checkpoint/replay they need for forensic analysis
- Cryptographic identity = provenance chain for which agent variant generated which output
- Economic slashing = cost enforcement when a generation exceeds budget

**Hyperagents needs AMAI.** The paper acknowledges the risk; AMAI is the infrastructure that makes it safe to run at scale. This is the pitch.

---

## Immediate Actions

- [ ] File cc-agent issue: evolutionary branching in create_plan (N variants → evaluate → best continues)
- [ ] File cc-agent issue: persistent learnings field in job hash
- [ ] File cc-agent issue: Docker isolation mode for agent runs
- [ ] Draft AMAI positioning doc: "Hyperagents needs accountability infrastructure"
- [ ] Read DGM paper (the prior work): arxiv.org/abs/2407.09435
