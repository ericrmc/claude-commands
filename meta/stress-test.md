# Stress Test

Structural stress-test for command files and process designs. Finds where the mechanism breaks — not whether the ideas are good, but whether the process that generates them is sound.

**Target:** $ARGUMENTS

---

## Preflight

1. Parse the target from the arguments above. The target should be a file path to a command file (`.md`), a prompt, or a process description. If none specified, ask the user what to stress-test.
2. Read the target file in full. You need to understand every rule, constraint, parameter, and implicit assumption.

---

## What This Is

This is not a code review or a content critique. This is a **mechanism audit**. You are looking for structural problems — places where the design's own rules conflict, where parameters hit invisible ceilings, where assumptions hold at defaults but fail at edges.

The agents inside a process can challenge each other's *ideas* but cannot challenge the *process itself*. They operate inside the frame the prompt sets. This tool operates outside that frame.

---

## Analysis Phases

Work through each phase sequentially. Write your findings as you go — do not wait until the end.

### Phase 1: Parameter Boundaries

Identify every configurable parameter (explicit flags, defaults, and implicit constants like pool sizes or list lengths).

For each parameter:
- What is its default value?
- What is its practical maximum before something breaks?
- What breaks first — a rule violation, resource exhaustion, or logical contradiction?
- Does the interface (flags, documentation) advertise limits that the mechanism can't actually support?

Test combinations, not just individual parameters. Two parameters that are fine alone may conflict at scale (e.g., rounds × agents > role pool size).

### Phase 2: Rule Consistency

List every rule, constraint, or protocol instruction in the target.

For each pair of rules, ask:
- Can both be satisfied simultaneously under all parameter values?
- Is there a parameter setting where rule A forces a violation of rule B?
- Are there implicit rules (unstated assumptions that the mechanism relies on)?

Pay special attention to:
- Uniqueness constraints vs. finite pools
- Ordering constraints vs. round counts
- Threshold rules vs. agent counts (e.g., "majority" with 2 agents)

### Phase 3: Degenerate Cases

Test what happens at the extremes:
- Minimum viable run (smallest possible parameters) — does the process still produce meaningful output?
- Single-agent case — do voting/challenge mechanics still work?
- Maximum parameters — where does it first become incoherent?
- Adversarial input — what if the problem statement is empty, absurdly broad, or self-referential?

### Phase 4: Structural Blind Spots

Ask what the process **cannot** surface by design:
- What categories of insight are structurally excluded by the role definitions, phase ordering, or information flow?
- If agents only see X, what problems require seeing Y to detect?
- Where does the file-based handoff lose important information?
- Are there feedback loops that should exist but don't? (e.g., can later phases inform earlier ones?)

### Phase 5: Failure Modes Under Real Use

Consider how the process behaves when things go wrong in practice:
- What happens when an agent doesn't respond or gives low-quality output?
- What happens when all agents converge too quickly (groupthink)?
- What happens when no ideas survive a round?
- Is there a recovery path, or does the process just produce garbage?

---

## Output

Write findings to `stress-test-output.md` in the working directory (or `--output PATH` if specified):

```markdown
# Stress Test: {target file name or description}

**Date:** {date}
**Target:** {file path or description}

---

## Critical Issues

Problems that will cause the mechanism to fail or produce incorrect results under realistic parameter settings.

### {issue title}
**Trigger:** {specific parameter values or conditions}
**What breaks:** {which rule, constraint, or assumption fails}
**Suggested fix:** {concrete change to the target}

---

## Structural Warnings

Problems that degrade quality or limit scalability but don't cause outright failure.

### {issue title}
**Trigger:** {conditions}
**Effect:** {what degrades}
**Suggested fix:** {concrete change}

---

## Blind Spots

Things the process cannot surface by design.

### {blind spot}
**Why it's hidden:** {structural reason}
**When it matters:** {scenario where this becomes a problem}
**Possible mitigation:** {how to address, if possible}

---

## Parameter Limits

| Parameter | Default | Practical Max | What Breaks First |
|-----------|---------|---------------|-------------------|
| {name} | {value} | {value} | {description} |

---

## Edge Case Behaviour

| Scenario | Expected | Actual | Verdict |
|----------|----------|--------|---------|
| {scenario} | {what should happen} | {what the mechanism actually does} | {ok / broken / degraded} |
```

### Present to user

Show:
- Count of critical issues, warnings, and blind spots
- The top 2-3 critical issues with one-line descriptions
- The output file path for full details
