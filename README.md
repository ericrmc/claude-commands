# Claude Agent Teams Pipeline Commands

Slash commands for structured multi-agent workflows in Claude Code. Inspired by [AgentCouncil](https://github.com/kiran-agentic/agentcouncil), built with commands instead of an MCP plugin.

Each run is self-improving. The pipeline brainstorms, implements, and reviews in a single pass — and re-running it on the same target progressively raises quality. Early runs surface high-severity structural issues, later runs find medium and low items, and eventually findings converge toward a well-rounded solution without manual steering. The pipeline successfully ran on itself to develop and verify the commands in this repo.

These commands guide Claude through structured protocols but are not deterministic — Claude interprets the instructions each run, so the exact execution path may vary. For stricter process guarantees, multiple LLM backends, or formal deliberation tracking, use [AgentCouncil](https://github.com/kiran-agentic/agentcouncil) directly.

| Command | What it does |
|---------|-------------|
| `/brainstorm` | Multi-round ideation with diverse agent roles. Produces ranked recommendations that have survived cross-challenge and voting. |
| `/review-fix` | Three independent reviewers find issues, then parallel developers fix them. Loops until clean. |
| `/pipeline` | Chains brainstorm → user approval → parallel implementation → review-fix. The full design-to-verified-code workflow. |
| `/meta/stress-test` | Structural audit of a command file or process design. Finds where the mechanism breaks, not whether the ideas are good. |

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
- `--rounds N` — number of rounds (default: 3)
- `--agents N` — agents per round (default: 4)
- `--keep N` — max ideas carried forward per round (default: 8)
- `--output PATH` — output file path

---

### `/review-fix <target>`

Three reviewers (correctness, robustness, quality) independently analyse the target, then cross-review each other's findings. You triage which findings to fix. Parallel developers implement fixes, verifiers confirm them. Repeats until clean or max iterations reached — with automatic escalation if the same finding keeps failing to resolve.

**Flags:**
- `--max-iterations N` — max review→fix loops (default: 3)
- `--auto` — auto-fix all medium+ findings without triage approval
- `--review-only` — stop after review, don't fix

---

### `/meta/stress-test <target>`

Reads a command file or process description and audits its *structure* — parameter boundaries, rule conflicts, degenerate cases, and blind spots. This is not a content review; it answers "where does this mechanism break?" rather than "are these ideas good?"

**When to use it:**
- After writing or significantly modifying a command file, before relying on it in production
- When a process produced surprising or low-quality results and you're not sure if the problem was the input or the mechanism
- When you've added configurable parameters (`--rounds`, `--agents`, etc.) and want to verify the design holds across the full parameter space
- Periodically, as a form of structural regression testing — the process may have accumulated assumptions that no longer hold

**Flags:**
- `--output PATH` — output file path (default: `stress-test-output.md`)

---

### `/pipeline <task>`

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

These workflows are token-intensive by design — the value is in the diversity of perspectives. Each agent gets its own context window (one full agent conversation).

| Command | Agent conversations (default settings) |
|---------|-------------|
| `/brainstorm` | ~13 (3 round leads + ~10 agents across rounds) |
| `/review-fix` | ~8 per iteration (3 reviewers + up to 4 developers + 2 verifiers) |
| `/pipeline` | ~40 for a full run (brainstorm + implementation + review) |
| `/meta/stress-test` | 1 (single agent, no teams) |

Reduce cost with `--rounds 2 --agents 3`, or skip phases: `--skip-brainstorm`, `--skip-review`.

---

## Tips

- **Use commands independently.** `/brainstorm` alone is great for architecture decisions. `/review-fix` alone works on any existing code.
- **Checkpoints.** `/pipeline` writes phase checkpoints to `.pipeline/`. If a run fails, `--resume` picks up from the last completed phase.
- **Triage gate.** `/review-fix` pauses before fixing and lets you choose which findings to address.
- **Design rationale persists.** The brainstorm output file survives after implementation — useful for onboarding or understanding why a particular approach was chosen.
- **Stress-test your commands.** Agents inside a process can challenge each other's ideas but cannot challenge the process itself — they operate inside the frame the prompt sets. `/meta/stress-test` operates outside that frame. Run it after writing or changing a command file to catch structural issues before they surface as bad output.
- **Re-run to converge.** Each pass catches what the previous one missed. Run `/pipeline` or `/review-fix` again on the same target and the severity of findings drops until there's nothing left worth fixing.
