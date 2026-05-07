# PLAN: Datomic-Style Temporal Database for Agentic Systems

## Task Restatement

Research and write a comprehensive spec document for a Datomic-inspired temporal database designed specifically for AI agents, accessed exclusively via MCP tools. Output: `temporal-db/RESEARCH.md`.

## Approach

Pure research task — fetch primary sources in parallel, synthesize into a complete spec.

1. Fetch Datomic docs (data model, query, transactions, time-travel)
2. Fetch existing simpler implementations (datahike, datascript, datalevin)
3. Analyze agent access patterns and design MCP tool interface
4. Design PostgreSQL EAV(T) schema and indexes
5. Write comprehensive RESEARCH.md covering all 10 required sections

## Files to Touch

- `PLAN.md` — this file
- `TODO.md` — task tracker
- `temporal-db/RESEARCH.md` — primary deliverable

## Risks

- Datomic docs may have changed structure/URLs
- Some GitHub repos may have limited README info
- Agent-specific simplifications require synthesis from first principles
