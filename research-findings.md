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

## 4. MemGPT/Letta Tiered Memory

### What It Is
MemGPT (Memory-GPT) is an OS-inspired approach to LLM context management that treats the context window as "RAM" and external databases as "disk storage." Originally a research project from UC Berkeley, it's now part of the Letta agent framework. The key insight: LLMs can manage their own memory through tool calls, moving data in and out of context as needed.

### How It Works

**Memory Hierarchy:**
```
┌─────────────────────────────────────────────┐
│          MAIN CONTEXT (In-Context)          │
├─────────────────────────────────────────────┤
│  System Instructions    │  ~1,076 tokens    │
│  Core Memory Blocks     │  ~86 tokens       │
│  Function Definitions   │  ~633 tokens      │
│  FIFO Message Queue     │  Variable         │
│  External Memory Summary│  ~107 tokens      │
├─────────────────────────────────────────────┤
│  TOTAL BASELINE:        │  ~2,000 tokens    │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│       EXTERNAL CONTEXT (Out-of-Context)     │
├─────────────────────────────────────────────┤
│  Recall Memory    │  Searchable conversation│
│                   │  history (vector DB)    │
│  Archival Memory  │  Long-term facts,       │
│                   │  documents (ChromaDB)   │
└─────────────────────────────────────────────┘
```

**Memory Tiers:**
1. **Core Memory (always in-context)**: Structured blocks with character limits, always accessible, self-edited by the LLM
2. **Recall Memory**: Complete conversation history, searchable via semantic search
3. **Archival Memory**: Long-term storage for facts and documents, requires explicit retrieval

**Paging Mechanism:**
- LLM receives memory-editing tools: `memory_replace`, `memory_insert`, `memory_rethink`
- **Eviction**: When context hits 70% (warning) or 100% (flush), messages are summarized and moved to recall
- **Retrieval**: LLM uses search tools to pull relevant memories back into context
- **Heartbeat**: LLM can request continued execution via `request_heartbeat=true` for multi-step reasoning

### Token Usage (GPT-4o-mini, 32K context)
| Component | Tokens | % of Context |
|-----------|--------|--------------|
| System instructions | 1,076 | 3.4% |
| Core memory | 86 | 0.3% |
| Function definitions | 633 | 2.0% |
| Messages/conversation | 179 | 0.6% |
| External memory summary | 107 | 0.3% |
| **Initial overhead** | **~2,081** | **6.5%** |

**Context Management Thresholds:**
- Warning at 70%: System prompts LLM to save important data
- Flush at 100%: Evict ~50% of messages, generate recursive summary

### Letta V1 Architecture (2025)
The latest Letta release moves away from tool-based control:
- Removes `thinking` and `request_heartbeat` tool parameters
- Uses native model reasoning capabilities (e.g., OpenAI Responses API)
- Model-agnostic (no longer requires tool-calling capability)
- Simpler prompting, relies on models trained for agentic patterns

### Benchmark Performance
- **Letta Filesystem approach**: 74.0% on LoCoMo benchmark (GPT-4o-mini)
- **Beats Mem0 graph variant**: 68.5% on same benchmark
- **Terminal-Bench**: Letta ranked #4 overall, #1 for open source
- Key insight: "Agent capabilities matter more than the tools" — filesystem operations outperformed specialized memory APIs

### Estimated Token Impact for OpenClaw
- **Base overhead**: ~2K tokens for MemGPT-style system
- **Trade-off**: Multi-step reasoning requires GPT calls for memory operations
- **Concern**: "Memory is actively managed, more context used by internal actions like updating or retrieving from memory"
- **Potential savings**: Unbounded context via virtualization, but with API call overhead

### Implementation Path for OpenClaw
1. **Study Letta patterns**: Core memory blocks + recursive summarization already align with OpenClaw's daily log + compaction approach
2. **Consider selective adoption**:
   - Memory blocks → Similar to OpenClaw's MEMORY.md structured sections
   - Recursive summarization → Could enhance existing compaction
   - Self-editing memory → Already in OpenClaw via /remember
3. **Evaluate overhead**: MemGPT's ~2K token baseline + function calls may not improve on OpenClaw's current approach
4. **Benchmark against OpenClaw**: Compare LoCoMo scores with current OpenClaw memory system

### Comparison with OpenClaw
| Feature | MemGPT/Letta | OpenClaw Current |
|---------|--------------|------------------|
| In-context memory | Core memory blocks | MEMORY.md + daily logs |
| Out-of-context | Archival + Recall DBs | Hybrid BM25 + vector |
| Compaction | Recursive summarization | Pre-flush compaction |
| Self-editing | Tool-based memory edits | /remember skill |
| Token overhead | ~2K baseline | Similar (bootstrap files) |

### Sources
- [MemGPT Research Paper](https://arxiv.org/abs/2310.08560)
- [Letta Documentation](https://docs.letta.com/concepts/memgpt/)
- [Letta GitHub](https://github.com/letta-ai/letta)
- [Letta V1 Agent Architecture](https://www.letta.com/blog/letta-v1-agent)
- [Agent Memory Blog](https://www.letta.com/blog/agent-memory)
- [Memory Benchmarking](https://www.letta.com/blog/benchmarking-ai-agent-memory)
- [Virtual Context Management](https://www.leoniemonigatti.com/blog/memgpt.html)

### Verdict: **INVESTIGATE**
MemGPT's OS-inspired approach is intellectually elegant but may not offer significant advantages over OpenClaw's current architecture. Key findings:
- OpenClaw already has similar patterns: structured in-context memory, external search, compaction
- Letta's benchmark shows filesystem-based approaches beat specialized memory APIs
- The ~2K token overhead is comparable to what OpenClaw already uses
- Multi-step memory operations add API call latency

**Recommendation**: Study Letta's recursive summarization algorithm for potential compaction improvements. Don't adopt full MemGPT framework — OpenClaw's simpler file-based approach aligns with Letta's own finding that "agent capabilities matter more than the tools."

---

## 5. A-MEM (Agentic Memory)

### What It Is
A-MEM is a 2025 NeurIPS-accepted paper introducing a Zettelkasten-inspired memory system for LLM agents. Unlike sequential memory storage, it creates interconnected knowledge networks through dynamic indexing and linking.

### How It Works
**Zettelkasten Principles:**
- Each memory becomes a "note" with structured attributes
- Notes are interconnected via semantic links
- System discovers relationships between new and historical memories
- Memory network evolves as new information arrives

**Memory Structure:**
```
Memory Note {
  content: string       // Primary information
  context: string       // Generated semantic description
  keywords: string[]    // LLM-extracted terms
  tags: string[]        // Category labels
  timestamp: string     // YYYYMMDDHHmm format
  connections: Note[]   // Bidirectional links to related memories
}
```

**Core Operations:**
1. **Note Generation**: When adding memory, generate structured attributes via LLM
2. **Link Discovery**: Analyze historical memories for semantic connections
3. **Memory Evolution**: New additions trigger updates to related memory metadata
4. **Retrieval**: ChromaDB vector search + graph traversal for connected context

### Benchmark Performance
- Accepted to NeurIPS 2025
- "Superior improvement against existing SOTA baselines" on 6 foundation models
- Evaluated on BLEU, ROUGE, METEOR, and semantic similarity metrics
- Outperforms MemGPT, LOCOMO, ReadAgent, MemoryBank on dialogue tasks

### Token Efficiency
The paper emphasizes efficiency through:
- ChromaDB vector indexing reduces full memory scans
- Selective retrieval via keywords/tags
- Connected context provides relevant information without loading everything

*Note: Specific token counts (e.g., the "4,600 tokens vs MemGPT's 16,900" from the prompt) were not confirmed in available sources — this may be from a different paper or earlier version.*

### Implementation Path for OpenClaw
1. **Structured memory notes**: Add metadata (keywords, context) to memory entries
2. **Memory linking**: Track which memories reference each other
3. **Evolution triggers**: When saving new memory, update related memories' context
4. **Hybrid retrieval**: Combine vector search with link traversal

### Comparison with OpenClaw
| Feature | A-MEM | OpenClaw Current |
|---------|-------|------------------|
| Storage | ChromaDB + links | BM25 + vector |
| Structure | Zettelkasten notes | Daily logs + MEMORY.md |
| Connections | Bidirectional links | None explicit |
| Evolution | Auto-updates related memories | Manual |

### Sources
- [A-MEM Paper (arXiv)](https://arxiv.org/abs/2502.12110)
- [A-MEM GitHub](https://github.com/agiresearch/A-mem)
- [Semantic Scholar](https://www.semanticscholar.org/paper/A-MEM:-Agentic-Memory-for-LLM-Agents-Xu-Liang/1f35a15fe9df43d24ec6ea551ec6c9766c17eccf)

### Verdict: **INVESTIGATE**
A-MEM's Zettelkasten approach is promising for reducing retrieval noise by surfacing connected context. However:
- Requires LLM calls for note generation and link discovery (overhead)
- ChromaDB dependency adds infrastructure
- Benchmark improvements are on dialogue tasks, not coding agents

**Recommendation**: Consider adding lightweight memory linking to OpenClaw's existing system. Track explicit "related to" connections between daily log entries and MEMORY.md sections without full Zettelkasten overhead.

---

## Topics Remaining
- Anthropic context engineering framework
- Knowledge graphs & RAG approaches

## Prioritized Recommendations
*(To be completed after all topics researched)*
