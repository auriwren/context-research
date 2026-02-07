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

## 3. Tool Schema Optimization

### What It Is
Tool schemas consume significant tokens in every API request — the full JSON schema for each tool is sent with every message. With 11+ skills and MCP servers, this can easily reach 15K-70K+ tokens before any conversation starts. Tool schema optimization refers to techniques to defer, compress, or selectively load tool definitions.

### The Problem
- GitHub MCP server alone: ~46K tokens
- 5-server setup with 58 tools: ~55K tokens
- Heavy MCP users: 50-100K+ tokens of overhead
- This consumes 25-50% of Claude's 200K context window before work begins

### Key Solutions

#### 1. MCP Tool Search (defer_loading)
**How it works:**
- Tools marked with `defer_loading: true` aren't loaded into context initially
- Claude receives a lightweight Tool Search Tool instead of full definitions
- When Claude needs a tool, it searches by keyword (regex or BM25)
- Only 3-5 relevant tools loaded on-demand (~3K tokens)
- Auto-activates when tool definitions exceed 10K tokens

**Token impact:**
- Before: ~77K tokens with 50+ tools
- After: ~8.7K tokens (85% reduction)
- Preserves 95% of context window

**Accuracy improvement:**
- Opus 4: 49% → 74%
- Opus 4.5: 79.5% → 88.1%

#### 2. Token-Efficient Tool Use (API Beta)
**How it works:**
- Uses beta header `token-efficient-tools-2025-02-19` (Claude 3.7)
- Built into Claude 4+ models by default
- Reduces tool call **output** tokens, not schema tokens

**Token impact:**
- Average: 14% reduction in output tokens
- Up to: 70% reduction in some cases

#### 3. Experimental MCP CLI Flag
**How it works:**
- Environment variable: `ENABLE_EXPERIMENTAL_MCP_CLI=true`
- Wraps MCP tools behind CLI commands instead of native tool calls
- Progressive schema discovery via Bash

**Token impact:**
- ~32K tokens recovered with 2 MCP servers
- Heavy users: 50-70K tokens saved

**Trade-offs:**
- Extra latency (schema fetch per tool)
- Bash escaping issues with complex JSON
- Tools invisible in UI until invoked
- Experimental/unsupported

#### 4. Programmatic Tool Calling
**How it works:**
- Tool orchestration moves to code execution environment
- Intermediate results don't pollute model context
- Only final summaries returned to context

**Token impact:**
- 37% reduction on complex research tasks
- Average usage: 43,588 → 27,297 tokens

### Comparison with OpenClaw's Current Approach
| Feature | OpenClaw Current | Tool Search | MCP CLI Flag |
|---------|-----------------|-------------|--------------|
| Skills | Lazy-loaded (metadata only) | N/A | N/A |
| MCP Tools | Full schemas upfront | Deferred (85% reduction) | CLI wrapper (100% reduction) |
| Tool Results | Trimmed | Unchanged | Via Bash |
| Maturity | Production | Production (default) | Experimental |

### Implementation Path for OpenClaw
1. **Immediate**: Verify MCP Tool Search is active (should be default when >10K tool tokens)
2. **Short-term**: Apply `defer_loading: true` to rarely-used MCP tools
3. **Investigate**: Token-efficient tool use for Claude 4+ (may already be default)
4. **Consider**: Programmatic tool calling for heavy operations (move orchestration to code)
5. **Monitor**: MCP CLI flag for potential future integration when stable

### Sources
- [Token-Efficient Tool Use - Claude Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/token-efficient-tool-use)
- [MCP Tool Search Guide - Cyrus](https://www.atcyrus.com/stories/mcp-tool-search-claude-code-context-pollution-guide)
- [Claude Code Hidden MCP Flag - Paddo](https://paddo.dev/blog/claude-code-hidden-mcp-flag/)
- [Advanced Tool Use - Anthropic Engineering](https://www.anthropic.com/engineering/advanced-tool-use)
- [Token Saving Updates - Anthropic](https://claude.com/blog/token-saving-updates)
- [GitHub Issue #7336: Lazy Loading for MCP](https://github.com/anthropics/claude-code/issues/7336)
- [GitHub Issue #11364: Lazy-load MCP tool definitions](https://github.com/anthropics/claude-code/issues/11364)

### Verdict: **RECOMMENDED**
MCP Tool Search is already production-ready and enabled by default. OpenClaw should ensure it's active and properly configured. The 85% reduction in tool schema tokens is substantial. For custom tools/skills, apply similar lazy-loading patterns. The experimental MCP CLI flag shows the direction Anthropic is heading but isn't ready for production use.

---

## Topics Remaining
- MemGPT/Letta tiered memory
- A-Mem paper analysis
- Anthropic context engineering framework
- Knowledge graphs & RAG approaches

## Prioritized Recommendations
*(To be completed after all topics researched)*
