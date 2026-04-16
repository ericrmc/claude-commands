# Design-Build-Review Pipeline

Structured brainstorm → user approval → parallel implementation → review-fix loop.

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
   - `--skip-brainstorm` — skip Phase 1 and 2. Arguments become the design brief directly.
   - `--from-brainstorm PATH` — skip Phase 1, use existing brainstorm output file. Enters Phase 2 directly.
   - `--skip-review` — stop after implementation
   - `--resume` — resume from most recent checkpoint in `.pipeline/`. See Appendix E for resume logic.
   - `--quick` — expands to `--rounds 2 --agents 3 --max-review-iterations 2`. Print expanded values. Explicit flags override `--quick` values.
   - `--dry-run` — print the full execution plan (phases, agent counts, estimated context windows, estimated tokens) and stop. No agents spawned.
4. Expand presets: if `--quick` is set, expand and print expanded values. Explicit flags override.
5. Validate parameters: `--rounds` ≥ 1, `--agents` ≥ 2, `--max-review-iterations` ≥ 0. Conflicts: `--skip-brainstorm` + `--from-brainstorm` (contradictory — print error, stop), `--resume` + `--skip-brainstorm` (ambiguous — print error, stop), `--skip-review` + `--max-review-iterations` (contradictory if iterations > 0 — print error, stop). If invalid, print the constraint and a usage example, then stop.
6. If `--dry-run` is set, print the execution plan and stop:
   > **Dry-run plan:**
   > Phase 1 (Brainstorm): {rounds} rounds, {agents} agents/round, ~{context windows}cw
   > Phase 2 (Approve): 1 approval gate
   > Phase 3 (Implement): up to 4 developers + conditional integration
   > Phase 4 (Review-Fix): up to {max-iterations} iterations, ~8cw/iteration
   > Total estimated context windows: ~{total}, Estimated input tokens: ~{total × 4}k
   > Flags: {all flags in effect after preset expansion}
   > Skipped phases: {list or "none"}
   > Run again without --dry-run to execute.

Ensure `.pipeline/` directory exists before writing the first checkpoint.

---

## Phase 1: BRAINSTORM

**If `--from-brainstorm PATH`:**
1. Read PATH. Verify it has: title, description, support count, dissent summary per recommendation. If missing, tell user and stop.
2. Print: `[Pipeline] Using existing brainstorm output: {PATH}. Skipping Phase 1.`
3. Write checkpoint: `phase_completed: brainstorm`, `brainstorm_output_path: {PATH}`.
4. Proceed to Phase 2.

**If `--skip-brainstorm`:** Skip to Phase 3. Arguments become the design brief.

**Otherwise:**

Print: `[Pipeline] Starting Phase 1: Brainstorm. Running {n} rounds with {agents} agents per round.`

Follow the `/brainstorm` protocol. Pass through `--rounds`, `--keep`, and `--agents` flags.

When complete:
1. Print: `[Pipeline] Phase 1 complete. Brainstorm output: {path}.`
2. Write checkpoint: `phase_completed: brainstorm`, `brainstorm_output_path: {path}`.

---

## Phase 2: APPROVE DESIGN

**If `--skip-brainstorm`:** Skip entirely — proceed to Phase 3.

**If zero recommendations survived:**

> **Brainstorm complete, but no recommendations survived.** You can:
> 1. Re-run with different parameters
> 2. Describe a design manually
> 3. Stop here

If `--auto`, stop and report.

**Normal approval:**

> **Brainstorm complete.** Here are the top recommendations:
>
> {number, title, description — Support: X/Y | Dissent: {role} ("{concern}") or None}
>
> Full details: {output file path}
>
> **How would you like to proceed?**
> 1. Implement the top recommendation *(default)*
> 2. Implement a combination — specify which
> 3. I'll describe what to build based on these
> 4. Stop here — brainstorm only

If `--auto`, proceed with option 1. Otherwise **wait for user response** — this is the most important approval gate.

If user picks option 3, their description replaces "what to build" but **retain all brainstorm-derived risks, rejected approaches, and dissenting views**.

Print: `[Pipeline] Design approved: {title}. Planning implementation.`

**Persist design brief:** Extract using the rules in Appendix A. Write to `.pipeline/design-brief.md`.

Write checkpoint: `phase_completed: design_approved`, `approved_design: {title}`, `design_brief_path: .pipeline/design-brief.md`.

---

## Phase 3: IMPLEMENT

### Step 1: Plan the work

Break implementation into discrete tasks. Each task: self-contained, owns specific files (no sharing), clear deliverable, defined dependencies.

> **Implementation plan:**
> {task number, description, files, dependencies}
> Estimated developers: {count}
> Proceed?

If `--auto`, proceed. Otherwise wait for approval.

### Step 2: Prepare design brief

If `.pipeline/design-brief.md` exists, read it directly.

If `--skip-brainstorm`, write user's arguments as the design brief with a note that brainstorm-derived sections are unavailable.

Otherwise, extract from brainstorm output using the rules in Appendix A.

### Step 3: Spawn developers

Print: `[Pipeline] Spawning {n} developers for {m} tasks.`

Create an agent team. Spawn **1 developer per task** (max 4). If more than 4, merge smallest or sequence them. Use the prompt in Appendix B.

Set up task dependencies so blocked tasks don't get claimed early.

### Step 4: Wait and collect

Wait for all developers. Collect summaries.

**Integration check:** If 2+ developers' interface references overlap:
1. Check for existing integration tests covering the shared interfaces.
2. If tests exist and are current, skip integration developer.
3. If no tests exist, spawn one integration developer using the prompt in Appendix C.

If no shared interfaces, skip entirely.

Shut down all developers, `TeamDelete`.

### Step 5: Developer concern gate

If any developer flagged design concerns:

> **Developer concern(s) raised:**
> {task, concern description}
>
> 1. Proceed to review *(default)*
> 2. Pause — address concerns first
> 3. Stop — concerns change the design

If `--auto`, proceed with option 1 but include concerns in final summary. If no concerns, skip.

Print: `[Status] phase=implementation files_created={n} files_modified={n} developer_concerns={count_or_none}`

Print: `[Pipeline] Implementation complete. {n} files changed.`

Write checkpoint: `phase_completed: implementation`, `files_created`, `files_modified`, `developer_concerns`.

### Step 6: Record and summarise

Compile list of files created/modified and tests added.

Write `.pipeline/implementation-summary.md` from developer summaries using the format in Appendix D. This scopes the review phase to actual changes.

---

## Phase 4: REVIEW-FIX

Unless `--skip-review`:

Follow the `/review-fix` protocol. Target files changed during Phase 3. Pass `--max-iterations {max-review-iterations}`. If `--auto`, also pass `--auto`.

**Scoping context:** Prepend the context block from Appendix D to each reviewer's prompt. This scopes reviewers to the implementation changes — not added when `/review-fix` runs standalone.

---

## Final Summary

```
## Pipeline Complete

### Design
Brainstorm rounds: {n}
Recommendations considered: {count}
Approved direction: {title}
Design document: {path}

### Implementation
Tasks completed: {count}
Files created: {list}
Files modified: {list}
Tests added: {list}

### Review
Review-fix iterations: {n}
Total findings: {count}
Fixed and verified: {count}
Remaining: {count, with IDs and reasons}

### Developer Concerns
{list or "None raised."}

### Summary
{2-3 sentence assessment}
```

---

## Appendix A: Design Brief Extraction

If `.pipeline/design-brief.md` already exists, read it directly — do not re-extract.

Extract from the brainstorm output. Do not pass the full document — it contains rejected ideas and deliberation noise. Extract only:

1. **What to build** — the recommendation's full description
2. **Why this approach** — short rationale covering why this was chosen over alternatives
3. **Known risks and mitigations** — risks the brainstorm identified with proposed mitigations
4. **Implementation notes** — concrete suggestions from agents
5. **Relevant dissenting views** — only from ops, security, and performance roles
6. **Rejected approaches** — 3-5 bullets with one-line reasons. These become anti-patterns.

Do NOT include: cut ideas, honourable mentions, process log, challenge history. Keep concise but do not truncate risks or dissenting views — they are load-bearing.

---

## Appendix B: Developer Spawn Prompt

> You are implementing part of a design produced through structured multi-agent brainstorming.
>
> <rules>
> 1. **Implement your task only.** Do not expand scope. Message the lead if something seems missing.
> 2. **Write tests.** Follow existing project conventions.
> 3. **Follow project conventions.** Read CLAUDE.md and existing patterns first.
> 4. **Design decisions are made.** If you disagree, message the lead rather than silently deviating.
> </rules>
>
> <design>
> {approved recommendation's full description}
> Why this approach: {rationale, alternatives rejected}
> </design>
>
> <task>
> {specific task — what to build, which files}
> </task>
>
> <risks>
> {brainstorm-identified risks with mitigations}
> </risks>
>
> <context>
> Implementation notes: {agent suggestions relevant to this task}
> Relevant concerns: {ops/security/performance dissenting views}
> </context>
>
> <rejected>
> {rejected approaches — do not re-introduce}
> </rejected>
>
> <interfaces>
> {contracts with dependent/dependency tasks}
> </interfaces>
>
> <completion>
> Message the lead: what you built, files changed, tests added, interface references, any concerns. Mark task complete.
> </completion>

---

## Appendix C: Integration Developer Prompt

> You are an integration developer. The parallel implementation phase is complete. Your job is narrow: verify interface contracts between tasks are consistent. Read developer completion summaries and the files they reference. Look for mismatched function signatures, inconsistent type definitions, module exports that don't match imports, shared config files that multiple tasks touched. Fix only interface-level mismatches — do not refactor. Run the full test suite after changes. Message the lead: what you changed, what you verified, files touched, test results.

---

## Appendix D: Implementation Summary

Write `.pipeline/implementation-summary.md` after collecting developer summaries:

```markdown
# Implementation Summary

## Design intent
{one-line description of approved design}

## Changes
- {file path}: {one-line description of what was changed and why}

## Tests added
- {test files created or modified}

## Files to review
{all files created or modified}
```

**Review scoping context** — prepend to each reviewer's prompt when running review as part of the pipeline:

> <change-context>
> You are reviewing changes made during a pipeline implementation phase, not reviewing the files from scratch.
>
> {contents of .pipeline/implementation-summary.md}
>
> **Scope your review to these changes.** Focus on whether the changes are correct, complete, and don't introduce new problems. Pre-existing issues outside scope are not findings unless they interact with the changes.
> </change-context>

---

## Appendix E: Resume Logic

`--resume` reads the most recent checkpoint in `.pipeline/` by filename timestamp. If multiple share a timestamp, use the highest `phase_completed` value.

Phase ordering: `brainstorm` < `design_approved` < `implementation` < `fix_iteration_1` < `fix_iteration_2` < ... < `review`

Extract from checkpoint:
1. `phase_completed` — where to resume from
2. `approved_design` — re-present and re-confirm before proceeding
3. `design_brief_path` — read `.pipeline/design-brief.md`
4. `brainstorm_output_path` — for reference
5. `files_created`/`files_modified` — review target if resuming after implementation

Missing fields → ask user. Re-confirm all approval gates unless `--auto` is also set. `--auto` takes precedence over re-confirmation.

**Checkpoint integrity check:** Before resuming, verify the checkpoint contains required fields (`phase_completed` and at least one of `approved_design`, `design_brief_path`, or `brainstorm_output_path`). If any required field is missing, abort with: "Checkpoint at {path} is malformed (missing {field}). Re-run without --resume or fix the checkpoint manually."

---

## Notes

- **Token economics.** Brainstorm (~10 agents) + Implementation (up to 4) + Review-fix (up to 8/iteration × 3 iterations). Full default run: ~35 context windows, ~150-200k input tokens on protocol overhead. Use `--rounds 2 --agents 3` or `--skip-brainstorm` to reduce.
- **Approval gates are load-bearing.** Brainstorm→implementation prevents building the wrong thing. Triage prevents fixing non-issues. Always pause unless `--auto`.
- **Artifacts persist.** Brainstorm output, design brief, and implementation summary survive after the pipeline completes. Useful for onboarding and understanding design decisions.
- **Design concerns from developers** are surfaced as a gate before review. This is where implementation-time discovery flows backward.
- **No design feedback loop.** Review cannot trigger re-brainstorm. If review reveals a design issue, stop and re-run `/brainstorm`. Deliberate simplicity trade-off.
- **Checkpoints** contain only structural metadata — no code, evidence, or transcripts. `--resume` picks up from last completed phase.

---

## Appendix F: When Things Go Wrong

| Failure | Detection | Recovery | Degraded behavior |
|---------|-----------|----------|-------------------|
| Brainstorm produces zero recommendations | No ideas survive final convergence | Report to user; offer to re-run with different parameters or describe a design manually | Cannot proceed to implementation without a design |
| Developer non-response | Other developers complete but one remains idle after nudge | Proceed without; reassign unfinished task to another developer or handle in integration pass | Partial implementation; note missing task in summary |
| Team creation failure | `TeamCreate` returns an error | Retry once; if still failing, tell user to check Agent Teams config | Cannot proceed — stop and report |
| Checkpoint corruption on --resume | Required fields missing from checkpoint file | Abort with clear message listing missing fields; suggest re-running without --resume | Cannot resume — full restart required |
| Developer flags design concern | Developer messages lead with design-level issue | Present to user at developer concern gate (Phase 3 Step 5) | User decides: proceed, pause, or stop |
