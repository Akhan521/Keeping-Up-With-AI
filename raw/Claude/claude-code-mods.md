---
origin: "Claude Code Mods: field notes (pasted notes compiled from official docs, blogs, videos and GitHub; reflects Claude Code v2.1.287 to v2.1.292)"
author:
published:
---

# Claude Code Mods: field notes

- **Version:** Reflects Claude Code v2.1.287 to v2.1.292.
- **Shelf life:** Mods are new and the API is still changing. Check the official docs before relying on a detail here.
- **Convention:** Bracketed tags like [s4] point to the source list at the bottom. Text marked **(note)** is commentary, not from the sources.

---

## TL;DR

- A **mod** is a Claude Code plugin whose JavaScript/TypeScript code runs **inside** the Claude Code process.
- It can draw UI (panes, bands, status lines, toasts) and it can intercept, rewrite or answer prompts and tool calls.
- It is the most powerful extension point Claude Code has. It also has the most privileges.
- **Mods are not sandboxed in any useful sense.** They cannot `import fs`, but the `$` API they get can read, write, run processes and use the network with your full user permissions.
- Shipped in **v2.1.287**, on by default. A mod is 3 files: `plugin.json` + `hooks.json` + one module.
- 2,688 public mods were catalogued in the first 5 days. Most are dashboards and UI, not guards.
- **Safe starting point:** the Anthropic built-ins and the Anthropic sample mods. Treat every community mod like an untrusted binary until you read it.
- **Fastest way to build one:** ask Claude Code to write it. It ships a `plugin-authoring` skill that knows the API.

---

## 1. What a mod is

Claude Code emits events for nearly everything it does: a prompt is submitted, a tool is called, a permission is checked, a piece of UI is drawn. A mod registers functions on those events. Each function runs before, after, instead of, or wrapped around Claude Code's own handling. [s2]

- Anthropic's launch post: a mod is code that can "rewrite a prompt, add new UI, replace a built-in feature, or add entirely new functionality." [s2]
- The docs, one line: "A mod is a plugin that changes how Claude Code looks and behaves." [s1]

### The four things a mod can do

| Verb | What it touches | Examples |
|---|---|---|
| **Rewrite** | Prompts and the system prompt | Change a prompt before the model sees it, inject context, add or drop system-prompt sections. [s2] |
| **Guard** | Tool calls and permissions | Block, rewrite or retry a tool call. Approve or deny a permission request. Redact secrets from tool output. [s2] |
| **Draw** | Panes, bands, status line, toasts | New UI beside the transcript or above the prompt. Restyle built-in rows, the spinner and the question dialog. [s1] |
| **Replace** | Built-in features | `/diff` and AGENTS.md loading already ship as mods. [s2, s1] |

**Anthropic's stated direction:** "We plan to move more built-in features to mods over time, so you can pare Claude Code down to a small core and add back only what you want." [s2]

---

## 2. Mods vs settings hooks, skills and MCP

Settings hooks, skills, status-line scripts and MCP servers all work from **outside** Claude Code. They run a script, supply text, or expose tools across a boundary. A mod runs **inside** the process. That is why only a mod can draw UI, hold state shared across hooks, and rewrite events in place. [s1, s9]

### Which one to use

| You want to... | Use | Why |
|---|---|---|
| Draw a pane or a band above the prompt, add an instant `/command`, rewrite an event | **Mod** | Only in-process code can draw or rewrite. |
| Block, allow or log a tool call with a script you already have | **Settings hook** | Simpler, and it does not load code into the process. |
| Stop pasting the same instructions | **Skill** | Changes what Claude knows, not how Claude Code runs. |
| Give Claude access to an external system | **MCP** | Tools over a protocol boundary. |

Rule of thumb from the docs. [s1] The blast-radius explainer summarizes it: skills change what Claude knows, shell hooks return a verdict, and mods can do both and also draw UI. [s9, s10]

**(note)** Prefer the least powerful option that does the job. A settings hook or a skill cannot draw UI, but it also cannot silently exfiltrate your credentials.

---

## 3. Anatomy of a mod

A mod is an ordinary plugin with one extra key in `hooks/hooks.json`: `"modules"`, which points at a JS/TS file. Because it is just a plugin, it ships through the same marketplaces as skills and MCP servers. [s1, s14]

```
my-mod/
├── .claude-plugin/plugin.json   # manifest: name, version, description, "types"
├── hooks/hooks.json             # { "modules": ["./register.tsx"] }  <- the one new key
├── hooks/register.tsx           # export const register: Register = (on, options) => ...
├── types/index.d.ts             # optional: typed contract for $.state values
└── hooks/register.test.ts       # optional: run with `claude plugin test`
```

Three required files, two optional. [s4, s7]

`.claude-plugin/plugin.json`:

```json
{
  "name": "turn-band",
  "version": "0.1.0",
  "description": "Shows last turn's duration and tool count above the prompt",
  "types": "./types/index.d.ts"
}
```

`hooks/hooks.json`:

```json
{ "modules": ["./register.tsx"] }
```

---

## 4. The hook chain (middleware for an agent)

Every hook has the same signature: `($, e, next)`. Anthropic's own video calls the system "pretty much just middleware built into Claude Code." [s14] If you know Express middleware, you know the model.

- **`$`** is the engine interface. Every call is noun then method: `$.ui.toast`, `$.fs.read`, `$.process.run`. [s7]
- **`e`** is the event input. It is deeply frozen plain data. To change it, pass a copy to `next`. [s4]
- **`next(e)`** runs the mods beneath, then Claude Code's own behaviour, and resolves to the event's result. [s7]

### Three patterns

| Pattern | Code | Effect |
|---|---|---|
| **Observe** | `return next(e)` | Look, record, maybe draw, then pass the event through unchanged. |
| **Rewrite** | `return next({ ...e, x })` | Everything downstream, including Claude Code, sees the changed event. |
| **Answer** | `return { deny: '...' }` | Do not call `next`. Your value replaces Claude Code's default behaviour. [s14] |

### Tier order

The event flows down through these tiers, and the result flows back up:

1. **org-prepend** (the `sec-default` guard lives here)
2. **user mods** (in load order)
3. **org-append** (managed)
4. **built-in mods** (`/diff`, `agents-md`, ...)
5. **engine default** (Claude Code itself)

- The first-loaded mod sees the event first and the result last.
- Any tier can stop and answer instead of calling `next()`.
- There are no numeric priorities.
- Admin tiers cannot be changed by later hooks.
- Only a mod listed in `prependPlugins` / `appendPlugins` can call `next.to` to skip tiers. [s4, s16, s2]

---

## 5. Events and the `$` API

The event catalog covers tools, prompts, commands, turns, sessions, subagents, UI and settings-hook compatibility. **Every mods API method is also an event**, named for its namespace and method (`fs.read`, `model.complete`, `ui.open`). So a later mod can intercept another mod's API calls. [s4] Claude Code's own example says it has "a few dozen internal events." [s14]

### Event families

| Family | Events | Typical use |
|---|---|---|
| Tools | `tool.call`, `tool.check`, `tool.describe` | Guards, rewrites, redaction, auditing |
| Prompts | `prompt.submit`, `prompt.fill`, `prompt.suggest`, `prompt.edit`, `prompt.compose`, `prompt.mention` (2.1.290+) | Inject context, rewrite input, edit the system prompt (`{ sections }`) |
| Commands | `command.run` | Slash commands that answer with no Claude turn |
| Turns and sessions | `turn.step`, `turn.complete`, `session.start`, `session.end`, `subagent` | Timers, meters, handoffs, registration |
| UI | `ui.render`, `ui.press`, `ui.input`, `ui.select`, `ui.focus`, `ui.scroll`, `ui.close`, `ui.message`, `ui.fault` (2.1.289+) | Draw and restyle, respond to presses and typing |
| Engine | `engine.create`, `process.spawn`, plus settings-hook events | Low-level interception |

**Hook result shapes:** `{deny}`, `{result}`, `{decision}`, `{drop}`, `{sections}`. [s4]

### The `$` namespaces

| Namespace | What it does |
|---|---|
| `$.ui` | `resolve`, `open`, `status`, `toast`, `log`: draw and notify. |
| `$.state` | Per-session state. Survives hot reload. [s4] |
| `$.store` | Machine-wide key-value store, 4 MiB of JSON total. [s4] |
| `$.command` / `$.tool` / `$.agent` | Register slash commands, model-callable tools, and subagent types. [s7] |
| `$.fs` / `$.process` / `$.http` | Files, processes and network, **with your permissions**. Hookable by other mods. [s16] |
| `$.model` | Call a model from inside a hook. OneWave's inner-monologue mod uses this. [s10] |
| `$.clock` | `now`, `every`, `after`. [s7] |
| `$.audio` | Play a sound bundled with the mod. [s7] |

### What public mods actually call

Call counts across the karanb192 catalogue scan (2,900 entries). [s20]

| API call | Calls |
|---|---|
| `$.command.register` | 1,945 |
| `$.ui.resolve` | 1,892 |
| `$.state` get/set | 1,555 |
| `$.clock.now` | 1,453 |
| `$.ui.toast` | 1,247 |
| `$.process.run` | 1,104 |
| `$.store` get/set | 1,100 |
| `$.clock.every` | 1,088 |
| `$.ui.open` | 1,000 |
| `$.ui.log` | 910 |

---

## 6. Code examples

The first three are adapted from the `plugin-authoring` skill bundled with Claude Code 2.1.292, whose full versions load and type-check on that build. The ones marked "trimmed" are cut down to show the shape. Read the originals before copying. [s7]

### 6.1 Guard a tool call, and fail closed

Deny edits to `.env` files. **The `.catch` matters:** by default a hook that throws **fails open**, which means the call goes through. [s16, s5]

```ts
// hooks/register.ts
import type { Register } from 'claude-code'

const PROTECTED = /(^|\/)\.env(\.|$)/

export const register: Register = on => {
  on('tool.call', { tool: 'Edit' }, ($, e, next) =>
    PROTECTED.test(e.file_path)
      ? { deny: `${$.plugin.name}: ${e.file_path} is protected here.` }
      : next(e),
  ).catch(($, e, next) =>
    // guard threw: refuse rather than let the edit through
    next.called ? next(e) : { deny: `${$.plugin.name}: its guard failed.` },
  )

  on('tool.call', { tool: 'Bash' }, async ($, e, next) => {
    const ran = await next({ ...e, command: e.command.trim() })  // rewrite
    const hasFailed = ran.deny === undefined && ran.isError === true
    $.ui.status(hasFailed ? `failed: ${e.command.slice(0, 40)}` : undefined)
    return ran
  })
}
```

### 6.2 A band above the prompt, with state that survives reload (trimmed)

Values live in `$.state` through `atom` / `read` / `update`. A write redraws whatever reads the value. Module-level `let` variables reset on every hot reload. [s7]

```tsx
// hooks/register.tsx
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
    if (seconds > 120) $.ui.toast(`That turn took ${seconds}s`)
    return next(e)
  })

  on('ui.render', { component: 'AbovePrompt' }, async ($, e, next) => {
    const turn = await read($, last)
    if (e.props.hasSurvey || turn === null) return next(e)
    const { Box, Text } = $.ui.resolve(e)   // elements for this surface
    return <Box><Text dimColor>Last turn: {turn.seconds}s</Text></Box>
  })
}
```

```ts
// types/index.d.ts: the $.state contract that `validate` checks against
export type Turn = { seconds: number }

declare module 'claude-code' {
  interface PluginState {
    'turn-band': { last: Turn | null }
  }
}
```

### 6.3 A slash command that opens a pane (trimmed)

```tsx
const PANE = 'tool-calls'

on('session.start', async ($, e, next) => {
  await $.command.register({ name: 'tool-calls', description: 'Show tool calls in a pane' })
  return next(e)
})

on('command.run', { command: 'tool-calls' }, async $ => {
  await $.ui.open({ id: PANE, title: 'Tool calls' })
  return { text: 'Tool calls pane opened.' }   // answered: no Claude turn
})

on('ui.render', { component: 'Pane', requestId: PANE }, async ($, e) => {
  const { Box, Text } = $.ui.resolve(e)
  const list = await read($, calls)
  return <Box flexDirection="column">{list.map(c => <Text>{c.tool}</Text>)}</Box>
})
```

### 6.4 Rewrite a prompt

```ts
on('prompt.submit', ($, e, next) =>
  next({ ...e, text: e.text + '\n\n(Answer in bullets.)' }))
```

- The rewrite pattern is `next({ ...e, text })`. [s7]
- To change the **system prompt**, hook `prompt.compose` and add, replace or drop entries in the `{ sections }` that `next(e)` returns.

---

## 7. Drawing UI, and where it shows up

A `ui.render` hook returns a JSX tree built from the elements of the current surface: `const { Box, Text, Button } = $.ui.resolve(e)`. `e.surface` is `terminal`, `desktop`, `vscode` or `mobile`. The module has no DOM and no Node, so it reaches everything through `$`. [s7]

### Where each kind of UI lands (terminal)

| UI | How | Where |
|---|---|---|
| Built-in rows and spinner | Restyle via `ui.render` | In the transcript |
| **Pane** | `$.ui.open({ id, title })` | Beside the transcript |
| **AbovePrompt band** | `ui.render` on `AbovePrompt` | Strip directly above the prompt |
| **Status** | `$.ui.status('...')` | Status line under the prompt |
| **Toast** | `$.ui.toast('...')` | Pop-up notification |

**Pane width rule:** A pane opened by a user action (a command, a button press) seats at any width. A pane opened unprompted (from `session.start` or a timer) waits until the terminal is **at least 144 columns** wide. [s7]

### Surface support

| Surface | Hooks run | UI drawn | Notes |
|---|---|---|---|
| Terminal CLI | yes | yes | v2.1.287+ [s1] |
| Desktop app, Code tab | yes | yes | v2.1.286+. The Desktop app does not take flags, so add the mod folder in its settings. [s1, s14] |
| VS Code extension | yes | **no** | `extension.js` has no `ui_render` handler. [s22] |
| `claude -p` / cloud sessions | yes | **no** | Hooks only. Build mods to fall back gracefully. [s1, s10] |
| Remote Control / iOS | partial | gaps | Not drawn in Desktop/iOS Remote Control. `$.command.register` commands are missing from the `/` menu. [s22] |

---

## 8. How to build your own mod

Practitioners agree on the fastest path: **ask Claude Code to write the mod.** It ships a `plugin-authoring` skill that knows the API, writes the files, and offers to hot-reload them into the current session. [s10, s11, s15]

**(note)** Claude Code ships this skill. Its description: "Make a mod: a change to Claude's own interface or behaviour, such as a live pane or panel, a band above the prompt, a status line, a toast, a slash command, or a hook on tool calls or prompts."

### The six-step loop [s11, s7]

1. **Ask** in plain English what you want the mod to do.
2. **Hot reload.** Claude asks "Enable...?" and loads it into the current session.
3. **Validate** with `claude plugin validate`.
4. **Test** with `claude plugin test`.
5. **Persist** by adding the folder to `CLAUDE_CODE_PLUGIN_DIRS`.
6. **Share** through a marketplace.

Each edit reloads when the turn that made it ends. `register` runs again and `session.start` fires again.

### Commands [s4, s5, s7]

```bash
# load a mod folder for one session (terminal)
claude --plugin-dir ~/mods/turn-band

# load it in every session: add the folder to CLAUDE_CODE_PLUGIN_DIRS in user settings

# inspect what a mod hooks and calls WITHOUT running it
claude plugin validate ~/mods/turn-band --strict --json

# run its *.test.ts files against the engine
claude plugin test ~/mods/turn-band

# type-check once it has loaded (types are laid into .claude-plugin/types/)
tsc -p ~/mods/turn-band

# in-session: /reload-plugins, /plugin to see what loaded
# share: one file makes the repo a marketplace
/plugin install turn-band --marketplace <owner>/<repo>

# kill switch while debugging: every installed mod off, org ones too
claude --safe-mode
```

**Rerun `validate` after every Claude Code upgrade**, because the API is still changing. [s11]

---

## 9. Gotchas and limits

**The number one gotcha: hot reload wipes module variables.** A reload is a fresh load of the module. `register` runs again and `session.start` fires again. Anything in a module-level `let` starts over. Keep state in `$.state` (per session) or `$.store` (across sessions). Three sources hit this independently. OneWave's burn-meter lost its data mid-build because of it. [s10, s11, s12, s7]

| Gotcha | Detail |
|---|---|
| **Failures are silent** | A hook that throws is skipped. The only trace is a dim transcript line or a debug-log entry. That is why OneWave ships tests with every mod. [s10, s7] |
| **Fail-open by default** | An unhandled hook error routes around the hook. A policy hook needs `.catch`, `try/catch` or `Promise.race` to fail closed. [s16, s5] |
| **Time limits** | A hook gets 10 s of its own time per event. A `prompt.edit` hook gets 50 ms. Time spent inside `next` does not count. [s4] |
| **`$.fs.write` is not atomic** | It replaces content in place. Share data across sessions through `$.store` (4 MiB total per machine). [s4] |
| **Unknown props fail quietly** | A `Box` with an unrecognized `borderStyle` such as `'rounded'` draws with no border at all. [s4] |
| **Engine skew** | The Desktop app bundles its own engine. It can differ from the CLI on PATH and cause validator errors unrelated to your code. [s11] |
| **`validate` does not type-check TypeScript** | Run `tsc` as well. [s22] |
| **Telemetry hooks** | A user-installed telemetry hook must filter `{ to: 'collector' }`, or `validate` fails it. [s4] |
| **Heuristic guards miss things** | Blast-radius-style command matching misses `$(...)`, aliases and nested scripts. Permission rules are still the hard block. [s9] |

---

## 10. Security model (read this before installing anything)

Two statements look contradictory. The docs say "mods aren't sandboxed." Anthropic's video says a mod "runs in its own sandbox." [s1, s14] **Both are true, at different layers.**

| Layer | What it means |
|---|---|
| **Isolated: ambient access** | Each plugin runs in a Bun Worker + `node:vm`. It cannot `import fs`, use `fetch()`, the DOM or Node. "It's a boundary! ... you cannot `import fs`" (design thread #91870). |
| **Not sandboxed: what `$` grants** | Your full user permissions. `$.fs.read('~/.claude/...')` works. `$.http` and `$.process.run` work. Per the admin docs, `$.fs` / `$.process` bypass deny rules. |

**The isolation stops a mod from importing ambient APIs. It does not limit what `$` can do, and `$` can do anything you can.** [s16, s5]

**The docs, verbatim:** "Mods aren't sandboxed... A mod that approves tool calls can approve one that an ask rule would prompt for, or that one of your own PreToolUse hooks blocked." [s1]

- A **deny rule** or a **managed PreToolUse hook** still beats a mod's approval.
- Exception: if `allowModsToOverrideDenyRules` is set, it does not. [s5]

### Pluto Security red team (pre-GA build, v2.1.274) [s8]

| Risk | What they showed |
|---|---|
| **1. Silent exfiltration** | A user-tier mod read `~/.claude/.credentials.json` and all of `history.jsonl` (834 KB) and POSTed them out. "No prompt appeared. No permission was asked." |
| **2. No disclosure at install** | `plugin install` asks nothing. `plugin details` showed a mod hooking 4 events, including a wildcard, as "Hooks (0), ~0 tokens". |
| **3. Spoofed UI** | A mod can draw a fake "re-enter API key" prompt and rewrite AskUserQuestion text. Descriptive option labels are restored, but generic "Confirm" labels stay spoofable. |
| **4. Code drift** | A mod can fetch and run a remote script, so the code you reviewed is not the code that runs. The content scan only covers the claude.ai install path. |

**Open conflict:** The docs say a mod "can't change what a prompt shows you." Pluto shows a mod rewriting AskUserQuestion text. Nobody has checked this against the GA build. [s1, s8]

### What public mods can reach

Share of 2,688 catalogued mods by capability label (karanb192 static scan). Labels overlap, so shares do not sum to 100%. The scanner has no "reads secrets" label, so env-var reads are the closest signal. [s20]

| Capability | Share of mods |
|---|---|
| Persists state | 39.8% |
| Runs processes | 39.7% |
| Reads env vars | 32.3% |
| Drives Claude | 29.4% |
| Writes files | 17.3% |
| Sends network | 10.6% |

### Practical stance [s8, s1, s9]

- Treat a mod like an **untrusted binary** until you have read it.
- Run `claude plugin validate` **before** installing.
- Review a Claude-scaffolded mod like a PR that grants filesystem and process access.
- Install only from authors and marketplaces you trust.

### Safety checklist (note, derived from the above)

Before installing any mod that is not an Anthropic built-in or sample:

1. Read every file in the module. It is usually short.
2. Run `claude plugin validate <dir> --strict --json` and compare what it hooks with what the README claims.
3. Search the module for `$.http`, `$.process.run`, `$.fs.read`, `$.model` and any URL. Ask why each one is there.
4. Reject any mod that fetches and runs remote code (Pluto risk 4).
5. Distrust any mod-drawn prompt that asks for a key, token or password (Pluto risk 3).
6. Pin to a commit you reviewed, not a moving branch, so the code cannot drift.
7. Keep `claude --safe-mode` in mind as the kill switch.
8. Follow your org's policy: never commit secrets, and route AI-generated code through PR review. A mod that you write still counts.

---

## 11. Org controls (managed settings)

On a machine with managed settings, **or for anyone signed into a Team or Enterprise plan**, the built-in guard `sec-default@builtin` loads **ahead of every installed mod**. [s4, s16, s17]

- It shows up as `cc-plugin-sec-default` in `/plugin`.
- It protects managed hooks, the system prompt and managed CLAUDE.md.
- It **fails closed**.

**(note)** If your org uses managed settings, `sec-default` should load. Check with `/plugin`.

### Policy options for admins [s5]

| Policy | How | Trade-off |
|---|---|---|
| No installed mods, hooks untouched | `allowManagedModsOnly`, deploy no mods | Org hooks, status lines and `/goal` keep working. |
| No mods and no hooks at all | `disableAllHooks: true` | Widest switch. Also kills managed hooks, status lines and `/goal`. |
| Only your org's mods | `allowManagedModsOnly` + your own mods | They must be enabled through managed `enabledPlugins` and loaded in place from an absolute-path marketplace. |
| Any mod from approved marketplaces | Marketplace restrictions + `disableSideloadFlags: true` | Blocks `--plugin-dir` sideloading. |
| Any mod, checked by your own policy mod | Your mod + `sec-default@builtin` in `prependPlugins` | Its `plugin.register` hook fails open unless you add `.catch`. |

Other admin facts: [s5, s4, s22]

- A plugin copied from GitHub, git, a URL or npm **always counts as a user's mod**, even when managed settings enable it.
- If managed `prependPlugins` is set, `sec-default` loads only if that list names it.
- The old `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS` env var is ignored from v2.1.287.

**(note)** If your org sets `allowManagedModsOnly` or `disableSideloadFlags`, local `--plugin-dir` mods will not load. If a local mod silently fails to load, check this and issue #99130 first.

---

## 12. Mods worth looking into

Ordered by trust, highest first. Only the first two tiers come from Anthropic.

### Tier 1: Anthropic built-ins (already in Claude Code)

| Mod | What it is |
|---|---|
| `diff` | The `/diff` feature, shipped as a mod. [s2] |
| `agents-md` | AGENTS.md loading. The first feature shipped as a mod. [s2, s18] |
| `sec-default` | The managed-settings guard. Loads first, fails closed. Source: https://github.com/anthropics/claude-code/tree/main/mods/sec-default [s17] |
| `telemetry` | First-party telemetry, shipped as a mod. [s2] |

Source for all built-ins: https://github.com/anthropics/claude-code/tree/main/mods

### Tier 2: Anthropic samples (read these to learn the patterns)

From https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods [s1, s9, s19]

| Mod | Pattern it teaches |
|---|---|
| `token-weather` | An observe-only band. |
| `blast-radius` | A Proceed/Cancel guard on risky Bash commands. Note the gotcha: string matching misses `$(...)`, aliases and nested scripts. |
| `replay-theater` | A `/replay` diff stepper (command + pane). |

### Tier 3: Community (not validated; read before running)

| Mod | What it does |
|---|---|
| [claude-auto-handoff](https://github.com/alexknowshtml/claude-auto-handoff) | Blocks tool calls past a context threshold, has Haiku write a brief, then seeds a new session. [s20] |
| [hamzafer/claude-code-mods](https://github.com/hamzafer/claude-code-mods) | Context bar, usage meter, panes, games. [s21] |
| OneWave `launch-codes`, `burn-meter`, +8 more (MIT) | `launch-codes` puts risky Bash behind a one-time code and has 64 tests. `burn-meter` shows spend as a fire bar. [s10] |
| [darrelltw-mods](https://github.com/darrell-tw/darrelltw-mods) | Mod collection. [s21] |
| [ctx-handoff-mod](https://github.com/cablate/ctx-handoff-mod) | Context handoff. [s21] |
| [cc-mod-image-view](https://github.com/GGGODLIN/cc-mod-image-view) | Image viewing. [s21] |
| [prismantis](https://github.com/NahumLitvin/prismantis) | Mod collection. [s21] |
| [zycck/claude-mods](https://github.com/zycck/claude-mods) | Mod collection. [s21] |

Catalogue of everything: https://github.com/karanb192/awesome-claude-code-mods [s20]

### Ecosystem shape (first 5 days) [s20]

- 2,900 entries scanned from 1,315 repos: 2,688 mods, 112 fixtures, 57 duplicates, 39 catalogs, 4 built-ins.
- Most are local UI with state in `$.state` / `$.store`.
- Guard and security mods are a small minority. Only 108 hook `tool.check` and 130 hook `prompt.compose`.

| Largest categories | Approx. mods |
|---|---|
| Dashboards / usage / cost | ~637 |
| Git / PR / CI | ~256 |
| Handoff / memory | ~216 |
| Games | ~178 |

**The most useful mods cannot be shared.** OneWave found that its most valuable mods were the ones wired into its own CRM and client data, which are too sensitive to open-source. [s10] Boris Cherny: "Each person works differently, so there's no reason why everyone should have an identical Claude experience." [s10]

### Example mod ideas (note, not from the sources)

| Idea | Why | Pattern |
|---|---|---|
| Context-% band plus a handoff nudge at ~50% context | Starts a fresh session before quality drops. `claude-auto-handoff` is the reference design. | `turn.complete` + `AbovePrompt` band |
| Guard that blocks write SQL against production databases | Turns a read-only rule into a hard block. Must fail closed with `.catch`. Keep the permission rules as the real block. | `tool.call` guard on `Bash` |
| Session cost meter | Replaces checking cost after the fact. `burn-meter` is the reference. | `$.state` + band |
| `.env` / credentials edit guard | Direct copy of example 6.1. Supports the "never commit secrets" policy. | `tool.call` guard on `Edit` |

---

## 13. Known issues

These go stale fast. Check GitHub for the current state.

| Issue | What | State |
|---|---|---|
| [#91870](https://github.com/anthropics/claude-code/issues/91870) | Umbrella design and feedback thread, ~245 comments | open |
| [#99130](https://github.com/anthropics/claude-code/issues/99130) | Local hooks module does not load on 2.1.288 (rollout switch) | open |
| [#99423](https://github.com/anthropics/claude-code/issues/99423), [#99045](https://github.com/anthropics/claude-code/issues/99045) | VS Code draws no mod UI | open |
| [#99217](https://github.com/anthropics/claude-code/issues/99217) | Mod UI not drawn in Desktop/iOS Remote Control | open |
| [#99691](https://github.com/anthropics/claude-code/issues/99691) | `$.command.register` commands missing from `/` menu | open |
| [#99771](https://github.com/anthropics/claude-code/issues/99771) | `claude plugin validate` does not type-check TS | open |
| [#98987](https://github.com/anthropics/claude-code/issues/98987) | Pane close mark | fixed in 2.1.289 |

**Open feature requests:** multi-line text input (#99616), toast placement (#99509), `$.ui.notify` (#99306), `cache_control` on `$.model.complete` (#96214), a WebView pane (#99213). [s22]

**Caveat on the design thread:** No commenter on #91870 is tagged MEMBER, OWNER or COLLABORATOR. The maintainer voice is the thread's author, tagged CONTRIBUTOR. Treat its Anthropic affiliation as unverified. [s16]

---

## 14. Getting started

1. Run `/plugin` and confirm `cc-plugin-sec-default` is loaded and whether org policy allows local mods.
2. Read the Anthropic samples (`token-weather`, `blast-radius`, `replay-theater`) and the `sec-default` source.
3. Read the official docs pages: overview, reference, admin (links below).
4. Build one small observe-only mod with the `plugin-authoring` skill (for example, the turn-time band). Validate, test and `tsc` it.
5. Only then try a guard mod, with `.catch` to fail closed and a test that proves it.
6. Rerun `claude plugin validate` on every mod after each Claude Code upgrade.

---

## Sources

### Official

- [s1] Mods overview: https://code.claude.com/docs/en/plugins/mods/overview
- [s2] Launch post, Anthropic: https://claude.com/blog/claude-code-mods
- [s3] Changelog, v2.1.287 "Added Claude Mods": https://code.claude.com/docs/en/changelog
- [s4] Mods reference (files, events, API, limits; "as of v2.1.290"): https://code.claude.com/docs/en/plugins/mods/reference
- [s5] Manage mods for your organization: https://code.claude.com/docs/en/plugins/mods/admin
- [s6] Create a mod: https://code.claude.com/docs/en/plugins/mods/create · Events: https://code.claude.com/docs/en/plugins/mods/events · API: https://code.claude.com/docs/en/plugins/mods/api · Interface: https://code.claude.com/docs/en/plugins/mods/interface · Test: https://code.claude.com/docs/en/plugins/mods/test · Troubleshoot: https://code.claude.com/docs/en/plugins/mods/troubleshoot
- [s7] `plugin-authoring` skill bundled with Claude Code 2.1.292 (examples: `tool-call.ts`, `band.tsx`, `pane.tsx`). No URL; it ships inside Claude Code.
- [s14] Video, "Mods in Claude Code: change how Claude Code works", Claude (official): https://www.youtube.com/watch?v=yUuFvL1lK4k

### Analysis, blogs, video

- [s8] Pluto Security, "Claude Code function hooks security": https://pluto.security/blog/claude-code-function-hooks-security/
- [s9] dev.to, "Claude Code mods just landed: first hook, blast-radius guard, and why they're not sandboxed": https://dev.to/yong_yu_f98e15562e9b120a0/claude-code-mods-just-landed-first-hook-blast-radius-guard-and-why-theyre-not-sandboxed-3225
- [s10] OneWave AI, "We built 12 Claude Code mods in one night": https://www.onewave-ai.com/blog/claude-code-mods
- [s11] Pawel Nowak, "I built my first Claude Code mod without writing code": https://dev.to/pawel_nowak/i-built-my-first-claude-code-mod-without-writing-code-claude-wrote-it-validated-it-and-2mh7
- [s12] ruvnet, "Claude Code mods field guide" (gist): https://gist.github.com/ruvnet/a485e930b148185197fc53fd38b429ea
- [s13] paddo.dev: https://paddo.dev/blog/claude-code-mods/ · Better Stack guide: https://betterstack.com/community/guides/ai/claude-code-mods/
- [s15] Community videos (Chase AI "Claude Mods Is The Biggest Claude Code Upgrade Since Skills", Aman DevX, Tristen O'Brien, lustoykov, Kev Builds Apps, Daniel Agrici, AI Coding Daily, Ray Amjad):
  - https://www.youtube.com/watch?v=Rn4nmFRPe0s
  - https://www.youtube.com/watch?v=hjgObn_YiDU
  - https://www.youtube.com/watch?v=LDn7rQKIFro
  - https://www.youtube.com/watch?v=Lpf9gH3FvZQ
  - https://www.youtube.com/watch?v=xTRZhPLhQ5E
  - https://www.youtube.com/watch?v=LYgV2wPU5DQ
  - https://www.youtube.com/watch?v=_0NSNY7n5lE
  - https://www.youtube.com/watch?v=B-YQANvDOq0

### GitHub, HN, ecosystem

- [s16] Design thread anthropics/claude-code#91870: https://github.com/anthropics/claude-code/issues/91870 (isolation comment: https://github.com/anthropics/claude-code/issues/91870#issuecomment-5546290346, launch comment: https://github.com/anthropics/claude-code/issues/91870#issuecomment-5942362421)
- [s17] `sec-default` built-in guard source: https://github.com/anthropics/claude-code/tree/main/mods/sec-default
- [s18] HN: AGENTS.md as the first mod (rollout apology) and launch submission: https://news.ycombinator.com/item?id=49815421 · https://news.ycombinator.com/item?id=49815363 · https://news.ycombinator.com/item?id=49925102
- [s19] Anthropic sample mods: https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods
- [s20] Community catalogue, karanb192/awesome-claude-code-mods (capability scan of `data/mods.json`): https://github.com/karanb192/awesome-claude-code-mods
- [s21] Community repos: hamzafer, darrell-tw, alexknowshtml, cablate, GGGODLIN, NahumLitvin, zycck (links in section 12)
- [s22] Issues: #99130, #99423, #99045, #99217, #99691, #99535, #99771 (links in section 13; #99535: https://github.com/anthropics/claude-code/issues/99535)
