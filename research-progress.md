# Research Progress

## Status: IN PROGRESS

## Iterations

### Iteration 1 (2026-02-07)
**Completed:**
- Researched Supermemory: architecture, Claude Code plugin, graph-based storage, pricing ($19/mo Pro)
- Researched Mem0: compared with Supermemory, architecture overview

**Key Findings:**
- Supermemory's hook-based approach (inject at start, capture at end) solves compaction survival elegantly
- External memory APIs reduce per-request context by ~70% vs full conversation replay
- Supermemory claims 25× faster retrieval than Mem0

### Iteration 2 (2026-02-07)
**Completed:**
- Researched Tool Schema Optimization: MCP Tool Search, defer_loading, token-efficient tool use beta

**Key Findings:**
- MCP Tool Search achieves 85% reduction in tool schema tokens (77K → 8.7K)
- Auto-activates when tool definitions exceed 10K tokens (now default in Claude Code)
- Token-efficient tool use built into Claude 4+ models (up to 70% output token reduction)
- Experimental MCP CLI flag (`ENABLE_EXPERIMENTAL_MCP_CLI=true`) can eliminate MCP overhead entirely but has trade-offs
- Accuracy improves with selective loading: Opus 4.5 went from 79.5% → 88.1%

### Iteration 3 (2026-02-07)
**Completed:**
- Researched MemGPT/Letta tiered memory architecture
- Researched A-MEM (Zettelkasten-inspired memory system)

**Key Findings:**
- MemGPT treats context as "RAM" with paging to external "disk" storage
- Base overhead: ~2K tokens for system instructions + core memory + functions
- Letta V1 moves away from tool-based control to native model reasoning
- Letta filesystem approach (74.0% LoCoMo) beats Mem0 graph (68.5%)
- Key insight: "Agent capabilities matter more than the tools"
- A-MEM uses Zettelkasten note-linking for connected context retrieval
- OpenClaw already has similar patterns to MemGPT (MEMORY.md, compaction, hybrid search)

**Next Steps:**
- Anthropic context engineering framework (Write/Select/Compress/Isolate)
- Dynamic relevance scoring and multi-agent context isolation
- Knowledge graphs & RAG approaches (Graphiti)
