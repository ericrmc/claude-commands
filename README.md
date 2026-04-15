# Claude Code Pipeline Commands

Three slash commands for structured multi-agent workflows in Claude Code:

| Command | What it does |
|---------|-------------|
| `/brainstorm` | Multi-round ideation with diverse agent teams. Each round: independent proposals, cross-challenge, convergence vote. Roles rotate to prevent groupthink. |
| `/build` | Full pipeline: brainstorm → user approves design → parallel implementation → review-fix loop. |
| `/review-fix` | Independent review by 3 agents (correctness, robustness, quality), triage, parallel fixes, verification. Loops until clean or max iterations reached. |

These commands were refined by running them on their own source. The original designs were enhanced through a `/build` pipeline run targeting the command files themselves: a 3-round `/brainstorm` produced 5 improvement recommendations, a parallel developer team implemented them across all three files, and a `/review-fix` loop validated and corrected the implementation — 22 findings found, 21 fixed and verified in a single iteration.

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

Copy the three `.md` files to your Claude Code commands directory:

```bash
cp brainstorm.md build.md review-fix.md ~/.claude/commands/
```

**3. Use them**

In any Claude Code session, type the slash command:

```
/brainstorm how should we structure our API authentication layer?
/review-fix src/auth/
/build add a rate limiting feature to the API
```

---

## Commands

### `/brainstorm <problem>`

Runs a configurable multi-round brainstorm with agents playing distinct roles (creative inventor, pragmatic engineer, security hardener, minimalist, devil's advocate, etc.). Each round: agents ideate independently, challenge each other's ideas, then vote — with self-voting excluded to prevent anchoring bias. Produces a ranked recommendation document.

**Flags:**
- `--rounds N` — number of rounds (default: 3)
- `--agents N` — agents per round (default: 4)
- `--keep N` — max ideas carried forward per round (default: 8)
- `--output PATH` — output file path

---

### `/review-fix <target>`

Spawns three independent reviewers (correctness, robustness, quality), each posting findings before reading the others'. Findings are triaged with the user, then fixed by parallel developer agents, then verified. Repeats until all findings are resolved or max iterations reached. Includes an oscillation circuit breaker that escalates to the user if the same finding fails to resolve in two consecutive iterations.

**Flags:**
- `--max-iterations N` — max review→fix loops (default: 3)
- `--auto` — auto-fix all medium+ findings without triage approval
- `--review-only` — stop after review, don't fix

---

### `/build <task>`

Chains `/brainstorm` and `/review-fix` around a parallel implementation phase. Brainstorm agents produce ranked design recommendations; the user approves one; developer agents implement it in parallel (with a conditional integration pass for cross-cutting interfaces); reviewer agents validate the result.

**Flags:**
- `--rounds N`, `--agents N` — passed to brainstorm phase
- `--max-review-iterations N` — passed to review-fix phase
- `--auto` — skip all approval gates
- `--skip-brainstorm` — go straight to implementation (supply a design description)
- `--skip-review` — stop after implementation
- `--resume` — resume from the most recent checkpoint in `.pipeline/`

---

## How it works

Each command is a markdown file that Claude Code loads as a slash command. When invoked, Claude follows the protocol in the file, using the Agent Teams API to spawn and coordinate subagents. All state lives in the conversation and in local files (brainstorm output, `.pipeline/` checkpoints) — nothing is sent to external services beyond the normal Claude API calls.

Agent Teams is an experimental Claude Code feature that lets a lead agent spawn, message, and coordinate multiple subagents within the same session.

---

## Cost

These workflows are token-intensive by design — the value is in the diversity of perspectives.

| Command | Default cost |
|---------|-------------|
| `/brainstorm` | ~12 context windows (3 rounds × 4 agents) |
| `/review-fix` | ~8 per iteration (3 reviewers + up to 4 developers + 2 verifiers) |
| `/build` | ~40 for a full default run |

For smaller problems: `--rounds 2 --agents 3`. For known designs: `--skip-brainstorm`.

---

## Tips

- **Use commands independently.** `/brainstorm` alone is great for architecture decisions. `/review-fix` alone works on any existing code.
- **Checkpoints.** `/build` writes phase checkpoints to `.pipeline/`. If a run fails partway through, `--resume` picks up from the last completed phase (re-confirming approval gates rather than trusting the checkpoint).
- **Triage gate.** `/review-fix` pauses before fixing and lets you choose which findings to address. Low-severity findings are surfaced but not automatically fixed.
- **The brainstorm document persists.** Even after implementation and review, the design rationale lives in the output file — useful for onboarding or understanding why a particular approach was chosen.
