---
name: loop-spec
description: "Create a technical specification with embedded research from a PRD for the Loop autonomous agent system. Use when you have a PRD and need to define HOW to build it - architecture, data model, API contracts, component design, and researched technical decisions. Triggers on: create spec from this prd, technical specification, how should we build this, architecture for, design spec, research how to build."
---

# Loop Technical Specification

Transform a PRD into a technical implementation specification with embedded research. This defines HOW we're going to build it — the architecture, data models, API contracts, component structure, and researched technical decisions.

**Important:** This skill requires an existing PRD. Use the `loop-prd` skill first if you don't have one.

---

## The Job

1. Check for a prior PRD document (input chaining — see below)
2. Identify technical decisions needed (marked as NEEDS CLARIFICATION)
3. **Phase 1 (Interactive):** Ask user up to 3 technical clarification questions
4. **Phase 2 (Autonomous):** Research technical decisions using Context7 MCP + WebSearch
5. **Phase 3 (Output):** Produce a single spec with research findings embedded inline
6. Translate quality gate intent from PRD into specific commands
7. Output to `loop-output/spec-[ISO-timestamp].md`

**Important:** Do NOT start implementing. Just create the technical spec.

---

## Input Chaining

On invocation, scan `loop-output/` for the most recent `prd-*.md` file.

**If found:**
1. Show the filename and date to the user (e.g., "I found `prd-2025-06-15T14-32-18Z.md` from June 15. Use this as the basis for the spec?")
2. If confirmed, read the PRD and use it as the primary input
3. Also check for a recent `concept-*.md` for additional context

**If not found:**
- Ask the user to provide a PRD or point to one. This skill needs a PRD to work from.

---

## Phase 1: Interactive — Technical Clarifications

Scan the PRD and the existing codebase to identify technical decisions needed.

### Technical Context Checklist

For each item, either specify the choice OR mark `[NEEDS CLARIFICATION: specific question]`:

```markdown
## Technical Context

| Aspect | Decision |
|--------|----------|
| Language/Runtime | [e.g., TypeScript 5.x, Node 20] |
| Framework | [e.g., Next.js 14, Express] |
| Database | [e.g., PostgreSQL, SQLite, none] |
| ORM/Query | [e.g., Drizzle, Prisma, raw SQL] |
| State Management | [e.g., React state, Zustand, Redux] |
| Testing | [e.g., Vitest, Jest, Playwright] |
| Deployment | [e.g., Vercel, Docker, local only] |
```

**Maximum 3 NEEDS CLARIFICATION markers.** Make informed defaults for everything else based on:
- Existing codebase patterns (check package.json, existing code)
- Industry standards for the project type
- Simplest solution that meets requirements

### Ask Clarifying Questions

If there are NEEDS CLARIFICATION items, present them as multiple-choice options:

```
1. Which database approach should we use?
   A. PostgreSQL with Drizzle ORM (recommended for production)
   B. SQLite for simplicity
   C. In-memory/file-based (no database)
   D. Other: [specify]

2. How should we handle authentication?
   A. Session-based with cookies
   B. JWT tokens
   C. OAuth only (delegate to provider)
   D. None needed for this feature
```

Wait for answers before proceeding. User can respond with "1A, 2B" format.

---

## Phase 2: Autonomous — Research

After resolving clarifications, research technical decisions autonomously. The user does not need to interact during this phase.

### Research Workflow

For each technical decision that benefits from validation:

#### A. Context7 Lookup (Official Documentation)

```
1. Use mcp__context7__resolve-library-id to find the library ID
   - Query: "[library name]"

2. Use mcp__context7__query-docs to get current documentation
   - Query: "[specific question about the topic]"
```

#### B. Web Search (Current Best Practices)

```
Use WebSearch for:
- "[topic] best practices [current year]"
- "[topic] vs [alternative] comparison [current year]"
- "[framework] [pattern] current recommended approach"
```

**Important:** Always include the current year in search queries to get up-to-date results.

#### C. Embed Findings Inline

Do NOT produce a separate research.md. Instead, embed research findings directly in the relevant spec sections:

- In **Technical Context**, add a "Rationale" column with research-backed reasoning
- In **Architecture Overview**, cite patterns found in official docs
- In **Data Model**, reference current ORM documentation
- Add a **Research Notes** subsection under any section where research informed the decision

### What to Research

Prioritize research for:
1. Library/framework version-specific patterns
2. Best practices that may have changed recently
3. Decisions where multiple valid approaches exist
4. Integration patterns between chosen technologies

Skip research for:
- Obvious choices dictated by existing codebase
- Simple CRUD patterns with well-established approaches
- Decisions the user has already made explicitly

---

## Phase 3: Output — Technical Specification

### Quality Gates: Translate Intent to Commands

Read the Quality Gate Intent section from the PRD. Based on the project's tech stack (detected from package.json, config files, or user input), translate each intent to specific commands:

| PRD Intent | Example Translation |
|------------|-------------------|
| "Must pass type checking" | `pnpm typecheck` or `npx tsc --noEmit` |
| "Must pass linting" | `pnpm lint` or `npx eslint .` |
| "Must pass automated tests" | `pnpm test` or `npx vitest run` |
| "UI must be visually verified" | "Verify in browser using dev-browser skill" |

Include the resolved commands in the spec's Quality Gates section.

### Spec Structure

```markdown
# Technical Specification: [Feature Name]

**Branch:** `loop/[feature-name]`
**Date:** [YYYY-MM-DD]
**PRD:** [path to PRD in loop-output/]
**Status:** Draft | Review | Approved

---

## Summary

[2-3 sentences: What we're building and the primary technical approach]

---

## Technical Context

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Language | TypeScript 5.x | Existing codebase |
| Framework | Next.js 14 | App router for server components |
| Database | PostgreSQL | Production-ready, existing setup |
| ... | ... | ... |

> **Research note:** [Any relevant finding that informed a decision above]

---

## Quality Gates

These commands must pass for every user story:
- `[specific command]` - [what it checks]
- `[specific command]` - [what it checks]

For UI stories, also include:
- [specific verification method]

---

## Architecture Overview

[High-level description of components and how they interact]

### Component Diagram (Text)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Client    │────▶│   Server    │────▶│  Database   │
│  (React)    │     │  (Actions)  │     │ (Postgres)  │
└─────────────┘     └─────────────┘     └─────────────┘
```

---

## Data Model

### Entities

#### [Entity Name]
| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | uuid | PK | Unique identifier |
| name | string | NOT NULL, max 255 | Display name |
| status | enum | DEFAULT 'pending' | pending, active, done |
| created_at | timestamp | NOT NULL | Creation time |
| updated_at | timestamp | NOT NULL | Last update |

**Relationships:**
- [Entity] has many [OtherEntity]
- [Entity] belongs to [ParentEntity]

**Indexes:**
- `idx_entity_status` on (status)
- `idx_entity_user` on (user_id)

---

## API Contracts

### [Endpoint Group]

#### `POST /api/[resource]`
**Purpose:** Create a new [resource]

**Request:**
```typescript
{
  name: string;       // Required, max 255 chars
  status?: string;    // Optional, defaults to 'pending'
}
```

**Response (201):**
```typescript
{
  id: string;
  name: string;
  status: string;
  createdAt: string;  // ISO 8601
}
```

**Errors:**
- 400: Invalid input (validation failed)
- 401: Unauthorized
- 500: Server error

---

## Component Structure

### New Components

| Component | Location | Purpose |
|-----------|----------|---------|
| FeatureList | `components/feature/` | Display list of items |
| FeatureForm | `components/feature/` | Create/edit form |

### Modified Components

| Component | Changes |
|-----------|---------|
| Header | Add navigation link |
| Sidebar | Add feature section |

---

## File Changes

### New Files
- `app/feature/page.tsx` - Main feature page
- `app/api/feature/route.ts` - API endpoints
- `components/feature/FeatureList.tsx` - List component

### Modified Files
- `lib/db/schema/index.ts` - Export new schema
- `app/layout.tsx` - Add navigation

---

## State Management

- Server state: [Approach]
- Client state: [Approach]
- Form state: [Approach]

---

## Error Handling

| Scenario | Handling |
|----------|----------|
| Network failure | Show retry button, preserve input |
| Validation error | Display inline field errors |
| Auth failure | Redirect to login |
| Not found | Show 404 component |

---

## Security Considerations

- [ ] Input validation on all endpoints
- [ ] Authorization checks
- [ ] CSRF protection (if using forms)
- [ ] Rate limiting (if public API)

---

## Testing Strategy

| Type | Coverage | Tool |
|------|----------|------|
| Unit | Business logic, utilities | Vitest |
| Integration | API routes, database | Vitest |
| E2E | Critical user flows | Playwright |

### Key Test Cases
1. [Test case from acceptance criteria]
2. [Edge case handling]

---

## Open Questions

- [Any remaining technical questions]
- [Dependencies on other teams/features]
```

---

## Spec Quality Checklist

Before saving, verify:

- [ ] Input chaining: checked for PRD in `loop-output/`
- [ ] All NEEDS CLARIFICATION resolved (max 3 asked as questions)
- [ ] Research performed for non-obvious technical decisions
- [ ] Research findings embedded inline (no separate research.md)
- [ ] Quality gate intent translated to specific commands
- [ ] Data model covers all entities from PRD
- [ ] API contracts cover all functional requirements
- [ ] Component structure matches user stories
- [ ] Error handling defined for each failure mode
- [ ] Testing strategy covers acceptance criteria
- [ ] Security considerations addressed
- [ ] No vague statements ("works correctly", "handles errors")
- [ ] Output saved to `loop-output/spec-[ISO-timestamp].md`

---

## Output Location

Save to: `loop-output/spec-[ISO-timestamp].md` (e.g., `loop-output/spec-2025-06-15T14-32-18Z.md`)

---

## Example: Task Status Feature

**Input PRD excerpt:**
```markdown
## User Stories
### US-001: Add status field to tasks table
As a developer, I need to store task status in the database.

## Quality Gate Intent
- Must pass type checking
- Must pass linting
- UI stories must be visually verified
```

**Phase 1 — Clarification question:**
```
1. Which ORM should we use for the migration?
   A. Drizzle ORM (matches existing codebase)
   B. Prisma
   C. Raw SQL
   D. Other
```

**Phase 2 — Research (autonomous):**
- Context7: Query Drizzle docs for migration syntax
- WebSearch: "Drizzle ORM enum column best practices 2026"

**Phase 3 — Output spec excerpt:**
```markdown
## Technical Context

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| ORM | Drizzle | Existing codebase pattern; Context7 docs confirm enum support via pgEnum |

> **Research note:** Drizzle supports native PostgreSQL enums via `pgEnum()`. Confirmed in current docs — no need for string column with check constraint.

## Quality Gates

These commands must pass for every user story:
- `pnpm typecheck` - Type checking (detected from package.json scripts)
- `pnpm lint` - Linting (detected from package.json scripts)

For UI stories, also include:
- Verify in browser using dev-browser skill

## Data Model

### Task (modified)
| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | uuid | PK | Existing |
| title | string | NOT NULL | Existing |
| **status** | pgEnum | DEFAULT 'pending' | NEW: pending, in_progress, done |
| updated_at | timestamp | NOT NULL | Existing |
```
