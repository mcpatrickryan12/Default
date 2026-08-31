# Running Verity's workflow on local hardware — GrokBots vs. Hermes vs. a hybrid

This is the companion research doc to the Grok Build driver in this repo. It answers:
*"Should the local agent layer be GrokBots or Hermes agents, and what runs where on a
DGX Spark + QNAP TS-1677X + two server towers?"*

## TL;DR recommendation

**Neither GrokBots nor Hermes replaces Verity's role runtimes — they solve a different
problem.** Verity's roles (vision, architect, build, review, verify, …) are invocations
of a *headless coding-agent CLI* with a strict contract: rendered prompt in, tool
allowlist enforced, JSONL transcript out, parseable result object, exit codes. Claude
Code, Codex CLI, and (with the driver in this repo) Grok Build satisfy that contract.

- **GrokBots do not fit that contract and are not local.** Each Bot is a cloud
  "coworker" on a dedicated xAI-hosted VM, driven by chat messages, priced
  $120–200+/month, with no headless CLI, no tool-permission flags, no transcript, and
  no exit-code semantics. There is nothing for Verity's `agent-exec` to drive. If you
  want xAI in the loop, the right integration point is **Grok Build** (which is exactly
  what the driver here adds) — same models, real CLI contract, runs on your machines.

- **Hermes Agent (Nous Research) is the right shape for the local, always-on layer** —
  open source, self-hosted, one process per "profile" (own config, identity, SQLite
  memory, cron jobs), A2A messaging, signed webhooks, and it can point at any
  OpenAI-compatible endpoint, including a model served on your own DGX Spark. But it is
  a general autonomous-agent framework, not a coding-agent CLI: it has no
  `-p / --output-format / --allow` surface either, so it also can't be a Verity role
  runtime without writing a much more speculative driver.

- **So the winning architecture is a hybrid:**
  1. Verity roles keep running on coding-agent CLIs — Claude Code and/or Grok Build —
     executed **locally on your own boxes** (the CLIs are local; only inference is
     remote).
  2. Verity's own **autonomy worker** and **gate runners** move onto your hardware —
     Verity already supports `gate_runner: localhost` (Docker + nektos/act) and
     `remote:<name>` runners over SSH, which is a direct fit for the two towers.
  3. **Hermes profiles** become the always-on operations layer around the workflow:
     watching STATUS/deployments, running the cron heartbeat, triaging failures, and
     messaging you — the places where persistent memory and 24/7 presence pay off and
     frontier coding ability doesn't matter.
  4. The **DGX Spark serves an open-weight model** (e.g. a Hermes-family or other
     strong open model via vLLM/SGLang) for the Hermes profiles and for any Verity
     role you deliberately point at local inference.

Running locally is *not* all-or-nothing: keep frontier cloud models on the roles where
capability compounds (build, review, architect) and use local inference where volume
and privacy matter more than peak capability (verify loops, monitoring, triage,
summarization). You can revisit per-role as open models improve.

## Why not GrokBots as the Verity runtime

| Verity needs (agent-exec contract) | GrokBots provide |
|---|---|
| Headless CLI: `bin -p <prompt> …` | Chat/message-driven cloud coworker |
| Tool allowlist enforced per role (T06, deny-by-default) | Bot signs into your tools with its own credentials |
| JSONL transcript + terminal result object | Conversational history in xAI's cloud |
| Exit codes, `--max-turns`, timeouts | No scriptable process boundary |
| Runs on your hardware next to the checkout | Dedicated xAI cloud VM |

GrokBots are genuinely useful as a *personal ops assistant* (email triage, scheduling,
web tasks) — but that sits beside your dev workflow, not inside it. For Verity + xAI,
Grok Build is the integration point, and it's open source (Apache 2.0,
[xai-org/grok-build](https://github.com/xai-org/grok-build)).

## Why Hermes for the always-on layer (and only that layer)

Hermes Agent ([NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent))
gives you, per profile: `config.yaml`, an identity document, SQLite-backed persistent
memory, its own gateway process and cron definitions, five deployment backends (local,
Docker, SSH, Singularity, Modal) with container hardening, A2A v1.0 for agent-to-agent
messaging, and signed outbound webhooks. That maps cleanly onto the parts of Verity's
Operate arc that want *presence* rather than peak coding ability:

- an **operator profile** that runs the heartbeat: polls `verity status` / STATUS.md,
  watches the live app the way the verify role does between releases, and pings you
  (webhook → phone) when something drifts;
- a **triage profile** that reads failed gate logs and opens/labels GitHub issues in
  Verity's vocabulary so the autonomy worker picks them up;
- optionally a **release-notes/comms profile** with memory of the project's history.

These profiles talk to a local model on the Spark, so the 24/7 loop costs electricity,
not tokens.

What Hermes should **not** do (yet): be the builder/reviewer. A local open-weight model
driving multi-hour repo mutations will burn your time in review; the Verity pipeline's
whole premise is that "done means proven," and today frontier models clear the gates in
dramatically fewer cycles.

## What runs where

```
┌─────────────────────────────────────────────────────────────────────┐
│ Tower 1 (64GB)                     Tower 2 (64GB)                   │
│  Verity autonomy worker             Verity gate runner              │
│  (verity-worker, supervised)        (Docker + nektos/act)           │
│  Claude Code / Grok Build CLIs      registered as gate_runner:      │
│  role execution (tier-1 local)      remote:tower2 over SSH          │
├─────────────────────────────────────────────────────────────────────┤
│ DGX Spark (128GB unified)                                           │
│  vLLM / SGLang serving an open-weight model (OpenAI-compatible API) │
│  → consumed by Hermes profiles + any role you point local           │
├─────────────────────────────────────────────────────────────────────┤
│ QNAP TS-1677X (40GB)                                                │
│  Hermes profiles (Docker): operator / triage / comms                │
│  Git mirror + artifact/backup store; webhook receiver; dashboards   │
└─────────────────────────────────────────────────────────────────────┘
            ↕ GitHub remains Verity's single source of truth
            ↕ Cloud inference (Anthropic / xAI) for build & review roles
```

Concrete Verity hooks that make this real, all already in the framework:

- `verity install --grok` / `--claude` on the towers; `verity doctor --agent grok`
  verifies each box.
- `.verity/autonomy.yml` → `verity-worker` on Tower 1 with the kill switch and
  supervised trust level to start.
- `gate_runner: remote:tower2` in the deployment-methods catalog: Verity probes
  Docker + act over SSH (`verity doctor` renders the four remote rows).
- `--substrate local` for fully-offline stretches: roles run against a bare git
  sibling with gh/network grants stripped from the allowlist automatically, syncing
  back to GitHub when you're ready.

## Sequencing

1. **Now**: apply the Grok Build driver (this repo) to your Verity install; run one
   role headless via `verity agent-exec build … --agent grok` next to your existing
   Claude runs and compare gate pass-rates and cost per merged PR.
2. **Week 1–2**: move the autonomy worker + gate runner onto the towers
   (`gate_runner: localhost` first, then `remote:` from your laptop).
3. **Week 2–4**: stand up vLLM on the Spark + one Hermes operator profile on the QNAP
   pointed at it; wire its webhook to your phone.
4. **Later**: experiment with pointing low-stakes Verity roles (verify, docs) at the
   Spark's endpoint and measure before promoting anything else.
