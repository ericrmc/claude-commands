# Structured Review-Fix Loop

Orchestrate a multi-phase Agent Team workflow: independent structured review, triage, parallel implementation, scoped verification, and loop until clean.

**Target:** $ARGUMENTS

---

## Preflight

1. Check that Agent Teams is enabled. If you cannot create an agent team, tell the user to add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` block in their settings.json, then stop.
2. Parse the target from the arguments above. If no target is specified, ask the user what to review.
3. Parse optional flags from the arguments:
   - `--max-iterations N` — max review→fix loops (default: 3, minimum: 1). If 0 is passed, treat as `--review-only`.
   - `--auto` — skip triage approval, auto-fix all medium+ findings
   - `--review-only` — stop after Phase 1, don't fix anything

---

## Phase 1: REVIEW

### Spawn the review team

Create an agent team with 3 reviewer teammates. Each reviewer gets a **distinct lens** and the **same protocol block**. The protocol enforces independent analysis before group discussion — this is the most important part.

Select 3 lenses following the rules below, then spawn each reviewer with their role name and the prompt. Replace `{TARGET}` with the actual target, and `{LENS}` with the reviewer-specific section.

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
> *(Calibrate severity as: critical = data loss, security breach, or crash. high = incorrect behavior for common inputs. medium = incorrect behavior for edge cases or under load. low = style, naming, or convention violation.)*
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

The reference pool below covers the most common review concerns. You may substitute or synthesise lenses when the target demands it — e.g., a **security reviewer** for auth code, a **performance reviewer** for hot paths, or a **precision reviewer** for financial calculations. Synthesised lenses are first-class and follow the same protocol.

**Selection rules:**
1. Always include at least one correctness-style lens (logic, contracts, state) and one robustness-style lens (failure modes, input validation, resource management).
2. The third lens should be chosen based on the target. Default to quality; substitute a domain-specific lens when the target has an obvious dominant concern.
3. State your lens selections and reasoning before spawning.

**Reference pool:**

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
   - **Compliance check:** Before triggering cross-review, verify each reviewer's post contains `[F{n}]` format markers. If a reviewer posted findings without the structured format, send a one-shot correction: *"Please reformat your findings using `[F{n}]` markers with the required fields (Severity, Impact, Evidence, Confidence)."* Proceed once corrected or after one reminder — do not loop.
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

**Required fields for triage:** Each finding passed to Phase 2 must have: ID *(mandatory)*, title *(mandatory)*, severity *(mandatory)*, impact *(mandatory)*, location *(mandatory)*, agreed count. Findings missing mandatory fields should be fixed during synthesis before proceeding.

### Checkpoint: review complete

Write a checkpoint to `.pipeline/checkpoint-{timestamp}.md` with: `phase_completed: review`, `iteration: {n}`, `finding_count: {total}`, `findings: {list of IDs, titles, and severities}`, `timestamp`, `pipeline_version: 1.0`. Do NOT include finding evidence (file:line quotes), code snippets, full deliberation transcripts, security-sensitive content, or implementation details — only IDs, titles, and severities.

### Exit checklist (silent)

Before proceeding to triage, confirm silently: (1) Shutdown request sent to all reviewer agents and all have confirmed completion or gone idle? (2) Findings synthesised into a single prioritised list with IDs assigned? (3) Duplicates merged, disputes resolved? If any check fails, complete it before presenting findings. **Do not print anything if all checks pass.**

---

## Phase 2: TRIAGE

If there are **no findings at all**, report a clean review and stop.

If `--review-only` was specified, present findings and stop.

If `--auto` was specified, select all medium+ findings and proceed to Phase 3. If there are only low findings with `--auto`, report them for awareness and stop.

Otherwise, present findings to the user and ask:

> Here are the review findings. What would you like to fix?
> 1. All medium and above *(default)*
> 2. All findings including low severity
> 3. I'll pick — list the IDs to fix
> 4. None — stop here

If there are only low-severity findings, still present them with the same options but note: *"All findings are low severity. These are typically worth fixing in aggregate but are individually non-urgent."*

**Required fields for implementation:** Each finding passed to Phase 3 must include: finding ID, title, severity, impact, evidence (with file:line), location. Developers cannot act on findings without specific evidence and location.

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
> <findings>
> {paste full finding details: ID, title, severity, impact, evidence, location}
> </findings>
>
> <rules>
> 1. **Fix only your findings.** Do not refactor, reformat, or improve surrounding code.
> 2. **Declare your files first.** Before writing any code, message the lead with all files you intend to modify or create (including test files). Wait for conflict-free confirmation before proceeding.
> 3. **Write a test for each fix** that would have caught the original issue. Follow existing test framework conventions.
> 4. **ESCALATE, don't patch.** If a finding requires architectural changes, changes to files owned by other developers, or design-level fixes, message the lead with: `ESCALATE [Fn] — {reason and scope assessment}`. Do not attempt a partial fix that masks the real issue.
> 5. Run the full test suite after your changes. If unrelated tests fail, message the lead.
> 6. If you disagree with a finding, message the lead with reasoning before skipping it.
> </rules>
>
> <completion>
> When done, message the lead with a summary:
> - Finding IDs fixed
> - Finding IDs skipped (with reason)
> - Finding IDs escalated (with scope assessment)
> - Tests added or modified
> - Test suite results (pass/fail count)
>
> Then mark your task as complete.
> </completion>

### File declaration coordination

After all developers have declared their intended files, check for conflicts. If two developers intend to modify the same file (including test files), reassign the conflicting finding to one developer and notify both. Only give the go-ahead once all file ownership is conflict-free.

### Wait and collect

Wait for all developers to complete. Collect their summaries — note which findings were fixed, which were skipped, and which were escalated.

**Escalated findings** are removed from the fix loop and presented to the user in the final summary as requiring manual intervention, with the developer's scope assessment. Do not attempt to fix escalated findings in subsequent iterations.

Shut down all developers and clean up the team.

**Required fields for verification:** Each developer summary must contain: finding IDs fixed, finding IDs skipped (with reasons), finding IDs escalated (with scope assessment), files changed, test results (pass/fail count). If a summary is missing required fields, follow up with the developer before proceeding.

### Checkpoint: fix iteration complete

Write a checkpoint to `.pipeline/checkpoint-{timestamp}.md` with: `phase_completed: fix_iteration_{n}`, `fixed_findings: {IDs}`, `skipped_findings: {IDs with reasons}`, `escalated_findings: {IDs with scope assessments}`, `files_changed: {list}`, `timestamp`, `pipeline_version: 1.0`. Do NOT include code snippets, full deliberation transcripts, or implementation details — only structural metadata.

---

## Phase 4: VERIFY

### Exit checklist (silent)

Before spawning verifiers, confirm silently: (1) Shutdown request sent to all fix developers and all have confirmed completion or gone idle? (2) All developer summaries collected (which findings fixed, which skipped, files changed)? (3) List of changed files compiled? If any check fails, follow up with developers first. **Do not print anything if all checks pass.**

### Spawn the verification team

Create a new agent team with **2 verifier teammates**.

#### Verifier spawn prompt

> You are verifying that code fixes correctly resolve their original findings.
>
> <findings>
> {list each finding: ID, title, severity, location, what the fix should have addressed}
> </findings>
>
> <changed-files>
> {list of modified files from the developer phase}
> </changed-files>
>
> <rules>
> 1. For each finding, check the changed code and report one of:
>    - **VERIFIED** — the fix resolves the finding
>    - **NOT_FIXED** — the original problem is still present
>    - **REGRESSED** — the fix made things worse or introduced a new bug
>
> 2. Check for **new issues** in the changed code only. If a fix introduced a new medium+ problem, report it as:
>    ```
>    [N{n}] {title}
>    Severity: ...
>    *(Calibrate severity as: critical = data loss, security breach, or crash. high = incorrect behavior for common inputs. medium = incorrect behavior for edge cases or under load. low = style, naming, or convention violation.)*
>    Impact: ...
>    Evidence: ...
>    ```
>
> 3. **Scope:** Only review the changes and their immediate context. Do NOT re-review the whole codebase.
>
> 4. **Discuss with the other verifier.** If you disagree on whether something is VERIFIED or NOT_FIXED, discuss and try to reach agreement. If you can't agree, flag it for the lead with both positions and evidence.
> </rules>
>
> <completion>
> Mark your task as complete when done.
> </completion>

**Lead resolution for verifier disagreements:** If verifiers flag a disagreement, read both positions and the relevant code. Rule in favour of the position with more specific evidence. If evidence is equal, rule NOT_FIXED — it's cheaper to re-fix than to miss a real issue. Note the disagreement in the iteration summary.

### Collect verification results

Wait for both verifiers to complete. Print: `[Verify] Iteration {n}: {verified} verified, {not_fixed} not fixed, {regressed} regressed.`

Synthesise:
- **VERIFIED** → done, remove from active list
- **NOT_FIXED** → goes back to Phase 3, preserving its original finding ID
- **REGRESSED** → goes back to Phase 3 with escalated severity, preserving its original finding ID. Escalate severity by one level: low → medium, medium → high, high → critical. Critical stays critical. A finding that REGRESSed in this iteration should be carried forward with its original ID — do not assign a new ID to a regressed finding.
- **New issues (medium+)** → added to the fix list for Phase 3. New IDs are only for issues first discovered during verification that were not in the original finding set.

Shut down verifiers and clean up the team.

---

## Phase 5: LOOP OR FINISH

Count remaining items: NOT_FIXED + REGRESSED + new medium+ issues. Exclude escalated findings — they have been removed from the loop.

### Oscillation check (runs first, before any other loop decision)

On the first iteration (iteration 1), there is no previous list — skip the oscillation check and proceed directly to the loop/stop decision.

On iteration 2+, two checks fire. Either one causes immediate escalation to the user, even if max iterations hasn't been reached.

**Check 1 — ID-based.** Compare the current NOT_FIXED/REGRESSED finding IDs against the previous iteration's list. If any finding ID (e.g., R3, R7) appears as NOT_FIXED or REGRESSED in **two consecutive iterations**, stop.

**Check 2 — Severity-weighted count.** Compare the total unresolved count against the previous iteration's, weighted by severity: critical = 4, high = 3, medium = 2, low = 1. If the weighted total has not decreased, stop. This catches new-issue chains (fixing R1 creates N1, fixing N1 creates N2 — different IDs but total never drops) while allowing progress where a high-severity fix introduces a low-severity side effect.

Report to the user:

> "Stopping: oscillation detected. {Check 1: 'The following findings have failed to resolve in two consecutive iterations: {IDs}.' | Check 2: 'Unresolved count is not decreasing ({previous} → {current}). Fixes are likely creating new issues at the same rate they resolve old ones.'} This typically indicates a structural conflict. Please review these findings manually: {details of each unresolved finding, including severity, impact, and location}."

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
Findings escalated: {count, with IDs and scope assessments}
Findings remaining: {count, with IDs and reasons}

Files changed:
- {list}

Tests added/modified:
- {list}
```

If any findings remain unresolved after max iterations, list them with their current status and recommend next steps.

If any findings were escalated, present them in a separate section with the developer's scope assessment and a recommendation for how to approach the structural fix.

---

## Notes

- **Token cost.** Each phase spawns a new team. A full iteration uses ~8 context windows (3 reviewers + up to 4 developers + 2 verifiers (developer count varies by finding count and grouping)). Multiple iterations multiply this. Be aware of cost on large targets.
- **File conflicts.** The file declaration step in Phase 3 catches conflicts before code is written. If two developers need the same file, reassign before giving the go-ahead. This adds a coordination round-trip but prevents overwrites.
- **Reviewer lenses.** The three reference lenses cover most targets, but substitute or synthesise lenses when the target has a dominant domain concern. Always keep at least one correctness-style and one robustness-style lens.
- **Escalation.** Developers may escalate findings that require architectural changes beyond their scope. Escalated findings exit the fix loop and are presented to the user separately. This prevents narrow patches that mask deeper issues.
- **Teammate discipline.** The structured finding format and independence protocol rely on prompt compliance. If a reviewer posts vague findings or doesn't follow the format, message them with a reminder before proceeding.
- **Scope creep.** Developers are told to fix only assigned findings. If they message about other issues they noticed, acknowledge them but do not expand scope mid-iteration. Note them for a future run.
