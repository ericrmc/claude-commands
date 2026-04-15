# Design-Build-Review Pipeline

Full pipeline: structured brainstorm → user approval → parallel implementation → review-fix loop.

Chains `/brainstorm` and `/review-fix` with an implementation phase between them. Use this for substantial new features or designs where the problem space benefits from multi-perspective deliberation before code is written.

For individual phases, use `/brainstorm` or `/review-fix` directly.

**Task:** $ARGUMENTS

---

## Preflight

1. Check that Agent Teams is enabled. If not, tell the user to add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` block in their settings.json, then stop.
2. Parse the task from the arguments above. If none specified, ask.
3. Parse optional flags:
   - `--rounds N` — brainstorm rounds (default: 3)
   - `--keep N` — ideas to carry per brainstorm round (default: 8)
   - `--agents N` — agents per brainstorm round (default: 4)
   - `--max-review-iterations N` — review-fix iterations (default: 3)
   - `--auto` — skip approval gates, proceed with top recommendation automatically
   - `--skip-brainstorm` — skip straight to implementation (provide a design description in the args)
   - `--skip-review` — stop after implementation, don't review
   - `--resume` — resume from the most recent checkpoint in `.pipeline/`. Read the most recent checkpoint (by filename timestamp; if multiple checkpoints share the same timestamp prefix, use the one with the highest `phase_completed` value — later phases take precedence; if still ambiguous, ask the user which checkpoint to resume from). Extract: (1) `phase_completed` — determines where to resume from; (2) `approved_design` — re-present this to the user and re-confirm before proceeding; (3) `brainstorm_output_path` — re-read this file to restore the design brief; (4) `files_created`/`files_modified` — use these as the Phase 4 review target if resuming after implementation. If a required field is missing from the checkpoint, ask the user to supply it. Re-confirms all outstanding approval gates before continuing — does not trust checkpoint's record of prior approvals.

Before writing the first checkpoint, ensure the `.pipeline/` directory exists. Create it if it does not.

---

## Phase 1: BRAINSTORM

Unless `--skip-brainstorm` was specified:

Print: `[Pipeline] Starting Phase 1: Brainstorm. Running {n} rounds with {agents} agents per round.`

Read `~/.claude/commands/brainstorm.md` and follow its full protocol, using the task as the problem statement. Pass through `--rounds`, `--keep`, and `--agents` flags.

This produces a brainstorm output markdown file with ranked recommendations, plus per-round working files (`brainstorm-rN-results.md` and `brainstorm-rN-transcript.md`).

When the brainstorm is complete:

1. Print: `[Pipeline] Phase 1 complete. Brainstorm output: {path}. Presenting recommendations.`
2. Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: brainstorm`, `brainstorm_output_path: {path}`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.
3. Before proceeding to Phase 2, confirm silently: (1) All round-lead agents have completed and written their handoff files? (2) Final output file written to disk? (3) Top recommendations extracted? If any check fails, resolve before presenting to user.

Proceed to Phase 2.

---

## Phase 2: APPROVE DESIGN

Present the brainstorm's top recommendations to the user:

> **Brainstorm complete.** Here are the top recommendations:
>
> {for each recommendation: number, title, one-line description — Support: X/Y agents | Dissent: {role} ("{concern}") or None}
>
> Full details with rationale, vote counts, and challenges survived: {output file path}
>
> **How would you like to proceed?**
> 1. Implement the top recommendation *(default)*
> 2. Implement a combination — specify which (e.g., "1 and 3")
> 3. I'll describe what to build based on these
> 4. Stop here — brainstorm only

If `--auto`, proceed with option 1.

**Wait for the user's response before continuing.** This is the most important approval gate — it determines what gets built.

Record the approved design direction for use in the implementation prompts.

Print: `[Pipeline] Design approved: {title}. Planning implementation.`

Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: design_approved`, `approved_design: {title of chosen recommendation}`, `approved_option: {option number chosen}`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.

Before proceeding to Phase 3, confirm silently: (1) User has explicitly chosen a design direction? (2) Approved recommendation recorded? If any check fails, re-present the approval gate.

---

## Phase 3: IMPLEMENT

### Plan the work

Based on the approved design, break the implementation into discrete tasks. Each task should:
- Be a self-contained unit of work (a module, a file group, a layer)
- Own specific files — **no two tasks share files**
- Have a clear deliverable (what exists when this task is done)
- Have defined dependencies (which tasks must complete first, if any)

Present the implementation plan:

> **Implementation plan:**
>
> {for each task: number, description, files it will create/modify, dependencies}
>
> Estimated developers: {count}
>
> Proceed? (or adjust the plan)

If `--auto`, proceed. Otherwise wait for approval.

### Spawn the developer team

Print: `[Pipeline] Spawning {n} developers for {m} tasks.`

Create an agent team. Spawn **1 developer per task** (max 4). If there are more than 4 tasks, merge the smallest or sequence them — a developer can handle 2-3 related tasks in order, but should never compete with another developer for the same files.

Set up task dependencies so blocked tasks don't get claimed until their dependencies are complete.

#### Prepare the design brief

Before spawning developers, extract a **design brief** from the brainstorm output. Do not pass the full document — it contains rejected ideas, process history, and deliberation noise that will confuse developers. Extract only:

1. **What to build** — the recommendation's description in full
2. **Why this approach** — a short rationale paragraph covering why this was chosen over the main alternatives (so developers don't silently re-introduce rejected approaches)
3. **Known risks and mitigations** — specifically the risks the brainstorm identified and any mitigations proposed
4. **Implementation notes** — concrete suggestions that emerged from agents
5. **Relevant dissenting views** — only from ops, security, and performance roles; these translate directly into implementation decisions. Skip philosophical or creative objections.
6. **Rejected approaches** — A short list (3-5 bullets) of approaches that were explicitly considered and rejected during brainstorming, with one-line reasons. Format: "- {approach}: {why rejected}". These become anti-patterns that developers should actively avoid, not silently re-introduce.

**Brief length guidance:** Aim for concise, focused language throughout. Do not truncate or summarize the "Known risks to build around" or "Relevant concerns" sections — these are load-bearing and developers who miss them will introduce the exact problems the brainstorm flagged. Cut background rationale prose instead.

Do NOT include: cut ideas, honourable mentions, the process log, or the full challenge history.

#### Developer spawn prompt

> You are implementing part of a design that was produced through structured multi-agent brainstorming.
>
> **Overall design:**
> {what's being built — the approved recommendation's full description}
>
> **Why this approach:**
> {one paragraph: why this was chosen, what alternatives were considered and why they were rejected}
>
> **Your task:**
> {specific task description — what to build, which files to create or modify}
>
> **Known risks to build around:**
> {risks from the brainstorm with their proposed mitigations — these are pre-identified failure modes, not speculative concerns}
>
> **Implementation notes:**
> {concrete suggestions from brainstorm agents relevant to your task}
>
> **Relevant concerns (from ops/security/performance reviewers):**
> {dissenting views from those roles that translate into implementation decisions — e.g., "the ops engineer flagged that this needs structured logging for incident response"}
>
> **Rejected approaches (do not re-introduce):**
> {list of rejected approaches from the design brief, each as: "- approach name: why it was rejected"}
>
> **Interfaces with other tasks:**
> {if this task has dependencies or dependents, describe the contracts: what this task expects to receive and what it should expose}
>
> ### Rules
>
> 1. **Implement your task only.** Do not expand scope. If you think something is missing from the design, message the lead.
> 2. **Write tests** for your implementation. Follow existing project conventions.
> 3. **Follow project conventions.** Read CLAUDE.md and look at existing patterns before writing new code.
> 4. **Design decisions are already made.** The brainstorm considered alternatives and this direction was chosen for documented reasons. If you disagree with a design choice, message the lead with your concern rather than silently deviating.
> 5. **When done**, message the lead with:
>    - What you built
>    - Files created or modified
>    - Tests added
>    - Interface references (if any): file paths, module names, or function names your task exposes or depends on from other tasks
>    - Any concerns or decisions you had to make that weren't covered by the design
> 6. Mark your task as complete.

### Wait and collect

Wait for all developers to complete. Collect their summaries. If any developer flagged design concerns, note them — they may surface as review findings.

**Integration check:** Review each developer's completion summary. If 2 or more developers' "Interface references" sections reference the same file path, module name, or function name, spawn one additional sequential integration developer with this prompt:

> You are an integration developer. The parallel implementation phase is complete. Your job is narrow: verify that the interface contracts between tasks are consistent. Read the developer completion summaries and the files they reference. Look for mismatched function signatures, inconsistent type definitions, module exports that don't match their imports, and shared config or entry-point files that multiple tasks touched. Fix only these interface-level mismatches — do not refactor implementations. When done, message the lead with what you changed and what you verified.

If all developers reported independent tasks with no shared interface references, skip this step.

Shut down all developers and clean up the team.

Print: `[Pipeline] Implementation complete. {n} files changed. Starting review.`

Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: implementation`, `files_created: {list}`, `files_modified: {list}`, `developer_concerns: {any concerns flagged}`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.

Before proceeding to Phase 4, confirm silently: (1) Shutdown request sent to all developer agents and all have confirmed completion or gone idle? (2) All developer completion summaries collected? (3) Full list of created/modified files compiled? If any check fails, follow up with the relevant developer before proceeding.

### Record what was built

Compile a list of all files created or modified and all tests added. This becomes the target for Phase 4.

---

## Phase 4: REVIEW-FIX

Unless `--skip-review` was specified:

Read `~/.claude/commands/review-fix.md` and follow its full protocol. Target the files changed during Phase 3. Pass `--max-iterations {max-review-iterations}`.

The review-fix loop will:
1. Review the implementation with 3 independent reviewers
2. Triage findings (present to user or auto-fix medium+)
3. Fix findings with parallel developers
4. Verify fixes
5. Loop until clean or max iterations reached

When the review-fix loop completes, proceed to the final summary.

---

## Final Summary

Present a complete pipeline report:

```
## Pipeline Complete

### Design
Brainstorm rounds: {n}
Recommendations considered: {count}
Approved direction: {title}
Design document: {path to brainstorm output file}

### Implementation
Tasks completed: {count}
Files created: {list}
Files modified: {list}
Tests added: {list}

### Review
Review-fix iterations: {n}
Total findings: {count}
Fixed and verified: {count}
Remaining: {count, with IDs and brief reasons}

### Summary
{2-3 sentence assessment: was the design implemented cleanly? are there open concerns? what should the user look at first?}
```

---

## Notes

- **Token cost.** This is the most resource-intensive workflow. Brainstorm (default: ~13 context windows — 3 round-leads + ~10 agents with tapering) + Implementation (up to 4) + Review-fix (up to 8 per iteration × 3 iterations). For a full default run: ~40 context windows. Round isolation in the brainstorm phase prevents cross-round context accumulation. Use `--rounds 2 --agents 3` for smaller problems, or `--skip-brainstorm` if the design is already decided.
- **Approval gates are load-bearing.** The brainstorm → implementation gate prevents building the wrong thing. The triage gate in review-fix prevents fixing non-issues. Unless `--auto` is set, always pause for user input at these points.
- **The brainstorm document persists.** Even after implementation and review, the design rationale lives in the output file. This is valuable for onboarding, future changes, or understanding why a particular approach was chosen over alternatives.
- **Incremental use.** You don't have to run the full pipeline. Use `/brainstorm` alone for design decisions. Use `/review-fix` alone for existing code. Use `/pipeline --skip-brainstorm` when you already know what to build. Use `/pipeline --skip-review` when you want design + implementation without the review loop.
- **Design concerns from developers.** If a developer flags a concern during implementation that contradicts the brainstorm's design, take it seriously — they have more concrete context than the brainstorm agents did. Note it in the final summary and let the user decide whether to address it.
- **Checkpoints.** Checkpoints do NOT contain finding evidence (file:line quotes), code snippets, full deliberation transcripts, or security-sensitive content — only structural decisions and artifact paths. The `--resume` flag reads the most recent checkpoint in `.pipeline/` and picks up from the last completed phase, but always re-confirms approval gates rather than trusting the checkpoint's record of prior approvals.
