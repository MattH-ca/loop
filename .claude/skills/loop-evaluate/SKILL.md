---
name: loop-evaluate
description: "Evaluate a completed Loop execution run: generate a closing report, audit the implementation against the spec, and promote to finalspec if the build matches. Use after Loop execution completes all stories. Triggers on: evaluate this run, closing report, audit the build, spec audit, promote to finalspec, loop evaluate."
---

# Loop Evaluate

Evaluate a completed Loop autonomous execution run. Generate a closing report on agent performance, audit the implementation against the original spec point-by-point, score deviations, and promote the spec to `finalspec` if the build is faithful.

**Important:** This skill requires a completed Loop execution. All stories in `prd.json` should have `passes: true`, and the codebase should be in its post-execution state.

---

## The Job

1. Check for prior artifacts (input chaining — see below)
2. Locate the target codebase where execution agents built the code
3. **Phase 1 (Closing Report):** Analyze execution artifacts and generate a performance report
4. **Phase 2 (Spec Audit):** Point-by-point comparison of spec vs actual implementation with scoring
5. **Phase 3 (Promotion Decision):** Promote spec to `finalspec` if audit passes, or present deductions
6. Output closing report to `loop-output/closing-report-[ISO-date].md`
7. Output `finalspec` to `loop-output/finalspec-[matching-spec-timestamp].md` if promoted

---

## Input Chaining

On invocation, scan `loop-output/` for these artifacts:

### Required
- Most recent `spec-*.md` — the specification given to execution agents
- Most recent `prd-*.json` — the task file with user stories

### Required from target codebase
- `progress.txt` — the execution progress log with iteration details
- All source files listed in the spec's module layout
- All test files

### Optional
- `prd-*.md` — the original PRD (for context)
- Screenshot evidence (if user provides)
- `finalspec-*.md` — if one already exists, this is a re-evaluation

**If spec or prd.json not found:**
- Ask the user to point to the artifacts. This skill cannot run without a spec to audit against.

**If target codebase path not known:**
- Check `prd.json` for project context, or ask the user where the code lives.

---

## Phase 1: Closing Report

Generate a comprehensive report on how the execution agents performed. This phase is **autonomous** — read all artifacts and produce the report without user interaction.

### Data Collection

Read these sources in parallel:

1. **`progress.txt`** — Extract per-iteration data:
   - Stories completed per iteration
   - Test counts (per iteration and cumulative)
   - Files changed per iteration
   - Learnings captured
   - Any rework or failed attempts

2. **`prd.json`** — Extract:
   - Total stories planned
   - Story pass/fail status
   - Story ordering and dependencies

3. **Target codebase** — Measure:
   - Source lines of code (per module)
   - Test lines of code (per test file)
   - Test-to-source ratio
   - Total test count from a `pytest` dry run or progress.txt

4. **Git log** (if available) — Extract:
   - Commit history on the execution branch
   - Commit messages and timestamps
   - Wall clock time (first commit to last)

### Report Structure

```markdown
# Loop Execution Report: [Project Name]

**Date:** [YYYY-MM-DD]
**Project:** [project field from prd.json]
**Codebase:** [path to target codebase]
**Branch:** [branchName from prd.json]

---

## Executive Summary

**[stories completed] user stories. [iterations] iterations. [test count] tests. [failures] failures. [first-run or rework needed].**

[2-3 sentence summary of the run]

---

## Pipeline Recap

| Phase | Artifact | Mode |
|-------|----------|------|
| **loop-idea** | `concept-*.md` | Interactive brainstorm |
| **loop-prd** | `prd-*.md` | Interactive PRD |
| **loop-spec** | `spec-*.md` | Technical spec with research |
| **loop-task** | `prd-*.json` | Executable stories |

---

## Iteration-by-Iteration Execution

| Iter | Story | What Shipped | Tests Added | Cumulative |
|------|-------|-------------|-------------|------------|
| 1 | US-001 | [summary] | [count] | [count] |
| ... | ... | ... | ... | ... |

---

## What the Agents Built

**[N] lines of source code** across [N] modules:

| Module | Lines | Responsibility |
|--------|-------|---------------|
| [module] | [lines] | [what it does] |

**[N] lines of tests** across [N] test files — a **[ratio]:1 test-to-source ratio**.

---

## Agent Behavioral Observations

### What went well
[Numbered list of positive patterns observed]

### Technical decisions the agents made well
[Bullet list of specific good decisions]

### Issues observed
[Bullet list of any problems, calibration issues, or UX gaps]

---

## Final Scorecard

| Metric | Value |
|--------|-------|
| Stories planned | [N] |
| Stories completed | [N] ([%]) |
| Iterations used | [N] |
| Total tests | [N] |
| Test pass rate | [%] |
| Source lines | [N] |
| Test lines | [N] |
| Test:source ratio | [ratio]:1 |
| Rework iterations | [N] |
| Wall clock time | [duration] |
| Quality gate | [command used] |

---

## Recommendations for Next Loop Run

[Numbered list of actionable recommendations for improving the next execution cycle. Focus on:]
1. PRD improvements (e.g., add codebase context section for existing projects)
2. Spec improvements (e.g., reference existing code patterns)
3. Task improvements (e.g., regression test gates)
4. Loop framework improvements (e.g., story-level rollback)
5. Code fixes to current build (e.g., data calibration issues)
```

### Report Quality Checklist

Before saving the closing report:
- [ ] Every iteration from progress.txt is accounted for
- [ ] Source line counts come from actual file reads, not estimates
- [ ] Test count matches progress.txt final cumulative number
- [ ] Recommendations are actionable and specific to this project
- [ ] No vague praise — observations cite specific code decisions or patterns

---

## Phase 2: Spec Audit

Systematic point-by-point comparison of the spec against the actual implementation. This is the core of the evaluation.

### Audit Method

**For every section of the spec**, read the corresponding source file(s) and verify each claim.

#### Reading Strategy

Read ALL source files listed in the spec's module layout. Do not sample — read every file. The audit must be exhaustive.

For each spec section, build a comparison table:

```markdown
| Spec Requires | Actual | Status |
|---------------|--------|--------|
| [specific requirement] | [what was found in code] | ✅ or ❌ |
```

### Scoring Rubric

**100-point scale distributed across spec sections:**

| Section | Points | What to Check |
|---------|--------|---------------|
| Module Layout | 10 | Every file in spec layout exists in codebase |
| Data Model | 15 | Struct formats, field mappings, type conversions, dataclass definitions |
| Database Schema | 15 | Tables, columns, types, constraints, indexes, default values |
| Component Structure | 25 | Function signatures, class methods, behavioral logic, code flow |
| API Contracts | 15 | Endpoints, HTTP methods, request/response shapes, status codes, error responses |
| Error Handling | 10 | Each error scenario from spec's error handling table is implemented |
| Testing Strategy | 5 | Test files exist, test categories covered, key test cases present |
| Security | 5 | Binding, parameterization, auth model, input validation |

### Scoring Rules

**Deduct points for:**
- Missing functionality specified in the spec
- Different behavior than what the spec requires
- Missing error handling that the spec defines
- Missing files from the module layout

**Do NOT deduct for:**
- Additions not in the spec (bonus work by agents — note these as "Enhancements")
- Agent fixing a spec bug (if the spec's suggested code was flawed but the functional requirement is met, the agent improved the spec — note as "Spec Bug Fixed")
- Different implementation approach that produces identical behavior (e.g., Python-side aggregation vs SQL aggregation when the API response contract is identical — note as "Alternate Approach")
- Style differences (naming, formatting) that don't affect functionality

### Distinguishing Deviation Types

Classify each finding as one of:

| Type | Description | Deduction? |
|------|-------------|------------|
| **Match** | Spec requirement met exactly | No |
| **Enhancement** | Agent added functionality beyond spec | No |
| **Spec Bug Fixed** | Agent corrected a flawed spec suggestion while meeting the functional requirement | No |
| **Alternate Approach** | Different implementation, identical behavior | No |
| **Minor Deviation** | Spec requirement partially met or edge case missed | Yes (proportional) |
| **Major Deviation** | Spec requirement not met, affects core functionality | Yes (full section weight) |
| **Missing** | Spec requirement completely absent | Yes (full item weight) |

### Audit Output Format

```markdown
## Spec vs Implementation Audit

**Spec:** `loop-output/spec-[timestamp].md`
**Codebase:** [path]
**Scoring:** 100-point scale across 8 spec sections

---

### Section 1: [Section Name] ([N] pts)

| Spec Requires | Actual | Status |
|---------------|--------|--------|
| [requirement] | [finding] | ✅ / ❌ |

**Score: [X]/[N]**

[Repeat for each section]

---

## Final Scorecard

| Section | Points | Score |
|---------|--------|-------|
| [section] | [max] | [actual] |
| **Total** | **100** | **[score]** |

---

## Deductions Summary

| # | Deviation | Points | Severity |
|---|-----------|--------|----------|
| 1 | [description] | -[N] | Minor/Major |

**Total deductions: -[N] points**

---

## Enhancements (No Deductions)

[Numbered list of things agents added beyond the spec]

## Spec Bugs Fixed (No Deductions)

[Numbered list of spec flaws the agents correctly worked around]
```

---

## Phase 3: Promotion Decision

### If score = 100/100 (zero deductions)

1. Copy spec to `loop-output/finalspec-[same-timestamp-as-spec].md`
2. Change `**Status:** Draft` to `**Status:** Final` in the finalspec
3. Report: "Spec promoted to finalspec. Zero deductions — implementation matches specification exactly."

### If score >= 90/100 (minor deductions only)

Present the deductions to the user with this framing:

> The deductions are against the **code**, not the spec. The spec correctly specified [N] behaviors that the agents didn't fully implement. These are execution gaps, not spec deficiencies.
>
> **Score: [X]/100** — [N] deductions, all in [section(s)].
>
> The spec qualifies for `finalspec` promotion — the deductions represent code bugs to fix, not spec problems. Promote?

If user confirms, promote. Log deductions as tech debt in the closing report's recommendations.

### If score < 90/100 (major deductions)

Present the deductions and recommend a fix cycle:

> **Score: [X]/100** — [N] deductions across [section(s)].
>
> [List deductions with severity]
>
> Recommendation: Fix the [major/minor] deviations before promoting to finalspec. These can be addressed in the next Loop execution cycle as a focused fix PRD.

Do NOT auto-promote. The user decides.

---

## Output Locations

| Artifact | Path | When |
|----------|------|------|
| Closing Report | `loop-output/closing-report-[ISO-date].md` | Always |
| Final Spec | `loop-output/finalspec-[spec-timestamp].md` | When promoted |

The finalspec timestamp matches the original spec's timestamp (not the current time) — this preserves traceability back to the spec that was used during execution.

---

## Checklist Before Completing

- [ ] **Input chaining**: found spec, prd.json, progress.txt, and target codebase
- [ ] **Closing report**: all iterations accounted for, line counts verified, recommendations specific
- [ ] **Audit**: every spec section scored, every source file read (not sampled)
- [ ] **Deductions**: each one cites the specific spec line/section and the specific code location
- [ ] **Enhancements**: agent bonus work acknowledged (no penalty)
- [ ] **Spec bugs**: agent fixes acknowledged (no penalty)
- [ ] **Promotion decision**: presented with correct framing (code gaps vs spec gaps)
- [ ] **Closing report saved** to `loop-output/closing-report-[ISO-date].md`
- [ ] **Finalspec saved** (if promoted) to `loop-output/finalspec-[spec-timestamp].md` with Status: Final
