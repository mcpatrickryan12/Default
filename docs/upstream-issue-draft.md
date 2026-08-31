# Draft issue for seanerama/verity-framework

Copy-paste everything below the line into a new issue at
<https://github.com/seanerama/verity-framework/issues/new>.

---

**Title:** Grok Build (xAI `grok` CLI) as a fourth host — working driver + tests offered

**Body:**

## What

I'd like to contribute Grok Build support to Verity: xAI's `grok` CLI
([xai-org/grok-build](https://github.com/xai-org/grok-build), Apache 2.0, v1.0 since
2026-08-07) as a provider beside `claude` and `codex`, with interactive install and
headless `agent-exec` at the same tier Codex first landed at (autonomy-worker
selection deferred).

Per CONTRIBUTING's "describe or sketch the change in an issue" — this one comes with
the change already built and tested against v1.3.0 (base `fe74312`), on a branch of
my fork:

- **Branch / diff:** <https://github.com/seanerama/verity-framework/compare/main...mcpatrickryan12:verity-framework:add-grok-build-provider>
- **Design notes:** <https://github.com/mcpatrickryan12/Default/blob/claude/verity-grok-build-hku1hr/docs/grok-build-driver.md>

Happy to reshape any of it for the development repo's conventions, split it, open a
reference PR, or hand it over as raw material — whatever fits the promotion pipeline
best.

## Why it's a small change

Two Grok Build design choices make the driver nearly a sibling of the reference
driver (verified against the user guide shipped in xai-org/grok-build):

1. `--output-format streaming-messages-json` emits the Messages stream-json wire
   format — `assistant` lines with `text`/`thinking`/`tool_use` blocks and a terminal
   `type:"result"` object — so `parseTranscript` / `countToolCalls` /
   `normalizeUsage` / `normalizeResult` carry over. Grok-specific touches:
   `input_tokens` is uncached-only (the reference formula is exactly the documented
   full-prompt identity), max-turns can surface as `stop_reason:
   "max_turn_requests"`, and omitted cost floats mean UNKNOWN → `est_usd: null`
   (ADR-0008 holds for free: Grok omits *all* cost floats when cost is partial).
2. `--allow` permission rules use the same `ToolPrefix(pattern)` grammar as the T06
   allowlists, and Grok's matcher explicitly accepts the `Bash(cmd:*)` prefix form —
   entries travel verbatim, one repeatable `--allow` per entry. The driver never
   passes an auto-approve flag, so headless stays deny-by-default and ADR-0029
   local-substrate narrowing applies unchanged.

## Shape of the change

ADR-0005 registry discipline — `agent-exec.cjs` untouched:

- `verity/bin/lib/agents/grok.cjs` (the driver) + a registry entry
- `verity install --grok`: flat `verity-<role>.md` under `~/.grok/commands`
  (`GROK_HOME` honored; Grok discovers Claude's legacy commands layout), handoffs
  flattened `/verity:x → /verity-x`, paired `.tools.json` copied beside, engine +
  install state
- `verity doctor --agent grok`: git + gh + `grok` registry (no claude row — a
  Grok-only box can be green) + three environment rows; `~/.grok` as an
  install-state selection root
- `verity.grokBuildMinVersion: "1.0.0"` in package.json
- `tests/agents-grok.test.cjs`: 27 characterization tests (exact argv incl. a
  never-auto-approve invariant, the rule-vocabulary projection end-to-end,
  wire-shape parsing with omitted-field tolerance, CLI end-to-end over a stub
  binary, host pass, installer layout, doctor rows), plus the two stage-8
  registry pins in `tests/agents.test.cjs` generalized for a third provider

`npm test` on the branch: **1195 passed, 0 failed** (v1.3.0 baseline 1168), and
`npm run lint` (Biome CI) is clean.

## Verified against Grok Build's source, not just its docs

Two behaviors were confirmed by reading xai-org/grok-build's Rust implementation:

- **Un-ruled tool calls fail closed headless.** Outside yolo mode, headless
  answers permission requests with `Cancelled` (`xai-grok-pager/src/headless.rs`)
  — refused, no hang, no auto-approve — so the driver's deny-by-default posture
  (never passing an auto-approve flag) is enforced by Grok itself.
- **Headless rule parsing is strict, and its vocabulary is finite.** One
  unrecognized `--allow` prefix aborts the invocation
  (`parse_permission_rules_strict`); the accepted prefixes
  (`permission/rules.rs`) are Bash, Read, Edit/Write, Grep/Glob, MCPTool,
  WebFetch, WebSearch, AgentMessage, plus `mcp__…` spellings. Verity's packaged
  allowlists include `Task`, which is not among them — so the driver projects
  each T06 list onto that vocabulary before argv construction: expressible
  entries travel byte-identical, `Task` is dropped (Grok subagents are not
  permission-gated; spawning is governed by `--disallowed-tools Agent`, never
  passed), and a list projecting to nothing refuses the dispatch
  (`unenforceable-policy`) rather than launching rule-less.

## Remaining caveat

Not yet exercised against a live authenticated `grok` binary (subscription-gated
login) — the residual risk is a release binary diverging from its own public
source. I can run that one smoke confirmation once pointed at a live login.

If you'd rather receive this as a mailed patch or a reference PR instead of the
branch link, say the word.
