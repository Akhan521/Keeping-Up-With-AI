# Keeping Up With AI: wiki schema

A personal knowledge base about AI concepts, tools and resources, built on Karpathy's LLM Wiki pattern (`raw/LLM-Wiki/karpathy-llm-wiki.md`). The user curates sources and asks questions. You maintain the wiki: summarizing, cross-referencing, filing, bookkeeping. This repo is also an Obsidian vault.

## Scope

Subjects the user is learning: AI first (concepts, tools, models, practices, ecosystem), plus software engineering. A topic gets a page, or a `(no page yet)` marker, when it is a subject in its own right. Tools a source mentions only in passing stay plain text. When unsure, ask during the Discuss step.

## Orient (every session)

1. Read this file.
2. Read recent activity: `grep "^## \[" wiki/log.md | tail -5`.

## Layout

```
raw/                     sources as captured; immutable after capture
  assets/                images
  <Folder>/              one per product family (Claude, Codex, Cursor) or broad topic (Agents, LLM-Wiki, Industry)
wiki/                    written and maintained by you
  index.md               catalog of tool, concept and synthesis pages, grouped by topic
  log.md                 append-only record of changes to the wiki
  tools/                 anything you can use: products, features, models, libraries
  concepts/              ideas that span tools
  sources/<Folder>/      one summary page per raw file; mirrors raw/<Folder>/
  syntheses/             answers the user asked to save
```

- **Raw folder choice**: the product family if the source is mainly about one; otherwise the closest broad topic. Create a folder only when a source needs it, and tell the user when you do.
- **Wiki folders** are fixed at the four above. Topics come from links, `topics:` frontmatter and `index.md`. Models go in `tools/`. Companies and people appear as `author:` metadata and in text, without their own pages.

## Naming

| Layer | Style | Example |
|---|---|---|
| raw files | kebab-case `author-short-title` | `raw/Claude/anthropic-agent-skills.md` |
| source pages | Title Case `Author - Title` | `wiki/sources/Claude/Anthropic - Agent Skills.md` |
| tool / concept / synthesis pages | Title Case | `Claude Code.md`, `MCP.md` |

The two styles keep raw and wiki filenames distinct, so every `[[wikilink]]` resolves to exactly one wiki page. On a same-folder raw collision, append `-2`.

## Pages

### Frontmatter

```yaml
type: tool | concept | source | synthesis
topics: [claude, agents]   # raw/ folder names, lowercased
updated: 2026-10-11        # last time the content changed
```

Additional fields:
- **tool**: `status: active | deprecated`, plus optional `docs:` and `repo:` URLs.
- **source**: `url`, `author`, `published`, `kind` (article | docs | video | podcast | paper | repo | thread | book | notes), `raw` (repo-relative path to the raw file).

`url:` always holds a fetchable link. When a source has no URL (a shared PDF, a book, private notes), omit `url:` and write a short plain-text citation in `origin:` instead, e.g. `origin: "AI Engineering (book), ch. 3"`.

### Templates

Tool and concept pages open with a skimmable header:

```markdown
# MCP
**TL;DR:** One sentence on what it is.
**Why it matters:** One sentence on its significance in the AI landscape.
**Builds on:** [[Tool Use]] · function calling *(no page yet; suggested source: OpenAI function-calling docs)*
```

Omit **Builds on** when there are no prerequisites. Required sections come first. Add optional sections only when there is real content for them, using these exact headings so they stay greppable:

| Type | Required | Optional |
|---|---|---|
| tool | `## What it does`, `## Getting started`, `## Related`, `## Sources` | `## Changelog`, `## Open questions` |
| concept | `## How it works`, `## Related`, `## Sources` | `## Across tools`, `## Open questions` |
| source | one-line description, `## Key takeaways`, `## Pages updated` | `## Details worth keeping` |
| synthesis | `## Question`, `## Answer`, `## Sources` | — |

- **Getting started**: install/setup, a minimal working example, key commands or config, gotchas, link to official docs. Written so an agent can implement from it.
- **Changelog**: notable changes only, dated bullets, newest first, each citing a source page.
- **Open questions**: what the wiki cannot yet answer about this page's subject.
- **Key takeaways**: scaled to the source, from 1–3 bullets for a post to ~10 for a paper.
- **Details worth keeping**: verbatim code and config snippets (with language tags), commands, API shapes.
- **Source pages** link their raw file with a markdown link: `[raw](../../../raw/Claude/anthropic-agent-skills.md)`.

### Linking and claims

- Every claim traces to a source page, inline or in `## Sources`. Every page is written from ingested sources.
- Use `[[wikilinks]]` for existing wiki pages only, on an entity's first mention on a page. Reference raw files by `raw:` path and markdown link.
- Write an in-scope topic that has no page as plain text followed by `*(no page yet)*`, adding `; suggested source: …` when you know a good one.
- Every page links to at least one other page and appears in `index.md` (source pages excepted).

### Keeping current

- Date anything an implementation depends on, plus numbers: versions, model IDs, CLI flags, API shapes, config formats, pricing, limits, benchmarks. Format: `(as of 2026-10, [[Source Page]])`.
- When a new source changes a fact, update the page and add a `## Changelog` entry. When sources genuinely disagree, keep both in a callout:
  ```markdown
  > [!warning] Conflict
  > [[Source A]] says X (2026-03); [[Source B]] says Y (2026-09).
  ```
- Set `status: deprecated` on retired tools and say what replaced them.

## index.md and log.md

**index.md**: one `## <topic>` heading per topic. Under each, one line per tool, concept and synthesis page with that topic; a page with several topics appears under each.
```markdown
## claude
- [[Claude Code]] (tool): Anthropic's agentic coding tool for terminal, IDE, desktop and web.
```

**log.md**: append-only, newest at the bottom. Record ingests, queries that changed the wiki, saved syntheses and lints.
```markdown
## [2026-10-11] ingest | Karpathy - LLM Wiki
Created: [[Karpathy - LLM Wiki]], [[LLM Wiki Pattern]]. Updated: none.
```
Entry types: `ingest`, `query`, `synthesis`, `lint`.

## Workflows

### Ingest

Triggers: "ingest <url | file | pasted text>" (discuss mode), "quick ingest …" (skip step 4), "ingest these: …" (batch: quick-ingest each in order).

1. **Check for duplicates.** Grep `url:` across `wiki/sources/`; for a source without a URL, look for an existing `Author - Title` source page. If the source is already ingested, tell the user and offer to re-ingest it as an updated version (a new raw file; the old one stays).
2. **Capture to raw.** Every input converges on the same state: in `raw/<Folder>/`, named in kebab-case, starting with a YAML header (`url` or `origin`, `author`, `published`).
   - URL: fetch the full text and save it. When the result is incomplete (paywall, X/Twitter, JS-rendered page, truncated output), stop and ask the user to clip it with Obsidian Web Clipper or paste it.
   - Clipped or dropped file: move it into `raw/<Folder>/` and rename it. Obsidian Web Clipper writes the link as `source:`; rename that field to `url:`.
   - Pasted text (markdown, transcript, thread): save it exactly as pasted.
   - Images go to `raw/assets/`. Binary files (PDFs) keep their content untouched; their metadata lives on the source page.
   - Fill header fields from what the source shows; ask the user for the rest, and leave a field blank rather than guess.
   - Source scope: a GitHub repo means its README plus key docs pages; a docs site means the specific pages the user chose.
   - From here on the raw file is immutable.
3. **Read the whole source.** Read long sources in parts.
4. **Discuss.** Give the user 3–5 key takeaways, the page plan (pages to create, pages to update and which sections) and any contradictions with existing pages. Incorporate their steering.
5. **Write.** The source page, then new and updated tool and concept pages, following *Pages*. Set `updated:` on every page you change.
6. **Backfill links.** For each newly created page, grep the wiki for plain-text mentions of its subject, including `(no page yet)` markers, and convert them to `[[wikilinks]]`.
7. **Update index.md and append to log.md** (one log entry per source).
8. **Report** the pages created and updated, then propose the commit (see *Commits*).

Done when: the raw file is in place, every page in the plan is written, every wikilink resolves, index and log are updated, and the commit is proposed.

### Query

1. Read `index.md`, then the relevant pages, then their sources as needed.
2. Answer in chat, citing pages with `[[wikilinks]]`. Label any part of the answer the wiki does not cover as general knowledge, and offer to find a source to ingest.
3. When the answer adds something the wiki lacks, offer to fold it into the page it belongs on. Folding in changes the wiki: log it as `query` and propose a commit.
4. When the user says "save this", write a `syntheses/` page, add it to `index.md`, log it as `synthesis` and propose a commit.

A query that changes no files leaves no trace.

### Lint

Trigger: "lint the wiki". When 10 or more `ingest` entries follow the last `lint` entry in `log.md`, suggest one at the end of an ingest.

1. Check every page for:
   - wikilinks that do not resolve
   - orphans, and tool/concept/synthesis pages missing from `index.md`
   - missing required frontmatter or sections
   - tool pages with `updated:` older than 90 days
   - "as of" dates older than 6 months
   - unresolved `[!warning] Conflict` callouts
   - subjects mentioned on 3+ pages without their own page
2. Fix mechanical issues directly: links, index entries, frontmatter.
3. Report content issues and ask before changing content.
4. Report **Suggested sources**: every `(no page yet)` topic and every `## Open questions` item, ranked by how many pages raise it.
5. Log it as `lint` and propose a commit.

## Commits

After any operation that changes files, propose the commit and wait for approval:

```
Proposed commit: ingest: Anthropic - Agent Skills
  + raw/Claude/anthropic-agent-skills.md
  + wiki/sources/Claude/Anthropic - Agent Skills.md
  ~ wiki/tools/Claude Code.md   (Changelog, Getting started)
  ~ wiki/index.md, wiki/log.md
Commit and push?
```

- On approval, commit and push. Without approval, leave the changes uncommitted.
- The message matches the log entry: `ingest: <Title>`, `query: <page>`, `synthesis: <Title>`, `lint: <summary>`.
- A batch ingest gets one proposal and one approval, committed as one commit per source.

## Using the wiki from another repo

Sessions working in other repositories use this wiki read-only:

1. Start from `wiki/index.md`, read the relevant pages, and follow `## Sources` for detail.
2. Before implementing anything marked "as of", check it against the page's `docs:` or the official documentation.
3. When something worth keeping comes up, suggest the user ingest it here; edits happen in sessions in this repo.

## Scaling

When `index.md` passes ~300 lines, propose adding [qmd](https://github.com/tobi/qmd) for search.
