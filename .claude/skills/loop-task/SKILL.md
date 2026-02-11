---
name: loop-task
description: "Convert PRDs to prd.json format for the Loop autonomous agent system. Use when you have an existing PRD and need to convert it to Loop's JSON format. Triggers on: convert this prd, turn this into loop format, create prd.json from this, loop json, create tasks."
---

# Loop PRD Converter

Converts existing PRDs (and optional specs) to the prd.json format that Loop uses for autonomous execution.

---

## The Job

1. Check for prior artifacts (input chaining — see below)
2. Take a PRD and optional spec and convert to `prd.json` in `loop-output/`
3. Add `references` pointing to relevant spec sections
4. Pre-populate `notes` with implementation hints from the spec

---

## Input Chaining

On invocation, scan `loop-output/` for the most recent `prd-*.md` AND `spec-*.md` files.

**If found:**
1. Show the filenames and dates to the user (e.g., "I found `prd-2025-06-15T14-32-18Z.md` and `spec-2025-06-15T15-00-00Z.md`. Use these?")
2. If confirmed, read both artifacts as input
3. Also check for a recent `concept-*.md` for additional context

**If PRD found but no spec:**
- Proceed with just the PRD. The `references` field will be empty arrays and `notes` will be minimal.

**If nothing found:**
- Ask the user to provide a PRD (markdown file or text).

---

## Output Format

```json
{
  "project": "[Project Name]",
  "branchName": "loop/[feature-name-kebab-case]",
  "description": "[Feature description from PRD title/intro]",
  "userStories": [
    {
      "id": "US-001",
      "title": "[Story title]",
      "description": "As a [user], I want [feature] so that [benefit]",
      "acceptanceCriteria": [
        "Criterion 1",
        "Criterion 2",
        "Typecheck passes"
      ],
      "references": [
        "loop-output/spec-TIMESTAMP.md#data-model",
        "loop-output/spec-TIMESTAMP.md#api-contracts"
      ],
      "notes": "Implementation hint from spec: use pgEnum for status field. Follow existing Card component pattern.",
      "priority": 1,
      "passes": false
    }
  ]
}
```

### Field Details

- **references**: Array of markdown anchor references pointing to relevant spec sections. Format: `"loop-output/spec-TIMESTAMP.md#section-anchor"`. Empty array if no spec available.
- **notes**: Pre-populated with implementation hints derived from the spec (e.g., "Use Drizzle migration pattern", "Component follows existing Card pattern"). Executing agents append their own learnings during implementation. Empty string if no spec available.

---

## Story Size: The Number One Rule

**Each story must be completable in ONE Loop iteration (one context window).**

Loop spawns a fresh instance per iteration with no memory of previous work. If a story is too big, the LLM runs out of context before finishing and produces broken code.

### Right-sized stories:
- Add a database column and migration
- Add a UI component to an existing page
- Update a server action with new logic
- Add a filter dropdown to a list

### Too big (split these):
- "Build the entire dashboard" - Split into: schema, queries, UI components, filters
- "Add authentication" - Split into: schema, middleware, login UI, session handling
- "Refactor the API" - Split into one story per endpoint or pattern

**Rule of thumb:** If you cannot describe the change in 2-3 sentences, it is too big.

---

## Story Ordering: Dependencies First

Stories execute in priority order. Earlier stories must not depend on later ones.

**Correct order:**
1. Schema/database changes (migrations)
2. Server actions / backend logic
3. UI components that use the backend
4. Dashboard/summary views that aggregate data

**Wrong order:**
1. UI component (depends on schema that does not exist yet)
2. Schema change

---

## Acceptance Criteria: Must Be Verifiable

Each criterion must be something Loop can CHECK, not something vague.

### Good criteria (verifiable):
- "Add `status` column to tasks table with default 'pending'"
- "Filter dropdown has options: All, Active, Completed"
- "Clicking delete shows confirmation dialog"
- "Typecheck passes"
- "Tests pass"

### Bad criteria (vague):
- "Works correctly"
- "User can do X easily"
- "Good UX"
- "Handles edge cases"

### Always include as final criterion:
```
"Typecheck passes"
```

For stories with testable logic, also include:
```
"Tests pass"
```

### For stories that change UI, also include:
```
"Verify in browser using dev-browser skill"
```

Frontend stories are NOT complete until visually verified by the agent. Loop will use the dev-browser skill to navigate to the page, interact with the UI, and confirm changes work.

---

## References: Linking Stories to Spec Sections

When a spec is available, link each story to the relevant spec sections:

1. Read the spec and identify section anchors (e.g., `#data-model`, `#api-contracts`, `#component-structure`)
2. For each story, determine which spec sections are most relevant
3. Add those as entries in the `references` array

**Example:**
```json
"references": [
  "loop-output/spec-2025-06-15T15-00-00Z.md#data-model",
  "loop-output/spec-2025-06-15T15-00-00Z.md#api-contracts"
]
```

This gives executing agents direct pointers to the technical details they need.

---

## Notes: Implementation Hints

When a spec is available, pre-populate notes with actionable hints:

- Mention specific patterns from the spec (e.g., "Use Drizzle `pgEnum()` for status field")
- Reference existing codebase patterns (e.g., "Follow existing Card component in `components/tasks/`")
- Call out gotchas from the spec's research notes (e.g., "SSE requires ReadableStream API, not ws library")

Keep notes concise — 1-3 sentences. Executing agents will append their own learnings during implementation.

---

## Conversion Rules

1. **Each user story becomes one JSON entry**
2. **IDs**: Sequential (US-001, US-002, etc.)
3. **Priority**: Based on dependency order, then document order
4. **All stories**: `passes: false`
5. **references**: Populated from spec section anchors (empty array if no spec)
6. **notes**: Pre-populated from spec hints (empty string if no spec)
7. **branchName**: Derive from feature name, kebab-case, prefixed with `loop/`
8. **Always add**: "Typecheck passes" to every story's acceptance criteria

---

## Splitting Large PRDs

If a PRD has big features, split them:

**Original:**
> "Add user notification system"

**Split into:**
1. US-001: Add notifications table to database
2. US-002: Create notification service for sending notifications
3. US-003: Add notification bell icon to header
4. US-004: Create notification dropdown panel
5. US-005: Add mark-as-read functionality
6. US-006: Add notification preferences page

Each is one focused change that can be completed and verified independently.

---

## Output Location

Save to: `loop-output/prd-[ISO-timestamp].json` (e.g., `loop-output/prd-2025-06-15T16-00-00Z.json`)

---

## Example

**Input PRD:** `loop-output/prd-2025-06-15T14-32-18Z.md`
**Input Spec:** `loop-output/spec-2025-06-15T15-00-00Z.md`

**Output `loop-output/prd-2025-06-15T16-00-00Z.json`:**
```json
{
  "project": "TaskApp",
  "branchName": "loop/task-status",
  "description": "Task Status Feature - Track task progress with status indicators",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add status field to tasks table",
      "description": "As a developer, I need to store task status in the database.",
      "acceptanceCriteria": [
        "Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')",
        "Generate and run migration successfully",
        "Typecheck passes"
      ],
      "references": [
        "loop-output/spec-2025-06-15T15-00-00Z.md#data-model"
      ],
      "notes": "Use Drizzle pgEnum() for the status column. See spec research note on native PostgreSQL enum support.",
      "priority": 1,
      "passes": false
    },
    {
      "id": "US-002",
      "title": "Display status badge on task cards",
      "description": "As a user, I want to see task status at a glance.",
      "acceptanceCriteria": [
        "Each task card shows colored status badge",
        "Badge colors: gray=pending, blue=in_progress, green=done",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "references": [
        "loop-output/spec-2025-06-15T15-00-00Z.md#component-structure"
      ],
      "notes": "Follow existing Card component pattern. StatusBadge goes in components/tasks/.",
      "priority": 2,
      "passes": false
    },
    {
      "id": "US-003",
      "title": "Add status toggle to task list rows",
      "description": "As a user, I want to change task status directly from the list.",
      "acceptanceCriteria": [
        "Each row has status dropdown or toggle",
        "Changing status saves immediately",
        "UI updates without page refresh",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "references": [
        "loop-output/spec-2025-06-15T15-00-00Z.md#api-contracts",
        "loop-output/spec-2025-06-15T15-00-00Z.md#component-structure"
      ],
      "notes": "Use PATCH /api/tasks/:id/status endpoint defined in spec. StatusDropdown component.",
      "priority": 3,
      "passes": false
    },
    {
      "id": "US-004",
      "title": "Filter tasks by status",
      "description": "As a user, I want to filter the list to see only certain statuses.",
      "acceptanceCriteria": [
        "Filter dropdown: All | Pending | In Progress | Done",
        "Filter persists in URL params",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "references": [
        "loop-output/spec-2025-06-15T15-00-00Z.md#component-structure",
        "loop-output/spec-2025-06-15T15-00-00Z.md#state-management"
      ],
      "notes": "Use URL search params for filter state. See spec state management section.",
      "priority": 4,
      "passes": false
    }
  ]
}
```

---

## Archiving Previous Runs

**Before writing a new prd.json, check if there is an existing one from a different feature:**

1. Read the current prd.json in `loop-output/` if it exists
2. Check if `branchName` differs from the new feature's branch name
3. If different AND `progress.txt` has content beyond the header:
   - Create archive folder: `archive/YYYY-MM-DD-feature-name/`
   - Copy current prd.json and `progress.txt` to archive
   - Reset `progress.txt` with fresh header

---

## Checklist Before Saving

Before writing prd.json, verify:

- [ ] **Input chaining**: checked for PRD and spec in `loop-output/`
- [ ] **Previous run archived** (if prd.json exists with different branchName, archive it first)
- [ ] Each story is completable in one iteration (small enough)
- [ ] Stories are ordered by dependency (schema to backend to UI)
- [ ] Every story has "Typecheck passes" as criterion
- [ ] UI stories have "Verify in browser using dev-browser skill" as criterion
- [ ] Acceptance criteria are verifiable (not vague)
- [ ] No story depends on a later story
- [ ] `references` populated from spec (or empty arrays if no spec)
- [ ] `notes` pre-populated with implementation hints (or empty strings if no spec)
- [ ] Output saved to `loop-output/prd-[ISO-timestamp].json`
