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

## 6. Anthropic Context Engineering Framework

### What It Is
Context engineering is the discipline of curating and maintaining the optimal set of tokens during LLM inference. Anthropic views it as the natural evolution of prompt engineering—answering the question: "What configuration of context is most likely to generate our model's desired behavior?" The framework recognizes that context, not model intelligence, is the primary bottleneck in AI applications.

### The Four Core Strategies: Write, Select, Compress, Isolate

#### 1. **Write** — Externalizing Context
Saving information outside the context window so it's available without consuming tokens.

**Techniques:**
- **Scratchpads**: Agents save intermediate results to disk/state objects via tool calls
- **Memory files**: Persistent artifacts (like MEMORY.md, CLAUDE.md) that capture decisions across sessions
- **External storage**: File systems, databases, vector stores for tool outputs

**Implementation example**: Rather than keeping all tool outputs in message history, write them externally and keep only references or summaries in context.

#### 2. **Select** — Pulling Relevant Information
Choosing which context to include at each step via dynamic retrieval.

**Techniques:**
- **Memory selection**: Embeddings/knowledge graphs retrieve task-relevant facts
- **Tool selection**: RAG-based filtering of tool descriptions (achieves 3× accuracy improvement)
- **Just-in-Time (JIT) context**: Load what you need when you need it, using lightweight identifiers and progressive disclosure

**Key insight**: Balance between too little context (poor decisions) and too much (bloated windows, context distraction).

#### 3. **Compress** — Reducing Size While Retaining Value
Summarizing, abstracting, or reformatting information to be more concise.

**Techniques:**
- **Summarization**: Recursive/hierarchical summarization of agent trajectories
- **Agent boundaries**: Compressed summaries at handoffs between multi-agent systems
- **Trimming**: Hard-coded heuristics like removing older messages
- **Auto-compact**: Claude Code triggers at 95% context usage, generating continuity summaries

**Claude's compaction**: Server-side API that automatically summarizes when approaching token threshold (default 150K tokens, minimum 50K).

#### 4. **Isolate** — Splitting Context Across Components
Using distinct agents/components for different subtasks to prevent unbounded growth.

**Techniques:**
- **Multi-agent**: Specialized sub-agents with isolated context windows running in parallel
- **Sandboxing**: Tool outputs run in isolated environments (e.g., HuggingFace CodeAgent)
- **State schemas**: Selective field exposure to LLM while preserving other data

**Anthropic's approach**: Teams of cooperating agents, each applying different context engineering techniques, orchestrated by a lead agent.

### Token Impact & Benchmarks

| Metric | Value | Source |
|--------|-------|--------|
| Multi-agent overhead | Up to 15× more tokens than chat | LangChain/Anthropic |
| Deep research naive | 500K tokens ($1-2 per run) | Anthropic example |
| Tool selection RAG | 3× accuracy improvement | Research papers |
| Auto-compact trigger | 95% context (default) | Claude Code |
| Server-side trigger | 150K tokens (configurable) | Compaction API |
| Context awareness | Real-time budget tracking | Claude 4.5 models |

### Context Failure Modes
Without proper engineering, agents experience:
- **Context Poisoning**: Hallucinations persist in history
- **Context Distraction**: Overwhelming irrelevant information
- **Context Confusion**: Superfluous data influences responses
- **Context Clash**: Conflicting information degrades output

### Server-Side Compaction API (Claude Opus 4.6)

**Key features:**
- Automatic summarization at configurable threshold (default 150K tokens)
- `pause_after_compaction` for preserving recent messages
- Custom summarization instructions
- Prompt caching on compaction blocks
- Detailed usage tracking per iteration

**Implementation:**
```python
context_management={
    "edits": [{
        "type": "compact_20260112",
        "trigger": {"type": "input_tokens", "value": 100000},
        "pause_after_compaction": True
    }]
}
```

### Context Awareness (Claude 4.5)

Claude Sonnet 4.5 and Haiku 4.5 feature built-in context awareness:
- Model receives token budget at start: `<budget:token_budget>200000</budget:token_budget>`
- Updates after each tool call: `<system_warning>Token usage: 35000/200000; 165000 remaining</system_warning>`
- Enables better task persistence and context management

### Implementation Path for OpenClaw

1. **Write**: OpenClaw already has MEMORY.md and daily logs (✓ implemented)
2. **Select**: Hybrid BM25 + vector search exists (✓ implemented) — consider JIT context for skills
3. **Compress**: Pre-flush compaction exists (✓ implemented) — evaluate server-side Compaction API upgrade
4. **Isolate**: Sub-agents via Task tool available — consider more aggressive isolation for heavy operations

**Specific opportunities:**
- Use `pause_after_compaction` to preserve critical recent context
- Apply context awareness prompts for long-running sessions
- Adopt JIT context patterns for tool descriptions (aligns with MCP Tool Search)
- Implement explicit failure mode detection (context clash, poisoning warnings)

### Comparison with OpenClaw

| Strategy | Anthropic Framework | OpenClaw Current |
|----------|-------------------|------------------|
| Write | Scratchpads, memory files | MEMORY.md, daily logs ✓ |
| Select | JIT context, RAG tools | BM25 + vector, skills lazy-load ✓ |
| Compress | Server-side compaction | Pre-flush compaction ✓ |
| Isolate | Multi-agent orchestration | Task tool sub-agents ✓ |
| Awareness | Token budget tracking | Not implemented |

### Sources
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [LangChain: Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/)
- [Claude Docs: Context Windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Claude Docs: Compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [FlowHunt: Context Engineering for AI Agents](https://www.flowhunt.io/blog/context-engineering-for-ai-agents/)
- [Context Engineering 101: What We Can Learn from Anthropic](https://omnigeorgio.beehiiv.com/p/context-engineering-101-what-we-can-learn-from-anthropic)

### Verdict: **RECOMMENDED**

OpenClaw already implements the core Write/Select/Compress/Isolate strategies. The main opportunities are:

1. **Upgrade to server-side Compaction API** (when available for target model) — provides pause-after-compaction, custom instructions, and better token tracking
2. **Add context awareness prompts** — enables models to track remaining budget and persist on tasks
3. **Strengthen isolation** — more aggressive sub-agent delegation for tool-heavy operations

The framework validates OpenClaw's current architecture while pointing to incremental improvements. Priority should be context awareness integration, as it's low-effort and provides immediate value for long-running sessions.

---

## 7. Dynamic Relevance Scoring

### What It Is
Dynamic relevance scoring is the practice of computing context priority at runtime based on multiple signals—recency, semantic relevance, importance—rather than using static ordering or naive truncation. The goal is to ensure the most valuable context items occupy limited token budgets while less relevant content is deferred or discarded.

### How It Works

**Composite Scoring Formula (from Generative Agents):**
The foundational approach from Stanford's Generative Agents combines three signals:
```
score(memory) = α × recency + β × importance + γ × relevance
```

| Signal | Computation | Purpose |
|--------|-------------|---------|
| **Recency** | Exponential decay (e.g., 0.995^hours) | Prioritize recent events |
| **Importance** | LLM-rated 1-10 score | Distinguish mundane from significant |
| **Relevance** | Cosine similarity to current query | Match semantic context |

**Reranking Pipeline:**
Modern systems add a reranking stage after initial retrieval:
1. **Retrieve**: Broad candidate set via embeddings (top 50-100)
2. **Score**: Cross-encoder or LLM evaluates query-document pairs
3. **Reorder**: Prioritize highest-scoring items for context inclusion
4. **Trim**: Fit within token budget by cutting lowest scores

**Token-Budget-Aware Reasoning (TALE):**
Recent research shows token budgets can guide reasoning itself:
- Estimate query complexity upfront
- Allocate proportional token budget
- Result: 68.9% token reduction with <5% accuracy loss

### Key Techniques

#### 1. Cross-Encoder Reranking
Cross-encoders process query+document jointly (rather than embedding separately), capturing subtle semantic relationships.

**Leading Models:**
| Model | Strengths | Trade-offs |
|-------|-----------|------------|
| **Cohere Rerank 4** | 32K context, self-learning, 100+ languages | API cost, vendor lock-in |
| **BGE Reranker** | Open-source, self-hostable | Requires infrastructure |
| **ColBERT** | Fast token-level matching | Less precise than cross-encoders |
| **RankZephyr** | LLM-based zero-shot (7B params) | Higher latency |
| **Jina Reranker v2** | Multilingual, code search | API-dependent |

**Impact**: Reranking typically improves RAG accuracy by 20-35% with 200-500ms additional latency.

#### 2. Just-In-Time (JIT) Context Selection
Rather than loading all context upfront, dynamically select based on current needs:
- Simple queries → more retrieved documents, less history
- Complex queries → more conversation context, less documents
- Reformulate queries as understanding evolves

#### 3. Hybrid Retrieval + Reranking
Elasticsearch's research shows optimal configuration:
- Combine sparse (ELSER) + dense vector search
- 84.3% recall@10, 0.53 MRR
- Retrieve 5 semantic chunks from top 5 documents
- Result: 93.3% task completion with 40% context size reduction

#### 4. ACAN (Auxiliary Cross Attention Network)
Novel approach using LLM-trained attention for memory retrieval:
- Agent state becomes query vector
- Memories are key-value pairs
- Attention weights determine retrieval priority
- LLM provides training signal for optimization

### Estimated Token Impact
| Technique | Token Savings | Accuracy Trade-off |
|-----------|---------------|-------------------|
| TALE budget-aware reasoning | 68.9% | <5% loss |
| Semantic chunking + proximity | 40% context reduction | 93.3% task completion |
| Reranking (top 20→5) | ~75% of retrieved context | +20-35% accuracy |
| JIT context vs full load | Variable (50-80%) | Depends on query type |

### Implementation Path for OpenClaw

1. **Immediate: Enhance memory retrieval scoring**
   - Add importance scores when saving to MEMORY.md (LLM-rated 1-10)
   - Implement exponential decay for daily log entries
   - Combine with existing BM25 + vector similarity

2. **Short-term: Add reranking layer**
   - Integrate BGE Reranker (open-source) or Cohere Rerank API
   - Apply to memory search results before context injection
   - Target: rerank top-20 results down to top-5

3. **Medium-term: JIT context patterns**
   - Classify query complexity before context allocation
   - Simple queries: minimal history, more tool results
   - Complex queries: fuller history, condensed tool output

4. **Investigate: Token-budget prompting**
   - Experiment with TALE-style budget hints in system prompts
   - "Respond concisely within ~500 tokens" for simple tasks

### Comparison with OpenClaw

| Feature | Current OpenClaw | With Dynamic Scoring |
|---------|-----------------|---------------------|
| Memory retrieval | BM25 + vector | + importance + recency decay |
| Reranking | None | Cross-encoder rerank layer |
| Context allocation | Fixed | Query-complexity aware |
| Budget awareness | None | Token budget hints |

### Sources
- [Elastic: Context Engineering Relevance](https://www.elastic.co/search-labs/blog/context-engineering-relevance-ai-agents-elasticsearch)
- [Token-Budget-Aware LLM Reasoning (TALE)](https://arxiv.org/abs/2412.18547)
- [Maxim: Context Window Management Strategies](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/)
- [Frontiers: Cross Attention Networks for Memory Retrieval](https://www.frontiersin.org/journals/psychology/articles/10.3389/fpsyg.2025.1591618/full)
- [ACM: Memory Mechanisms in LLM Agents Survey](https://dl.acm.org/doi/10.1145/3748302)
- [n8n: Implementing Rerankers in AI Workflows](https://blog.n8n.io/implementing-rerankers-in-your-ai-workflows/)
- [NVIDIA: Introduction to LLM Agents](https://developer.nvidia.com/blog/introduction-to-llm-agents/)

### Verdict: **RECOMMENDED**

Dynamic relevance scoring directly addresses context bloat by ensuring only the most valuable tokens are included. Key recommendations:

1. **Adopt composite scoring** (recency + importance + relevance) for memory retrieval—this is a well-validated pattern from Generative Agents research
2. **Add reranking layer** to existing search—20-35% accuracy improvement with minimal latency cost
3. **Implement JIT context selection**—query-aware allocation prevents over-fetching

OpenClaw's hybrid BM25 + vector search provides a strong foundation. Adding importance scoring, recency decay, and a reranking layer would significantly improve context quality without major architectural changes.

---

## Topics Remaining
- Multi-agent context isolation patterns
- Knowledge graphs & RAG approaches (Graphiti)

## Prioritized Recommendations
*(To be completed after all topics researched)*
