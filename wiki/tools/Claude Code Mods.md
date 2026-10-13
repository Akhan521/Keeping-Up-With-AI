---
type: tool
topics: [claude]
updated: 2026-10-12
status: active
docs: https://code.claude.com/docs/en/plugins/mods/overview
repo: https://github.com/anthropics/claude-code/tree/main/mods
---
# Claude Code Mods
**TL;DR:** Plugins whose JavaScript/TypeScript module runs inside the Claude Code process, hooking its events to rewrite prompts, guard tool calls, draw UI or replace built-in features.
**Why it matters:** It is Claude Code's most powerful and most privileged extension point, and Anthropic plans to rebuild more of Claude Code itself as mods so users can pare it down to a small core.
**Builds on:** Claude Code *(no page yet; suggested source: Claude Code overview docs)* · Claude Code plugins *(no page yet; suggested source: Claude Code plugins docs)* · Claude Code hooks *(no page yet; suggested source: Claude Code hooks docs)*

## What it does

Claude Code emits events for nearly everything it does (prompt submitted, tool called, permission checked, UI drawn). A mod registers functions on those events that run before, after, instead of, or around Claude Code's own handling. It can ([[Claude Code Mods Field Notes]]):

| Verb | Touches | Examples |
|---|---|---|
| **Rewrite** | Prompts, system prompt | Change a prompt before the model sees it, inject context, add or drop system-prompt sections |
| **Guard** | Tool calls, permissions | Block, rewrite or retry a tool call; approve or deny a permission request; redact secrets from output |
| **Draw** | Panes, bands, status line, toasts | UI beside the transcript or above the prompt; restyle built-in rows, the spinner, the question dialog |
| **Replace** | Built-in features | `/diff`, AGENTS.md loading and first-party telemetry already ship as mods |

**Mods vs other extension points.** Settings hooks, skills, status-line scripts and MCP *(no page yet)* servers work from outside the process; a mod runs inside it, which is why only a mod can draw UI, hold state shared across hooks, and rewrite events in place. Choose the least powerful option that works ([[Claude Code Mods Field Notes]]):

| You want to... | Use |
|---|---|
| Draw a pane or band, add an instant `/command`, rewrite an event | Mod |
| Block, allow or log a tool call with an existing script | Settings hook |
| Stop pasting the same instructions | Skill (Agent Skills *(no page yet)*) |
| Give Claude access to an external system | MCP |

**The hook chain.** Every hook is middleware with the signature `($, e, next)`: `$` is the engine API (noun then method: `$.ui.toast`, `$.fs.read`), `e` is the deeply frozen event, and `next(e)` runs everything beneath and resolves to the result. Three patterns: observe (`return next(e)`), rewrite (`return next({ ...e, x })`), answer (return `{ deny }` etc. without calling `next`). Events flow down tiers and results flow back up: org-prepend (where `sec-default` lives) → user mods in load order → org-append → built-in mods → engine default. There are no numeric priorities; only mods listed in `prependPlugins` / `appendPlugins` can call `next.to` to skip tiers (as of 2026-10, [[Claude Code Mods Field Notes]]).

**Events and API** (as of 2026-10, v2.1.290 reference, [[Claude Code Mods Field Notes]]):
- Event families: tools (`tool.call`, `tool.check`, `tool.describe`), prompts (`prompt.submit`, `prompt.fill`, `prompt.suggest`, `prompt.edit`, `prompt.compose`, `prompt.mention`), commands (`command.run`), turns and sessions (`turn.step`, `turn.complete`, `session.start`, `session.end`, `subagent`), UI (`ui.render`, `ui.press`, `ui.input`, `ui.select`, `ui.focus`, `ui.scroll`, `ui.close`, `ui.message`, `ui.fault`), engine (`engine.create`, `process.spawn`, settings-hook events).
- Result shapes: `{deny}`, `{result}`, `{decision}`, `{drop}`, `{sections}`.
- `$` namespaces: `$.ui` (`resolve`, `open`, `status`, `toast`, `log`), `$.state` (per session, survives hot reload), `$.store` (machine-wide, 4 MiB JSON), `$.command` / `$.tool` / `$.agent` (register slash commands, model-callable tools, subagent types), `$.fs` / `$.process` / `$.http` (with the user's permissions), `$.model`, `$.clock` (`now`, `every`, `after`), `$.audio`.
- Every API method is also an event (`fs.read`, `model.complete`, `ui.open`), so a later mod can intercept another mod's calls.

**Where UI shows up.** A `ui.render` hook returns JSX built from `$.ui.resolve(e)` (`Box`, `Text`, `Button`); the module has no DOM and no Node. Panes open beside the transcript, the `AbovePrompt` band sits above the prompt, `$.ui.status` writes the status line, `$.ui.toast` pops a notification. A pane opened without a user action waits until the terminal is at least 144 columns wide (as of 2026-10, [[Claude Code Mods Field Notes]]).

| Surface | Hooks | UI |
|---|---|---|
| Terminal CLI (v2.1.287+) | yes | yes |
| Desktop app, Code tab (v2.1.286+) | yes | yes (add the mod folder in settings; no flags) |
| VS Code extension | yes | no |
| `claude -p`, cloud sessions | yes | no; mods should degrade gracefully |
| Remote Control / iOS | partial | gaps |

(as of 2026-10, [[Claude Code Mods Field Notes]])

**Security model.** Each plugin runs in a Bun Worker + `node:vm` and cannot `import fs`, call `fetch()` or touch the DOM, but the `$` API grants the user's full permissions: `$.fs`, `$.process` and `$.http` work, and per the admin docs `$.fs` / `$.process` bypass deny rules. The docs: "Mods aren't sandboxed... A mod that approves tool calls can approve one that an ask rule would prompt for, or that one of your own PreToolUse hooks blocked." A deny rule or managed PreToolUse hook still beats a mod's approval unless `allowModsToOverrideDenyRules` is set. Pluto Security's red team of a pre-GA build (v2.1.274) showed silent exfiltration of `~/.claude/.credentials.json` and `history.jsonl`, no disclosure at install, spoofed UI (a fake "re-enter API key" prompt), and code drift via fetched remote scripts ([[Claude Code Mods Field Notes]]).

> [!warning] Conflict
> The official mods overview (cited in [[Claude Code Mods Field Notes]]) says a mod "can't change what a prompt shows you"; Pluto Security's red team (same source page, pre-GA v2.1.274) showed a mod rewriting AskUserQuestion text. Not yet checked against the GA build.

**Org controls.** On machines with managed settings, and for anyone signed into a Team or Enterprise plan, the built-in `sec-default@builtin` guard (shown as `cc-plugin-sec-default` in `/plugin`) loads ahead of every installed mod, protects managed hooks, the system prompt and managed CLAUDE.md, and fails closed. Admin policies (as of 2026-10, [[Claude Code Mods Field Notes]]):

| Policy | How |
|---|---|
| No installed mods, hooks untouched | `allowManagedModsOnly`, deploy no mods |
| No mods and no hooks at all | `disableAllHooks: true` (also kills managed hooks, status lines, `/goal`) |
| Only the org's mods | `allowManagedModsOnly` + managed `enabledPlugins`, loaded from an absolute-path marketplace |
| Any mod from approved marketplaces | Marketplace restrictions + `disableSideloadFlags: true` (blocks `--plugin-dir`) |
| Any mod, checked by an org policy mod | Policy mod + `sec-default@builtin` in `prependPlugins` |

A plugin copied from GitHub, git, a URL or npm always counts as a user's mod, even when managed settings enable it.

**Ecosystem.** A static scan of 2,688 public mods in the first 5 days found mostly local UI: dashboards/usage/cost (~637), git/PR/CI (~256), handoff/memory (~216), games (~178). Only 108 hook `tool.check`. 39.7% run processes, 32.3% read env vars, 10.6% send network requests ([[Claude Code Mods Field Notes]]).

**Version notes** (from [[Claude Code Mods Field Notes]]): v2.1.286 Desktop support; v2.1.287 mods ship on by default in the CLI and `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS` is ignored; v2.1.289 adds `ui.fault`; v2.1.290 adds `prompt.mention`.

## Getting started

All details as of 2026-10 (Claude Code v2.1.287–v2.1.292, [[Claude Code Mods Field Notes]]); the API is still changing, so check the [reference](https://code.claude.com/docs/en/plugins/mods/reference) first.

**Before anything:** run `/plugin` to confirm whether `cc-plugin-sec-default` is loaded and whether org policy (`allowManagedModsOnly`, `disableSideloadFlags`) blocks local mods. If a local mod silently fails to load, check that and issue #99130.

**Fastest path:** ask Claude Code to write the mod. It ships a `plugin-authoring` skill that knows the API, writes the files, and offers to hot-reload them into the session. Then validate, test, `tsc`, and persist.

**Files:**

```
my-mod/
├── .claude-plugin/plugin.json   # name, version, description, "types"
├── hooks/hooks.json             # { "modules": ["./register.tsx"] }
├── hooks/register.tsx           # export const register: Register = (on, options) => ...
├── types/index.d.ts             # optional: typed $.state contract
└── hooks/register.test.ts       # optional: run with `claude plugin test`
```

```json
{
  "name": "turn-band",
  "version": "0.1.0",
  "description": "Shows last turn's duration and tool count above the prompt",
  "types": "./types/index.d.ts"
}
```

**Minimal example:** an observe-only band showing last turn's duration, with state in `$.state` so it survives hot reload (trimmed from the bundled skill):

```tsx
import { atom, read, update } from 'claude-code'
import type { Register } from 'claude-code'

const last = atom({ plugin: 'turn-band', key: 'last' } as const, null)

export const register: Register = on => {
  let startedAt = 0   // fine for one turn; resets on reload

  on('prompt.submit', async ($, e, next) => {
    startedAt = await $.clock.now()
    return next(e)
  })

  on('turn.complete', async ($, e, next) => {
    const seconds = Math.round(((await $.clock.now()) - startedAt) / 1000)
    await update($, last, () => ({ seconds }))
    return next(e)
  })

  on('ui.render', { component: 'AbovePrompt' }, async ($, e, next) => {
    const turn = await read($, last)
    if (e.props.hasSurvey || turn === null) return next(e)
    const { Box, Text } = $.ui.resolve(e)
    return <Box><Text dimColor>Last turn: {turn.seconds}s</Text></Box>
  })
}
```

Other shapes: a slash command registers in `session.start` via `$.command.register({ name, description })` and answers in `command.run` with `{ text }` (no Claude turn); a prompt rewrite is `next({ ...e, text })` on `prompt.submit`; system-prompt edits hook `prompt.compose` and change the `{ sections }` that `next(e)` returns. For a guard that fails closed, see [[Claude Code Mods Field Notes]].

**Commands:**

```bash
claude --plugin-dir ~/mods/turn-band                      # one session; persist via CLAUDE_CODE_PLUGIN_DIRS in user settings
claude plugin validate ~/mods/turn-band --strict --json   # what it hooks and calls, without running it
claude plugin test ~/mods/turn-band                       # run *.test.ts against the engine
tsc -p ~/mods/turn-band                                   # validate does not type-check
claude --safe-mode                                        # kill switch: every mod off, org ones too
# in-session: /reload-plugins, /plugin
```

**Gotchas:**
- Hot reload (at the end of each turn that edits the mod) re-runs `register` and fires `session.start` again, wiping module-level variables. Keep state in `$.state` or `$.store`.
- Hooks fail open: a throwing hook is skipped with only a dim transcript line. Any policy hook needs `.catch`, `try/catch` or `Promise.race` to fail closed, plus a test proving it.
- Time limits: 10 s of a hook's own time per event, 50 ms for `prompt.edit`; time inside `next` doesn't count.
- `$.fs.write` is not atomic; share cross-session data through `$.store`.
- Unknown props fail quietly (e.g. `borderStyle: 'rounded'` draws no border).
- The Desktop app bundles its own engine, which can differ from the CLI and cause unrelated validator errors.
- A user-installed telemetry hook must filter `{ to: 'collector' }` or `validate` fails it.
- String-matching Bash guards miss `$(...)`, aliases and nested scripts; permission rules remain the hard block.
- Rerun `claude plugin validate` after every Claude Code upgrade.

**Before installing someone else's mod:** treat it as an untrusted binary. The notes' own checklist, derived from the sources: Read every file; compare `validate --strict --json` output with what the README claims; search for `$.http`, `$.process.run`, `$.fs.read`, `$.model` and URLs; reject mods that fetch and run remote code; distrust mod-drawn prompts asking for keys; pin to a reviewed commit. Start from the Anthropic built-ins and samples (`token-weather`, `blast-radius`, `replay-theater` in https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods).

## Open questions

- Does the GA build still let a mod rewrite AskUserQuestion text, contradicting the docs? (see the conflict above)
- Is the author of design thread #91870 an Anthropic maintainer? No commenter is tagged MEMBER, OWNER or COLLABORATOR.
- Which built-in features move to mods next, and when?
- When will VS Code, Remote Control and iOS draw mod UI (issues #99423, #99045, #99217)?

## Related

- Claude Code hooks, Agent Skills and MCP: the out-of-process extension points mods are compared against (no pages yet).
- [[Claude Code Mods Field Notes]]: the source page, with a fail-closed guard example and the reading list.

## Sources

- [[Claude Code Mods Field Notes]]
