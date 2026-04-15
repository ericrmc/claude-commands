# Structured Brainstorm

Multi-round brainstorming with diverse agent teams. Each round: independent ideation, cross-challenge, convergence. Roles rotate between rounds to bring fresh perspectives. Ideas are refined and narrowed each round until the strongest recommendations survive.

**Problem:** $ARGUMENTS

---

## Preflight

1. Check that Agent Teams is enabled. If you cannot create an agent team, tell the user to add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` block in their settings.json, then stop.
2. Parse the problem statement from the arguments above. If none specified, ask the user what to brainstorm.
3. Parse optional flags:
   - `--rounds N` — brainstorm rounds (default: 3)
   - `--keep N` — max ideas to carry forward per round (default: 8)
   - `--agents N` — agents per round (default: 4)
   - `--output PATH` — output file path (default: `brainstorm-output.md` in working directory)

---

## Role Pool

Select agents from this pool each round. Each role thinks about problems in a fundamentally different way.

| Role | Perspective |
|------|-------------|
| **Creative inventor** | Novel combinations, lateral thinking, "what if we did X instead of the obvious thing" |
| **Pragmatic engineer** | Simplest working solution, build cost, what's maintainable long-term |
| **Security hardener** | Threat model, attack surface, trust boundaries, data exposure, abuse cases |
| **End user advocate** | Usability, friction, accessibility, what real people actually do vs. what we assume |
| **Systems architect** | Integration points, scalability, emergent behaviour, component boundaries, data flow |
| **Minimalist** | What to remove, YAGNI, smallest viable version, complexity budget, "do we even need this" |
| **Domain specialist** | Industry patterns, prior art, how others have solved this, standards and conventions |
| **Devil's advocate** | Hidden assumptions, failure cascades, second-order effects, "yes but what about..." |
| **Ops engineer** | Deployment, monitoring, maintenance burden, incident response, observability, toil |
| **Performance engineer** | Efficiency, bottlenecks, resource constraints, scaling limits, cost at volume |
| **API designer** | Interface contracts, extensibility, backward compatibility, developer ergonomics |
| **Day-1 shipper** | MVP scope, cut aggressively, ship and iterate, what's good-enough-for-now |

### Role selection rules

Before each round, select {agents} roles following these rules:

1. **Never repeat the same role in consecutive rounds.** Fresh eyes each time.
2. **Balance expansive and constraining roles.** Each round MUST include at least one expansive role (creative inventor, systems architect, domain specialist) AND at least one constraining role (minimalist, security hardener, devil's advocate, day-1 shipper). Without this balance, rounds either generate fluff or kill everything.
3. **Adapt to the problem.** If the problem is security-focused, include security in Round 1 but bring in usability in Round 2 to balance. If it's a greenfield design, front-load creative roles. If it's an optimisation problem, bring in performance early.
4. **Shift toward refinement in later rounds.** Round 1 should maximise divergence. Final round should stress-test for production readiness (ops, performance, API design).
5. **State your selections.** Before spawning each round, tell the user which roles you picked and why.

---

## Round Structure

Repeat this structure for each round. Between rounds, the surviving ideas carry forward with their challenge history.

### Phase A: DIVERGE

Before spawning, emit a status line: `[Round {n}/{total}] Spawning {agent count} agents: {role list}`

Spawn {agents} teammates with the selected roles.

#### Agent spawn prompt template

Use this template for every agent. Replace `{PROBLEM}`, `{ROLE}`, `{PERSPECTIVE}`, and the round-specific context block.

> You are a brainstorming agent. Your job is to think about this problem through a specific lens and propose ideas.
>
> **Problem:** {PROBLEM}
>
> **Your role:** {ROLE}
> **Your perspective:** {PERSPECTIVE}
>
> {ROUND CONTEXT — include one of the following blocks:}
>
> **For Round 1:**
> This is the first round. There are no prior ideas — start fresh.
>
> **For Rounds 2+:**
> Ideas surviving from previous rounds (these have already been challenged and defended):
>
> {For each surviving idea:}
> - [{ID}] {title} — {one-line description}. Supported by {vote count} agents. Survived challenges: {key objections it weathered}.
>
> You may: build on these, propose modifications, argue to keep or cut, or propose entirely new ideas.
>
> {END ROUND CONTEXT}
>
> ### Protocol — follow exactly
>
> **Step 1 — Think independently.** Consider the problem through your lens. DO NOT read messages from other teammates yet. Take your time.
>
> **Step 2 — Post your ideas.** Message all teammates. For each idea:
>
> ```
> [I{n}] {title}
> What: {2-3 sentences — what it is and how it works}
> Why: {why this is worth doing, from your perspective}
> Risk: {the biggest risk or weakness you see in your own idea}
> ```
>
> Round 1 agents: propose 3-5 ideas.
> Later round agents: propose new ideas AND/OR argue for modifications to surviving ones. You may also argue to cut a surviving idea if you think it's weak.
>
> **Step 3 — Wait for the challenge phase.** Do NOT respond to other agents yet. Wait for the lead's signal.

If any agent has not posted after the others have all completed, proceed with the agents that did respond and note the non-response.

### Phase B: CHALLENGE

Once all agents have posted their ideas, emit a status line: `[Round {n}] All {agent count} agents posted ideas. Starting challenge phase.`

Then broadcast:

> *"All ideas are posted. Challenge phase: read every idea from every agent. Stress-test them. Your job is to find weaknesses and poke holes — but be specific and fair. A good challenge identifies a concrete failure mode, not a vague concern."*

Agents respond to each other's ideas using:
- **STRONG [In]** — this holds up, here's why: {evidence or reasoning}
- **WEAK [In]** — this has a specific problem: {the problem and why it matters}
- **MODIFY [In]** — this would be stronger if: {specific change and why}
- **MERGE [Ia]+[Ib]** — these two ideas combine well: {how and why}

Allow 1-2 rounds of back-and-forth. If agents start repeating themselves, broadcast: *"Finalise your positions — we're moving to convergence."*

### Phase C: CONVERGE

Broadcast:

> *"Challenge phase complete. Each agent: vote on which ideas should survive to the next round. Post your votes with a one-line justification for each. Vote for as many as you genuinely believe deserve to continue — don't vote strategically. **You may not vote for ideas you originally proposed in this round.** Vote only for ideas proposed by other agents. Voting for your own ideas is not permitted — this prevents anchoring bias from inflating your proposals' survival chances."*

After all agents post their votes (if any agent has not posted votes after the others have all completed, proceed with available votes and note the non-response):

1. **Shut down all agents and clean up the team.**
2. **Count support.** For each idea, tally how many agents voted for it.
3. **Survival threshold: majority support (external votes only).** Self-votes are not permitted. If an agent proposed a MERGE [Ia]+[Ib], they may not vote for either constituent idea, but may vote for other merged combinations they did not propose. With 4 agents, each idea can receive at most 3 external votes. With 3 agents, each idea can receive at most 2 external votes (so survival requires 2/2 — effectively unanimous among external voters; the minority-idea preservation rule in step 5 catches genuinely strong ideas that fall just short). With 2 agents, each idea can receive at most 1 external vote; an idea needs 1 external vote (unanimous agreement) to survive — the minority-idea preservation rule in step 5 still applies. When agents ≥ 3, the majority threshold is ≥2 external votes. When only 2 agents are in a round, the threshold is ≥1 external vote. An idea survives if it meets this threshold. This self-calibrates — if 3 ideas are genuinely strong, 3 survive; if 9 are, 9 survive.
4. **Apply a soft cap of 10.** If more than 10 ideas clear the threshold (agents were too generous), keep only the top 10 by vote count. Break ties by preferring ideas that survived challenges without needing modification.
5. **Preserve one strong minority idea.** If exactly one idea fell just below threshold but its justifications were substantive and it raises a concern no surviving idea covers, carry it forward marked as `[minority]`. Do not use this to rescue weak ideas.
6. **Record cuts.** For each cut idea, record the vote count and the strongest objection against it.

The natural trajectory with these rules: Round 1 typically produces 6-9 survivors from 12-20 proposals. Round 2's harder roles cut further to 4-6. The final round should yield 3-5 strong recommendations. If you end with more than 8 after the final round, the convergence was too loose — trim by applying stricter majority (e.g., require 3 out of 4 votes) and re-cut.

After tallying votes and applying the survival rules, emit a status line: `[Round {n}] Convergence complete. {survived} ideas survive, {cut} cut. {next action}.`

Where `{next action}` is either "Starting Round {n+1}." or "Moving to final output." for the last round.

### Between rounds

Before starting the next round, present to the user:

```
## Round {n} Complete

Surviving ideas ({idea count}):
{for each: [ID] title — Support: {vote count}/{total} external votes | Dissent: {agent role name} ("{one-line concern}") or "none"}

Cut:
{for each: [ID] title — reason (strongest objection)}

Next round roles: {selected roles and one-line reason for each selection}
```

Dissent is any WEAK or MODIFY challenge that was not fully addressed during the challenge phase. `{agent role name}` is the role that raised the concern (e.g., "minimalist"). If no agent raised unresolved concerns, show "Dissent: none". Always reference the full output file path so the user can review complete agent transcripts rather than relying solely on this summary.

---

## After All Rounds

### Write output file

Write the full brainstorm results to `{output path}`:

```markdown
# Brainstorm: {problem title}

**Date:** {date}
**Rounds:** {n}
**Agents per round:** {agent count}
**Total ideas proposed:** {idea count across all rounds}
**Final recommendations:** {idea count}

---

## Recommendations

Ranked by strength of support across all rounds.

### 1. {title}

**Support:** {count across final round} | **First proposed:** Round {n} | **Survived:** {n} rounds

{Full description — what it is and how it works}

**Strengths:**
- {list}

**Risks and mitigations:**
- {risk} — {how it was addressed during challenges, or why it's acceptable}

**Implementation notes:**
{Any concrete suggestions that emerged from agents — structure, tools, sequencing}

**Key challenges weathered:**
- Round {n}: {challenge and defense}

---

### 2. {title}
...

---

## Honourable Mentions

Ideas that were cut but had notable support or raised interesting angles worth remembering.

### {title}
**Cut in round:** {n} | **Peak support:** {vote count}
**Why it was cut:** {strongest objection}
**Why it's worth noting:** {what was interesting or what scenario would make it relevant}

---

## Dissenting Views

Minority positions that didn't win consensus but raised valid concerns the implementation should account for. Include only concerns that were raised as WEAK or MODIFY challenges and were not fully resolved during cross-challenge — same criteria as the dissent shown in between-round summaries.

- {concern} — raised by {agent role name}, Round {n}. Relevant if: {condition}.

---

## Process Log

### Round 1
**Roles:** {list}
**Ideas proposed:** {idea count}
**Ideas carried forward:** {idea count}
**Key debate:** {one-line summary of the most substantive disagreement}

### Round 2
...
```

### Present summary to user

Show:
- The top 3-5 recommendations with one-line descriptions
- The output file path for full details
- Ask if they want to proceed with implementation or if this was brainstorm-only

---

## Notes

- **Diversity is the mechanism.** If you pick 4 similar roles, the brainstorm collapses into groupthink. The value comes from genuinely different perspectives colliding. Review your role selections critically.
- **History carries forward.** Later rounds MUST see what survived and what was challenged. Without this, agents re-propose cut ideas or miss known weaknesses. Include the surviving ideas and their challenge history in the spawn prompt.
- **Idea count trajectory.** Round 1 with 4 agents × 3-5 ideas = 12-20 raw ideas. Converging to 8 cuts ~50-60%. By the final round you should have 3-5 strong, well-tested recommendations. If you end with more than 8, your convergence is too loose. If you end with fewer than 3, it's too aggressive.
- **Token cost.** {rounds} rounds × {agents} agents = {rounds × agents} context windows. Default is 12. This is token-intensive by design — the value is in the diversity of perspectives. For simpler problems, use `--rounds 2 --agents 3`.
- **Not just for code.** This skill works for architecture decisions, product direction, process design, or any problem that benefits from structured multi-perspective analysis. The output file serves as a design rationale document regardless of whether implementation follows.
