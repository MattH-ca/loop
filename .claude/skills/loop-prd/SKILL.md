---
name: loop-prd
description: "Create a PRD through interactive questioning (use question mode whenever possible) for the Loop autonomous agent system. Use when the user wants to create a PRD, write product requirements, define a feature spec, or mentions 'user stories', 'acceptance criteria', or 'product requirements document'. Triggers on: create a prd, write requirements for, define this feature, I need a prd for, help me spec out."
model: opus
argument-hint: "path/to/concept-*.md"
---

# Loop PRD Creator

Create structured PRDs through adaptive questioning. The output PRD is designed for conversion to Loop's prd.json format for autonomous execution.

**Important:** This skill creates the PRD. To convert it to Loop's JSON format, use the `loop-task` skill.

---

## The Job

1. Check for a prior concept document (input chaining — see below)
2. Receive or confirm a feature description from the user
3. Ask essential clarifying questions (with lettered options) — one set at a time
4. Always ask about quality gate intent (what standards must be met)
5. After each answer, ask follow-up questions if needed (adaptive exploration)
6. Generate a structured PRD when you have enough context

**Important:** Do NOT start implementing. Just create the PRD.

---

## Input Chaining

On invocation, use $ARGUMENTS if not present then scan `loop-output/` for the most recent `concept-*.md` file.

**If found:**
1. Show the filename and date to the user (e.g., "I found `concept-2025-06-15T14-32-18Z.md` from June 15. Use this as context?")
2. If confirmed, read the concept document and use it as context

**If not found:**
- Explain to the User there is no argument.

---

## Step 1: Clarifying Questions (Iterative)

Ask questions one set at a time. Each answer should inform your next questions. THe concept will have some of this overred but you now need to expand.  Focus on:

- **Problem/Goal:** What problem does this solve? (skip if covered in concept doc)
- **Core Functionality:** What are the key actions?
- **Scope/Boundaries:** What should it NOT do?
- **Success Criteria:** How do we know it's done?
- **Integration:** How does it fit with existing features?
- **Success Metrics:** How do we know this worked? What does "good" look like?
- **Scope & Timeline:** Is this a weekend project or a full product? Any deadlines?
- **Constraints:** Budget, time, resources, regulatory limits?
- **Quality Gate Intent:** What quality standards must every story meet? (REQUIRED)


### Quality Gate Intent Question (REQUIRED)

Ask about quality standards in terms of **intent**, not specific commands. The spec phase will translate these to concrete commands based on the tech stack.

```
What quality standards must every user story meet?

Examples:
   A. Must pass type checking
   B. Must pass linting
   C. Must pass automated tests
   D. UI changes must be visually verified


```

### Adaptive Questioning

After each response, decide whether to:

- **Ask follow-up questions** (if answers reveal complexity)
- **Ask about a new aspect** (if current area is clear)
- **Generate the PRD** (if you have enough context)

Typically 4-8 rounds of questions are needed. Fewer if a concept doc was loaded.

---

## Step 2: PRD Structure

Generate the PRD with these sections:

### 1. Overview
Brief description of the feature and the problem it solves.

### 2. Goals
Specific, measurable objectives (bullet list).

### 3. Quality Gate Intent

**CRITICAL:** Capture the quality standards that must be met for every user story. Express these as intent, not specific commands — the spec phase will resolve them to concrete commands.

```markdown
## Quality Gate Intent

Every user story must meet these standards:
- Must pass type checking
- Must pass linting
- UI stories must be visually verified in browser
```

This section is consumed by the spec phase (loop-spec) to determine the exact commands for the project's tech stack.

### 4. User Stories

Each story needs:

- **Title:** Short descriptive name
- **Description:** "As a [user], I want [feature] so that [benefit]"
- **Acceptance Criteria:** Verifiable checklist of what "done" means

Each story should be small enough to implement in one focused AI agent session.

**Format:**
```markdown
### US-001: [Title]
**Description:** As a [user], I want [feature] so that [benefit].

**Acceptance Criteria:**
- [ ] Specific verifiable criterion
- [ ] Another criterion
```

**Note:** Do NOT include quality gate commands in individual story criteria — they are defined once in the Quality Gate Intent section and applied during task conversion.

**Important:**
- Acceptance criteria must be verifiable, not vague
- "Works correctly" is bad
- "Button shows confirmation dialog before deleting" is good
- Each story should be independently completable

### 5. Functional Requirements
Numbered list of specific functionalities:
- "FR-1: The system must allow users to..."
- "FR-2: When a user clicks X, the system must..."

Be explicit and unambiguous.

### 6. Non-Goals (Out of Scope)
What this feature will NOT include. Critical for managing scope.

### 87. Success Metrics
How will success be measured? (Absorbed from concept's "what good looks like" if available.)

### 8. Open Questions
Remaining questions or areas needing clarification.

---

## Writing for AI Agents

The PRD will be executed by AI coding agents via Loop. Therefore:

- Be explicit and unambiguous
- User stories should be small (completable in one session)
- Acceptance criteria must be machine-verifiable where possible
- Include specific file paths if you know them
- Reference existing code patterns in the project

---

## Output Format

Save to: `loop-output/prd-[ISO-timestamp].md` (e.g., `loop-output/prd-2025-06-15T14-32-18Z.md`)

```markdown
# PRD: Feature Name

## Overview
...

## Goals
...

## Quality Gate Intent
...

## User Stories
...

## Functional Requirements
...

## Non-Goals
...

## Success Metrics
...

## Open Questions
...
```


