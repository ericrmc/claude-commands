# Structured Brainstorm

Multi-round brainstorming with diverse agent teams. Each round: independent ideation, cross-challenge, convergence. Roles rotate between rounds. Ideas are refined and narrowed until the strongest recommendations survive.

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

## Execution

Repeat the following sequence for each round. Agent count tapers: Round 1 uses `{agents}`, Rounds 2+ use `{agents - 1}` (minimum 2).

### Step 1: Select roles

Pick `{agent count}` roles from the role pool (Appendix A). Rules:
- Never repeat a role from the previous round.
- Each round MUST have at least one expansive role (creative inventor, systems architect, domain specialist, first principles thinker) AND one constraining role (minimalist, security hardener, devil's advocate, assumption challenger, day-1 shipper).
- Round 1: maximise divergence. Final round: stress-test for production readiness.
- **For non-code problems** (architecture decisions, product direction, process design) or when the problem statement came from a user request: include assumption challenger or first principles thinker in Round 1. User requests encode assumptions about the solution — these roles challenge whether the problem is correctly framed before solution-space roles optimize within that frame. They pair well together: the assumption challenger clears false premises, the first principles thinker rebuilds from actual needs.
- Synthesise new roles when the problem demands it. They are first-class.
- Tell the user which roles you picked and why.

### Step 2: Create team and spawn agents

1. Print: `[Round {n}/{total}] Spawning {agent count} agents: {role list}`
2. `TeamCreate` with `team_name: "brainstorm-r{n}"`
3. Spawn each agent using the Agent tool with `team_name: "brainstorm-r{n}"`, a descriptive `name`, `run_in_background: true`, and the spawn prompt from Appendix B.

Do not summarise or analyse the codebase before spawning. Agents can read code themselves if needed.

### Step 3: Broadcast ideation

`SendMessage` to `"*"`:

*"Begin ideation. Think independently about the problem through your specific lens, then post your ideas in a single message to all teammates using the [I{n}] format. Do NOT read other agents' messages until you've posted your own ideas."*

Wait for all agents to post. If one hasn't posted after the rest complete, proceed and note the non-response.

**Compliance check:** Scan posts for `[I{n}]` tags. If missing, send one correction: *"Your response is missing the required [I{n}] format markers. Please repost."* Proceed after one attempt regardless.

### Step 4: Broadcast challenge & vote

Print: `[Round {n}] All {agent count} agents posted ideas. Starting challenge-and-vote phase.`

`SendMessage` to `"*"`:

> *"All ideas are posted. Read every idea from every agent. In a SINGLE message, do both:*
>
> *1. Challenge: stress-test each idea with concrete failure modes.*
> *2. Vote: which ideas should survive? One-line justification each. You may NOT vote for your own ideas. Vote only for ideas proposed by other agents.*
>
> *Challenge labels: STRONG [In], WEAK [In], MODIFY [In], MERGE [Ia]+[Ib]*
> *Vote format: VOTE [In] — {justification}*
>
> *50 words max per challenge. Everything in ONE message."*

If agents post follow-ups, broadcast: *"Finalise your positions — we're moving to convergence."*

**Compliance check:** Scan for STRONG/WEAK/MODIFY/MERGE labels. If missing, send one correction. If still unformatted, manually extract what you can and note it.

### Step 5: Convergence

1. Send shutdown request to all agents: `SendMessage` with `message: {type: "shutdown_request"}`
2. Tally external votes per idea (no self-votes). Threshold: ≥2 votes when agents ≥ 3, ≥1 vote when 2 agents.
3. Apply `--keep` cap. If more ideas survive than the `--keep` value (default 8), keep only the top `--keep` by vote count. Break ties by preferring unchallenged ideas.
4. **Rescue** up to 2 ideas cut by role-structural opposition rather than evidence. Mark as `[lead-rescued]`.
5. Record cut ideas with vote counts and strongest objections.

### Step 6: Write handoff files

Write `brainstorm-r{n}-results.md` and `brainstorm-r{n}-transcript.md` using the formats in Appendix C.

**Self-check:** Verify results file contains: surviving idea IDs with titles, vote counts (X/Y format), unresolved dissent section, cut ideas with reasons. Fix any gaps now.

Print: `[Status] round={n} agents_responded={n/m} ideas_proposed={count} ideas_surviving={count} ideas_cut={count}`

Print: `[Round {n}] Convergence complete. {survived} ideas survive, {cut} cut. {Starting Round {n+1} | Moving to final output}.`

### Step 7: Tear down and present

1. `TeamDelete` to clean up `brainstorm-r{n}`.
2. Present between-rounds summary to user:

```
## Round {n} Complete

Surviving ideas ({count}):
{[ID] title — Support: X/Y | Dissent: {role} ("{concern}") or "none"}

Cut:
{[ID] title — reason}

Next round roles: {selections and reasoning}

Full transcript: brainstorm-r{n}-transcript.md
```

**Early stopping:** If the surviving idea set is unchanged from the previous round (same ideas, no new dissent), skip remaining rounds and proceed to final synthesis. Print: `[Status] early_stop=true reason=converged round={n} remaining_rounds_skipped={count}`

**Idle notifications:** Treat the first idle notification after a substantive message as "agent is done." Ignore repeats.

---

## After All Rounds

### Synthesise

Read all transcript files and the final results file. Transcripts contain the reasoning chains that compact results files omit. Use them to write a high-quality final output.

### Write output file

Write to `{output path}` using the format in Appendix D.

### Present to user

Show:
- Top 3-5 recommendations with one-line descriptions
- Output file path
- Ask if they want to proceed with implementation or stop here

---

## Appendix A: Role Pool

| Role | Perspective |
|------|-------------|
| **Creative inventor** | Novel combinations, lateral thinking, "what if we did X instead of the obvious thing" |
| **Pragmatic engineer** | Simplest working solution, build cost, what's maintainable long-term |
| **Security hardener** | Threat model, attack surface, trust boundaries, data exposure, abuse cases |
| **End user advocate** | Usability, friction, accessibility, what real people actually do vs. what we assume |
| **Systems architect** | Integration points, scalability, emergent behaviour, component boundaries, data flow |
| **Minimalist** | What to remove, YAGNI, smallest viable version, complexity budget, "do we even need this" |
| **Domain specialist** | Industry patterns, prior art, how others have solved this, standards and conventions |
| **Assumption challenger** | What are we taking as given that might be wrong? False premises in the problem statement, historical accidents mistaken for requirements, solutions looking for problems |
| **First principles thinker** | Strip away the existing implementation. What does the user actually need to accomplish? Rebuild from the ground up, ignoring what exists today |
| **Devil's advocate** | Hidden assumptions, failure cascades, second-order effects, "yes but what about..." |
| **Ops engineer** | Deployment, monitoring, maintenance burden, incident response, observability, toil |
| **Performance engineer** | Efficiency, bottlenecks, resource constraints, scaling limits, cost at volume |
| **API designer** | Interface contracts, extensibility, backward compatibility, developer ergonomics |
| **Day-1 shipper** | MVP scope, cut aggressively, ship and iterate, what's good-enough-for-now |

Synthesise new roles when the problem demands it. Give each a name and one-line perspective. They carry the same weight as pool roles.

---

## Appendix B: Agent Spawn Prompt

> You are a brainstorming agent. Think about this problem through your specific lens and propose ideas.
>
> **Problem:** {PROBLEM}
>
> **Your role:** {ROLE}
> **Your perspective:** {PERSPECTIVE}
>
> {For Round 1:}
> This is the first round. No prior ideas — start fresh.
>
> {For Rounds 2+:}
> Ideas surviving from previous rounds (already challenged and defended):
> - [{ID}] {title} — {description}. Dissent: {concern or "none"}.
>
> You may: build on these, propose modifications, argue to cut, or propose new ideas.
>
> **Post your ideas** in a single message to all teammates. For each:
> ```
> [I{n}] {title}
> What: {2-3 sentences — what it is and how it works}
> Why: {1 sentence — why worth doing from your perspective}
> Risk: {1 sentence — biggest weakness you see}
> ```
>
> 100 words max per idea. Round 1: propose 3-5. Later rounds: new ideas and/or modifications.
> Mark your strongest idea with `[LEAD]`.
>
> After posting, **stop**. Do NOT read other agents' messages. You will receive a new message when the challenge phase begins.

---

## Appendix C: Handoff File Formats

### Results file (`brainstorm-rN-results.md`)

```markdown
# Round N Results

## Surviving Ideas
- [S1] Title — one-line description. Support: X/Y votes. Key challenge weathered: ...
- [S2] [lead-rescued] Title — description. Support: X/Y votes. Rescued because: ...

## Cut Ideas
- [C1] Title — reason (strongest objection)

## Unresolved Dissent
- Concern — raised by role, relevant if: condition

## Role Selection Notes for Next Round
- Suggested roles and reasoning
```

Required fields: surviving idea IDs/titles, one-line descriptions, vote counts, unresolved dissent section (even if "None."), cut ideas with reasons. The role selection notes section is optional.

### Transcript file (`brainstorm-rN-transcript.md`)

```markdown
# Round N Transcript

## Roles: [list]

## Phase A: Ideas
### [Agent Name] — [Role]
[Full text of ideas as posted]

## Phase B: Challenge & Vote
### [Agent Name] — [Role]
[Full text of challenges and votes as posted]

## Lead Notes
[Observations: non-responsive agents, consensus patterns, merge proposals]
```

---

## Appendix D: Final Output Format

```markdown
# Brainstorm: {problem title}

**Date:** {date}
**Rounds:** {n}
**Agents per round:** {agent count}
**Total ideas proposed:** {count across all rounds}
**Final recommendations:** {count}

---

## Recommendations

Ranked by strength of support across all rounds.

### 1. {title}

**Support:** {count} | **First proposed:** Round {n} | **Survived:** {n} rounds

{Full description}

**Strengths:**
- {list}

**Risks and mitigations:**
- {risk} — {how addressed or why acceptable}

**Implementation notes:**
{Concrete suggestions from agents}

**Key challenges weathered:**
- Round {n}: {challenge and defense}

---

## Honourable Mentions

### {title}
**Cut in round:** {n} | **Peak support:** {count}
**Why it was cut:** {strongest objection}
**Why it's worth noting:** {what scenario makes it relevant}

---

## Dissenting Views

- {concern} — raised by {role}, Round {n}. Relevant if: {condition}.

---

## Process Log

### Round 1
**Roles:** {list}
**Ideas proposed:** {count}
**Ideas carried forward:** {count}
**Key debate:** {one-line summary}
```

---

## Notes

- **Don't pre-summarise the codebase.** Agents can read code themselves if they need to.
- **Diversity is the mechanism.** Similar roles collapse into groupthink. The value is in different perspectives colliding.
- **History carries forward.** Later rounds see what survived via results files. They don't need full transcripts.
- **Idea count trajectory.** Round 1: 12-20 proposals → 6-9 survivors. Final round: 3-5 strong recommendations. More than 8 after the final round = convergence too loose.
- **Token economics.** Default (3 rounds, 4/3/3 agents) = ~10 agent context windows. Each agent receives ~4k tokens of protocol and context. Full default brainstorm: ~50-60k input tokens on protocol overhead. Use `--rounds 2 --agents 3` for cost-sensitive runs.
- **Intermediate files.** Per-round results and transcript files are working files for synthesis. Clean up after final output is written.
- **Non-compliance is accepted degraded behavior.** If an agent fails format compliance after one correction, proceed with reduced diversity. Extract what you can, note it in lead notes. The `[Status]` line shows `agents_responded=N/M`.
