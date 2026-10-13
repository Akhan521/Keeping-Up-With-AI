---
type: source
topics: [claude]
updated: 2026-10-12
origin: "Claude Code Mods: field notes (pasted notes compiled from official docs, blogs, videos and GitHub; reflects Claude Code v2.1.287 to v2.1.292)"
author:
published:
kind: notes
raw: raw/Claude/claude-code-mods.md
---
# Claude Code Mods Field Notes

Field notes on [[Claude Code Mods]], the in-process plugin extension point Claude Code *(no page yet; suggested source: Claude Code overview docs)* shipped in v2.1.287, compiled from 22 sources (official docs and launch post, the bundled `plugin-authoring` skill, the Pluto Security red team, practitioner blogs, the #91870 design thread and a community catalogue scan). Text marked "(note)" in the raw file is the compiler's commentary, not from the sources. [raw](../../../raw/Claude/claude-code-mods.md)

## Key takeaways

- A mod is a Claude Code plugin whose JS/TS module runs **inside** the Claude Code process. It can rewrite prompts and the system prompt, guard tool calls and permissions, draw UI (panes, bands, status line, toasts), and replace built-in features. `/diff` and AGENTS.md loading already ship as mods, and Anthropic plans to move more built-ins to mods.
- Settings hooks, skills and MCP work from outside the process; only a mod can draw UI, keep state shared across hooks, and rewrite events in place. Rule of thumb: use the least powerful option that does the job.
- A mod is an ordinary plugin plus one key, `"modules"`, in `hooks/hooks.json`, so it ships through the same marketplaces as skills and MCP servers.
- Every hook is middleware with the signature `($, e, next)`: observe (`return next(e)`), rewrite (`return next({ ...e, x })`) or answer (return a result without calling `next`). Events pass down tiers: org-prepend, user mods, org-append, built-in mods, engine default. There are no numeric priorities.
- Every `$` API method is itself an event, so one mod can intercept another mod's `fs.read`, `model.complete` or `ui.open`.
- **Not sandboxed in any useful sense.** Mods run in a Bun Worker + `node:vm` and cannot `import fs` or call `fetch()`, but `$.fs`, `$.process` and `$.http` carry the user's full permissions, and per the admin docs `$.fs` / `$.process` bypass deny rules. A pre-GA red team showed silent credential exfiltration, no disclosure at install, spoofed UI and remote code drift.
- Hooks **fail open** by default: a throwing hook is skipped silently. Guards need `.catch` to fail closed.
- The top gotcha: hot reload wipes module-level variables. Keep state in `$.state` (per session) or `$.store` (machine-wide, 4 MiB).
- UI draws in the terminal and the Desktop Code tab only; VS Code, `claude -p`, cloud sessions and Remote Control run hooks but draw no (or partial) UI.
- The fastest way to build one is to ask Claude Code, which ships a `plugin-authoring` skill, then `claude plugin validate`, `claude plugin test` and `tsc`.
- Ecosystem after 5 days: 2,688 public mods, mostly dashboards and UI; guard mods are a small minority. Team/Enterprise and managed-settings machines load a fail-closed `sec-default` guard ahead of every mod.

## Details worth keeping

Layout of a mod (three required files, two optional):

```
my-mod/
├── .claude-plugin/plugin.json   # manifest: name, version, description, "types"
├── hooks/hooks.json             # { "modules": ["./register.tsx"] }  <- the one new key
├── hooks/register.tsx           # export const register: Register = (on, options) => ...
├── types/index.d.ts             # optional: typed contract for $.state values
└── hooks/register.test.ts       # optional: run with `claude plugin test`
```

A guard that fails closed (adapted from the bundled `plugin-authoring` skill, Claude Code 2.1.292):

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
}
```

Commands:

```bash
claude --plugin-dir ~/mods/turn-band                      # load for one session
claude plugin validate ~/mods/turn-band --strict --json   # inspect hooks/calls without running
claude plugin test ~/mods/turn-band                       # run *.test.ts against the engine
tsc -p ~/mods/turn-band                                   # validate does not type-check
claude --safe-mode                                        # kill switch: all mods off, org ones too
# in-session: /reload-plugins, /plugin
/plugin install turn-band --marketplace <owner>/<repo>
```

Admin settings named in the source: `allowManagedModsOnly`, `disableAllHooks`, `disableSideloadFlags`, `prependPlugins` / `appendPlugins`, `enabledPlugins`, `allowModsToOverrideDenyRules`. `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS` is ignored from v2.1.287.

Reading list from the source: the Anthropic built-ins (https://github.com/anthropics/claude-code/tree/main/mods), the Anthropic samples `token-weather`, `blast-radius`, `replay-theater` (https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods), the official docs (https://code.claude.com/docs/en/plugins/mods/overview, `/reference`, `/admin`), the Pluto Security write-up (https://pluto.security/blog/claude-code-function-hooks-security/) and the community catalogue (https://github.com/karanb192/awesome-claude-code-mods).

## Pages updated

- [[Claude Code Mods]] (created)
