# The Universal Loop Pattern

> Three systems: autoresearch (ML training), marketing experiments, cosmic string physics.
> Same underlying structure. Different domains. Same engine.

---

## The Pattern (Extracted from All Three)

```
LOCKED ORACLE         ←  immutable reality check
      ↓
SEARCH ENGINE         ←  navigates combinatorial space
      ↓
CANDIDATE             ←  one specific modification
      ↓
NUMERICAL FEEDBACK    ←  hard reality prunes immediately
      ↓
KEEP OR DISCARD       ←  advance state or reset
      ↓
LOOP                  ←  forever
```

**The key tension:** The search space is too large for exhaustive enumeration. The oracle is too slow for every candidate to be validated cheaply. The feedback signal is the bridge — it prunes the space fast enough that systematic search becomes tractable.

---

## Side-by-Side: The Three Systems

### What Each Component Maps To

| Component | autoresearch | Cosmic String Paper | Marketing Loop |
|-----------|-------------|---------------------|----------------|
| **Locked oracle** | `evaluate_bpb()` in `prepare.py` (immutable, agent can't touch) | High-precision numerical baseline (ground truth integral value) | Scoring rule in `baseline.md` (metric definition never changes) |
| **Search engine** | Claude/Codex agent reading `program.md` | Gemini Deep Think + PUCT tree search | Claude agent reading `loop_program.md` |
| **Candidate** | Modified `train.py` (one change per experiment) | LaTeX mathematical expression + auto-generated Python eval function | Modified `template.json` (one variable, one direction) |
| **Feedback signal** | `val_bpb` after 5-min training run | Exception traceback + absolute error penalty injected to context | Reply rate / CTR / metric after fixed window |
| **Keep/discard** | git commit (keep) / git reset (discard) | Advance tree node (keep) / prune branch (discard) | Template file advances (keep) / restored to last commit (discard) |
| **Cumulative state** | Git branch history = stacked wins | Tree structure = map of explored solution space | experiment_log.tsv + git history of template.json |
| **Human interface** | `program.md` (iterate on the research org instructions) | System prompts + search constraints | `loop_program.md` (iterate on the research org instructions) |

---

## What Makes a Problem Amenable to This Pattern

The cosmic string paper is explicit about this. The problem had specific structural properties that made the approach work. These are the conditions:

**Condition 1: Computable ground truth exists**
"High-precision numerical evaluation provides objective validation criteria."

You need a fast, cheap, unambiguous oracle. In physics: numerical integration to high precision. In ML: run training for 5 minutes, check val_bpb. In marketing: count replies after 72h.

If ground truth is expensive, slow, or ambiguous — the feedback loop degrades. Each cycle takes longer. Signal is noisier. Pruning becomes unreliable.

**Condition 2: Solution space is combinatorially large but structured**
The problem is too hard to solve by thinking alone, but not random. There's a space of valid mathematical moves (basis expansions, substitutions, transformations) — finite types, infinite combinations. The LLM navigates this space using pattern-matched knowledge of which moves tend to work where.

Marketing equivalent: the space of one-variable template changes is combinatorially large but structured. Subject lines can vary in length, framing, personalization depth, question vs statement, loss vs gain, social proof vs authority — each is a known dimension with known directions. LLM navigates using pattern-matched knowledge of what tends to work in what contexts.

**Condition 3: Solutions exist within a known space (not requiring genuinely new objects)**
The paper: "Closed-form solutions exist within standard special functions."

The Gegenbauer breakthrough wasn't the invention of new math — it was the recognition that existing mathematical machinery (Gegenbauer polynomials) had the right structure to cancel the singularity. The LLM found a non-obvious connection within the space of known tools.

Marketing equivalent: you're not inventing new human psychology. Loss aversion, social proof, authority, specificity, urgency — these mechanisms exist. The loop finds non-obvious combinations and framings that activate them in your specific ICP context.

**Condition 4: Feedback prunes fast**
The cosmic string paper: "The automated Python verifier successfully caught and pruned over 80% of these branches due to algebraic errors or numerical divergence."

80% of the 600 candidates were eliminated automatically before any human review. The feedback signal is doing the heavy lifting. The LLM proposes. Reality culls. Human reviews the survivors.

In autoresearch: the 5-minute training run is the pruning mechanism. Crashes = immediate discard. val_bpb worse = git reset. Human wakes up to survivors.

In marketing: fast-fail triggers (0 replies after 24h on 50 sends) prune early. The 72h window is the full evaluation. Human reviews the experiment_log.

**Condition 5: The "negative prompting" / local maxima escape mechanism**
The cosmic string paper used an explicit technique: "Once valid paths emerged, the system forced exploration of alternative approaches by explicitly forbidding successful methods, compelling the model to discover distinct solution classes."

This is how they found SIX solutions instead of one. Once Class I methods (monomial) started working, they told the system: "don't use monomial expansions." Forced discovery of spectral methods. Forced discovery of Gegenbauer.

autoresearch equivalent: the agent is instructed to "think harder, read papers, try more radical changes" when stuck. Not to keep refining the same approach.

Marketing equivalent: Flow sessions that explicitly ask "what are we NOT testing?" Banning the current winning approach in loop_program.md to force exploration of a new dimension.

---

## The Deeper Pattern: What Role the LLM Actually Plays

All three systems reveal the same truth about LLMs in loops:

**LLM ≠ oracle. LLM = combinatorial space navigator.**

The LLM doesn't "understand" physics, training dynamics, or buyer psychology. It pattern-matches across training data of mathematical techniques / ML experiments / copywriting patterns and proposes combinations that haven't been tried yet.

The thing that makes this work isn't LLM intelligence. It's:
1. The LLM can generate valid candidates in the structured space fast
2. Numerical feedback prunes invalid candidates immediately
3. The loop runs long enough to explore the space systematically

"The LLM navigates the space of valid mathematical transformations, and hard numerical reality continuously prunes the search. Neither alone would work."

**Neither alone would work.** LLM without feedback loop = hallucination and false confidence. Feedback loop without LLM = brute force enumeration of an intractable space.

The architecture is the intelligence. Not the model.

---

## What This Unlocks (The Generalization)

The pattern applies wherever:

1. You have a well-defined metric (the oracle)
2. You have a structured space of valid modifications (the search space)
3. You can evaluate candidates cheaply and unambiguously (the feedback)
4. Solutions are improvements in the existing space (not requiring invention of new dimensions)

**Where it works right now:**
- ML architecture/hyperparameter search (autoresearch)
- Mathematical problem solving with numerical ground truth (cosmic strings)
- Marketing asset optimization with measurable conversion signals (our loops)
- Code optimization (fix the bug, run tests = feedback)
- Drug molecule screening (modify scaffold, run assay = feedback)
- Materials science (modify composition, run simulation = feedback)

**Where it breaks:**
- No cheap oracle (validation requires human judgment, long timelines, expensive experiments)
- Ambiguous feedback (signal is noisy, multi-factor, hard to attribute)
- Solutions require genuinely new mathematical/conceptual objects (the solution doesn't exist in pattern-matching space of training data)
- The search space isn't structured (pure novelty problems)

**Gap (unsettled):** The paper explicitly hedges: "While the mathematical resolution of this integral is interesting in its own right, the primary contribution of this paper is methodological." They don't claim this generalizes. The gap is real: problems where the oracle is expensive, noisy, or ambiguous are a different regime. Pulsar timing arrays (the actual physics validation) will take decades. The fast numerical oracle was the key enabler.

---

## The Meta-Layer: You Iterate on the Loop, Not the Output

All three systems share this structure:

```
WHAT AGENT ITERATES ON:   train.py  /  LaTeX expressions  /  template.json
WHAT HUMAN ITERATES ON:   program.md  /  system prompts + constraints  /  loop_program.md
```

The human's job is not to run experiments. The human's job is to write better instructions for the agent — to build a better research org.

In the cosmic string paper, the researchers designed the system prompts, defined the search constraints, created the negative prompting strategy. They didn't do the math. They built the machine that does the math.

In autoresearch, you don't touch train.py. You iterate on program.md — the "research org code" that determines how the agent approaches the search.

In marketing loops, you don't write the email variations. You write loop_program.md — the instructions that determine what variable the agent explores, how it scores results, when it decides to keep vs discard, what dimensions it explores next when stuck.

**The evolved loop_program.md after 2,000 experiments is the product.** It encodes institutional knowledge that took thousands of iterations to build. This is what compounds. This is what's hard to replicate.

---

## The Compounding Asymmetry

Classical research (human team): serial, slow, intuition-driven, expensive, sleeps.

Loop pattern: parallel, fast, systematic, cheap, never sleeps.

**The cosmic string problem sat open for 30+ years.** Expert physicists had partial solutions. The system found 6 complete solutions in one run. Not because the LLM is smarter than physicists — because systematic tree search across 600 candidates with automated pruning covers ground that serial human intuition can't.

This is the same as the marketing claim: not that the agent is a better marketer, but that 2,000 systematic experiments beat 30 intuition-driven campaigns.

**The asymmetry compounds because the oracle is cheap.**

Physics: numerical integration is microseconds. 600 candidates explored.
ML: 5-minute training run. 100 candidates overnight.
Marketing: 72h window. 1 candidate per window per channel, 17 channels = 17 candidates per 72h.

The faster the oracle, the larger the search space you can cover, the more likely you find the non-obvious combination that works.

---

## What "Non-Obvious" Means Mechanically

The Gegenbauer breakthrough: the singularity in the cosmic string integral comes from the denominator `(1-e₁²)(1-e₂²)`. Gegenbauer polynomials have weight function `(1-t²)`. When you expand in Gegenbauer, the weight function algebraically cancels the singularity. The problem evaporates.

This connection — Gegenbauer expansion cancels this specific singularity — is non-obvious to human intuition but findable by a system that can systematically try all basis expansions in its training data.

The marketing analog: "revenue at risk" framing → 3x reply rate. This is non-obvious. It requires knowing that loss aversion > gain framing in cold outreach, AND that "revenue" is more specific than "growth", AND that "at risk" creates urgency without being aggressive. A human might find this with luck and good intuition. The loop finds it systematically by testing each element.

The mechanism is the same. The domain changes. The pattern holds.
