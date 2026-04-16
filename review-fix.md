# Structured Review-Fix Loop

Independent structured review → triage → parallel fix → scoped verification → loop until clean.

**Target:** $ARGUMENTS

---

## Preflight

1. Check that Agent Teams is enabled. If you cannot create an agent team, tell the user to add `"CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"` to the `env` block in their settings.json, then stop.
2. Parse the target from the arguments above. If no target is specified, ask the user what to review.
3. Parse optional flags:
   - `--max-iterations N` — max review→fix loops (default: 3, minimum: 1). If 0 is passed, treat as `--review-only`.
   - `--auto` — skip triage approval, auto-fix all medium+ findings
   - `--review-only` — stop after Phase 1, don't fix anything
   - `--quick` — expands to `--max-iterations 2`. Print expanded values. Explicit flags override `--quick` values.
   - `--dry-run` — print the execution plan (reviewer count, estimated iterations, estimated context windows) and stop. No agents spawned.
4. Expand presets: if `--quick` is set, expand and print expanded values. Explicit flags override.
5. Validate parameters: `--max-iterations` ≥ 0 (0 treated as `--review-only`). Conflicts: `--review-only` + `--auto` is ambiguous — print error and stop. If invalid, print the constraint and a usage example, then stop.
6. If `--dry-run` is set, print the execution plan and stop:
   > **Dry-run plan:**
   > Reviewers: 3, Max iterations: {n}, Estimated context windows per iteration: ~8, Estimated input tokens: ~{iterations × 30}k
   > Flags: {all flags in effect}
   > Run again without --dry-run to execute.

---

## Phase 1: REVIEW

### Step 0: Select lenses

Select 3 lenses from the lens pool (Appendix B) using the decision table below. State selections and reasoning before spawning.

| Target characteristic | Lens recommendation | Rationale |
|---|---|---|
| Default (no strong signal) | correctness, robustness, quality | Baseline coverage for general code |
| Auth, secrets, or trust boundaries | correctness, robustness, **security** (swap quality) | Security concerns dominate quality concerns |
| New API surface or public interface | correctness, robustness, **API contract** (swap quality) | Interface stability matters more than internal quality |
| Data pipeline or ETL | correctness, **idempotency/precision**, robustness | Data correctness and replay safety are primary concerns |
| Performance-critical path | correctness, robustness, **performance** | Bottleneck code needs efficiency review |
| Pipeline-invoked with change context | correctness, robustness (keep defaults) | Scoped review; quality lens less relevant when reviewing targeted changes |

If none of the above match, synthesise a domain-specific lens. Synthesised lenses inherit the same protocol and scope constraints as pool lenses.

### Step 1: Spawn reviewers

Using the lenses selected in Step 0:

Print: `[Review] Spawning 3 reviewers ({lens1}, {lens2}, {lens3}).`

Create an agent team. Spawn 3 reviewers using the prompt in Appendix A, each with their selected lens.

### Step 2: Wait for independent findings

Wait for all 3 reviewers to post `[F{n}]` findings. Do not rush — wait until all 3 have posted.

**Compliance check:** Verify each post contains `[F{n}]` markers. If missing, send one correction: *"Please reformat using `[F{n}]` markers with Severity, Impact, Evidence, Confidence."* Proceed after one attempt.

Print: `[Review] All 3 reviewers posted findings. Starting cross-review.`

### Step 3: Cross-review

Broadcast: *"All independent analyses are in. Please read each other's findings and respond with CONFIRM, DISPUTE, or ADD."*

Allow 1-2 rounds of back-and-forth. If going in circles, broadcast: *"Please finalise your positions — we're moving to synthesis."*

### Step 4: Shut down and synthesise

Shut down all reviewers (`SendMessage` with `message: {type: "shutdown_request"}`), then `TeamDelete`.

Synthesise findings:
1. **Merge duplicates.** Same issue found by 2+ reviewers → one finding, note agreement count.
2. **Resolve disputes.** Disputed with evidence → downgrade or remove. Disputed without evidence → keep.
3. **Assign severity.** Multi-reviewer confirmation → higher confidence. Adjust if cross-review changed assessment.
4. **Sort.** Critical → high → medium → low.
5. **Assign IDs.** R1, R2, R3, etc.

Present as summary table:

| ID | Title | Severity | Agreed | Impact | Location |
|----|-------|----------|--------|--------|----------|

Print: `[Status] phase=review_synthesis findings={n} critical={n} high={n} medium={n} low={n}`

Each finding must have: ID, title, severity, impact, location, agreed count. Fix any gaps before proceeding.

### Checkpoint

Write to `.pipeline/checkpoint-{timestamp}.md`: `phase_completed: review`, `iteration: {n}`, `finding_count: {total}`, `findings: {IDs, titles, severities}`, `timestamp`, `pipeline_version: 1.0`. No code snippets or evidence — only structural metadata.

---

## Phase 2: TRIAGE

If **no findings**, report clean review and stop.

If `--review-only`, present findings and stop.

If `--auto`, select all medium+ findings and proceed. If only low findings with `--auto`, report for awareness and stop.

Otherwise, present to user:

> Here are the review findings. What would you like to fix?
> 1. All medium and above *(default)*
> 2. All findings including low severity
> 3. I'll pick — list the IDs to fix
> 4. None — stop here

Each finding passed to Phase 3 must include: ID, title, severity, impact, evidence (file:line), location.

---

## Phase 3: IMPLEMENT

### Step 1: Group findings by file ownership

Group so no two developers edit the same file (including test files). When in doubt, assign to one developer.

### Step 2: Spawn developers

Print: `[Status] phase=fix iteration={n} developers={m} findings={k}`

Create a new agent team. Spawn **1 developer per group** (max 4). If more than 4 groups, merge the smallest. Use the prompt in Appendix C.

### Step 3: File declaration coordination

After developers declare intended files, check for conflicts. If two developers claim the same file, reassign and notify both. Go-ahead only when conflict-free.

### Step 4: Wait and collect

Wait for all developers to complete. Collect summaries: which findings fixed, skipped, escalated.

**Escalated findings** exit the fix loop — presented to user in final summary with scope assessment.

Shut down developers, `TeamDelete`.

Each summary must contain: finding IDs fixed, skipped (with reasons), escalated (with scope), files changed, test results. Follow up if missing.

### Checkpoint

Write to `.pipeline/checkpoint-{timestamp}.md`: `phase_completed: fix_iteration_{n}`, `fixed_findings`, `skipped_findings`, `escalated_findings`, `files_changed`, `timestamp`, `pipeline_version: 1.0`.

---

## Phase 4: VERIFY

### Step 1: Spawn verifiers

Create a new agent team with **2 verifiers**. Use the prompt in Appendix D.

### Step 2: Collect results

Wait for both verifiers.

Print: `[Status] phase=verify iteration={n} verified={n} not_fixed={n} regressed={n}`

**Lead resolution for disagreements:** Rule in favour of more specific evidence. If equal, rule NOT_FIXED.

Synthesise:
- **VERIFIED** → done, remove from active list
- **NOT_FIXED** → back to Phase 3, keep original ID
- **REGRESSED** → back to Phase 3, escalate severity one level, keep original ID
- **New issues (medium+)** → added to fix list with new IDs

Shut down verifiers, `TeamDelete`.

---

## Phase 5: LOOP OR FINISH

Count remaining: NOT_FIXED + REGRESSED + new medium+ issues. Exclude escalated.

### Oscillation check (iteration 2+ only)

**Check 1 — ID-based.** Any finding ID NOT_FIXED or REGRESSED in two consecutive iterations → stop.

**Check 2 — Severity-weighted.** Weight: critical=4, high=3, medium=2, low=1. If weighted total hasn't decreased → stop.

Report: *"Stopping: oscillation detected. {details}. Please review these findings manually: {finding details}."*

### Loop or stop

**Items remain AND iteration < max AND no oscillation:** Report and return to Phase 3. Preserve original finding IDs. New issues get next available ID.

**All verified OR max reached OR no medium+ remain OR oscillation:** Present final summary.

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

List unresolved findings with status and next steps. List escalated findings with scope assessments.

---

## Appendix A: Reviewer Spawn Prompt

> You are a code reviewer with a specific focus area. Your job is to find problems, not propose fixes.
>
> **What to review:** {TARGET}
>
> **Your focus:** {LENS}
>
> ### Protocol — follow exactly
>
> **Step 1 — Independent analysis.** Read the target code. Form your findings on your own. DO NOT read messages from other teammates yet. Take your time.
>
> **Step 2 — Post findings.** Message all teammates. For each finding:
>
> ```
> [F{n}] {short title}
> Severity: critical | high | medium | low
> *(critical = data loss, security breach, or crash. high = incorrect behavior for common inputs. medium = incorrect behavior for edge cases or under load. low = style, naming, or convention violation.)*
> Impact: {what goes wrong if unfixed}
> Evidence: {file:line — quote the problematic code}
> Confidence: high | medium | low
> ```
>
> **Step 3 — Cross-review.** Read other reviewers' findings and respond:
> - **CONFIRM [Fn]** — agree, optionally add evidence
> - **DISPUTE [Fn]** — disagree with counter-evidence
> - **ADD** — new findings others missed
>
> **Step 4 — Mark complete.**
>
> **Rules:** Don't propose fixes. Don't review outside your lens. Be specific — "This could be improved" is not a finding.

---

## Appendix B: Reviewer Lenses

**correctness-reviewer:**
> Logic errors, edge cases, off-by-one errors, incorrect assumptions, missing boundary validation, race conditions, wrong return types, broken contracts between functions, state that can become inconsistent.

**robustness-reviewer:**
> Error handling gaps, failure modes, unvalidated external input, resource leaks, missing try/catch around I/O, exception swallowing, data integrity under partial failure, security concerns (injection, auth bypass, secrets exposure).

**quality-reviewer:**
> Unnecessary complexity, duplication that could cause drift, misleading names, dead code, missing or inadequate test coverage, unclear API boundaries, coupling that will make changes painful, violations of the project's own conventions.

Synthesise domain-specific lenses when needed (security reviewer, performance reviewer, precision reviewer). They follow the same protocol.

---

## Appendix C: Developer Spawn Prompt

> You are fixing specific code review findings. Fix only what's assigned — nothing else.
>
> <findings>
> {paste full finding details: ID, title, severity, impact, evidence, location}
> </findings>
>
> <rules>
> 1. **Fix only your findings.** Do not refactor, reformat, or improve surrounding code.
> 2. **Declare your files first.** Message the lead with all files you intend to modify or create (including test files). Wait for confirmation before proceeding.
> 3. **Write a test for each fix** that would have caught the original issue.
> 4. **ESCALATE, don't patch.** If a finding requires architectural changes or files owned by others: `ESCALATE [Fn] — {reason and scope}`. Do not attempt a partial fix.
> 5. Run the full test suite after changes. If unrelated tests fail, message the lead.
> 6. If you disagree with a finding, message the lead before skipping.
> </rules>
>
> <completion>
> Message the lead: finding IDs fixed, skipped (with reason), escalated (with scope), tests added, test results (pass/fail count). Mark task complete.
> </completion>

---

## Appendix D: Verifier Spawn Prompt

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
> 1. **Independent analysis first.** Form your own assessment before reading the other verifier's messages.
> 2. For each finding, report: **VERIFIED**, **NOT_FIXED**, or **REGRESSED**.
> 3. Check for **new issues** in changed code only. Report as `[N{n}]` with severity, impact, evidence.
>    *(critical = data loss, security breach, or crash. high = incorrect behavior for common inputs. medium = edge cases or under load. low = style or convention.)*
> 4. **Scope:** Only review changes and immediate context. Do NOT re-review the whole codebase.
> 5. **Cross-check.** After posting, read the other verifier's assessment. Discuss disagreements. If you can't agree, flag for the lead with both positions.
> </rules>
>
> <completion>
> Mark task complete when done.
> </completion>

---

## Notes

- **Token economics.** Each phase spawns a new team. A full iteration uses ~8 context windows (3 reviewers + up to 4 developers + 2 verifiers). Reviewers each receive ~3k tokens of protocol. Protocol overhead: ~25-30k input tokens per iteration. Iterations 2+ skip the review phase (only fix + verify), using ~6 context windows. For cost-sensitive use, target smaller files or use `--review-only`.
- **File conflicts.** The file declaration step catches conflicts before code is written. If two developers need the same file, reassign before giving the go-ahead.
- **Escalation.** Developers may escalate findings beyond their scope. Escalated findings exit the fix loop and are presented separately.
- **Non-compliance is accepted degraded behavior.** If a reviewer or developer fails format compliance after one correction, proceed with reduced coverage. Extract what you can, note the degradation. The `[Status]` line shows agent response counts.

---

## Appendix E: When Things Go Wrong

| Failure | Detection | Recovery | Degraded behavior |
|---------|-----------|----------|-------------------|
| Reviewer non-response | Other reviewers complete but one remains idle after nudge | Proceed with 2 reviewers | Reduced lens coverage; note in synthesis |
| Format non-compliance | Reviewer posts findings without `[F{n}]` markers or severity | Send one correction; if still non-compliant, manually extract findings | Lead synthesises from unstructured output |
| Team creation failure | `TeamCreate` returns an error | Retry once; if still failing, tell user to check Agent Teams config | Cannot proceed — stop and report |
| Developer file conflict | Two developers declare overlapping files | Reassign files before giving go-ahead; merge groups if needed | One developer handles both sets of findings |
| Checkpoint corruption on --resume | Required fields missing from checkpoint file (`phase_completed`, `finding_count`) | Abort resume with clear message; suggest re-running without --resume | Cannot resume — full restart required |
