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

## Roles

Roles are the diversity mechanism. Each agent gets a role that defines how it thinks about the problem. The reference pool below covers common perspectives, but it is not exhaustive — you may and should synthesise new roles when the problem demands it.

### Reference pool

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

### Synthesising new roles

When the problem has a domain or dimension not well covered by the reference pool, create a role for it. Give it a name and a one-line perspective description in the same format as the table above. Examples:

- **Cost analyst** — Budget constraints, unit economics, cost-at-scale, "can we afford this?"
- **Regulatory specialist** — Compliance requirements, legal exposure, audit trails, data sovereignty
- **Accessibility engineer** — Screen readers, motor impairment, cognitive load, WCAG compliance

Synthesised roles are first-class — they follow the same selection rules and carry the same weight as reference pool roles.

### Role selection rules

Before each round, select {agents} roles following these rules:

1. **Never repeat the same role in consecutive rounds.** A role used in Round N cannot appear in Round N+1. Re-use after a gap is fine — a "creative inventor" in Round 1 and Round 3 sees completely different idea landscapes.
2. **Balance expansive and constraining roles.** Each round MUST include at least one expansive role (creative inventor, systems architect, domain specialist, or equivalent) AND at least one constraining role (minimalist, security hardener, devil's advocate, day-1 shipper, or equivalent). Without this balance, rounds either generate fluff or kill everything.
3. **Adapt to the problem.** If the problem is security-focused, include security in Round 1 but bring in usability in Round 2 to balance. If it's a greenfield design, front-load creative roles. If it's an optimisation problem, bring in performance early. Synthesise roles when the reference pool doesn't cover the problem's key dimensions.
4. **Shift toward refinement in later rounds.** Round 1 should maximise divergence. Final round should stress-test for production readiness (ops, performance, API design).
5. **State your selections.** Before spawning each round, tell the user which roles you picked and why. For synthesised roles, include the one-line perspective.

---

## Agent Count Tapering

The default `--agents N` value applies to Round 1. Later rounds use fewer agents since convergence strengthens and diminishing returns set in:

- **Round 1:** {agents} agents (maximum divergence)
- **Round 2:** {agents - 1} agents (minimum 2)
- **Round 3+:** {agents - 1} agents (minimum 2)

For example, with the default of 4: Round 1 uses 4, Rounds 2+ use 3. With `--agents 3`: Round 1 uses 3, Rounds 2+ use 2.

---

## Round Structure

Each round is orchestrated directly by the main orchestrator. The orchestrator creates a team, spawns agents, broadcasts instructions, collects responses, tallies votes, writes handoff files, and cleans up the team.

### Two-file handoff per round

Each round produces two files:

1. **`brainstorm-rN-results.md`** (compact, structured) — what the next round reads as input. ~1-2k tokens.
2. **`brainstorm-rN-transcript.md`** (detailed, append-only) — full agent ideas, challenges, and votes. Reference archive for final synthesis.

**Required fields for downstream consumption.** Every results file MUST include all of the following. If any are missing, the next round cannot reliably consume the handoff:

- **Surviving idea IDs and titles** — each surviving idea must have a unique `[S{n}]` tag and a title.
- **One-line descriptions** — every surviving idea needs a description, not just a title.
- **Vote counts** — `Support: X/Y votes` for each surviving idea. The next round uses these to gauge strength.
- **Unresolved dissent** — the `## Unresolved Dissent` section must be present even if empty (write "None." if no dissent). Later rounds use this to assign scrutiny.
- **Cut ideas with reasons** — the `## Cut Ideas` section must list every idea that didn't survive with its strongest objection.

The `## Role Selection Notes for Next Round` section is optional but recommended. All other sections above are mandatory.

**Results file format:**
```markdown
# Round N Results

## Surviving Ideas
- [S1] Title — one-line description. Support: X/Y votes. Key challenge weathered: ...
- [S2] ...
- [S3] [lead-rescued] Title — one-line description. Support: X/Y votes. Rescued because: ...

## Cut Ideas
- [C1] Title — reason (strongest objection)

## Unresolved Dissent
- Concern — raised by role, relevant if: condition

## Role Selection Notes for Next Round
- Suggested roles and reasoning
```

**Transcript file format:**
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
[Any observations the orchestrator made during the round — non-responsive agents,
 consensus patterns, merge proposals that emerged organically]
```

### Orchestration flow

```
Main orchestrator (per round):
  1. TeamCreate brainstorm-r{n}
  2. Spawn {agents} brainstorm agents into team (Agent with team_name="brainstorm-r{n}")
  3. Broadcast ideation instructions via SendMessage to="*"
  4. Wait for all agents to post ideas
  5. Compliance check on [I{n}] tags
  6. Broadcast challenge-and-vote instructions via SendMessage to="*"
  7. Wait for all agents to post challenges and votes
  8. Compliance check on STRONG/WEAK/MODIFY/MERGE labels
  9. Send shutdown requests to all agents
  10. Tally votes, determine survivors, write brainstorm-r{n}-results.md + brainstorm-r{n}-transcript.md
  11. Self-check: verify results file contains all required fields before proceeding
  12. TeamDelete brainstorm-r{n}
  13. Print between-rounds summary, proceed to next round
```

The file-based handoff (`brainstorm-r{n}-results.md`) is the only communication channel between rounds. The main orchestrator passes previous round results to the next round's agents via their spawn prompts.

---

## Running a Round

### Set up

Before each round, the main orchestrator:

1. Selects roles for this round (see Role Selection Rules above).
2. Emits a status line: `[Round {n}/{total}] Spawning {agent count} agents: {role list}`
3. Creates the team: `TeamCreate` with `team_name: "brainstorm-r{n}"`.
4. Spawns each brainstorm agent into the team using the Agent tool with `team_name: "brainstorm-r{n}"` and a descriptive `name` (e.g., `"creative-inventor"`, `"minimalist"`). Use the agent spawn prompt template below. Spawn agents with `run_in_background: true`.

**Do not summarise or analyse the codebase before spawning agents.** The problem statement and any surviving ideas from previous rounds are sufficient context. Reading and summarising the codebase dilutes the prompt with potentially inaccurate information and wastes context window. If agents need to understand specific code to brainstorm effectively, they can read it themselves.

### Phase A: DIVERGE

After all agents are spawned, broadcast to all agents (`SendMessage` with `to: "*"`) to begin ideation:

*"Begin ideation. Think independently about the problem through your specific lens, then post your ideas in a single message to all teammates using the [I{n}] format. Do NOT read other agents' messages until you've posted your own ideas."*

#### Agent spawn prompt template

The main orchestrator uses this template when spawning each brainstorm agent. Replace `{PROBLEM}`, `{ROLE}`, `{PERSPECTIVE}`, and the round-specific context block.

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
> - [{ID}] {title} — {one-line description}. Dissent: {unresolved concern or "none"}.
>
> You may: build on these, propose modifications, argue to keep or cut, or propose entirely new ideas.
>
> {END ROUND CONTEXT}
>
> ### Protocol — follow exactly
>
> **Step 1 — Think independently.** Consider the problem through your lens. DO NOT read messages from other teammates yet. Take your time.
>
> **Step 2 — Post your ideas.** Message all teammates in a **single message**. For each idea:
>
> ```
> [I{n}] {title}
> What: {2-3 sentences max — what it is and how it works}
> Why: {1 sentence — why this is worth doing, from your perspective}
> Risk: {1 sentence — the biggest risk or weakness you see in your own idea}
> ```
>
> **Length limits:** 100 words max per idea. Round 1 agents: propose 3-5 ideas. Later round agents: propose new ideas AND/OR argue for modifications to surviving ones. You may also argue to cut a surviving idea if you think it's weak.
>
> **Lead proposal:** Mark your strongest idea with `[LEAD]` after the idea tag (e.g., `[I2] [LEAD] Title`). This signals to other agents where to focus scrutiny during the challenge phase. Mark exactly one idea as lead.
>
> **Step 3 — Stop.** Your task for this phase is complete. Do NOT read or respond to other agents' messages. You will receive a new message from the lead when the challenge-and-vote phase begins.

Wait for all agents to post their ideas. If any agent has not posted after the others have all completed, proceed with the agents that did respond and note the non-response.

#### Compliance check

Before moving to Phase B, scan each agent's post for `[I{n}]` tags. If any agent's post is missing the required format markers, send a one-shot correction: *"Your response is missing the required [I{n}] format markers. Please repost your ideas using the format specified in your instructions."* Proceed after one correction attempt regardless — do not loop. If an agent cannot repost (idle or unresponsive), note the non-compliance and proceed with available formatted posts.

### Phase B: CHALLENGE & VOTE (combined)

Once all agents have posted their ideas, emit a status line: `[Round {n}] All {agent count} agents posted ideas. Starting challenge-and-vote phase.`

Then broadcast (`SendMessage` with `to: "*"`):

> *"All ideas are posted. Read every idea from every agent. In a SINGLE message, do both:*
>
> *1. Challenge: stress-test each idea. Be specific and fair — identify concrete failure modes, not vague concerns.*
>
> *2. Vote: state which ideas should survive to the next round with a one-line justification for each. Vote for as many as you genuinely believe deserve to continue — don't vote strategically. **You may not vote for ideas you originally proposed in this round.** Vote only for ideas proposed by other agents.*
>
> *Use these labels for challenges:*
> - *STRONG [In] — this holds up, here's why: {evidence}*
> - *WEAK [In] — specific problem: {the problem}*
> - *MODIFY [In] — would be stronger if: {specific change}*
> - *MERGE [Ia]+[Ib] — these combine well: {how}*
>
> *Then list your votes:*
> - *VOTE [In] — {one-line justification}*
>
> *50 words max per challenge assessment. Post everything in ONE message."*

If agents start posting follow-ups, broadcast: *"Finalise your positions — we're moving to convergence."* Allow at most one round of brief back-and-forth.

#### Compliance check

Before moving to convergence, scan each agent's challenge-and-vote post for STRONG/WEAK/MODIFY/MERGE labels. Also check for VOTE lines — but note that an agent may legitimately post only challenge labels with no votes if they find no ideas worth supporting. Only flag as non-compliant if challenge labels are missing. If challenge labels are missing, send a one-shot correction: *"Your response is missing the required challenge labels (STRONG/WEAK/MODIFY/MERGE). Please repost using the format specified in your instructions."* Proceed after one correction attempt regardless — do not loop. If an agent's challenge-and-vote post remains unformatted after correction, manually extract any discernible votes or challenge assessments from the unformatted text and include them in the convergence tally. Note the extraction in the lead notes.

### Convergence

After all agents post their combined challenge-and-vote messages (if any agent has not posted after the others have all completed, proceed with available responses and note the non-response):

1. **Shut down all brainstorm agents.** Send a shutdown request to each agent via `SendMessage` with `message: {type: "shutdown_request"}`.
2. **Count support.** For each idea, tally how many agents voted for it.
3. **Survival threshold: majority support (external votes only).** Self-votes are not permitted. If an agent proposed a MERGE [Ia]+[Ib], they may not vote for either constituent idea, but may vote for other merged combinations they did not propose. When agents ≥ 3, the majority threshold is ≥2 external votes. When only 2 agents are in a round, the threshold is ≥1 external vote. An idea survives if it meets this threshold.
4. **Apply a soft cap of 10.** If more than 10 ideas clear the threshold, keep only the top 10 by vote count. Break ties by preferring ideas that survived challenges without needing modification.
5. **Lead curation.** Review ideas that fell below threshold. If an idea was cut primarily by role-structural opposition (e.g., a minimalist opposing on complexity grounds without citing a concrete failure mode) rather than evidence-based challenge, the orchestrator may rescue it. This is a light touch — only intervene when the opposition lacked specific evidence. Mark rescued ideas as `[lead-rescued]` with a one-line reason. Do not rescue more than 2 ideas per round.
6. **Record cuts.** For each cut idea, record the vote count and the strongest objection against it.

The natural trajectory with these rules: Round 1 typically produces 6-9 survivors from 12-20 proposals. Round 2's harder roles cut further to 4-6. The final round should yield 3-5 strong recommendations. If you end with more than 8 after the final round, the convergence was too loose — trim by applying stricter majority (e.g., require 3 out of 4 votes) and re-cut.

### Write handoff files

Write both handoff files (`brainstorm-r{n}-results.md` and `brainstorm-r{n}-transcript.md`) using the formats defined above.

**Self-check:** Before proceeding, verify the results file contains all required fields: surviving idea IDs with titles, vote counts (X/Y format), unresolved dissent section, cut ideas with reasons. If any are missing, fix them now.

Emit a status line: `[Status] round={n} agents_responded={n/m} ideas_proposed={count} ideas_surviving={count} ideas_cut={count}`

Followed by: `[Round {n}] Convergence complete. {survived} ideas survive, {cut} cut. {next action}.`

Where `{next action}` is either "Starting Round {n+1}." or "Moving to final output." for the last round.

### Tear down

After writing handoff files, call `TeamDelete` to clean up `brainstorm-r{n}`.

### Between rounds

After team cleanup, read `brainstorm-r{n}-results.md` and present to the user:

```
## Round {n} Complete

Surviving ideas ({idea count}):
{for each: [ID] title — Support: {vote count}/{total} external votes | Dissent: {agent role name} ("{one-line concern}") or "none"}

Cut:
{for each: [ID] title — reason (strongest objection)}

Next round roles: {selected roles and one-line reason for each selection}

Full transcript: brainstorm-r{n}-transcript.md
```

#### Early stopping check

If the surviving idea set is unchanged from the previous round (same ideas, no new dissent, no new modifications), skip remaining rounds and proceed to final synthesis. Note the early stop: `[Status] early_stop=true reason=converged round={n} remaining_rounds_skipped={count}`

### Idle notification handling

Treat the **first idle notification** from an agent after a substantive message as "this agent is done." Ignore subsequent idle notifications from the same agent until you send them a new message. Do not respond to or acknowledge repeated idle notifications — they carry zero additional information.

---

## After All Rounds

### Synthesise from transcript files

Read all transcript files (`brainstorm-r1-transcript.md` through `brainstorm-r{n}-transcript.md`) and the final round's results file. The transcripts contain the full reasoning chains — specific challenge exchanges, defenses, and merge proposals — that the compact results files omit. Use these to write a high-quality final output.

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

- **Don't pre-summarise the codebase.** The problem statement is the prompt. Do not read, analyse, or summarise the codebase to "give agents context" — this dilutes the prompt with a lossy summary that introduces errors and burns context window. Agents can read code themselves if they need to.
- **Diversity is the mechanism.** If you pick 4 similar roles, the brainstorm collapses into groupthink. The value comes from genuinely different perspectives colliding. Review your role selections critically.
- **History carries forward.** Later rounds MUST see what survived and what was challenged. The results file handoff carries this between rounds — agents don't need the full transcript, just the compact surviving-idea list.
- **Idea count trajectory.** Round 1 with 4 agents × 3-5 ideas = 12-20 raw ideas. Converging to 8 cuts ~50-60%. By the final round you should have 3-5 strong, well-tested recommendations. If you end with more than 8, your convergence is too loose. If you end with fewer than 3, it's too aggressive.
- **Token economics.** Default (3 rounds, 4/3/3 agents) = ~10 agent context windows. Each agent receives ~4k tokens of protocol and context before doing any reasoning. A full default brainstorm consumes ~50-60k input tokens on protocol overhead alone. For cost-sensitive use, `--rounds 2 --agents 3` cuts this roughly in half.
- **Intermediate files.** Each round produces `brainstorm-rN-results.md` and `brainstorm-rN-transcript.md`. These are working files consumed during synthesis and can be cleaned up after the final output is written. The transcript files preserve the detailed reasoning chains that make the final output high-quality.
- **Not just for code.** This skill works for architecture decisions, product direction, process design, or any problem that benefits from structured multi-perspective analysis. The output file serves as a design rationale document regardless of whether implementation follows.
- **Non-compliance is accepted degraded behavior.** If an agent fails to comply with the required format after one correction, the round proceeds with reduced diversity. The orchestrator extracts what it can from non-compliant output and notes the degradation in lead notes. The `[Status]` line shows `agents_responded=N/M` to make this visible. This is an inherent limitation of prompt-only orchestration, not a bug.
