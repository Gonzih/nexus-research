# PLAN: Nexus Research Migration + Geometry of Language

## Task Restatement

Migrate research from the `Gonzih/money-brain` GitHub repo into this new dedicated nexus-research repo. Then write an original research document on the geometry of language, based on a gist conversation between Maksim and Max. Finally, write a README for the repo.

## Constraint Discovered

**The `Gonzih/money-brain` repo returns 404** — it is private (or doesn't exist). Without an auth token, raw file fetches and the GitHub API both return "Not Found". Cannot migrate files that are inaccessible. Will create the folder structure and placeholder README files documenting what should live there, then proceed with everything else.

## Approaches Considered

### Approach A: Block on migration, don't proceed
- Pro: Completeness
- Con: Kills the whole task; geometry research and README are still valuable

### Approach B (chosen): Proceed without migration content; create folder scaffolding
- Create `_research/`, `research/`, `x-research/` directories with a `MIGRATION_NOTE.md` explaining the repo was private
- Create all standalone file placeholders with notes
- Do the geometry research and README fully
- Pro: Delivers maximum value; folder structure ready for when user grants access

### Approach C: Ask user for auth token
- Pro: Could actually migrate
- Con: User is an automated agent run; can't interactively ask

## Files to Touch

- `PLAN.md` — this file
- `TODO.md` — task tracker
- `_research/MIGRATION_NOTE.md` — migration status
- `research/MIGRATION_NOTE.md`
- `x-research/MIGRATION_NOTE.md`
- Standalone placeholder files: `nexus-soul.txt`, `NEXUS.md`, `EDGE_BOY.md`, `MYTHOLOGY_UNDERSTANDING.md`, `FRICTION_POINT_FRAMEWORK.md`, `8GAMES.md`, `AI_PSYCHOSIS_RESEARCH.md`, `ROBOTICS_INTELLIGENCE_SYNTHESIS.md`
- `geometry/geometry-of-language.md` — original research document (primary deliverable)
- `README.md` — repo overview

## Risks

- money-brain is private → handled by placeholder approach
- Geometry research quality: using the gist as source material, will synthesize carefully
