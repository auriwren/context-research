# Research Findings: Reducing Context Bloat

## 1. Supermemory

### What It Is
Supermemory is a "Memory API for the AI era" — a managed service providing graph-based memory storage with knowledge graph structure instead of simple vector embeddings. Unlike RAG which answers "What do I know?", Supermemory answers "What do I remember about you?" with temporal tracking and entity relationships.

### How It Works
- **Graph-based storage**: Maintains entity recognition, relational mappings, temporal boundaries, and invalidation markers
- **Memory vs Documents**: Documents are stateless universal knowledge; Memories are stateful, user-specific insights with temporal validity filtering
- **Dual-layer approach**: Combines semantic search with graph traversal for context-aware retrieval
- **Auto-capture**: Conversations saved on session end via signal extraction (keywords like "remember", "architecture", "decision")
- **Context injection**: User profile (up to configurable `maxProfileItems`) injected at session start

### Claude Code Integration
The `@supermemory/claude-supermemory` plugin provides:
- **super-search**: Query past work and previous sessions
- **super-save**: Store information for team access
- **Hooks-based capture**: Uses Claude Code hooks for session-end capture
- **Compaction survival**: Memories stored externally — context is *injected* at session start, so compaction doesn't lose memory state (the memory service is the source of truth, not the context window)

### Estimated Token Impact
- `maxProfileItems: 5` default — roughly 500-2000 tokens injected at session start depending on memory content
- Reduces in-context memory burden by externalizing to API
- Claims 70% savings on token costs via intelligent context management vs full conversation replay

### Cost
- **Pro**: $19/month for 3M tokens
- **Scale**: $399/month for enterprise
- Free tier available for development
- Retrieval latency: sub-400ms

### Comparison with Mem0
| Aspect | Supermemory | Mem0 |
|--------|-------------|------|
| Speed | 10× faster than Zep, 25× faster than Mem0 | Slower but mature |
| Funding | $2.6M | $24M |
| Approach | Graph-based structured insights | Vector-based with graph option |
| Claude Plugin | Yes (official) | MCP integration available |
| Token savings | 70% claimed | 80% claimed (compressed snippets) |

### Implementation Path for OpenClaw
1. Install `@supermemory/claude-supermemory` plugin
2. Configure API key via `SUPERMEMORY_CC_API_KEY`
3. Set `maxProfileItems` to balance context usage
4. Configure signal keywords for selective capture
5. Use container tags for project/team separation

### Sources
- [Supermemory Docs](https://supermemory.ai/docs/introduction)
- [Claude Supermemory GitHub](https://github.com/supermemoryai/claude-supermemory)
- [Supermemory Blog: Claude Code Integration](https://supermemory.ai/blog/we-added-supermemory-to-claude-code-its-insanely-powerful-now/)
- [Memory vs RAG Concepts](https://supermemory.ai/docs/concepts/memory-vs-rag)

### Verdict: **RECOMMENDED**
Strong candidate for reducing context bloat. External memory survives compaction by design. Graph-based approach provides better context relevance than simple vector search. $19/month Pro tier is reasonable for individual developers. The hook-based auto-capture and context injection pattern is exactly what's needed to maintain memory without per-request overhead.

---

## 2. Mem0

### What It Is
Mem0 is a "Universal memory layer for AI" — positions as a portable "memory passport" for AI applications. Offers self-hosted and cloud-hosted options with broader framework integrations.

### How It Works
- **Memory types**: User memory (cross-conversation), Session memory (single conversation), Agent memory (per-agent instance)
- **Multi-store architecture**: Supports vector stores, graph stores, and hybrid
- **Auto-inference**: Extracts structured memories from message arrays
- **Scoped ownership**: user_id, agent_id, app_id, run_id, project_id, org_id for fine-grained control

### Token Impact
- Claims 90% lower token usage than full-context approaches
- +26% accuracy over OpenAI Memory on LOCOMO benchmark

### Cost
- Free tier: 10K memories
- Pro: Unlimited memories (price not clearly documented)
- Self-hosted option available (infrastructure costs only)

### Implementation Path for OpenClaw
- Integrate via MCP server or direct API
- Self-hosting possible with mem0ai/mem0 open source

### Sources
- [Mem0 GitHub](https://github.com/mem0ai/mem0)
- [Mem0 Documentation](https://docs.mem0.ai/)
- [Mem0 Pricing](https://mem0.ai/pricing)

### Verdict: **INVESTIGATE**
More mature with larger funding, but Supermemory claims significantly faster retrieval. Self-hosting option is attractive for cost control. Worth evaluating if self-hosted deployment is preferred over managed service.

---

## Topics Remaining
- Tool schema optimization (OpenClaw issue #6691)
- MemGPT/Letta tiered memory
- A-Mem paper analysis
- Anthropic context engineering framework
- Knowledge graphs & RAG approaches

## Prioritized Recommendations
*(To be completed after all topics researched)*
