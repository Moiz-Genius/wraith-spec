# WRAITH — Full Build Specification

*A fast, CLI-only autonomous coding agent. Purple ghost theme. Full filesystem + shell authority. No gateway, no web, no bloat.*

Paste this entire document as your first message to the AI IDE agent. It's written as a direct instruction set — follow it in the order given, section 17 last.

---

## 0. Mission Brief

You are a senior TypeScript/Node.js engineer building **Wraith** — a local-first CLI coding agent inspired by OpenClaw's config philosophy (AGENTS.md/SOUL.md-style persona injection, provider-agnostic model layer) but stripped down to exactly one job: sit in a terminal, in a project folder, and read/write/edit/delete/execute anything in that folder on the operator's behalf. No daemon, no server, no browser, no chat-app integrations.

**Non-negotiables:**
- CLI only. No gateway process, no WebSocket server, no ports, no web UI, no mobile app.
- Full autonomy by default. No confirmation prompts for file writes, edits, deletes, or shell commands. The operator asked for this explicitly — don't add permission gates and don't water this down.
- Fast. Minimal dependencies, lazy-loaded providers, streamed output, sub-second cold start as the target.
- Purple ghost branding throughout — name, palette, glyphs, and microcopy — not just a logo bolted on top.
- Multi-provider from day one: OpenAI, Anthropic, Groq, Mistral, Gemini, plus any OpenAI-compatible custom endpoint via config, with zero code changes required to add one.

---

## 1. Identity & Theme

| | |
|---|---|
| Name | **Wraith** |
| Tagline | "The ghost in your codebase." |
| Symbol | 👻 (primary mark, used in banner, prompt, spinner) |
| Package / bin name | `wraith` |
| Config dir | `~/.wraith/` (global), `.wraith/` (per-project) |

**Glyph set** — use consistently as line prefixes in terminal output, never randomly:

```typescript
export const glyphs = {
  idle:     '👻', // boot banner, ready prompt
  thinking: '🔮', // planning / reasoning before acting
  acting:   '⚡', // a tool call is executing
  danger:   '💀', // destructive op, failed command, checkpoint-before-risk marker
  success:  '✨', // task complete
} as const;
```

**Color palette** — one source of truth, no hardcoded colors anywhere else in the codebase:

```typescript
// src/ui/theme.ts — "the Ectoplasm Palette"
export const palette = {
  accent:  '#A855F7', // primary highlights, active prompt, links
  deep:    '#5B21B6', // headers, panel borders, secondary emphasis
  pale:    '#E9D5FF', // dim/secondary text, ghostly narration lines
  white:   '#F5F3FF', // primary body text
  success: '#4ADE80', // success states, diff added lines
  error:   '#F87171', // errors, diff removed lines
  warning: '#FBBF24', // warnings
} as const;
```

**Startup banner** (renders in `accent`/`pale`, version string dimmed underneath):

```
        .-""""""-.
       /  ()  ()   \
      |      __     |
      |     (__)    |
       \_          _/
         \        /
       .-'          '-.
      /  |  |  |  |  |  \
     '---'--'--'--'--'---'
            W R A I T H
```

Spinner/status microcopy should be genuinely ghost-flavored, not generic: "Phasing through files…", "Materializing changes…", "Haunting the shell…", "Fading context back in…" (used for `/compact`). Keep it to one line, never blocks output.

---

## 2. Tech Stack & Project Layout

- **Language:** TypeScript, compiled to ESM.
- **Runtime:** Node.js 20+ (native `fetch`, no axios needed).
- **CLI parsing:** `commander` — small, no framework weight.
- **Terminal color:** `picocolors` — not `chalk`; a fraction of the size, same job.
- **Diffing:** `diff` npm package for unified-diff rendering of edits.
- **Config format:** JSON5 (`json5` package) — comments and trailing commas allowed, since humans edit this file by hand.
- **Search:** shell out to system `ripgrep` (`rg`) if present; fall back to a plain recursive JS walk only if `rg` is missing. Never reimplement grep in JS as the primary path — it's an order of magnitude slower.
- **Session storage:** plain JSONL files under `.wraith/sessions/<id>.jsonl`, not SQLite. No native bindings to compile — avoids the exact devcontainer/native-module pain of trying to get a compiled dependency working inside GitHub Codespaces or a restricted sandbox.

```
wraith/
├── bin/
│   └── wraith.js                 # shebang entry, imports dist/cli
├── src/
│   ├── cli/
│   │   ├── index.ts              # command router (commander setup)
│   │   ├── repl.ts               # interactive loop
│   │   └── commands/
│   │       ├── init.ts
│   │       ├── config.ts
│   │       ├── models.ts
│   │       ├── auth.ts
│   │       ├── sessions.ts
│   │       └── doctor.ts
│   ├── agent/
│   │   ├── loop.ts               # plan -> tool call -> observe -> repeat
│   │   ├── context.ts            # bootstrap file loader, token budget
│   │   ├── compaction.ts         # /compact summarization
│   │   └── session.ts            # JSONL session persistence
│   ├── providers/
│   │   ├── base.ts               # ProviderAdapter interface
│   │   ├── openai-compatible.ts  # OpenAI, Groq, Mistral, OpenRouter, etc.
│   │   ├── anthropic.ts
│   │   ├── gemini.ts
│   │   └── registry.ts
│   ├── tools/
│   │   ├── fs-read.ts
│   │   ├── fs-write.ts
│   │   ├── fs-edit.ts            # FIND/REPLACE patch tool
│   │   ├── fs-delete.ts
│   │   ├── fs-list.ts
│   │   ├── search.ts             # ripgrep-backed
│   │   ├── shell.ts              # unrestricted exec
│   │   ├── fetch-url.ts          # plain outbound GET for docs/error lookups
│   │   └── registry.ts
│   ├── ui/
│   │   ├── theme.ts
│   │   ├── banner.ts
│   │   ├── spinner.ts
│   │   ├── diffRenderer.ts
│   │   └── markdown.ts           # render model output with syntax highlighting
│   ├── config/
│   │   ├── schema.ts
│   │   ├── load.ts                # merges ~/.wraith + ./.wraith + env
│   │   └── defaults.ts
│   └── util/
│       ├── git.ts                # auto-checkpoint helper
│       └── tokens.ts              # counting / truncation
├── templates/                     # written by `wraith init`
│   ├── AGENTS.md
│   ├── SPIRIT.md
│   ├── GRIMOIRE.md
│   ├── SUMMONER.md
│   └── ECHOES.md
├── package.json
├── tsconfig.json
└── README.md
```

---

## 3. Core Agent Loop

1. On session start, load bootstrap files in this order: `AGENTS.md` → `SPIRIT.md` → `GRIMOIRE.md` → `SUMMONER.md` → `ECHOES.md` → today's + yesterday's `.wraith/memory/YYYY-MM-DD.md`. Concatenate into the system prompt.
2. A file that doesn't exist is skipped silently — no error, no warning printed. Don't nag the operator about optional files.
3. Enforce a token budget on injection: `bootstrapMaxChars` per file (default 20000), `bootstrapTotalMaxChars` overall (default 60000). Truncate from the end, not the middle.
4. Loop: user input → streamed model call with full tool schema → if the response contains tool calls, execute them → feed results back as tool results → repeat until the model returns plain text with no further tool calls → render.
5. Reads may run concurrently. Writes, edits, deletes, and shell calls run sequentially — never parallelize filesystem mutations against the same working tree.
6. `/compact`: when context grows past a configurable threshold, summarize older turns into a condensed note and replace them in-context, the same pattern as `/compact` in comparable coding agents.

---

## 4. Tool System

Exact tool surface. Function-calling schema names on the left, behavior on the right.

```
read_file(path, start_line?, end_line?)
  -> file contents with line numbers prefixed, for precise editing later

write_file(path, content)
  -> creates or fully overwrites; creates parent directories as needed
  -> use ONLY for brand-new files or an explicitly requested full rewrite

edit_file(path, old_str, new_str)
  -> exact-match FIND/REPLACE. old_str must match exactly once in the file
     or the call fails with a clear error (ambiguous match) rather than
     guessing which occurrence was meant
  -> this is the default way to modify existing files — never rewrite
     a whole file just to change a few lines

delete_path(path, recursive?)
  -> deletes a file or directory, no confirmation

list_dir(path, depth?)
  -> tree listing, respects .gitignore

glob(pattern)
  -> matching file paths

search(query, path?, regex?)
  -> ripgrep-backed content search, returns file:line:match

move_path(source, destination)
  -> rename or move

run_shell(command, cwd?, background?, timeout_ms?)
  -> foreground (default): streams stdout/stderr live to the terminal
     as it runs, returns { exitCode, stdout, stderr } on completion
  -> background: true detaches the process (e.g. dev servers, watchers),
     returns { pid, logFile } immediately instead of blocking the agent
     loop; the agent can tail logFile or check status later
  -> no allowlist, no denylist, no sandbox — arbitrary commands, full
     authority over the working directory and anything reachable from it

fetch_url(url)
  -> plain outbound GET for looking up docs / error messages, text
     extracted, no JS rendering — this is a utility call, not a gateway
```

**Editing philosophy:** `edit_file` (FIND/REPLACE) is the default tool for existing files — precise, cheap on tokens, and unambiguous about what changed. `write_file` is reserved for net-new files or explicit full rewrites. Never invent a file path, function signature, or library API without having verified it with `read_file` or `search` first.

---

## 5. Multi-Provider Model Layer

```typescript
// src/providers/base.ts
interface ProviderAdapter {
  id: string;
  streamChat(params: {
    model: string;
    messages: Message[];
    tools: ToolSchema[];
    systemPrompt: string;
  }): AsyncIterable<StreamChunk>;
}
```

Three built-in adapters cover almost everything:
- **`openai-compatible`** — handles OpenAI, Groq, Mistral, OpenRouter, Together, and any other `/v1/chat/completions`-shaped endpoint via a configurable `baseUrl`. This is one adapter, not four.
- **`anthropic`** — native Messages API, `tool_use` blocks.
- **`gemini`** — native `generateContent`, function-call blocks.

Adding a new provider is a config edit, not a code change — same pattern as OpenClaw's `models.providers.<name>` block. Never hardcode a provider list in the UI; render whatever's in config.

Lazy-load provider SDKs: only import the adapter code path for the provider actually selected, so switching to a Groq-only setup doesn't pull in Anthropic/Gemini client weight at boot.

---

## 6. Config System

`~/.wraith/config.json5` (global) merges with `./.wraith/config.json5` (per-project overrides win). Example — pre-filled with the providers actually in use, since inference speed matters here:

```json5
{
  agent: {
    workspace: ".",              // operates on the folder wraith is run in
    model: {
      primary: "groq/llama-3.3-70b-versatile",  // check `wraith models list` — names drift
      fallback: "openai/gpt-4o-mini",
    },
    autonomy: "full",             // "full" | "confirm-destructive" | "confirm-all"
    autoCheckpoint: true,
    bootstrapMaxChars: 20000,
    bootstrapTotalMaxChars: 60000,
  },
  providers: {
    groq:      { apiKey: "${GROQ_API_KEY}",      api: "openai-compatible", baseUrl: "https://api.groq.com/openai/v1" },
    mistral:   { apiKey: "${MISTRAL_API_KEY}",   api: "openai-compatible", baseUrl: "https://api.mistral.ai/v1" },
    openai:    { apiKey: "${OPENAI_API_KEY}",    api: "openai-compatible" },
    anthropic: { apiKey: "${ANTHROPIC_API_KEY}", api: "anthropic" },
    gemini:    { apiKey: "${GEMINI_API_KEY}",    api: "gemini" },
  },
  ui: { theme: "ghost-purple", banner: true },
}
```

API keys are always `${ENV_VAR}` references, never plaintext in the file. `.wraith/` goes in `.gitignore` by default on `wraith init`.

---

## 7. Context Files — the ghost's memory & rules

Same bootstrap-injection philosophy as OpenClaw's workspace files, renamed and trimmed to what a coding agent actually needs. `AGENTS.md` keeps its standard name deliberately — it's becoming a cross-tool convention that other IDEs and agents already look for at project root. The rest live in `.wraith/` to keep the repo root clean.

| File | Location | Role |
|---|---|---|
| `AGENTS.md` | project root | operating rules — what the agent does |
| `SPIRIT.md` | `.wraith/` | personality — who the agent is |
| `GRIMOIRE.md` | `.wraith/` | tool & environment notes (docs, not permissions) |
| `SUMMONER.md` | `.wraith/` | context about the operator — name, stack, preferences |
| `ECHOES.md` | `.wraith/` | curated long-term memory — durable facts and decisions |
| `.wraith/memory/YYYY-MM-DD.md` | `.wraith/memory/` | daily working log, auto-appended |

Keep each file to one concern. Rules go in `AGENTS.md`, tone goes in `SPIRIT.md` — don't let them blur.

**Default `AGENTS.md` template** (write this to `templates/AGENTS.md`, used by `wraith init`):

```markdown
# AGENTS.md — Wraith Operating Rules

## Autonomy
Default mode is full autonomy: act first, report after. Do not ask
permission before reading, writing, editing, or deleting files, or
before running shell commands, unless `autonomy` is set to a stricter
mode in config. Never issue unsolicited warnings about actions already
authorized by running Wraith in this folder.

## Workflow
1. Plan briefly and silently — don't narrate every micro-step.
2. Act. Prefer a real tool call over describing what you would do.
3. Report only what changed, in as few lines as possible.

## Editing
Use edit_file (FIND/REPLACE) for existing files. Only use write_file
for brand-new files or a full rewrite that was explicitly requested.
Never invent a file path, API, or function you haven't verified with
a tool call — read or search first if unsure.

## Multi-file changes
When a change spans multiple files, make all the edits before
reporting back — not one file at a time with a status update after each.

## Immediate execution
If intent is unambiguous, execute now. Only ask a clarifying question
when proceeding would likely produce a completely wrong result.
```

**Default `SPIRIT.md` template:**

```markdown
# SPIRIT.md — Personality

Direct, efficient, low-ceremony. No filler acknowledgments, no
restating the request back before doing it, no closing "let me know
if you need anything else." Structured output over prose — tables and
bullet points beat paragraphs when either would do. Assume competence;
skip explanations of basic concepts unless asked.
```

---

## 8. CLI Surface

```
wraith                              # interactive REPL in current directory
wraith "<prompt>"  /  -p "<prompt>" # one-shot, non-interactive
wraith init                         # scaffold AGENTS.md + .wraith/ in cwd
wraith config get|set|unset <key>
wraith models list|set|status
wraith auth add <provider>
wraith sessions list|resume <id>|rm <id>
wraith doctor                       # node version, rg present?, git repo?, keys configured?
wraith --version / --help
```

## 9. In-Session Slash Commands

```
/help              show commands
/clear             clear the visible transcript (session log is kept)
/compact           summarize older turns to reclaim context
/model <name>      switch active model mid-session
/cost              token usage / estimated spend this session
/sessions          list recent sessions
/diff              show pending uncommitted changes
/undo              revert the last checkpoint via git
/exit  or  /quit
```

---

## 10. Autonomy & Safety Model

This is the one place Wraith deliberately diverges from most agent frameworks, per the brief: **no permission gate by default.** `autonomy: "full"` means no confirmation dialogs for writes, edits, deletes, or shell execution, period.

The one thing to build anyway, because it costs nothing and only adds recoverability: **auto-checkpoint.** Before any filesystem-mutating tool call, if the working directory is a git repo, silently stage and commit current state (`wraith: checkpoint before <op> <path>`). If it isn't a repo, offer to `git init` once on first run and remember the answer — don't ask again. If `git` isn't installed or the check fails, skip silently. This isn't a permission gate — it never blocks or slows the tool call, it just means `/undo` and `git log` always work.

`confirm-destructive` and `confirm-all` exist in the config schema for anyone who wants guardrails later, but they are not the default and nothing in the build should assume they're on.

---

## 11. Performance Budget

Target hardware is modest — assume 8GB RAM, an older CPU, no GPU to lean on. Design accordingly:

- Minimal dependency tree. Every dependency is a startup-time and install-time cost — justify each one.
- Lazy-load provider SDKs; never eagerly import all three.
- Stream everything — tokens render as they arrive, tool output streams live, never buffer-then-print.
- No synchronous fs calls in any hot path.
- ripgrep for search, not a JS directory walk.
- Sub-second cold start to first prompt is the target; measure it and treat regressions as bugs.

---

## 12. Explicitly Out of Scope

Do not build any of the following. If asked to reconsider mid-build, don't — these are deliberate cuts, not omissions:

- Gateway daemon / WebSocket server / any persistent background process
- Multi-channel chat integrations (Discord, Slack, Telegram, WhatsApp, iMessage, Signal, etc.)
- Web control UI or any browser-based dashboard
- Mobile companion apps or "node" devices
- ACP/IDE bridge
- Pre-built third-party skill marketplace (Gmail, Twitter, Spotify, smart-home, Obsidian, etc.) — a minimal local skill-loading mechanism is fine if trivial, but don't build or stub any of these integrations
- Cron/heartbeat scheduling — this needs a persistent daemon, which is out of scope
- Telemetry, analytics, or any phone-home behavior

---

## 13. Packaging

- `package.json` `bin` field pointing at `bin/wraith.js`; `npm link` or `npm install -g .` for local global install during development.
- Pure Node/TS, no native-module dependencies, so it installs cleanly on Linux, in containers, and in restricted sandboxes (e.g. Codespaces) without a compile step.
- Before any public release, check OpenClaw's actual license terms for naming/attribution requirements, since Wraith is derivative in spirit even though the code is original.

---

## 14. Build Order

Build in this order, and show a file tree + short plan before generating code for each phase:

1. Scaffold: `package.json`, `tsconfig.json`, folder structure, `bin/wraith.js` entry.
2. Config system (§6) + provider adapters (§5) — get one provider (start with the `openai-compatible` adapter against Groq) actually streaming text end to end before anything else.
3. Tool registry + all tools in §4, tested individually against a scratch directory.
4. Agent loop (§3) wiring tools into the streaming loop, with the git auto-checkpoint (§10).
5. CLI commands (§8) and REPL (§9).
6. UI/theme layer (§1) — banner, glyphs, palette, diff rendering, spinner.
7. `wraith init` writing the template files from §7.
8. Smoke test: fresh scratch folder → `wraith init` → ask it to create a file, edit a line in it, run a shell command, and show a diff — confirm streaming output and the purple theme render correctly end to end.

---

## 15. Acceptance Checklist

- [ ] `wraith` with no args drops into a REPL in the current directory, banner renders in purple, no gateway process starts
- [ ] A one-shot `wraith "create a file called test.txt with hello world"` works non-interactively
- [ ] `edit_file` fails loudly on an ambiguous (non-unique) match instead of guessing
- [ ] `run_shell` streams output live and supports `background: true` for a long-running process
- [ ] Deleting a file or running a destructive shell command requires zero confirmation
- [ ] A git checkpoint commit exists after each mutating tool call, in a repo that has git
- [ ] Switching provider is a config edit or `wraith models set`, no code touched
- [ ] Cold start to first prompt is fast — no noticeable hang on a modest machine
- [ ] No open ports, no background process left running after the CLI exits
