---
type: concept
topics: [llm-wiki]
updated: 2026-10-11
---
# LLM Wiki Pattern

**TL;DR:** An LLM incrementally builds and maintains a persistent, interlinked markdown wiki from your sources, so knowledge is compiled once and kept current instead of re-derived on every query.
**Why it matters:** It turns the LLM from a per-question retriever into the maintainer of a knowledge base that compounds with every source and question, removing the bookkeeping that makes people abandon wikis.

## How it works

**Contrast with retrieval.** RAG *(no page yet; suggested source: Lewis et al. 2020, "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks")* retrieves relevant chunks of raw documents at query time and synthesizes an answer, so nothing accumulates between questions. In this pattern, cross-references, contradictions and synthesis are worked out at ingest time and stored in the wiki.

**Three layers**
- **Raw sources:** a curated, immutable collection (articles, papers, images, data). The LLM reads them and never modifies them; they are the source of truth.
- **The wiki:** LLM-generated markdown pages: summaries, entity and concept pages, comparisons, an overview, a synthesis. The LLM owns this layer; the human reads it.
- **The schema:** a file such as CLAUDE.md or AGENTS.md that defines structure, conventions and workflows, and is co-evolved by human and LLM. It makes the LLM a disciplined maintainer rather than a generic chatbot.

**Three operations**
- **Ingest:** read a new source, discuss takeaways, write a summary page, update related pages across the wiki, update the index and log. One source may touch 10–15 pages. Sources can be ingested one at a time with supervision or batched.
- **Query:** read the index, drill into relevant pages, answer with citations. Valuable answers (comparisons, analyses, discovered connections) are filed back as pages, so explorations compound like sources do.
- **Lint:** a periodic health check for contradictions, stale claims, orphan pages, concepts without their own page, missing cross-references, and gaps a web search could fill.

**Navigation**
- **index.md:** a content catalog, one line per page, organized by category and read first when querying. Enough at about 100 sources and hundreds of pages, with no embedding infrastructure.
- **log.md:** an append-only, chronological record with a consistent, greppable entry prefix.
- At larger scale, a search engine over the pages, e.g. qmd *(no page yet)*.

**Roles.** The human curates sources, directs the analysis, asks questions and thinks about what it means. The LLM does the summarizing, cross-referencing, filing and bookkeeping. Karpathy keeps the agent open on one side and Obsidian on the other, browsing results as they change.

## Open questions

- How do other people's implementations of this pattern differ in structure, schema and tooling, and which choices hold up over time?
- Where exactly does the index-only approach break down, and what does migrating to search look like in practice?

## Related

- This repository implements the pattern; its rules are in the [schema](../../CLAUDE.md).

## Sources

- [[Karpathy - LLM Wiki]]
