---
type: source
topics: [llm-wiki]
updated: 2026-10-11
url: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
author: Andrej Karpathy
published: 2026-04-04
kind: article
raw: raw/LLM-Wiki/karpathy-llm-wiki.md
---
# Karpathy - LLM Wiki

Andrej Karpathy's "idea file" describing the [[LLM Wiki Pattern]]: a persistent, LLM-maintained markdown wiki as an alternative to query-time retrieval. [raw](../../../raw/LLM-Wiki/karpathy-llm-wiki.md)

## Key takeaways

- RAG-style tools (NotebookLM, ChatGPT file uploads, most RAG systems) rediscover knowledge from scratch on every question; nothing accumulates.
- Instead, the LLM incrementally builds and maintains a wiki: each new source updates entity pages, revises summaries and flags contradictions. Knowledge is "compiled once and then *kept current*."
- The human curates sources, directs analysis and asks questions; the LLM does all the writing and bookkeeping. "Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."
- Three layers: immutable raw sources, the LLM-owned wiki, and a schema file (CLAUDE.md or AGENTS.md) that human and LLM co-evolve.
- Three operations: ingest (a single source may touch 10–15 pages), query (good answers are filed back as pages), lint (contradictions, stale claims, orphans, missing pages and cross-references, data gaps).
- `index.md` (a content catalog, read first when querying) and `log.md` (an append-only timeline) replace embedding-based RAG at moderate scale: about 100 sources and hundreds of pages.
- Past that scale, add search. qmd *(no page yet; suggested source: the qmd README)* is suggested: local hybrid BM25/vector search with LLM re-ranking, usable as a CLI or an MCP server.
- Why it works: people abandon wikis because maintenance grows faster than value, and LLMs make maintenance nearly free. The idea echoes Vannevar Bush's Memex (1945), which never solved who does the maintenance.
- The document is deliberately abstract: directory structure, conventions and tooling are meant to be worked out with your own agent.

## Details worth keeping

Log entries with a consistent prefix stay parseable with unix tools:

```markdown
## [2026-04-02] ingest | Article Title
```

```bash
grep "^## \[" log.md | tail -5   # last 5 entries
```

Obsidian setup tips from the source:
- The Obsidian Web Clipper browser extension converts web articles to markdown for the raw collection.
- Local images: Settings → Files and links → "Attachment folder path" set to a fixed folder (e.g. `raw/assets/`); then Settings → Hotkeys → "Download attachments for current file", bound to e.g. Ctrl+Shift+D. LLMs can't read markdown with inline images in one pass: read the text first, then view the images separately.
- Graph view shows hubs and orphans; Dataview queries YAML frontmatter; Marp renders markdown slide decks.

## Pages updated

- [[LLM Wiki Pattern]] (created)
