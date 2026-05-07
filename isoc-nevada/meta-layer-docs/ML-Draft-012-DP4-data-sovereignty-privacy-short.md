# ML-Draft-012: DP4 - Data Sovereignty and Privacy (short version)

**Draft Number:** ML-Draft-012
**Revision:** 00
**Date:** 2026-04-22
**Authors:** The Meta-Layer Initiative
**Pages:** 5
**Word Count:** 1701
**Status:** approved
**Source URL:** https://govhub.live/doc/draft/rhg0086m/

---

## **DP4 – Data Sovereignty and Privacy**

## **1. Purpose of This Draft**

This draft articulates Desirable Property 4 (DP4) as the condition under which participants and communities can meaningfully govern data about themselves and their activity in the meta-layer.

DP4 does not treat privacy as a settings menu, a compliance ritual, or a legal disclaimer. It defines the conditions under which claims of ownership, consent, confidentiality, deletion, and portability remain meaningful in practice.

The core claim is that sovereignty over data depends on more than access controls. It depends on whether collection, inference, retention, sharing, and reuse are bounded by visible purposes, governed by revocable permissions, and constrained by structures that communities can understand and audit.

If DP4 is weak, predictable failures follow: consent theater, surveillance-by-default, inference without accountability, lock-in through broken portability, deletion promises that stop at the first vendor boundary, and community rules that cannot survive contact with underlying data pipelines.

DP4 therefore functions as a precondition for multiple later properties. Agency cannot be exercised over invisible data flows. Governance cannot constrain systems that communities cannot inspect. Ethical AI cannot be meaningful where the data it sees, stores, or trains on is structurally uncontrolled.

DP4 does not resolve all legal, jurisdictional, or sector-specific privacy questions. It defines the minimum conditions under which sovereignty and privacy remain real at the interface where data is created, combined, interpreted, and acted upon.

## **2. Problem Statement**

In today’s web, privacy is often presented as disclosure without control.

Participants are shown banners, terms updates, and granular-looking toggles, yet the underlying system still optimizes for maximal collection, indefinite retention, behavioral inference, and partner expansion. In many cases, the formal interface of consent exists while the operational reality of choice does not.

This produces recurring failures:

*   participants cannot tell what is being collected, for what purpose, for how long, or by whom 
*   data collected for one function expands into advertising, analytics, resale, or model training 
*   sensitive inferences are derived from behavior without clear notice, recourse, or deletion pathways 
*   data exports preserve liability but not usability, making exit technically legal but practically costly 
*   account deletion rarely maps cleanly to embeddings, downstream processors, backups, or partner copies 
*   communities cannot enforce stricter data norms inside their spaces because underlying systems remain vendor-shaped

These failures are not edge cases. They are structural consequences of architectures designed to treat data accumulation as default value creation.

DP4 addresses this by defining data sovereignty as an operational condition. Privacy becomes meaningful only when participants and communities can see the active terms of data use, limit those terms in practice, revoke permissions without fiction, and move or leave without losing the structure of their digital lives.

## **3. Threats and Failure Modes**

### **3.1 Consent theater**

Interfaces bundle unrelated processing into a single act of acceptance.

**Example:** A participant accepts a terms update to continue using a service and, in doing so, silently authorizes secondary uses of behavioral data for recommendation tuning, advertising, and model training.

**Why this matters:** The system records consent, but the participant did not experience a meaningful choice. DP4 treats this as a sovereignty failure, not a paperwork issue.

### **3.2 Purpose creep and secondary use**

Data collected for one function expands into new products, ranking systems, partner programs, or model behaviors without a fresh social contract.

**Example:** Location data collected for safety or delivery is later used for engagement scoring, ad targeting, or brokered partner analytics.

**Why this matters:** The participant’s mental model of risk becomes false. Trust erodes even where no obvious breach has occurred.

### **3.3 Illusory portability**

Export exists formally but fails functionally.

**Example:** A participant downloads an archive that contains files and timestamps but omits social graph edges, permission history, role context, provenance, or schemas needed to restore meaningful continuity elsewhere.

**Why this matters:** Exit is made to look possible while dependency is preserved. DP4 requires portability that preserves usable structure, not only raw payloads.

### **3.4 Inference without accountability**

Systems derive high-stakes conclusions from behavioral traces without clearly governing how those inferences are created, used, challenged, or removed.

**Example:** A wellness application infers stress or depression risk from typing cadence and browsing patterns, then shares a derived score with an advertising or insurance intermediary.

**Why this matters:** The participant never explicitly submitted the sensitive category, yet is still acted upon as if they had.

### **3.5 Retention without sunset**

Data persists because retention is cheap, deletion is operationally inconvenient, and analytics cultures prefer indefinite memory.

**Example:** A participant deletes an account, but vector embeddings, partner datasets, abuse-model features, and backup systems continue to retain traces with no coherent deletion pathway.

**Why this matters:** Sovereignty requires time bounds. Without them, institutions remember indefinitely while participants bear the burden of asymmetrical memory.

### **3.6 Cross-context correlation**

Identifiers, device graphs, and fingerprinting techniques merge activity across settings that participants experienced as distinct.

**Example:** Pseudonymous participation in a civic forum is quietly linked to shopping behavior, social browsing, or location history through shared infrastructure.

**Why this matters:** Plural identity becomes decorative. Communities cannot sustain contextual integrity if correlation silently defeats boundaries.

### **3.7 False anonymity and weak de-identification**

Organizations describe datasets as anonymized even where re-identification remains plausible or contractually enabled downstream.

**Example:** A mobility dataset stripped of names still exposes sparse routines in a small town, allowing individuals to be reconstructed through outside knowledge.

**Why this matters:** DP4 requires honesty about residual risk. “De-identified” cannot be treated as a magic word that dissolves responsibility.

### **3.8 Partner sprawl without propagation**

Deletion, revocation, and correction stop at the first layer of control.

**Example:** A participant deletes messages in one tool, but analytics vendors, cloud backups, and SDK partners continue to retain copies without visibility or participant recourse.

**Why this matters:** Sovereignty that fails at the first subcontractor boundary is not sovereignty.

### **3.9 Youth and vulnerable-context overexposure**

Defaults optimized for adult engagement expose minors and vulnerable users to data-intensive patterns they are less equipped to assess or contest.

**Example:** A youth-oriented social tool enables location sharing, behavioral profiling, or AI-mediated emotional inference by default.

**Why this matters:** DP4 requires higher baselines where stakes are higher. Uniform defaults can produce unequal harm.

## **4. Core Principle**

Data sovereignty and privacy in the meta-layer require that personal and community data be collected, inferred, stored, shared, and reused only under visible, bounded, and governable conditions.

Those conditions must include:

*   clear purpose binding 
*   minimization of collection, access, and retention 
*   meaningful consent and withdrawal 
*   portability with practical utility 
*   deletion or attenuation pathways that propagate as far as technically possible 
*   auditability of significant access, inference, and transfer events 
*   community capacity to impose stricter norms within governed zones

In today’s web, these conditions rarely hold together. A system may disclose collection without limiting reuse, provide deletion without propagation, or offer export without restoration value. DP4 treats such partial compliance as insufficient.

The meta-layer reframes privacy as operational control at the point of interaction.

**Example:** A participant opens a data lens and sees active purposes, relevant processors, current retention clocks, sensitive inferences attached to their account, and downstream systems that have accessed their data. They can revoke training permission, export their activity in an interoperable format, contest a high-risk inference, and receive a propagation receipt for deletion requests.

**What this feels like:** Privacy stops being a maze of legal text and becomes a set of understandable levers tied to real system behavior.

**Without this:** Privacy becomes trust in opacity, and opacity fails precisely where accountability matters most.

## **5. Primary Mechanisms and Structural Conditions**

### **5.1 Purpose binding**

Every collection and processing pathway must declare its purpose in terms legible to both participants and communities. Material changes in purpose require visible reauthorization, reclassification, or zone-level review.

### **5.2 Data minimization by design**

Systems must begin from the least collection, retention, and sharing compatible with the function being offered, and expand only through visible, justified choices.

### **5.3 Consent stack**

Permission must be layered, granular, and revocable, with separate scopes for distinct categories of data use.

### **5.4 Meaningful portability**

Portability must preserve enough structure to support continuity, not just compliance.

### **5.5 Retention clocks and deletion propagation**

Retention must be bounded and deletion must propagate to known downstream systems with auditable outcomes.

### **5.6 Sensitive inference governance**

High-risk inferences must be disclosed, bounded, and contestable.

### **5.7 Zone-scoped privacy norms**

Communities must be able to define stricter privacy norms within their zones while remaining interoperable.

### **5.8 Auditability and provenance of use**

Significant data access and use must be inspectable.

### **5.9 Training and model-use boundaries**

Data used for model training must be separately governed and revocable where possible.

### **5.10 Jurisdictional and transfer transparency**

Cross-border transfers and legal conditions must be visible to participants.

## **6. Governance, Accountability, and Agency Surfaces**

Participants must be able to:

*   see active data uses 
*   revoke permissions 
*   export usable data 
*   contest inferences

Communities must be able to:

*   define privacy norms 
*   enforce them in zones 
*   audit outcomes

## **7. Incentives and Power Analysis**

Data accumulation is often economically incentivized. DP4 requires that these incentives become visible and contestable rather than hidden.

## **8. Community Signals Informing DP4**

Recurring signals include:

*   fatigue with consent banners 
*   demand for real portability 
*   concern about invisible inference

## **9. Non-Goals and Explicit Boundaries**

DP4 does not:

*   guarantee anonymity in all contexts 
*   prohibit all inference or sharing 
*   replace legal frameworks

## **10. Minimum Alignment (Non-Normative)**

A DP4-aligned system should:

*   expose purposes and processors 
*   enable revocation 
*   support meaningful portability 
*   define retention limits

## **11. Open Questions and Future Work**

Key open questions include:

*   interoperability of privacy profiles 
*   model deletion and inference correction 
*   balancing privacy with accountability

## **12. Relationship to Other Desirable Properties**

DP4 underpins:

*   DP2 (agency) 
*   DP11 (ethical AI) 
*   DP12 (governance) 
*   DP13 (containment)

## **13. Path Toward ML-RFC**

Advancement requires:

*   standardized mechanisms 
*   implementation evidence 
*   governance adoption

## **14. Closing Orientation**

DP4 defines the conditions under which privacy and sovereignty are real, not symbolic. Without it, higher-order trust, governance, and AI safety properties cannot reliably function.
