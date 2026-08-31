# Verity × Grok Build — provider driver + local-agents plan

This repo carries two deliverables:

1. **A Grok Build provider driver for [Verity](https://github.com/seanerama/verity-framework)** —
   making Verity's roles run on xAI's `grok` CLI beside Claude Code, Codex CLI, and
   OpenCode. Shipped as a git patch against upstream (ready to apply or open as a PR),
   plus standalone copies of the new files.
2. **A researched plan for running Verity's workflow on local hardware**
   (DGX Spark + QNAP TS-1677X + two server towers), answering the GrokBots-vs-Hermes
   question. Short version: GrokBots can't be a Verity runtime (cloud coworker bots, no
   CLI contract); Hermes agents are the right always-on local layer but not the coding
   runtime; Grok Build/Claude Code stay the role runtimes and Verity's own worker +
   gate runners move onto your boxes. Details in
   [docs/local-agents-plan.md](docs/local-agents-plan.md).

## Layout

| Path | Contents |
|---|---|
| `patches/0001-add-grok-build-provider-driver.patch` | The full change against `seanerama/verity-framework` main (base commit `fe74312`, v1.3.0) |
| `verity-grok/agents/grok.cjs` | The driver (standalone copy) |
| `verity-grok/tests/agents-grok.test.cjs` | Its 25-test characterization suite (standalone copy) |
| `docs/grok-build-driver.md` | Design notes: wire-format mapping, permission model, install layout, what still needs a live-binary check |
| `docs/local-agents-plan.md` | GrokBots vs. Hermes analysis + what-runs-where topology and sequencing |

## Using it

```bash
git clone https://github.com/seanerama/verity-framework
cd verity-framework
git am /path/to/patches/0001-add-grok-build-provider-driver.patch
npm test          # full suite incl. the new grok tests
npm i -g .        # or npm pack / link

verity install --grok          # roles → ~/.grok/commands/verity-<role>.md
verity doctor --agent grok     # git + gh + grok + environment rows
verity agent-exec build 7 --run-id r1 --agent grok --max-turns 40 --json
```

The driver targets Grok Build ≥ 1.0.0 (`curl -fsSL https://x.ai/cli/install.sh | bash`;
requires a SuperGrok or X Premium+ subscription).

## Status

- Verified against the documentation shipped in
  [xai-org/grok-build](https://github.com/xai-org/grok-build) (v1.x user guide);
  upstream test suite green with the driver added.
- Not yet exercised against a live authenticated `grok` binary — the two behaviors to
  confirm on first real run (headless deny fallback; `--allow` acceptance of every
  packaged allowlist spelling) are listed at the end of
  [docs/grok-build-driver.md](docs/grok-build-driver.md), and both are isolated inside
  the driver.
