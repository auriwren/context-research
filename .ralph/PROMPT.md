# Research: Reducing Auri's Context Bloat

## Goal
Research memory architectures and context management techniques that can be added to OpenClaw to reduce per-request token overhead without losing functionality. Write findings to `research-findings.md`. Update `research-progress.md` each iteration with status and next steps.

## What's Already in OpenClaw (Do NOT research these)
- File-based memory (MEMORY.md, daily logs)
- Compaction with pre-flush
- Skills lazy-loading (metadata only, SKILL.md on demand)
- Hybrid BM25 + vector memory search
- Tool result trimming, bootstrap file trimming

## Research Topics (in priority order)
1. **Supermemory** (https://supermemory.ai/docs/introduction) -- External memory API with graph-based storage, dual-layer timestamping, auto-recall/auto-capture. Has an OpenClaw plugin (`@supermemory/openclaw-supermemory`). How does it survive compaction? What does it cost? How does it compare to Mem0's plugin?
2. **Tool schema optimization** -- Tool schemas (~15K tokens for 11+ skills) are sent in full every request, unlike skill instructions which are lazy-loaded. What solutions exist? Check OpenClaw issue #6691. What do toolPolicy allowlists actually save?
3. **MemGPT/Letta tiered memory** -- LLM-driven paging between active context and archival storage. Also check A-Mem (https://arxiv.org/pdf/2502.12110) which achieved similar results in ~4,600 tokens vs MemGPT's 16,900.
4. **Context prioritization** -- Anthropic's context engineering framework (Write/Select/Compress/Isolate). Dynamic relevance scoring. Multi-agent context isolation for heavy operations.
5. **Emerging approaches** -- Knowledge graphs (Graphiti), RAG-based context injection, prompt caching strategies for larger effective contexts.

## Output Format
For each topic in `research-findings.md`, include: what it is, how it works, estimated token impact, implementation path for OpenClaw, sources with URLs, and a verdict (RECOMMENDED / INVESTIGATE / SKIP). End the file with a prioritized recommendation list.

## Completion
Write `STATUS: COMPLETE` in `research-progress.md` when all 5 topics have findings with verdicts, or when returns are diminishing.
