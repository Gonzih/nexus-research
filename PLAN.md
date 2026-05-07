# PLAN: Archive govhub.live Meta-Layer Initiative Governance Docs

## Task Restatement

Scrape and archive all 27 governance documents from https://govhub.live/doc/all/ for the Meta-Layer Initiative (ISOC Nevada). Save as structured markdown files in `isoc-nevada/meta-layer-docs/`, create an index and summary, then PR+merge.

## Documents (27 total)

| Draft | Slug | Title |
|-------|------|-------|
| ML-Draft-027 | 73088jr6 | DP21 - Multi-Modal Interactions & Experiences |
| ML-Draft-026 | q8q132sr | DP10 - Education |
| ML-Draft-025 | fo686uq9 | DP20 - Community Ownership |
| ML-Draft-024 | nn2shgy8 | DP19 - Amplifying Presence & Community Engagement |
| ML-Draft-023 | xxf5ny7l | DP18 - Feedback Loops & Reputation |
| ML-Draft-022 | t4q8puic | DP17 - Financial Sustainability |
| ML-Draft-021 | 1n34m2mg | DP16 — Roadmap & Milestones |
| ML-Draft-020 | yu9ajp3x | DP15 — Security & Provenance |
| ML-Draft-019 | qnbbe3ki | DP14 - Trust & Transparency |
| ML-Draft-018 | b18kbksn | DP9 - Developer & Community Incentives |
| ML-Draft-017 | blzqopyd | DP8 - Meta-communities |
| ML-Draft-016 | 3kud2xmo | DP7 - Interoperability |
| ML-Draft-015 | 8a37qe9r | The Teilhard Test |
| ML-Draft-014 | bilz9fmo | Enduring Artifacts in the Age of AI |
| ML-Draft-013 | mjvwlfqx | Civic Digital Artifacts (v2) |
| ML-Draft-012 | rhg0086m | DP4 – Data Sovereignty and Privacy (short) |
| ML-Draft-011 | bptucnnk | DP3 - Adaptive Governance |
| ML-Draft-010 | egl2wnnf | DP6 - Commerce |
| ML-Draft-009 | kceeygnx | DP5 - Decentralized Namespace |
| ML-Draft-008 | rzif7bim | DP4 - Data Sovereignty & Privacy (full) |
| ML-Draft-007 | tdr7e0xe | DP2 - Participant Agency & Empowerment |
| ML-Draft-006 | fmpzlexh | DP1 - Federated Auth & Accountability (Rev 01) |
| ML-Draft-005 | 3f4o6c7u | Civic Digital Artifacts |
| ML-Draft-004 | 2069sg0n | DP13 – AI Containment (Rev 01) |
| ML-Draft-003 | 8bye41qq | DP12 - Community Governance of AI (Rev 01) |
| ML-Draft-002 | n1sy3631 | DP11 - Safe and Ethical AI (Rev 01) |
| ML-Draft-001 | qdupkive | Foundational Governance Practices (Rev 01) |

## Approach

1. Fetch all 27 documents in parallel batches via `curl -s "https://r.jina.ai/URL"`
2. Save each as a markdown file in `isoc-nevada/meta-layer-docs/`
3. Create INDEX.md and SUMMARY.md
4. Git: branch → commit → push → PR → merge

## Files to Touch

- `PLAN.md`, `TODO.md`
- `isoc-nevada/meta-layer-docs/ML-Draft-NNN-*.md` (27 files)
- `isoc-nevada/meta-layer-docs/INDEX.md`
- `isoc-nevada/meta-layer-docs/SUMMARY.md`

## Risks

- Rate limiting on r.jina.ai with 27 parallel requests — batch into groups of 6-7
- Document content may vary in quality (some are short: ML-Draft-012 is 1701 words, ML-Draft-014 is 806 words)
- ML-Draft-012 and ML-Draft-008 appear to cover the same topic (DP4) — note both
