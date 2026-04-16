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
   - `--auto` — skip approval gates, proceed with top recommendation automatically. Passed through to review-fix as `--auto` (auto-fix medium+ findings without triage).
   - `--skip-brainstorm` — skip Phase 1 and Phase 2. Your arguments become the design brief directly — be specific about what to build and how. Phase 3 uses the arguments as the design description, with no brainstorm-derived risks, rejected approaches, or dissenting views available.
   - `--from-brainstorm PATH` — skip Phase 1, use an existing brainstorm output file. PATH is the brainstorm output markdown file (e.g., `brainstorm-output.md`). The pipeline reads this file and enters Phase 2 (design approval) directly. All brainstorm-derived context (risks, rejected approaches, dissenting views) is extracted from the file as normal. Transcript files (`brainstorm-rN-transcript.md`) are used during final synthesis if they exist alongside the output file, but are not required.
   - `--skip-review` — stop after implementation, don't review
   - `--resume` — resume from the most recent checkpoint in `.pipeline/`. Read the most recent checkpoint (by filename timestamp; if multiple checkpoints share the same timestamp prefix, use the one with the highest `phase_completed` value — later phases take precedence; if still ambiguous, ask the user which checkpoint to resume from). Phase ordering for comparison: `brainstorm` < `design_approved` < `implementation` < `fix_iteration_1` < `fix_iteration_2` < ... < `review`. Extract: (1) `phase_completed` — determines where to resume from; (2) `approved_design` — re-present this to the user and re-confirm before proceeding; (3) `design_brief_path` — read `.pipeline/design-brief.md` to restore the design brief deterministically; (4) `brainstorm_output_path` — for reference; (5) `files_created`/`files_modified` — use these as the Phase 4 review target if resuming after implementation. If a required field is missing from the checkpoint, ask the user to supply it. Re-confirms all outstanding approval gates before continuing — does not trust checkpoint's record of prior approvals. **Exception:** if `--auto` is also set, skip re-confirmation and proceed automatically. `--auto` takes precedence over `--resume`'s re-confirmation requirement.

Before writing the first checkpoint, ensure the `.pipeline/` directory exists. Create it if it does not.

---

## Phase 1: BRAINSTORM

If `--from-brainstorm PATH` was specified:

1. Read the file at PATH. Verify it contains the required fields for Phase 2 (title, one-line description, support count, dissent summary for each recommendation). If any are missing, tell the user what's missing and stop.
2. Print: `[Pipeline] Using existing brainstorm output: {PATH}. Skipping Phase 1.`
3. Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: brainstorm`, `brainstorm_output_path: {PATH}`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.
4. Proceed to Phase 2.

Unless `--skip-brainstorm` or `--from-brainstorm` was specified:

Print: `[Pipeline] Starting Phase 1: Brainstorm. Running {n} rounds with {agents} agents per round.`

Follow the `/brainstorm` protocol, using the task as the problem statement. Pass through `--rounds`, `--keep`, and `--agents` flags.

This produces a brainstorm output markdown file with ranked recommendations, plus per-round working files (`brainstorm-rN-results.md` and `brainstorm-rN-transcript.md`).

**Required fields for Phase 2:** The brainstorm output must contain, for each surviving recommendation: title, one-line description, support count (X/Y external votes), and dissent summary (role + concern, or "None"). Phase 2 cannot present recommendations without these.

When the brainstorm is complete:

1. Print: `[Pipeline] Phase 1 complete. Brainstorm output: {path}. Presenting recommendations.`
2. Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: brainstorm`, `brainstorm_output_path: {path}`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.
3. Before proceeding to Phase 2, confirm silently: (1) Final output file written to disk? (2) Top recommendations extracted? If any check fails, resolve before presenting to user.

Proceed to Phase 2.

---

## Phase 2: APPROVE DESIGN

If `--skip-brainstorm` is set (and `--from-brainstorm` is not), skip this phase entirely — proceed to Phase 3 with the arguments as the design description.

### Zero-recommendation fallback

If the brainstorm produced zero surviving recommendations, present:

> **Brainstorm complete, but no recommendations survived.** All ideas were cut during convergence. You can:
> 1. Re-run with different parameters (e.g., `--rounds 2 --agents 3`)
> 2. Describe a design manually — I'll use the brainstorm's honourable mentions and dissenting views as context
> 3. Stop here

If `--auto`, stop and report: "Auto mode cannot proceed — brainstorm produced no recommendations. Re-run with adjusted parameters or without `--auto`."

Wait for the user's response before continuing.

### Normal approval

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

**If the user picks option 3:** Their description replaces the "what to build" and "why this approach" sections of the design brief. However, **retain all brainstorm-derived risks, rejected approaches, and dissenting views** in the brief — these are load-bearing and apply regardless of which design direction the user chooses. Prompt the user: *"Your description will be used for what to build. Risks, rejected approaches, and dissenting views from the brainstorm will still be included in the developer brief."*

Record the approved design direction for use in the implementation prompts.

Print: `[Pipeline] Design approved: {title}. Planning implementation.`

### Persist design brief

Extract the design brief (see Phase 3's "Prepare the design brief" section) and write it to `.pipeline/design-brief.md`. This makes the brief a concrete artifact — deterministic on resume, reviewable, and editable.

Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: design_approved`, `approved_design: {title of chosen recommendation}`, `approved_option: {option number chosen}`, `design_brief_path: .pipeline/design-brief.md`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.

Before proceeding to Phase 3, confirm silently: (1) User has explicitly chosen a design direction? (2) Approved recommendation recorded? (3) Design brief written to `.pipeline/design-brief.md`? If any check fails, re-present the approval gate.

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

If `.pipeline/design-brief.md` already exists (from a previous run or Phase 2), read it directly — do not re-extract. This ensures deterministic briefs across resumes.

If `--skip-brainstorm` is set, the user's arguments are the design brief. Write them to `.pipeline/design-brief.md` with a note that brainstorm-derived sections (risks, rejected approaches, dissenting views) are unavailable. Developers should apply their own judgement for these.

Otherwise, extract a **design brief** from the brainstorm output. Do not pass the full document — it contains rejected ideas, process history, and deliberation noise that will confuse developers. Extract only:

1. **What to build** — the recommendation's description in full
2. **Why this approach** — a short rationale paragraph covering why this was chosen over the main alternatives (so developers don't silently re-introduce rejected approaches)
3. **Known risks and mitigations** — specifically the risks the brainstorm identified and any mitigations proposed
4. **Implementation notes** — concrete suggestions that emerged from agents
5. **Relevant dissenting views** — only from ops, security, and performance roles; these translate directly into implementation decisions. Skip philosophical or creative objections.
6. **Rejected approaches** — A short list (3-5 bullets) of approaches that were explicitly considered and rejected during brainstorming, with one-line reasons. Format: "- {approach}: {why rejected}". These become anti-patterns that developers should actively avoid, not silently re-introduce.

**Brief length guidance:** Aim for concise, focused language throughout. Do not truncate or summarize the "Known risks to build around" or "Relevant concerns" sections — these are load-bearing and developers who miss them will introduce the exact problems the brainstorm flagged. Cut background rationale prose instead.

Do NOT include: cut ideas, honourable mentions, the process log, or the full challenge history.

If the brief was not already written to `.pipeline/design-brief.md` during Phase 2, write it now.

**Required fields for developers:** The design brief must contain: (1) what to build, (2) why this approach, (3) known risks, (4) implementation notes, (5) relevant dissenting views, (6) rejected approaches. If `--skip-brainstorm` is set, items 3-6 may be unavailable — note this explicitly so developers apply their own judgement.

#### Developer spawn prompt

> You are implementing part of a design that was produced through structured multi-agent brainstorming.
>
> <rules>
> 1. **Implement your task only.** Do not expand scope. If you think something is missing from the design, message the lead.
> 2. **Write tests** for your implementation. Follow existing project conventions.
> 3. **Follow project conventions.** Read CLAUDE.md and look at existing patterns before writing new code.
> 4. **Design decisions are already made.** The brainstorm considered alternatives and this direction was chosen for documented reasons. If you disagree with a design choice, message the lead with your concern rather than silently deviating.
> </rules>
>
> <design>
> {what's being built — the approved recommendation's full description}
>
> Why this approach: {one paragraph: why this was chosen, what alternatives were considered and why they were rejected}
> </design>
>
> <task>
> {specific task description — what to build, which files to create or modify}
> </task>
>
> <risks>
> {risks from the brainstorm with their proposed mitigations — these are pre-identified failure modes, not speculative concerns}
> </risks>
>
> <context>
> Implementation notes: {concrete suggestions from brainstorm agents relevant to your task}
>
> Relevant concerns (from ops/security/performance reviewers): {dissenting views from those roles that translate into implementation decisions — e.g., "the ops engineer flagged that this needs structured logging for incident response"}
> </context>
>
> <rejected>
> {list of rejected approaches from the design brief, each as: "- approach name: why it was rejected". Do not re-introduce these.}
> </rejected>
>
> <interfaces>
> {if this task has dependencies or dependents, describe the contracts: what this task expects to receive and what it should expose}
> </interfaces>
>
> <completion>
> When done, message the lead with:
> - What you built
> - Files created or modified
> - Tests added
> - Interface references (if any): file paths, module names, or function names your task exposes or depends on from other tasks
> - Any concerns or decisions you had to make that weren't covered by the design
>
> Mark your task as complete.
> </completion>

### Wait and collect

Wait for all developers to complete. Collect their summaries.

**Integration check:** Review each developer's completion summary. If 2 or more developers' "Interface references" sections reference the same file path, module name, or function name:

1. **Check for existing integration tests.** Look for test files that import or reference the modules being changed (e.g., test files containing import statements or references to the shared interfaces). Search the project's test directories and any co-located test files.
2. **If integration tests exist** for the changed modules, verify they are not stale — they should have been modified within the current session or reference the specific interfaces from the developer completion summaries, not just the module name in general. If test relevance is uncertain, err on the side of spawning the integration developer. If the tests are relevant, skip the integration developer — the existing test suite will catch interface mismatches. Note in the implementation summary: "Integration developer skipped — existing integration tests cover {modules}."
3. **If no integration tests exist**, spawn one additional sequential integration developer with this prompt:

> You are an integration developer. The parallel implementation phase is complete. Your job is narrow: verify that the interface contracts between tasks are consistent. Read the developer completion summaries and the files they reference. Look for mismatched function signatures, inconsistent type definitions, module exports that don't match their imports, and shared config or entry-point files that multiple tasks touched. Fix only these interface-level mismatches — do not refactor implementations. Run the full test suite after making changes. When done, message the lead with: what you changed, what you verified, which files you touched, and test results.

If all developers reported independent tasks with no shared interface references, skip this step entirely.

Shut down all developers and clean up the team.

### Developer concern gate

If any developer flagged design concerns during implementation, present them to the user before proceeding to review:

> **Developer concern(s) raised during implementation:**
>
> {for each: developer task, concern description}
>
> These may indicate the brainstorm's design assumptions don't hold in practice. How would you like to proceed?
> 1. Proceed to review — address concerns later if they surface as findings *(default)*
> 2. Pause — I'll address these concerns before review
> 3. Stop — these concerns change the design direction

If `--auto`, proceed with option 1 but include the concerns prominently in the final summary.

If no developers flagged concerns, skip this gate.

Print: `[Status] phase=implementation files_created={n} files_modified={n} developer_concerns={count_or_none}`

Print: `[Pipeline] Implementation complete. {n} files changed. Starting review.`

Write a checkpoint to `.pipeline/checkpoint-{YYYYMMDD-HHmmss}.md` with: `phase_completed: implementation`, `files_created: {list}`, `files_modified: {list}`, `integration_files_touched: {list, if integration developer ran}`, `developer_concerns: {any concerns flagged}`, `timestamp: {ISO timestamp}`, `pipeline_version: 1.0`.

Before proceeding to Phase 4, confirm silently: (1) Shutdown request sent to all developer agents and all have confirmed completion or gone idle? (2) All developer completion summaries collected? (3) Full list of created/modified files compiled? If any check fails, follow up with the relevant developer before proceeding.

### Record what was built

Compile a list of all files created or modified and all tests added. This becomes the target for Phase 4.

**Required fields for Phase 4:** The review-fix target must include: list of files created, list of files modified, and list of tests added. If any list is empty, state "None" explicitly.

### Write implementation summary

Write `.pipeline/implementation-summary.md` from the developer completion summaries. This scopes the review phase to the actual changes rather than reviewing entire files from scratch.

```markdown
# Implementation Summary

## Design intent
{one-line description of the approved design direction}

## Changes
{for each file changed:}
- {file path}: {one-line description of what was changed and why}

## Tests added
- {list of test files created or modified}

## Files to review
{list of all files created or modified}
```

Keep each file's description to one line — enough for a reviewer to know what to look for, not a full diff.

---

## Phase 4: REVIEW-FIX

Unless `--skip-review` was specified:

Follow the `/review-fix` protocol. Target the files changed during Phase 3 (including any files touched by the integration developer). Pass `--max-iterations {max-review-iterations}`. If `--auto` is set, also pass `--auto`.

**Scoping context for reviewers:** When spawning reviewers, prepend the following context block to each reviewer's prompt (before their lens-specific instructions). Read `.pipeline/implementation-summary.md` and include it:

> <change-context>
> You are reviewing changes made during a pipeline implementation phase, not reviewing the files from scratch.
>
> {contents of .pipeline/implementation-summary.md}
>
> **Scope your review to these changes.** Focus on whether the described changes are correct, complete, and don't introduce new problems. Pre-existing issues outside the scope of these changes are not findings unless they directly interact with or are worsened by the changes.
> </change-context>

This context block is added by the pipeline only — when `/review-fix` is run standalone (without a pipeline), no scoping context is added and reviewers review the full target.

The review-fix loop will:
1. Review the implementation with 3 independent reviewers (scoped to changes)
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

### Developer Concerns
{if any: list each concern with the developer's assessment and current status (addressed during review / still open)}
{if none: "None raised."}

### Summary
{2-3 sentence assessment: was the design implemented cleanly? are there open concerns? what should the user look at first?}
```

---

## Notes

- **Token cost.** This is the most resource-intensive workflow. Brainstorm (default: ~10 agent context windows with tapering) + Implementation (up to 4) + Review-fix (up to 8 per iteration × 3 iterations). For a full default run: ~35 context windows. Each agent receives ~4k tokens of protocol overhead. A full default pipeline consumes ~150-200k input tokens on protocol alone. Use `--rounds 2 --agents 3` for smaller problems, or `--skip-brainstorm` if the design is already decided.
- **Approval gates are load-bearing.** The brainstorm → implementation gate prevents building the wrong thing. The triage gate in review-fix prevents fixing non-issues. Unless `--auto` is set, always pause for user input at these points. `--auto` takes precedence over `--resume`'s re-confirmation — when both are set, resume proceeds without pausing.
- **The brainstorm document persists.** Even after implementation and review, the design rationale lives in the output file. This is valuable for onboarding, future changes, or understanding why a particular approach was chosen over alternatives.
- **The design brief persists.** The extracted design brief is written to `.pipeline/design-brief.md` during Phase 2. This makes the brief deterministic on resume, reviewable, and editable. If you want to adjust what developers see, edit this file before resuming.
- **Incremental use.** You don't have to run the full pipeline. Use `/brainstorm` alone for design decisions. Use `/review-fix` alone for existing code. Use `/pipeline --skip-brainstorm` when you already know what to build (your arguments become the design brief directly). Use `/pipeline --from-brainstorm brainstorm-output.md` to resume from a completed brainstorm session — the pipeline reads the output file and enters design approval directly. Use `/pipeline --skip-review` when you want design + implementation without the review loop.
- **Design concerns from developers.** If a developer flags a concern during implementation that contradicts the brainstorm's design, the pipeline surfaces it as a gate before review. This is the mechanism for implementation-time discovery to influence the pipeline — the only point where information flows backward.
- **No design feedback loop.** The pipeline is strictly linear — review cannot trigger re-brainstorm. If review reveals a design-level issue, stop the pipeline and re-run `/brainstorm` with the new information. This is a deliberate simplicity trade-off.
- **Checkpoints.** Checkpoints do NOT contain finding evidence (file:line quotes), code snippets, full deliberation transcripts, or security-sensitive content — only structural decisions and artifact paths. The `--resume` flag reads the most recent checkpoint in `.pipeline/` and picks up from the last completed phase, re-confirming approval gates unless `--auto` is also set.
