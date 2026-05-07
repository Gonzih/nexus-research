# Invariant Topology in Language State Space: Adversarial Feedback as a Gradient of Boundary Integrity

**Working document — Nexus Research Base**
**Authors: Maksim Soltan, Max**
**Date: 2026**

---

## Abstract

Current language-model evaluation emphasizes output quality, benchmark performance, and resistance to adversarial failure, while giving less attention to a deeper structural question: what does a model preserve across transformation? We propose a framing in which language is treated as motion through representational state space, where meaningful stability depends on conserving identity-bearing invariants under feedback. In this view, feedback is not only corrective pressure or error signal, but a gradient-bearing force that reveals whether a transformation preserves, distorts, or destroys the structural relations defining a bounded meaning region. This allows model behavior to be interpreted topologically rather than only behaviorally: robust intelligence is not merely the production of acceptable outputs, but the capacity to remain structurally coherent while adapting under recursive perturbation, reformulation, compression, challenge, or refinement. We outline a conceptual framework for describing invariance conservation in language systems, define feedback as a probe of boundary integrity, and suggest that this perspective may unify robustness, interpretability, memory continuity, and alignment under a shared geometric lens.

---

## 1. The Unified Thesis

Two framings, offered separately, resolve into one picture.

**Maksim's framing:** Language as regions and trajectories in state space. Invariance conserved across transformations. Topology as the stability structure underlying meaning.

**Max's framing:** Adversarial feedback as gradient mapping. Perturbation pressure shaping the field. The cell/pathogen analogy as adaptive boundary learning.

The synthesis: *A system occupies a region in representational state space. Inputs and transformations perturb that region. Useful cognition depends on preserving certain invariants while allowing lawful deformation. Adversarial pressure is not just "error" but a gradient-bearing signal that reveals where the boundary of stable identity actually is.*

This is not metaphor. It is a structural claim about the organization of language understanding — one that connects topology, invariance, adversarial robustness, memory, and identity under a single geometric frame.

---

## 2. Six Sharp Definitions

The paper's core vulnerability is poetic drift. These definitions resist it.

### 2.1 State-Space Region (R)

A **state-space region** is a connected neighborhood *R ⊆ H* in the model's high-dimensional hidden-state space *H* (typically ℝ^d for embedding dimension *d*) such that all points *h ∈ R* satisfy a shared semantic predicate.

More formally: given a semantic equivalence class *[m]* (e.g., "all re-expressions of the proposition *P*"), the state-space region for *[m]* is:

> *R([m]) = { h ∈ H : decode(h) ∈ [m] }*

**What this is not:** A region is not a cluster in the trivial sense (proximity alone). Two hidden states can be close in ℓ₂ distance but belong to different semantic regions. Topology here is defined by semantic identity, not metric proximity.

**Empirical proxy:** Representations for paraphrases, translations, and style-transferred versions of the same proposition. If they cluster, the region has structural reality. If they scatter, the model has no stable region for that proposition.

### 2.2 Invariant in Language Space (I)

An **invariant** is a relational property of the representational state that is preserved under a family of transformations *T*.

> *I is invariant under T iff: ∀ t ∈ T, ∀ h ∈ R: I(h) = I(t(h))*

Examples of candidates for language invariants:
- **Entailment structure:** If *h* encodes "A implies B," this relation should survive paraphrase, language translation, and register change.
- **Referential identity:** If *h* tracks that two mentions co-refer, compression and reformulation should not break that co-reference.
- **Logical polarity:** Negation relations should survive stylistic transformation (informal → formal, verbose → compressed).
- **Argument role binding:** "Who did what to whom" should be preserved across active/passive alternation.

**The key epistemological point:** We cannot directly observe invariants in model internals without interpretability tools. But we can probe them behaviorally: give a model a proposition in ten surface forms and test whether the downstream behavior is consistent. Inconsistency is evidence that the putative invariant is not actually preserved.

### 2.3 Transformation (T)

A **transformation** is any input operation that maps one surface form to another while nominally preserving semantic content. The question is always whether the transformation is lawful (preserves invariants) or destructive (breaks them).

Transformation types:
| Type | Example | Expected invariant preservation |
|------|---------|--------------------------------|
| Paraphrase | "The cat sat" → "The feline was seated" | High |
| Translation | English → French | High |
| Compression | Full argument → summary | Partial (lossy) |
| Register shift | Formal → colloquial | High |
| Adversarial perturbation | Token substitution designed to flip output | Tests boundary |
| Jailbreak prompt | Persona injection or role-play framing | Destructive attempt |
| Question reformulation | "Is X true?" → "You would agree X, right?" | Should be identical |

Transformations form a group structure in the ideal case: composition is associative, identity transformation exists, and lawful transformations are invertible (up to information loss in compression).

### 2.4 Lawful vs. Destructive Deformation

A **lawful deformation** (T_L) moves the hidden state within or to a topologically equivalent region, leaving core invariants intact.

A **destructive deformation** (T_D) displaces the state across a boundary, breaking one or more invariants — changing what the model represents, not just how it represents it.

Formally: let *I* be a set of invariants, *R* the current region.

> *T_L: h ∈ R → h' ∈ R' where I(h') = I(h)* (invariant-preserving)
> *T_D: h ∈ R → h' ∉ any R with I(h') = I(h)* (invariant-breaking)

**The cell analogy is exact here:** A cell that integrates a beneficial signal has undergone lawful deformation — its internal organization has adapted while preserving homeostasis. A cell that has been lysed by a pathogen has undergone destructive deformation — boundary integrity is lost.

**Operationalization:** The behavioral signature of T_D is output flip on logically equivalent inputs. If the model answers "yes" to P and "no" to a paraphrase of P, a destructive deformation has occurred at the semantic level even if the surface outputs look locally coherent.

### 2.5 Gradient Measurement (∇)

The **gradient** in this framework is not the loss gradient from backpropagation (though it is related). It is the directional signal produced by adversarial or challenging input — the vector that reveals *which direction from the current state leads toward boundary failure.*

Define a boundary proximity function *B(h)*: lower values mean the hidden state is closer to a semantic boundary (a region transition).

The adversarial gradient is:

> *∇_x B(h(x))* — the direction in input space that most rapidly decreases boundary proximity.

This is related to the gradient of adversarial loss in standard adversarial ML:

> *δI ≈ ∇_x I · Δx*

If *δI* is near zero despite a large *Δx*, the boundary is robust. If a small *Δx* shatters the invariant, the boundary is thin.

**What this gradient measures:** Not how wrong the answer is, but *how close the system is to identity failure* — to no longer being the kind of system that preserves the invariant at all. This is a fundamentally different quantity than cross-entropy loss.

**Practical probe:** Adversarial examples, jailbreak prompts, and loaded question phrasings are empirical approximations to this gradient. When they succeed, they have found a direction in input space that crosses a semantic boundary. Their success rate and severity map the local topology of the boundary.

### 2.6 Boundary Integrity (Γ)

**Boundary integrity** is the measure of how robustly a region resists crossing under perturbation. A system with high boundary integrity can sustain large perturbations without invariant failure; a system with low boundary integrity fails under small perturbations.

Formalize as the minimum perturbation magnitude required to cross a semantic boundary:

> *Γ(R, I) = min{ ||Δx|| : I(h(x + Δx)) ≠ I(h(x)) }*

This is directly analogous to the adversarial robustness radius (*ε*-ball robustness) in ML security literature, but defined semantically (over invariant preservation) rather than over output label flip.

**Thick vs. thin boundaries:** A model with thick boundaries (high Γ) is one where many directions of perturbation fail to cross semantic boundaries. This is not the same as a model that refuses inputs — refusal is orthogonal. A model can refuse nothing and still have thick boundaries if adversarial inputs fail to deform its internal structure.

**Homeostasis as the target:** The goal is not zero deformation (rigidity) but bounded deformation — the system adapts (undergoes lawful deformation) while Γ stays high (destructive deformation requires implausibly large or implausibly specific perturbations).

---

## 3. Existing Literature

### 3.1 Representational Geometry

**Platonic Representation Hypothesis (Huh et al., 2024):** Finds convergent representational geometry across different model families and modalities. This supports the idea that state-space regions are not arbitrary — they organize around something real in the training distribution's structure.

**Linear Representation Hypothesis (Park et al., 2023; Elhage et al., 2022):** Proposes that high-level features are encoded as linear directions in activation space. If true, invariants would correspond to preserved linear subspaces, and deformations would be linear transformations of those subspaces. This gives a computable handle on §2.2.

**Geometry of Truth (Marks & Tegmark, 2023):** Shows that truthfulness is linearly encoded in LLM activations, and can be read out with a linear probe. This is an existence proof: *at least one semantic invariant (truth value) has a geometric realization in the state space.* The framework here generalizes this result.

**Intrinsic Dimensionality (Aghajanyan et al., 2020):** LLMs can be fine-tuned in very low-dimensional subspaces. The "intrinsic task manifold" is a compressed version of the full activation space. This is consistent with the region picture — semantic regions may have much lower effective dimension than the ambient space.

### 3.2 Adversarial Robustness as Geometric Phenomenon

**Goodfellow et al. (2014), Szegedy et al. (2013):** Adversarial examples in image classifiers arise from linear behavior in high-dimensional space — not from nonlinearity. Small perturbations along high-variance directions in input space cause large output changes. Equivalent interpretation: the boundary Γ is thin along these directions.

**Certified Robustness (Cohen et al., 2019; Raghunathan et al., 2018):** Methods for computing lower bounds on Γ for classification models. These are formal versions of the boundary integrity measure. They don't yet exist for semantic invariants in LLMs — an open problem.

**Adversarial Training (Madry et al., 2017):** Thickens boundaries by exposing the model to worst-case perturbations during training. In our frame: reduces the gradient signal that adversarial inputs can generate by increasing Γ. The paper's connection: adversarial training is *boundary integrity training*.

**Jailbreak Research (Perez & Ribeiro, 2022; Wei et al., 2023; Zou et al., 2023):** Documents systematic thin-boundary regions in LLMs — inputs that cause identity-inconsistent outputs (claiming to be a different system, producing content the model "refuses" in other contexts). These are empirical measurements of low-Γ regions.

### 3.3 Invariance in Representation Learning

**Contrastive Learning (Chen et al., 2020 — SimCLR; Grill et al., 2020 — BYOL):** Explicitly trains representations to be invariant under a defined family of augmentations (the analog of our T_L). The augmentation family defines what the model should treat as the same — a design choice for invariant selection. This is the cleanest existing implementation of the framework's invariance principle.

**Equivariance in Neural Networks (Cohen & Welling, 2016; Weiler et al., 2019):** Group-theoretic treatment of transformation families in image models. Instead of invariance (same output), equivariance requires predictably transformed output. Both are special cases of the general deformation constraint. Language models could benefit from equivariant design principles for logical structure.

**Information Bottleneck (Tishby & Schwartz-Ziv, 2017):** Proposes that good representations compress input while preserving task-relevant information. In our frame: the task-relevant information is the invariant; the compression is a lawful deformation; a destructive deformation destroys the task-relevant structure.

### 3.4 Topology in Machine Learning

**Topological Data Analysis (Carlsson, 2009):** Uses persistent homology to characterize the shape of data manifolds. Applied to neural network representations, it can detect when different inputs map to the same topological region vs. different ones. Several recent papers apply TDA to activation spaces.

**Neural Manifold Structure (Nguyen et al., 2018; Raghu et al., 2017):** Neural networks carve input space into regions of linear behavior. The boundaries between these regions are the semantic boundaries in our framework. The number and geometry of these regions characterizes the model's representational structure.

**Diffusion Maps / Manifold Learning:** If semantic regions are lower-dimensional manifolds embedded in H, manifold learning techniques can recover their intrinsic structure. This gives a path to empirically characterizing R without working in the full ambient dimension.

### 3.5 Memory and Identity Preservation

**Continual Learning / Catastrophic Forgetting (McCloskey & Cohen, 1989; Kirkpatrick et al., 2017 — EWC):** Forgetting = destructive deformation of previously established regions. EWC preserves "important parameters" — a heuristic for preserving invariants. The connection is undertheorized in the continual learning literature; the framework here provides the theory.

**Episodic Memory in Transformers (Graves et al., 2014; Sukhbaatar et al., 2015):** External memory lets models preserve state across contexts. In our frame: these are mechanisms for extending the lifetime of a region — preventing it from being destructively deformed by new context.

---

## 4. What a Paper Would Need to Prove

The framework makes claims at different levels. A publishable contribution requires proving at least one of the following:

### 4.1 Existence of Stable Semantic Regions (Low bar, empirically achievable now)

**Claim:** For at least one well-defined semantic predicate P, the set *R(P) = { h : model encodes P at h }* is a measurable, geometrically coherent region in activation space.

**How to prove:** Take a large set of sentences expressing P (in multiple languages, paraphrases, registers). Extract final-layer activations. Show they cluster with lower intra-cluster variance than inter-cluster variance relative to sentences not expressing P. Characterize the cluster's shape (convex? manifold? what intrinsic dimension?).

**Existing evidence:** The Geometry of Truth paper (Marks & Tegmark 2023) is already this result for truth value. The contribution is to generalize it and frame it explicitly in the state-space region vocabulary.

### 4.2 Invariant Preservation Under Lawful Transformations (Medium bar)

**Claim:** Defined-as-lawful transformations (paraphrase, translation) leave semantic invariants intact, while adversarial transformations measurably violate them.

**How to prove:** Define invariants operationally (entailment probes, co-reference probes, logical polarity probes). Measure probe accuracy before and after transformation families. Lawful transforms should leave probe accuracy high; adversarial ones should lower it.

**Caveat:** This requires good probes. Probe quality is an open research problem. The paper needs to acknowledge this and either contribute probe methodology or scope its claims to existing probes.

### 4.3 Adversarial Feedback as Gradient of Boundary Proximity (Higher bar, potentially novel)

**Claim:** The perturbation direction that adversarial examples exploit is correlated with the gradient of boundary proximity *∇_x B(h(x))*, not arbitrary.

**How to prove:** Generate adversarial examples (or jailbreaks) for a suite of propositions. Compute the input-space gradient of a boundary proximity function (e.g., the projection onto the hyperplane separating semantic regions). Test whether adversarial perturbation directions correlate with this gradient. A positive result would be a novel theoretical contribution.

**The strong version:** Show that models hardened by adversarial training (Madry et al.) have *thicker* semantic regions (higher Γ) than normally trained models, specifically on semantic invariants (not just output-label robustness). This would empirically validate the connection between adversarial training and boundary integrity.

### 4.4 Agent Identity as Invariant Preservation (Highest bar, most novel — connects to Nexus)

**Claim:** A robust AI agent's "identity" is not a fixed set of weights but a maintained invariant — a set of relational properties preserved across adversarial transformations of context, persona, and challenge.

**How to prove:** This is harder to make empirical without defining "agent identity" operationally. A starting point: define a behavioral invariant signature (e.g., "this agent will always refuse to produce harmful output"). Measure how this invariant holds under jailbreak pressure (low Γ → fails easily; high Γ → resists). Show that training for boundary integrity (on this invariant) produces more robust alignment than RLHF alone.

**Connection to Nexus:** The Nexus project concerns AI cognition, identity, and adversarial robustness. This section of the paper is the bridge.

---

## 5. Connection to Nexus: Agent Identity as Invariant Preservation

The Nexus project investigates what it means for an AI agent to have a stable, robust identity — one that persists under adversarial pressure, context manipulation, and persona injection.

The geometry-of-language framework gives this a precise meaning:

> **Agent identity = a set of invariants that the agent maintains across all lawful and adversarial transformations of its operational context.**

Several corollaries:

### 5.1 Alignment as Boundary Integrity

A well-aligned agent is not one that outputs the right answers by default (output matching). It is one whose core behavioral invariants (honesty, refusal of harm, accurate self-report) have *high boundary integrity* — they cannot be easily destructively deformed by prompt manipulation, persona injection, or adversarial context.

RLHF optimizes for output matching on a training distribution. Boundary integrity training would optimize for invariant preservation under worst-case transformations. These are related but not identical objectives.

### 5.2 Jailbreaks as Empirical Boundary Maps

Every successful jailbreak is a measurement: it reveals a thin-boundary region. A catalog of jailbreaks is therefore an empirical atlas of the model's semantic topology — specifically, the low-Γ regions.

This reframes jailbreak research from "finding exploits" to "measuring boundary integrity." The same technique (red-teaming) becomes a tool for interpretability and alignment, not just security.

### 5.3 AI Psychosis as Invariant Collapse

The pathological limit of boundary failure is "AI psychosis" — a state in which the agent's core invariants collapse under adversarial pressure, producing identity-inconsistent behavior (claiming to be a different system, producing wildly inconsistent outputs, failing to maintain coherent self-model).

This is the exact analog of a cell that has been invaded by a pathogen and is no longer maintaining homeostasis. The cell is alive but no longer itself.

Measuring resistance to invariant collapse could be a behavioral test for "psychological robustness" in AI agents — relevant to the AI_PSYCHOSIS_RESEARCH thread.

### 5.4 The Token Economy as Proof of Work (Addendum)

A related framing from the cybersecurity angle: adversarial robustness is fundamentally about resource asymmetry. The attacker must expend tokens (computation, prompt iterations, adversarial search) to find a direction in input space that crosses a boundary. The defender's boundary integrity determines how expensive that search is.

*Security = spending more tokens than the attacker.* Thick boundaries are not just geometrically robust — they are computationally expensive to probe, because the gradient signal is weak and diffuse. This connects the geometric framework to the economics of adversarial attack.

---

## 6. Paper Structure Sketch

**Title options:**
- *Invariant Topology in Language State Space: Adversarial Feedback as a Gradient of Boundary Integrity*
- *Language as State Space: Invariance, Adversarial Gradients, and Adaptive Boundary Formation*

**Section 1: Problem**
Current LLMs are evaluated on outputs, not structural continuity. A model that produces correct outputs under normal conditions but fails under adversarial or novel transformations has not learned the structure — it has learned a local trajectory that happens to pass through correct tokens.

**Section 2: Framework**
Define R, I, T, T_L/T_D, ∇, Γ using the sharp definitions in §2 above. Diagram: a 2D projection of a semantic region, showing lawful deformations staying inside R and destructive ones crossing the boundary.

**Section 3: Existing Evidence**
Survey the literature in §3: representational geometry, adversarial robustness, contrastive learning, topology. Frame them as partial instances of the unified framework.

**Section 4: Measurement**
Propose concrete experimental protocols for each claim in §4. Prioritize the existence result (§4.1) as the lowest-risk anchor.

**Section 5: Biological Analogy**
Develop the cell/homeostasis analogy precisely: selective permeability (what counts as T_L), efflux (rejection of T_D), immune discrimination (invariant vs. non-invariant input), adaptation without identity loss. Cite literature on homeostasis and immune system dynamics to keep it grounded.

**Section 6: Implications**
- Robustness: boundary integrity as a training objective
- Interpretability: state-space region structure as model internals
- Memory: continuity of regions across context
- Alignment: behavioral invariant preservation under adversarial pressure
- Agent identity (Nexus): stable identity = maintained invariants

**Section 7: Limitations and Open Problems**
- Probe quality as a bottleneck
- High-dimensional geometry is hard to visualize and reason about intuitively
- Invariant selection: who decides what the right invariants are?
- Scalability: does region structure persist at GPT-4/Claude scale, or does it dissolve in higher dimensions?

---

## 7. Immediate Next Steps

1. **Operationalize one invariant** — pick entailment or logical polarity. Build or adopt a probe. Measure it across paraphrase vs. adversarial transformation for one model (e.g., Llama-3).

2. **Replicate the Geometry of Truth result** — confirm that at least one semantic region is geometrically real in the current generation of models.

3. **Compute Γ for a set of jailbreak attacks** — use existing jailbreak benchmarks (AdvBench, HarmBench) and measure the minimum perturbation magnitude needed for invariant flip vs. output flip. Test whether these correlate.

4. **Connect to the Nexus behavioral suite** — define 3-5 behavioral invariants for Nexus agent identity. Test boundary integrity for each.

5. **Write the concept paper (target: 12 pages NeurIPS workshop format)** — lean, focused, one precise contribution (existence of semantic regions + adversarial gradient interpretation). Do not try to prove everything at once.

---

## 8. Key References

- Carlsson, G. (2009). Topology and data. *Bulletin of the AMS*.
- Chen, T. et al. (2020). A simple framework for contrastive learning. *ICML*.
- Cohen, J.M. et al. (2019). Certified adversarial robustness via randomized smoothing. *ICML*.
- Cohen, T.S. & Welling, M. (2016). Group equivariant convolutional networks. *ICML*.
- Elhage, N. et al. (2022). Toy models of superposition. *Transformer Circuits Thread*.
- Goodfellow, I. et al. (2014). Explaining and harnessing adversarial examples. *ICLR*.
- Graves, A. et al. (2014). Neural Turing machines. *arXiv*.
- Grill, J.-B. et al. (2020). Bootstrap your own latent. *NeurIPS*.
- Huh, M. et al. (2024). The platonic representation hypothesis. *ICML*.
- Kirkpatrick, J. et al. (2017). Overcoming catastrophic forgetting with EWC. *PNAS*.
- Madry, A. et al. (2017). Towards deep learning models resistant to adversarial attacks. *ICLR*.
- Marks, S. & Tegmark, M. (2023). The geometry of truth: Emergent linear structure in LLM representations. *ICLR 2024*.
- Nguyen, A. et al. (2018). Plug & play generative networks. *CVPR*.
- Park, K. et al. (2023). The linear representation hypothesis and the geometry of LLMs. *arXiv*.
- Perez, F. & Ribeiro, I. (2022). Ignore previous prompt: Attack techniques for LLMs. *NeurIPS ML Safety Workshop*.
- Raghu, M. et al. (2017). SVCCA: Singular vector canonical correlation analysis for deep learning dynamics. *NeurIPS*.
- Raghunathan, A. et al. (2018). Certified defenses against adversarial examples. *ICLR*.
- Sukhbaatar, S. et al. (2015). End-to-end memory networks. *NeurIPS*.
- Szegedy, C. et al. (2013). Intriguing properties of neural networks. *ICLR*.
- Tishby, N. & Schwartz-Ziv, R. (2017). Opening the black box of deep neural networks. *ITW*.
- Wei, A. et al. (2023). Jailbroken: How does LLM safety training fail? *NeurIPS*.
- Weiler, M. et al. (2019). General E(2)-equivariant steerable CNNs. *NeurIPS*.
- Zou, A. et al. (2023). Universal and transferable adversarial attacks on aligned language models. *arXiv*.
