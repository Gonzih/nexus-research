# Meta-Layer Initiative — Corpus Summary

**Archived:** 2026-05-07
**Documents:** 27 ML-Drafts
**Source:** https://govhub.live/doc/all/

---

## What Is the Meta-Layer Initiative?

The Meta-Layer Initiative is designing a governance infrastructure for the open web — a reflexive layer that sits *above* existing platforms, apps, and protocols without replacing them. Think of it as the web's missing governance stratum: a set of shared standards, identity primitives, and community coordination mechanisms that allow participants to carry their identity, data, reputation, and governance rights across digital environments without losing them when they change platforms.

The meta-layer is described as "the condition under which the web functions as a shared system rather than a collection of connected but incompatible domains" (DP7). It is explicitly positioned as a non-extractive alternative to today's surveillance-capitalism model: participants own their digital artifacts, communities self-govern their spaces, and value flows are transparent.

The initiative is organized under ISOC Nevada and operates through an Infrastructure SIG. Governance documents are published as ML-Drafts, community-reviewed, and promoted to ML-RFCs when rough consensus is reached.

---

## What These Documents Collectively Describe

The 27 documents fall into three categories:

### 1. Foundational Philosophy (5 docs)
- **ML-Draft-001** establishes the SIG's governance process itself: how documents are written, reviewed, and promoted; what projects, guilds, and workgroups are; what stabilization means.
- **ML-Draft-015 (The Teilhard Test)** provides the philosophical framework: technology should be evaluated not just for capability but for whether it advances humanity's "developmental maturity" — reflexivity, convergence without coercion, preservation of differentiation, and alignment of power with responsibility. The Teilhard Test is the civilizational benchmark against which the meta-layer's own design is measured.
- **ML-Draft-014 (Enduring Artifacts)** diagnoses the epistemic crisis caused by generative AI disrupting the noosphere's pace layers, and argues for immutable digital artifacts as stabilizing infrastructure.
- **ML-Drafts-005/013 (Civic Digital Artifacts)** define the artifact type that underpins the system: verifiable, persistent, community-owned records that serve as anchors for identity, reputation, and governance.

### 2. Desirable Properties (DPs) — The Normative Core (21 docs)
Twenty-one documents specify the "Desirable Properties" (DP1–DP21) that any meta-layer implementation should satisfy. These are detailed normative specifications that each:
- Name a failure mode the property prevents
- Define the core principle
- Specify primary mechanisms and structural conditions
- Describe governance/accountability surfaces
- Analyze incentives and power dynamics
- Identify open questions for future ML-RFC development

### 3. Operational Specs Embedded in DPs
Several DPs contain detailed technical or operational specifications:
- **DP5** specifies the decentralized namespace system with meta-domains and personal identifiers
- **DP9** specifies incentive schemas for contributors, maintainers, and open-source developers
- **DP10** specifies the PEARL digital badge schema for lifelong learning credentials
- **DP13** specifies AI containment tiers, rate limits, and rollback mechanisms
- **DP15** specifies provenance artifact formats and security investment requirements

---

## Key Themes Across the Corpus

### 1. Sovereignty and Non-Extraction
The corpus is built on a consistent rejection of the extractive web model. Every DP that touches data (DP4), commerce (DP6), incentives (DP9), or finance (DP17) contains explicit "non-goals" and "failure modes" for extractive behaviors: hidden data harvesting, dark patterns, attention rents, invisible monetization. The meta-layer is explicitly designed so participants *own* their data, names, artifacts, and community infrastructure — not renting it from platforms.

### 2. Layered, Federated Governance
The meta-layer is not a central authority. It is a set of coordination standards and governance primitives that communities instantiate locally. DP3 (Adaptive Governance) and DP8 (Meta-communities/Governance Zones) together define how communities self-govern while remaining interoperable. DP7 (Interoperability) ensures that governance decisions made locally don't trap participants — things must remain portable and enforceable across boundaries.

### 3. AI as Both Tool and Subject of Governance
Three full DPs (DP11, DP12, DP13) and parts of seven others address AI governance. The framework treats AI as something communities must be able to *define rules for*, *audit in real time*, *contain when it misbehaves*, and *assign reputation to*. DP12 is particularly notable: it insists that communities — not platforms or model providers — must hold the runtime authority to define what AI can and cannot do in their spaces.

### 4. Trust Infrastructure Built from Verifiable Artifacts
The technical substrate is always "verifiable artifacts with provenance." DP1 (federated identity), DP15 (security and provenance), DP5 (decentralized namespace), and the civic digital artifacts papers all converge on the same primitive: cryptographically verifiable, immutable records that carry authorship, timestamps, and lineage. This is the technical response to the epistemic crisis described in ML-Draft-014 and ML-Draft-015.

### 5. Feedback Loops as Governance Mechanism
DP18 (Feedback Loops & Reputation) is the learning layer: the system must be able to update itself based on participant experience. Reputation is treated not as a score but as contextual memory that decays, repairs, and carries provenance. Feedback must close: participants must be able to see whether their input was received, routed, acted upon, or not — with explanations when not. This connects to DP3 (Adaptive Governance): governance evolves through feedback, not just through committee decisions.

### 6. Human-First Multi-Modal Design
DP21 (Multi-Modal Interactions) frames the interface layer with an accessibility-first, graceful degradation requirement. The meta-layer must work across text, audio, visual, haptic, and spatial modalities without losing semantic continuity. This is the forward-looking extension of DP2 (Participant Agency): meaningful agency requires interfaces that work for all participants, not just those with optimal hardware.

### 7. Financial Sustainability as a Design Constraint
DP17 (Financial Sustainability) treats funding transparency as a first-class governance requirement, not a business consideration. Revenue models, allocation breakdowns, and maintenance budgets must be disclosed. Funding must be sufficient to maintain core functions — governance, safety, and infrastructure — and those budgets must be ring-fenced. The failure mode "assurance starvation" (underfunding security to inflate growth metrics) is explicitly prohibited.

---

## The DP Dependency Graph (Key Relationships)

The DPs are not independent specifications — they reference each other extensively. The most critical dependencies:

- **DP1** (identity) is the foundation for everything: reputation (DP18), governance (DP12), commerce (DP6), and interoperability (DP7) all depend on accountable identity
- **DP3** (adaptive governance) provides the mechanism by which all other DPs evolve over time
- **DP4** (data sovereignty) is a constraint on DP6 (commerce), DP9 (incentives), DP18 (reputation), and DP19 (engagement)
- **DP13** (AI containment) is the enforcement layer for DP11 (ethical AI) and DP12 (community AI governance)
- **DP15** (security/provenance) provides the verification substrate for DP7 (interoperability) and DP1 (identity)
- **DP17** (financial sustainability) is the resource layer that makes all other DPs viable over time

---

## Status and Maturity

All 27 documents are marked `approved` as ML-Drafts, with 5 documents at Revision 01 (ML-Draft-001, 002, 003, 004, 006) indicating community review has already produced updates. No documents have yet been promoted to ML-RFC status. Each DP document contains a "Path Toward ML-RFC" section specifying what implementation evidence and community consensus is required for promotion.

The corpus represents a coherent, mutually-reinforcing governance framework at an early but substantive stage of development — past initial ideation, actively seeking implementors and community validation.
