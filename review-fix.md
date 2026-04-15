# Structured Review-Fix Loop

Orchestrate a multi-phase Agent Team workflow: independent structured review, triage, parallel implementation, scoped verification, and loop until clean.

**Target:** $ARGUMENTS

---

## Preflight

1. Check that Agent Teams is enabled. If you cannot create an agent team, tell the user to add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` block in their settings.json, then stop.
2. Parse the target from the arguments above. If no target is specified, ask the user what to review.
3. Parse optional flags from the arguments:
   - `--max-iterations N` — max review→fix loops (default: 3)
   - `--auto` — skip triage approval, auto-fix all medium+ findings
   - `--review-only` — stop after Phase 1, don't fix anything

---

## Phase 1: REVIEW

### Spawn the review team

Create an agent team with 3 reviewer teammates. Each reviewer gets a **distinct lens** and the **same protocol block**. The protocol enforces independent analysis before group discussion — this is the most important part.

Spawn each reviewer with their role name and the prompt below. Replace `{TARGET}` with the actual target, and `{LENS}` with the reviewer-specific section.

---

#### Spawn prompt template (use for all 3 reviewers)

> You are a code reviewer with a specific focus area. Your job is to find problems, not propose fixes.
>
> **What to review:** {TARGET}
>
> **Your focus:** {LENS}
>
> ### Protocol — follow exactly
>
> **Step 1 — Independent analysis.** Read the target code. Form your findings on your own. DO NOT read messages from other teammates yet. Take your time — thoroughness matters more than speed.
>
> **Step 2 — Post findings.** Message all teammates with your findings. Use this format for each one:
>
> ```
> [F{n}] {short title}
> Severity: critical | high | medium | low
> Impact: {what goes wrong if unfixed}
> Evidence: {file:line — quote the problematic code}
> Confidence: high | medium | low
> ```
>
> **Step 3 — Cross-review.** After posting, read the other reviewers' findings. Respond to each:
> - **CONFIRM [Fn]** — you agree, optionally add supporting evidence
> - **DISPUTE [Fn]** — you disagree, explain why with counter-evidence
> - **ADD** — post new findings others missed
>
> **Step 4 — Mark complete.** When discussion has settled, mark your task as complete.
>
> **Rules:**
> - Do NOT propose fixes or refactors. Describe problems and their impact only.
> - Do NOT review things outside your focus lens. Other reviewers cover other areas.
> - Be specific. "This could be improved" is not a finding. "This returns null on line 47 when the input array is empty, causing a TypeError in the caller at line 52" is a finding.

---

#### Reviewer lenses

**correctness-reviewer:**
> Logic errors, edge cases, off-by-one errors, incorrect assumptions, missing boundary validation, race conditions, wrong return types, broken contracts between functions, state that can become inconsistent.

**robustness-reviewer:**
> Error handling gaps, failure modes, unvalidated external input, resource leaks, missing try/catch around I/O, exception swallowing, data integrity under partial failure, security concerns (injection, auth bypass, secrets exposure).

**quality-reviewer:**
> Unnecessary complexity, duplication that could cause drift, misleading names, dead code, missing or inadequate test coverage, unclear API boundaries, coupling that will make changes painful, violations of the project's own conventions.

---

### Coordinate the review

Print: `[Review] Spawning 3 reviewers (correctness, robustness, quality). This may take a moment.`

1. **Wait for independent findings.** Monitor messages. Each reviewer should post their `[Fn]` findings before reading others. Do not rush this — wait until all 3 have posted. Then print: `[Review] All 3 reviewers posted findings. Starting cross-review.`
2. **Trigger cross-review.** Once all independent findings are posted, broadcast: *"All independent analyses are in. Please read each other's findings and respond with CONFIRM, DISPUTE, or ADD."*
3. **Let discussion settle.** Allow 1-2 rounds of back-and-forth. If reviewers are going in circles, broadcast: *"Please finalise your positions — we're moving to synthesis."*
4. **Shut down and clean up.** Ask all reviewers to shut down, then clean up the team.

### Synthesise

After the review team is cleaned up, synthesise all findings into one prioritised list:

- **Merge duplicates.** If 2+ reviewers found the same issue independently, merge into one finding and note the agreement count.
- **Resolve disputes.** If a finding was disputed with valid counter-evidence, downgrade or remove it. If disputed without evidence, keep it.
- **Assign severity.** Findings confirmed by multiple reviewers get higher confidence. Adjust severity if cross-review changed the assessment.
- **Sort.** Critical → high → medium → low.
- **Assign IDs.** R1, R2, R3, etc.

Present as a summary table:

| ID | Title | Severity | Agreed | Impact | Location |
|----|-------|----------|--------|--------|----------|

Print: `[Review] Synthesis complete. {n} findings ({critical} critical, {high} high, {medium} medium, {low} low).`

### Checkpoint: review complete

Write a checkpoint to `.pipeline/checkpoint-{timestamp}.md` with: `phase_completed: review`, `iteration: {n}`, `finding_count: {total}`, `findings: {list of IDs, titles, and severities}`, `timestamp`, `pipeline_version: 1.0`. Do NOT include finding evidence (file:line quotes), code snippets, full deliberation transcripts, security-sensitive content, or implementation details — only IDs, titles, and severities.

### Exit checklist (silent)

Before proceeding to triage, confirm silently: (1) Shutdown request sent to all reviewer agents and all have confirmed completion or gone idle? (2) Findings synthesised into a single prioritised list with IDs assigned? (3) Duplicates merged, disputes resolved? If any check fails, complete it before presenting findings. **Do not print anything if all checks pass.**

---

## Phase 2: TRIAGE

If there are **no medium+ findings**, report the review found no medium or higher findings (list any low-severity findings for awareness) and stop.

If `--review-only` was specified, present findings and stop.

If `--auto` was specified, select all medium+ findings and proceed to Phase 3.

Otherwise, present findings to the user and ask:

> Here are the review findings. What would you like to fix?
> 1. All medium and above *(default)*
> 2. All findings including low severity
> 3. I'll pick — list the IDs to fix
> 4. None — stop here

---

## Phase 3: IMPLEMENT

### Group findings by file ownership

Group the approved findings so that findings affecting the same files go to the same developer. No two developers should edit the same file. Also consider test files. If two findings would require adding tests to the same test file, assign those findings to the same developer. When in doubt about test file ownership, assign to one developer rather than risk a conflict. If findings are entangled across many files, assign them to one developer rather than risking conflicts.

### Exit checklist (silent)

Before spawning fix developers, confirm silently: (1) User has approved which findings to fix (or `--auto` is set)? (2) Findings grouped by file ownership with no two developers sharing files (including test files)? If any check fails, resolve before spawning. **Do not print anything if all checks pass.**

### Spawn the developer team

Print: `[Fix] Iteration {n}: Spawning {m} developers for {k} findings.`

Create a new agent team. Spawn **1 developer per finding group** (max 4 developers). If there are more groups than 4, merge the smallest groups.

#### Developer spawn prompt

> You are fixing specific code review findings. Fix only what's assigned — nothing else.
>
> **Your assigned findings:**
>
> {paste full finding details: ID, title, severity, impact, evidence, location}
>
> ### Rules
>
> 1. **Fix only your findings.** Do not refactor, reformat, or improve surrounding code.
> 2. **Write a test for each fix** that would have caught the original issue. If a test framework is already in use, follow its conventions. If a finding is about missing tests, add the tests that were missing.
> 3. **Run tests** to check you haven't broken anything. If tests fail for reasons unrelated to your changes, message the lead.
> 4. **If you disagree with a finding** or believe it's a false positive, message the lead with your reasoning before skipping it.
> 5. **When done**, message the lead with a short summary:
>    - Which findings you fixed (by ID)
>    - Which you skipped (with reason)
>    - Which tests you added or modified
> 6. Mark your task as complete.

### Wait and collect

Wait for all developers to complete. Collect their summaries — note which findings were fixed, which were skipped with reasons.

Shut down all developers and clean up the team.

### Checkpoint: fix iteration complete

Write a checkpoint to `.pipeline/checkpoint-{timestamp}.md` with: `phase_completed: fix_iteration_{n}`, `fixed_findings: {IDs}`, `skipped_findings: {IDs with reasons}`, `files_changed: {list}`, `timestamp`, `pipeline_version: 1.0`. Do NOT include code snippets, full deliberation transcripts, or implementation details — only structural metadata.

---

## Phase 4: VERIFY

### Exit checklist (silent)

Before spawning verifiers, confirm silently: (1) Shutdown request sent to all fix developers and all have confirmed completion or gone idle? (2) All developer summaries collected (which findings fixed, which skipped, files changed)? (3) List of changed files compiled? If any check fails, follow up with developers first. **Do not print anything if all checks pass.**

### Spawn the verification team

Create a new agent team with **2 verifier teammates**.

#### Verifier spawn prompt

> You are verifying that code fixes correctly resolve their original findings.
>
> **Original findings that were fixed:**
>
> {list each finding: ID, title, severity, location, what the fix should have addressed}
>
> **Files that were changed:**
>
> {list of modified files from the developer phase}
>
> ### Your task
>
> 1. For each finding, check the changed code and report one of:
>    - **VERIFIED** — the fix resolves the finding
>    - **NOT_FIXED** — the original problem is still present
>    - **REGRESSED** — the fix made things worse or introduced a new bug
>
> 2. Check for **new issues** in the changed code only. If a fix introduced a new medium+ problem, report it as:
>    ```
>    [N{n}] {title}
>    Severity: ...
>    Impact: ...
>    Evidence: ...
>    ```
>
> 3. **Scope:** Only review the changes and their immediate context. Do NOT re-review the whole codebase.
>
> 4. **Discuss with the other verifier.** If you disagree on whether something is VERIFIED or NOT_FIXED, discuss and try to reach agreement. If you can't agree, flag it for the lead.
>
> 5. Mark your task as complete when done.

### Collect verification results

Wait for both verifiers to complete. Print: `[Verify] Iteration {n}: {verified} verified, {not_fixed} not fixed, {regressed} regressed.`

Synthesise:
- **VERIFIED** → done, remove from active list
- **NOT_FIXED** → goes back to Phase 3, preserving its original finding ID
- **REGRESSED** → goes back to Phase 3 with escalated severity, preserving its original finding ID. A finding that REGRESSed in this iteration should be carried forward with its original ID — do not assign a new ID to a regressed finding.
- **New issues (medium+)** → added to the fix list for Phase 3. New IDs are only for issues first discovered during verification that were not in the original finding set.

Shut down verifiers and clean up the team.

---

## Phase 5: LOOP OR FINISH

Count remaining items: NOT_FIXED + REGRESSED + new medium+ issues.

### Oscillation check (runs first, before any other loop decision)

On the first iteration (iteration 1), there is no previous list — skip the oscillation check and proceed directly to the loop/stop decision.

On iteration 2+, compare the current NOT_FIXED/REGRESSED finding IDs against the previous iteration's NOT_FIXED/REGRESSED list. If any finding ID (e.g., R3, R7) appears as NOT_FIXED or REGRESSED in **two consecutive iterations**, stop immediately — do not spawn another fix cycle. Report to the user:

> "Stopping: oscillation detected. The following findings have failed to resolve in two consecutive iterations: {finding IDs and titles}. This typically indicates a structural conflict where fixing one finding breaks another. Please review these findings manually: {details of each oscillating finding, including severity, impact, and location}."

This check fires **before** checking max iterations. Even if max iterations hasn't been reached, oscillation causes immediate escalation to the user.

### Loop or stop

**If items remain AND iteration < max iterations AND no oscillation detected:**
- Report: *"Iteration {n} complete. {x} verified, {y} remaining. Starting next fix cycle."*
- Return to Phase 3 with the remaining items. **When carrying NOT_FIXED or REGRESSED items into the next iteration, preserve their original finding IDs.** R3 stays R3, R7 stays R7 — do not re-assign IDs. New issues discovered during verification get the next available ID (e.g., if R1–R5 existed, a new issue becomes R6).

**If all verified OR max iterations reached OR no medium+ items remain OR oscillation detected:**
- Stop and present the final summary.

### Final summary

```
## Review-Fix Complete

Iterations: {n}
Findings found: {total}
Findings fixed and verified: {count}
Findings remaining: {count, with IDs and reasons}

Files changed:
- {list}

Tests added/modified:
- {list}
```

If any findings remain unresolved after max iterations, list them with their current status and recommend next steps.

---

## Notes

- **Token cost.** Each phase spawns a new team. A full iteration uses ~8 context windows (3 reviewers + up to 4 developers + 2 verifiers (developer count varies by finding count and grouping)). Multiple iterations multiply this. Be aware of cost on large targets.
- **File conflicts.** The grouping step in Phase 3 is critical. If two developers edit the same file, one will overwrite the other. When in doubt, assign overlapping work to one developer.
- **Teammate discipline.** The structured finding format and independence protocol rely on prompt compliance. If a reviewer posts vague findings or doesn't follow the format, message them with a reminder before proceeding.
- **Scope creep.** Developers are told to fix only assigned findings. If they message about other issues they noticed, acknowledge them but do not expand scope mid-iteration. Note them for a future run.
