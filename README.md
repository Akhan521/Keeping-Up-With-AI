# Keeping-Up-With-AI

A personal knowledge base for keeping up with AI concepts, tools and resources, plus the software engineering around them. It follows Andrej Karpathy's [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): I collect sources, and Claude Code turns them into an interlinked wiki that it keeps up to date. I read it in Obsidian.

## Layout

| Path | What it holds | Who writes it |
|---|---|---|
| `raw/` | Sources as captured: articles, docs, transcripts, papers, grouped by product family or topic | Me (Claude files them) |
| `wiki/` | Tool, concept, source-summary and synthesis pages, plus `index.md` and `log.md` | Claude |
| `CLAUDE.md` | The schema: structure, conventions and workflows Claude follows | Both of us |

`CLAUDE.md` is the single source of truth for how the wiki works.

## Using it

Open the repo in Claude Code and say:

- **`ingest <url>`**, or point Claude at a clipped/dropped file, or paste text. Claude captures it to `raw/`, discusses the key takeaways, then updates the wiki.
- **`quick ingest <url>`** skips the discussion. **`ingest these: …`** handles a batch.
- **Ask anything.** Answers cite wiki pages, and anything the wiki doesn't cover is labeled. Say **`save this`** to keep an answer as a page.
- **`lint the wiki`** health-checks links, staleness and gaps, and suggests what to read next.

Claude proposes every commit and waits for my approval before committing and pushing.

**Obsidian setup:** open the repo folder as a vault, then set Settings → Files and links → Attachment folder path to `raw/assets`. Obsidian's own config (`.obsidian/`) stays out of git.

## Design notes

- **Sources by topic, wiki by kind of page.** Each source has one main subject, so `raw/` is grouped by topic. Wiki pages connect across tools, so they're grouped by kind (tools, concepts, sources, syntheses), and topics come from links and tags.
- **Raw files change only when a newer version is re-captured,** and git keeps every earlier version. The wiki can always be traced back to what was actually read.
- **Kebab-case raw files, Title Case wiki pages.** The names can't collide, so every `[[wikilink]]` points to exactly one page.
- **Every claim cites a source; no pages from general knowledge.** Topics without a page yet appear as plain text with `(no page yet)`, which doubles as a reading list.
- **Facts that go stale carry "as of" dates.** Versions, APIs, pricing and limits change fast; dated facts tell a reader, or a future Claude session implementing something, what to re-check.
- **Answers stay in chat by default.** New insights get folded into existing pages, so the wiki gets richer instead of just bigger.
- **The log records changes only.** It's a readable timeline of what was learned, and commit messages mirror it.
