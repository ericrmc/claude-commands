# Claude Agent Teams Pipeline Commands

Slash commands for structured multi-agent workflows in Claude Code. Inspired by [AgentCouncil](https://github.com/kiran-agentic/agentcouncil), built with commands instead of an MCP plugin.

Each run is self-improving. The pipeline brainstorms, implements, and reviews in a single pass — and re-running it on the same target progressively raises quality. Early runs surface high-severity structural issues, later runs find medium and low items, and eventually findings converge toward a well-rounded solution without manual steering. The pipeline successfully ran on itself to develop and verify the commands in this repo.

These commands guide Claude through structured protocols but are not deterministic — Claude interprets the instructions each run, so the exact execution path may vary. For stricter process guarantees, multiple LLM backends, or formal deliberation tracking, use [AgentCouncil](https://github.com/kiran-agentic/agentcouncil) directly.

| Command | What it does |
|---------|-------------|
| `/brainstorm` | Multi-round ideation with diverse agent roles. Produces ranked recommendations that have survived cross-challenge and voting. |
| `/review-fix` | Three independent reviewers find issues, then parallel developers fix them. Loops until clean. |
| `/pipeline` | Chains brainstorm → user approval → parallel implementation → review-fix. The full design-to-verified-code workflow. |

**Meta commands** — for developing and improving the commands themselves:

| Command | What it does |
|---------|-------------|
| `/meta/stress-test` | Structural audit of a command file. Finds where the mechanism breaks — parameter boundaries, rule conflicts, execution ergonomics. |
| `/meta/feedback` | Post-session reflection. Run after a brainstorm or pipeline session to surface execution friction and protocol gaps. |

---

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop app, or IDE extension)
- Agent Teams enabled (experimental feature — one config change, see below)

No plugins, extensions, or API keys beyond what Claude Code already uses.

---

## Setup

**1. Enable Agent Teams**

Open your Claude Code settings (`/config` or edit `~/.claude/settings.json`) and add the following to the `env` block:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**2. Install the commands**

Copy the `.md` files to your Claude Code commands directory:

```bash
cp brainstorm.md pipeline.md review-fix.md ~/.claude/commands/
cp -r meta/ ~/.claude/commands/meta/
```

**3. Use them**

In any Claude Code session, type the slash command:

```
/brainstorm how should we structure our API authentication layer?
/review-fix src/auth/
/pipeline add a rate limiting feature to the API
/meta/stress-test brainstorm.md
```

---

## Commands

### `/brainstorm <problem>`

Agents with distinct roles (creative inventor, pragmatic engineer, security hardener, minimalist, devil's advocate, etc.) independently propose ideas, then challenge and vote on each other's proposals. Runs multiple rounds with rotating roles — ideas that survive repeated cross-examination rise to the top. Produces a ranked recommendation document with full reasoning chains.

**Flags:**
- `--rounds N` — number of rounds (default: 4)
- `--agents N` — agents per round (default: 4)
- `--keep N` — max ideas carried forward per round (default: 8)
- `--output PATH` — output file path
- `--quick` — cost-sensitive preset: `--rounds 2 --agents 3`
- `--dry-run` — print the execution plan (agents, rounds, estimated tokens) and stop

---

### `/review-fix <target>`

Three reviewers independently analyse the target with lenses selected from a decision table (defaults: correctness, robustness, quality — overridden based on target characteristics like auth code, API surfaces, or data pipelines). You triage which findings to fix. Parallel developers implement fixes, verifiers confirm them. Repeats until clean or max iterations reached — with automatic escalation if the same finding keeps failing to resolve.

**Flags:**
- `--max-iterations N` — max review→fix loops (default: 3)
- `--auto` — auto-fix all medium+ findings without triage approval
- `--review-only` — stop after review, don't fix
- `--quick` — cost-sensitive preset: `--max-iterations 2`
- `--dry-run` — print the execution plan and stop

---

### `/pipeline <task>`

Chains `/brainstorm` and `/review-fix` around a parallel implementation phase. Brainstorm agents produce ranked design recommendations; the user approves one; developer agents implement it in parallel (with a conditional integration pass for cross-cutting interfaces); reviewer agents validate the result.

**Flags:**
- `--rounds N`, `--agents N` — passed to brainstorm phase
- `--max-review-iterations N` — passed to review-fix phase
- `--auto` — skip all approval gates
- `--skip-brainstorm` — go straight to implementation (supply a design description)
- `--from-brainstorm PATH` — skip brainstorm, use an existing brainstorm output file (e.g., from a previous `/brainstorm` session)
- `--skip-review` — stop after implementation
- `--resume` — resume from the most recent checkpoint in `.pipeline/`
- `--quick` — cost-sensitive preset: `--rounds 2 --agents 3 --max-review-iterations 2`
- `--dry-run` — print the full execution plan across all phases and stop

---

## Meta Commands

These are for developing and improving the command files themselves. The core commands (`/brainstorm`, `/review-fix`, `/pipeline`) do the work; the meta commands help you improve how that work gets done.

### `/meta/stress-test <target>`

Audits a command file's *structure* — parameter boundaries, rule conflicts, degenerate cases, execution ergonomics, and blind spots. Answers "where does this mechanism break?" rather than "are the ideas good?"

**When to use it:**
- After writing or significantly modifying a command file
- When a process produced surprising or low-quality results
- Periodically, as structural regression testing

**Flags:**
- `--output PATH` — output file path (default: `stress-test-output.md`)

### `/meta/feedback`

Post-session reflection. Run this after a `/brainstorm`, `/review-fix`, or `/pipeline` session to capture what worked, what was hard, and what should change — from the executor's perspective.

Brainstorm agents evaluate architecture theoretically. Stress-tests analyze mechanisms structurally. Neither captures the operational experience of actually running the protocol. This command fills that gap.

**When to use it:**
- After any session, especially early in a project when the commands are still being tuned
- When a session produced good results but felt harder than it should have
- When you want to propose changes to the command files based on real execution experience

Run it in the same conversation where the session happened — the execution context is already in your history.

**Flags:**
- `--output PATH` — output file path (default: `feedback-{session-type}-{date}.md`)

---

## How it works

Each command is a markdown file that Claude Code loads as a slash command. When invoked, Claude follows the protocol in the file, using the Agent Teams API to spawn and coordinate subagents. All state lives in the conversation and in local files (brainstorm output, `.pipeline/` checkpoints) — nothing is sent to external services beyond the normal Claude API calls.

Agent Teams is an experimental Claude Code feature that lets a lead agent spawn, message, and coordinate multiple subagents within the same session.

---

## Cost

These workflows are token-intensive by design — the value is in the diversity of perspectives. Each agent gets its own context window (one full agent conversation), plus ~4k tokens of protocol overhead per spawn.

| Command | Agent conversations (default settings) | Protocol overhead |
|---------|-------------|-------------|
| `/brainstorm` | ~14 (4/4/3/3 agents across 4 rounds) | ~70-80k input tokens |
| `/review-fix` | ~8 per iteration (3 reviewers + up to 4 developers + 2 verifiers) | ~25-30k input tokens |
| `/pipeline` | ~39 for a full run (brainstorm + implementation + review) | ~180-230k input tokens |
| `/meta/stress-test` | 1 (single agent, no teams) | ~2k input tokens |

Reduce cost with `--quick`, skip phases (`--skip-brainstorm`, `--skip-review`), or resume from a previous brainstorm (`--from-brainstorm`).

---

## Tips

- **Use commands independently.** `/brainstorm` alone is great for architecture decisions. `/review-fix` alone works on any existing code.
- **Checkpoints.** `/pipeline` writes phase checkpoints to `.pipeline/`. If a run fails, `--resume` picks up from the last completed phase.
- **Triage gate.** `/review-fix` pauses before fixing and lets you choose which findings to address.
- **Design rationale persists.** The brainstorm output file survives after implementation — useful for onboarding or understanding why a particular approach was chosen.
- **Re-run to converge.** Each pass catches what the previous one missed. Run `/pipeline` or `/review-fix` again on the same target and the severity of findings drops until there's nothing left worth fixing.
- **Improve the commands themselves.** Run `/meta/feedback` after sessions to capture execution friction. Run `/meta/stress-test` after modifying a command file to catch structural issues.
- **Early stopping.** If brainstorm rounds converge early (same ideas, no new dissent), remaining rounds are skipped automatically to save tokens.
- **Observability.** Each phase prints a `[Status]` line with agent response counts and context estimates. Use these to tune parameters and diagnose quality issues.

---

## Why the pipeline doesn't auto-loop

Within a single run, `/review-fix` already loops up to `--max-review-iterations` times with oscillation detection. Raise the default of 3 if needed.

Across runs, the pipeline is deliberately manual. Chained brainstorms have no memory of prior rounds, no convergence guarantee, and no rollback point without a commit between cycles. The intended workflow is: run, review the diff, commit, then re-run if another pass is worthwhile.
