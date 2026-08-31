# Grok Build driver for Verity — design notes

Adds `grok` as a third provider beside `claude` and `codex` in
[verity-framework](https://github.com/seanerama/verity-framework), following the
registry design in the framework's ADR-0005: a driver module under
`verity/bin/lib/agents/`, a one-line registry entry, an installer host pass, and
doctor rows. `agent-exec.cjs` itself is untouched — exactly as the registry's header
comment promises ("Adding a runtime is a registry entry plus a driver plus fixtures").

## Why the driver is small

Grok Build was verified against the docs shipped in
[xai-org/grok-build](https://github.com/xai-org/grok-build)
(`crates/codegen/xai-grok-pager/docs/user-guide/`, Grok Build ≥ 1.0.0). Two design
choices by xAI make Verity integration nearly free:

1. **`--output-format streaming-messages-json` emits the Messages API stream-json
   wire format** — `assistant` lines whose `message.content[]` holds
   `text`/`thinking`/`tool_use` blocks, `user` lines carrying `tool_result`, and a
   terminal `{"type":"result","subtype":"success","is_error":false,…,"usage":{…},
   "total_cost_usd":…}` line. That is the same grammar Verity's reference (Claude)
   driver already parses, so `parseTranscript`, `countToolCalls`, `normalizeUsage`,
   and `normalizeResult` carry over with only two Grok-specific touches:
   - `input_tokens` is documented as *uncached-only*; summing it with both cache
     buckets (the reference driver's formula) is exactly the documented full-prompt
     identity.
   - max-turns exhaustion can surface as `stop_reason: "max_turn_requests"` in
     addition to the Claude-style `subtype: "error_max_turns"`; both map to the same
     operator-facing message.
   - Grok *omits* fields it has no real data for instead of zero-filling; the parsers
     key only on `type`, so omission costs nothing (and `total_cost_usd` absent means
     UNKNOWN → `est_usd: null`, per the framework's ADR-0008 — conveniently, Grok
     omits **all** cost floats when cost is partial, so a present value is a complete
     bill).

2. **`--allow` permission rules use the same `ToolPrefix(pattern)` grammar as
   Verity's T06 allowlists**, and Grok's matcher explicitly accepts Claude's
   `Bash(cmd:*)` prefix form. Expressible T06 entries therefore travel
   **verbatim**, one repeatable `--allow` flag per entry (see the rule-vocabulary
   projection below for the one exception). The driver never passes `--yolo` /
   `--always-approve` / `--permission-mode`: headless runs cannot prompt, so a tool
   call no rule covers fails closed — deny-by-default stays honest, which also lets
   the local-substrate narrowing (ADR-0029: strip `Bash(gh …)`/`WebFetch`/`WebSearch`
   on `--substrate local`) work identically to the reference driver.

## The invocation

```
grok -p "<rendered role prompt>" --verbatim \
     --output-format streaming-messages-json \
     --max-turns N [--model M] \
     --allow <entry> [--allow <entry> …]
```

`--verbatim` keeps the rendered role text from being re-interpreted as
slash-command/attachment syntax — the prompt *is* the program.

Binary resolution: `VERITY_GROK_BIN` → `VERITY_AGENT_BIN` (the test seam) → `grok`
on PATH. Version floor: `verity.grokBuildMinVersion` (1.0.0 — Grok Build's v1.0 left
beta 2026-08-07 with this headless flag surface) via the shared `doctor.checkBinary`
probe. Transcripts are `<role>.grok.jsonl` (the codex convention — never clobbers a
Claude transcript of the same role).

## Interactive install (`verity install --grok`)

Grok Build discovers flat `*.md` files under `~/.grok/commands/` as user-invocable
slash commands (filename stem = command name — Claude's legacy layout; see the
user guide's skills chapter). The `grok` host pass therefore mirrors the OpenCode
flattening: roles install as `verity-<role>.md` (invoked `/verity-<role>`),
frontmatter reduced to `description:`, cross-role handoffs rewritten
`/verity:x → /verity-x`, and the engine fallback path moved to
`${GROK_HOME:-$HOME/.grok}/verity`. The paired `.tools.json` allowlists are copied
beside the commands under the same flattened stem so the pair never separates.

(Fun fact: Grok Build also scans `~/.claude/commands/` for Claude compatibility by
default, so an existing `verity install --claude` is already partially visible in
the Grok TUI — but the dedicated host gets correct handoff names, engine paths, and
install state, and lets `verity doctor --agent grok` reason about the box.)

## Doctor

`verity doctor --agent grok` probes git + gh + `grok --version` (no claude/codex row
— a Grok-only machine can be green; remediation is the official installer,
`curl -fsSL https://x.ai/cli/install.sh | bash`) plus three environment rows:
flattened command discovery, the engine fallback, and install state. `resolveAgent`
also learns `~/.grok` (honoring `GROK_HOME`) as an install-state root, so a box where
only `verity install --grok` ran selects grok without a flag.

## Host matrix position

Same tier as Codex's initial landing: **interactive + headless (`agent-exec`)
supported; autonomy worker/Actions selection deferred** — the worker's provider
policy is a separate, deliberate step (the framework gates worker-selectable
providers on trust tiers).

## The rule-vocabulary projection (a bug the source review caught)

The original driver passed T06 entries to `--allow` unconditionally. Reading Grok
Build's Rust sources showed that is wrong in one specific way: **headless mode
parses `--allow`/`--deny` strictly** (`xai-grok-pager/src/headless.rs`,
`parse_permission_rules_strict`) — a single unrecognized rule prefix aborts the
whole invocation — and the rule parser's vocabulary
(`xai-grok-workspace/src/permission/rules.rs`, `tool_name_to_filter`) is exactly
`Bash`, `Read`, `Edit`/`Write` (folded), `Grep`/`Glob` (folded), `MCPTool`,
`WebFetch`, `WebSearch`, `AgentMessage` aliases, plus bare `mcp__…` spellings.
Verity's packaged allowlists include `Task`, which is not in that vocabulary, so
every role granting `Task` would have failed to launch.

The driver therefore projects each allowlist onto that verified vocabulary before
building argv (`projectAllowlist`): expressible entries travel byte-identical,
inexpressible ones are dropped — narrowing only, since an uncovered call then
fails closed at runtime rather than being allowed — and a list that projects to
nothing refuses the dispatch (`unenforceable-policy`), never launching rule-less.
Dropping `Task` costs nothing: Grok subagents are not permission-gated (spawning
is governed by `--disallowed-tools Agent`, which the driver never passes).

## Validation

- Both fail-closed behaviors the docs implied are now **verified against Grok
  Build's source** (xai-org/grok-build @ v1.x main): headless answers un-ruled
  permission requests with `Cancelled` outside yolo mode (`headless.rs`,
  `RequestPermission` handling) — refused, no hang, no auto-approve — and the
  strict/lenient parser split plus prefix vocabulary above.
- `tests/agents-grok.test.cjs` — 27 tests in the framework's characterization
  style: registry, env precedence, version gate, exact argv (including the
  never-auto-approve invariant), the vocabulary projection (verbatim keeps,
  `Task` dropped end-to-end at the process boundary, empty projection refused),
  allowlist errors, substrate narrowing, wire-shape parsing/counting/normalization
  (including omitted-field tolerance and the `max_turn_requests` spelling), CLI
  end-to-end over a stub binary, host pass, installer layout, and doctor rows.
- Full upstream suite green with the driver added: **1195 passed, 0 failed**
  (baseline before changes: 1168 passed); `npm run lint` (Biome CI) clean.

## Live-binary confirmation

Smoke-run against a live authenticated `grok` release binary (2026-08-31,
`verity agent-exec build 7 --run-id smoke-1 --agent grok --max-turns 10 --json`):

```json
{"schema":1,"role":"build","outcome":"failed","tokens":{"in":356903,"out":3941},
 "est_usd":0.05248172,"wall_secs":111,"tool_calls":28,"artifacts":{},
 "error":"max turns (10) exhausted"}
```

The failed outcome is the 10-turn ceiling, deliberately low for a smoke run —
what it proves is the driver surface: the release binary accepted the full argv
including the projected `--allow` rules (28 tool calls ran, so the permission
grants work), the streaming-messages-json transcript parsed, usage normalized,
Grok stamped a real cost that surfaced as `est_usd`, and max-turns exhaustion
mapped to the contract's error string. The one path not exercised live is the
success-path result marker (the role never finished within the ceiling); that
code is byte-identical to the reference driver's. The stamped cost also matters
for autonomy later: the worker's budget breaker gates on unknown cost "until a
provider's cost accounting is proven" — for this login, it is.
