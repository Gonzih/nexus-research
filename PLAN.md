# PLAN: Tool-Call Memory System Research

## Task Restatement

Deep research + spec document for a system that intercepts every Claude Code tool call via hooks, stores them in PostgreSQL, embeds/chunks results, and exposes an MCP server for querying "global brain" memory across sessions and projects. Output: `memory-system/RESEARCH.md`.

## Approach

Pure research task — no code to write. Steps:
1. Fetch primary sources (mem0, Zep/Graphiti, Letta, nexus-reasoning-graph, Claude Code hooks docs)
2. Analyze each system: PostgreSQL support, MCP interface, tool-call indexing, project tagging
3. Write comprehensive RESEARCH.md covering all 8 required sections

## Files to Touch

- `PLAN.md` — this file
- `TODO.md` — task tracker  
- `memory-system/RESEARCH.md` — primary deliverable

## Risks

- Some repos/docs may be private or return 404
- Claude Code hooks documentation may not be on GitHub; may need to search alternatives
- nexus-reasoning-graph may be private
