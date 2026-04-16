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
- Each round MUST have at least one expansive role AND one constraining role.
- Round 1: maximise divergence. Final round: stress-test for production readiness.
- Synthesise new roles when the problem demands it. They are first-class.
- Tell the user which roles you picked and why.

**Match roles to the problem type.** The pool covers many perspectives — pick the ones that matter for this problem, don't default to the same set every time:

| Problem type | Include in Round 1 | Why |
|---|---|---|
| User request or feature spec | Assumption challenger, first principles thinker | User requests encode assumptions about the solution. Challenge the frame before optimising within it. |
| Architecture or technology choice | Future-state thinker, systems architect | Decisions that compound over time need temporal perspective. |
| Build-vs-buy, investment, prioritisation | Cost analyst, opportunity cost thinker | Economic lens is often the deciding factor. |
| Greenfield or high-uncertainty | Uncertainty mapper, experiment designer | Identify what you don't know before committing to a direction. |
| Multi-team, API, or org-wide change | Stakeholder analyst, downstream impact thinker | Who else is affected? Decisions made in isolation break at boundaries. |
| Process change, tool rollout, adoption | Adoption realist, change management thinker | The best solution fails if nobody can operate it. |
| Code-focused (bugs, refactoring, features) | Pragmatic engineer, minimalist | Stay in solution/implementation space. |

### Step 2: Gather constraints (Round 1 only)

Before the first round, ask the user:

> **Before we start — any constraints, prior decisions, or ruled-out approaches I should give agents as context?** For example: approaches already tried, known technical constraints, decisions already made. Type "none" to start immediately.

If the user provides input, include it in every agent's spawn prompt as a `<constraints>` block (see Appendix B). This is ground truth, not steering toward a solution — it prevents agents from wasting time on dead ends you've already explored.

If `--auto` is set, skip this step.

### Step 3: Create team and spawn agents

1. Print: `[Round {n}/{total}] Spawning {agent count} agents: {role list}`
2. `TeamCreate` with `team_name: "brainstorm-r{n}"`
3. Spawn each agent using the Agent tool with `team_name: "brainstorm-r{n}"`, a descriptive `name`, `run_in_background: true`, and the spawn prompt from Appendix B. Include any user constraints and between-round steering in the prompt.

Do not summarise or analyse the codebase before spawning. Agents can read code themselves if needed.

### Step 4: Broadcast ideation

`SendMessage` to `"*"`:

*"Begin ideation. Think independently about the problem through your specific lens, then post your ideas in a single message to all teammates using the [I{n}] format. Do NOT read other agents' messages until you've posted your own ideas."*

Wait for all agents to post. If one hasn't posted after the rest complete, proceed and note the non-response.

**Compliance check:** Scan posts for `[I{n}]` tags. If missing, send one correction: *"Your response is missing the required [I{n}] format markers. Please repost."* Proceed after one attempt regardless.

### Step 5: Broadcast challenge & vote

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

### Step 6: Convergence

1. Send shutdown request to all agents: `SendMessage` with `message: {type: "shutdown_request"}`
2. Tally external votes per idea (no self-votes). Threshold: ≥2 votes when agents ≥ 3, ≥1 vote when 2 agents.
3. Apply `--keep` cap. If more ideas survive than the `--keep` value (default 8), keep only the top `--keep` by vote count. Break ties by preferring unchallenged ideas.
4. **Rescue** up to 2 ideas cut by role-structural opposition rather than evidence. Mark as `[lead-rescued]`.
5. Record cut ideas with vote counts and strongest objections.

### Step 7: Write handoff files

Write `brainstorm-r{n}-results.md` and `brainstorm-r{n}-transcript.md` using the formats in Appendix C.

**Self-check:** Verify results file contains: surviving idea IDs with titles, vote counts (X/Y format), unresolved dissent section, cut ideas with reasons. Fix any gaps now.

Print: `[Status] round={n} agents_responded={n/m} ideas_proposed={count} ideas_surviving={count} ideas_cut={count}`

Print: `[Round {n}] Convergence complete. {survived} ideas survive, {cut} cut. {Starting Round {n+1} | Moving to final output}.`

### Step 8: Tear down, present, and steer

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

**Between-round steering (if rounds remain).** After presenting the summary, ask:

> **Any corrections or context before the next round?** For example: "drop S3, we tried that last quarter and it failed because X" or "agents are assuming Y but actually Z." Type "continue" to proceed without changes.

If the user provides input, include it in the next round's agent spawn prompts as a `<steering>` block (see Appendix B). Steering provides constraints and ground truth — it rules out dead ends and corrects false assumptions. It does not direct agents toward a preferred solution.

If `--auto` is set, skip the steering prompt and continue.

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

**Problem space** — challenge whether the problem is correctly framed:

| Role | Perspective |
|------|-------------|
| **Assumption challenger** | What are we taking as given that might be wrong? False premises, historical accidents mistaken for requirements, solutions looking for problems |
| **First principles thinker** | Strip away the existing implementation. What does the user actually need? Rebuild from the ground up, ignoring what exists today |

**Solution space** — what could we build, what should we not:

| Role | Perspective |
|------|-------------|
| **Creative inventor** | Novel combinations, lateral thinking, "what if we did X instead of the obvious thing" |
| **Systems architect** | Integration points, scalability, emergent behaviour, component boundaries, data flow |
| **Domain specialist** | Industry patterns, prior art, how others have solved this, standards and conventions |
| **Minimalist** | What to remove, YAGNI, smallest viable version, complexity budget, "do we even need this" |
| **Devil's advocate** | Hidden assumptions, failure cascades, second-order effects, "yes but what about..." |
| **Day-1 shipper** | MVP scope, cut aggressively, ship and iterate, what's good-enough-for-now |

**Implementation space** — how should we build it:

| Role | Perspective |
|------|-------------|
| **Pragmatic engineer** | Simplest working solution, build cost, what's maintainable long-term |
| **Security hardener** | Threat model, attack surface, trust boundaries, data exposure, abuse cases |
| **Ops engineer** | Deployment, monitoring, maintenance burden, incident response, observability, toil |
| **Performance engineer** | Efficiency, bottlenecks, resource constraints, scaling limits, cost at volume |
| **API designer** | Interface contracts, extensibility, backward compatibility, developer ergonomics |
| **End user advocate** | Usability, friction, accessibility, what real people actually do vs. what we assume |

**Impact space** — who else is affected and how:

| Role | Perspective |
|------|-------------|
| **Stakeholder analyst** | Who has power, interest, or dependency on this decision? Whose needs are we not hearing? |
| **Downstream impact thinker** | What does this change for the teams, processes, or systems that depend on what we're changing? |

**Temporal space** — what does this look like over time:

| Role | Perspective |
|------|-------------|
| **Future-state thinker** | What does the 6-month maintenance burden look like? Will this still make sense when the team is 3x larger? |
| **Technical debt forecaster** | What shortcuts are we taking now, what will they cost later, and is that trade-off explicit? |

**Economic space** — what does this cost and who pays:

| Role | Perspective |
|------|-------------|
| **Cost analyst** | Budget constraints, unit economics, total cost of ownership, cost at scale |
| **Opportunity cost thinker** | What are we NOT doing by choosing this? What's the cost of inaction vs. the cost of building wrong? |

**Uncertainty space** — what don't we know:

| Role | Perspective |
|------|-------------|
| **Uncertainty mapper** | What are we assuming to be true without evidence? Where should the decision be reversible? |
| **Experiment designer** | What's the cheapest way to validate our riskiest assumption before committing? |

**Adoption space** — can people actually execute this:

| Role | Perspective |
|------|-------------|
| **Adoption realist** | Does the team have the skills? Will the process actually be followed? Will the docs be maintained? |
| **Change management thinker** | How do we get from here to there? What resistance will this face and how do we address it? |

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
> {Include if user provided constraints in Step 2:}
> <constraints>
> The following are ground truth from the user — known facts, prior decisions, or ruled-out approaches. Do not propose ideas that contradict these.
> {user's constraints}
> </constraints>
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
> {Include if user provided steering after the previous round:}
> <steering>
> The user provided the following corrections or context after the last round. Treat these as ground truth:
> {user's steering input}
> </steering>
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
