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

### Iteration 4 (2026-02-07)
**Completed:**
- Researched Anthropic's Context Engineering Framework (Write/Select/Compress/Isolate)

**Key Findings:**
- Framework groups strategies into 4 buckets: Write (externalize), Select (JIT retrieval), Compress (summarize), Isolate (multi-agent)
- Server-side Compaction API available for Claude Opus 4.6 with `pause_after_compaction` feature
- Claude 4.5 models have built-in context awareness (token budget tracking)
- Multi-agent systems can use 15× more tokens than chat — isolation helps but costs latency
- OpenClaw already implements all 4 strategies; main opportunities are context awareness and server-side compaction upgrade

### Iteration 5 (2026-02-07)
**Completed:**
- Researched Dynamic Relevance Scoring techniques

**Key Findings:**
- Composite scoring (recency + importance + relevance) is the validated approach from Generative Agents research
- Recency via exponential decay (0.995^hours), importance via LLM-rating (1-10), relevance via cosine similarity
- Cross-encoder reranking improves RAG accuracy by 20-35% with 200-500ms latency
- TALE framework achieves 68.9% token reduction with <5% accuracy loss via budget-aware reasoning
- Semantic chunking + proximity scoring: 40% context reduction with 93.3% task completion
- Leading rerankers: Cohere Rerank 4 (32K context), BGE Reranker (open-source), RankZephyr (LLM-based)
- JIT context selection: allocate context based on query complexity, not fixed ratios

### Iteration 6 (2026-02-07)
**Completed:**
- Researched Multi-agent Context Isolation patterns

**Key Findings:**
- Subagent isolation achieves 10-40× compression on heavy operations (raw output → summary)
- 67% fewer tokens in parent context with proper delegation
- Target 40-60% context utilization in parent agent for optimal reasoning
- Anthropic's multi-agent research system: 90.2% outperformance vs single-agent
- Token usage explains 80% of performance variance in research tasks
- Claude Code Task tool provides production-ready subagent infrastructure
- Artifact-based communication: subagents write to files, return references (avoids transit through context)
- Parallel subagents run concurrently for independent analyses (security + tests + docs)
- Use Haiku/Sonnet for subagent work, reserve Opus for parent reasoning
- Emerging feature: `isolated: true` parameter for zero-parent-context (GitHub #20304)

**Next Steps:**
- Knowledge graphs & RAG approaches (Graphiti)
- Compile final recommendations
