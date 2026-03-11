# Loop

An autonomous AI coding agent pipeline built on Claude Code. You describe what you want to build through a series of guided skills, then Loop executes it — story by story, commit by commit — with no human in the loop.

## How It Works

Loop has two phases: **design** (you + AI collaborating) and **execution** (AI working autonomously).

During design, you walk through 4 interactive skills that progressively refine a raw idea into an implementation-ready task list. Each skill produces a revisioned artifact in `loop-output/` and can iterate on itself — refining the document into successive minor revisions (0A → 0B → 0C) before you move on. When you're satisfied, the next skill chains forward by finding the highest-revision prior artifact and asking you to confirm.

During execution, `loop.sh` spawns Claude Code in a headless loop. Each iteration picks up the next unfinished user story from `prd.json`, implements it, runs quality checks, commits, marks the story as done, and logs what it learned. The next iteration reads those learnings before starting. When every story passes, the loop stops.

After execution, an optional evaluation skill audits the implementation against the original spec, scores deviations, and can promote the spec to a `finalspec` if the build is faithful.

```mermaid
flowchart TD
    subgraph design["Design Phase — Interactive"]
        A["↻ 1. loop-idea<br/>→ concept.md"]
        B["↻ 2. loop-prd<br/>→ prd.md"]
        C["↻ 3. loop-spec<br/>→ spec.md"]
        D["↻ 4. loop-task<br/>→ prd.json"]
        A --> B --> C --> D
    end

    NOTE["↻ = refine before advancing<br/>Each step writes new revisions:<br/>0A → 0B → 0C"]
    B -.- NOTE

    subgraph exec["Execution Phase — Autonomous"]
        E["Run loop.sh"]
        F["Pick next story"]
        G["Implement + quality gates"]
        H["Commit, mark passes: true"]
        I["Log learnings"]
        K{"More stories?"}
    end

    subgraph eval["Evaluation Phase — Post-Execution"]
        M["6. loop-evaluate<br/>Audit spec vs code"]
        N["Closing report + score"]
        O{"Promote?"}
        P["finalspec.md"]
    end

    D --> E
    E --> F --> G --> H --> I --> K
    K -- "Yes" --> F
    K -- "No" --> L["All stories complete"]
    L -.-> M
    M --> N --> O
    O -- "Yes" --> P
    O -- "No" --> Q["Fix in next cycle"]

    classDef blue fill:#dbeafe,stroke:#93c5fd,color:#1e3a5f
    classDef gray fill:#f3f4f6,stroke:#d1d5db,color:#374151
    classDef yellow fill:#fef9c3,stroke:#fde68a,color:#713f12
    classDef green fill:#d1fae5,stroke:#6ee7b7,color:#065f46
    classDef orange fill:#ffedd5,stroke:#fdba74,color:#7c2d12
    classDef note fill:#f8fafc,stroke:#94a3b8,stroke-dasharray:5 5,color:#475569

    class A,B,C,D blue
    class E,F,G,H,I gray
    class K,O yellow
    class L,P green
    class M,N,Q orange
    class NOTE note
```

---

## Prerequisites

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** — Anthropic's CLI for Claude. Loop shells out to `claude` in headless mode during execution.
- **[Context7 MCP Server](https://github.com/upstash/context7)** — Provides live library documentation lookup. The `loop-spec` skill uses Context7 to research technical decisions (e.g., querying current Drizzle ORM docs for migration syntax, checking Next.js patterns). Without it, the spec phase falls back to web search only and may produce less accurate technical recommendations. Set up as an MCP server in your Claude Code configuration.
- **jq** — Used by `loop.sh` to read `branchName` from `prd.json` for branch management and archiving.

---

## The Pipeline: Design Phase

Each design skill can **iterate on itself** — if you want changes, the skill writes a new minor revision (0A → 0B → 0C) rather than overwriting. When you're satisfied, move to the next skill.

| # | Skill | Mode | Input | Output |
|---|-------|------|-------|--------|
| 1 | `loop-idea` | Interactive | Your rough idea | `[slug]-concept-[rev].md` |
| 2 | `loop-prd` | Interactive | Highest-rev concept + your answers | `[slug]-prd-[rev].md` |
| 3 | `loop-spec` | Interactive → Research → Output | Highest-rev PRD + codebase + research | `[slug]-spec-[rev].md` |
| 4 | `loop-task` | Autonomous | Highest-rev PRD + spec | `[slug]-prd-[rev].json` |

### Skill 1: `loop-idea` — Brainstorm

Turns a vague idea into a clear concept document through guided dialogue. The agent asks 6–7 focused questions — one at a time, with multiple-choice options when helpful — covering the core concept, problem, audience, distinctiveness, and trade-offs. You end up with a plain-English document that anyone can read. This skill also creates the **slug** — a short kebab-case identifier (e.g., `draft-feedback`) that becomes the permanent name for all artifacts in the pipeline.

If you want refinements, the agent writes a new revision (e.g., `0A` → `0B`) rather than editing the original.

**Output:** `loop-output/[slug]-concept-[rev].md` (e.g., `draft-feedback-concept-0A.md`)

### Skill 2: `loop-prd` — Product Requirements

Creates a structured Product Requirements Document. Chains from the highest-revision concept doc, then asks iterative questions about goals, scope, timeline, constraints, success metrics, and quality gate intent. The quality gate question is required — it captures what standards every story must meet (type checking, linting, tests, visual verification) without specifying exact commands yet.

If you want refinements, the agent writes a new revision. The next skill will automatically pick up whichever revision is highest.

**Output:** `loop-output/[slug]-prd-[rev].md` (e.g., `draft-feedback-prd-0A.md`) with user stories, acceptance criteria, functional requirements, and quality gate intent.

### Skill 3: `loop-spec` — Technical Specification

Transforms the PRD into a technical implementation spec. Runs in three phases:

1. **Interactive** — Scans the PRD and existing codebase, identifies up to 3 technical decisions that need your input, and asks as multiple-choice questions.
2. **Autonomous research** — Uses the Context7 MCP server to query live library documentation and web search to validate technical decisions. Finds current best practices, version-specific patterns, and integration approaches.
3. **Output** — Produces a spec with architecture, data models, API contracts, component structure, error handling, testing strategy, and security considerations. Research findings are embedded inline (no separate research doc). Quality gate intent from the PRD gets translated into specific commands for the project's tech stack (e.g., "must pass type checking" → `pnpm typecheck`).

If you want refinements, the agent writes a new revision.

**Output:** `loop-output/[slug]-spec-[rev].md` (e.g., `draft-feedback-spec-0A.md`)

### Skill 4: `loop-task` — Task Conversion

Converts the PRD + spec into the `prd.json` format that `loop.sh` consumes. Each user story becomes a JSON entry with:

- **`references`** — Links to relevant spec sections (e.g., `task-status-spec-0A.md#data-model`) so executing agents can look up technical details.
- **`notes`** — Pre-populated implementation hints from the spec (e.g., "Use Drizzle pgEnum for status column").
- **`priority`** — Dependency-ordered. Schema changes first, then backend logic, then UI.
- **`passes: false`** — Every story starts unfinished.

The key constraint: each story must be completable in **one Loop iteration** (one context window). If a story is too big, the agent runs out of context before finishing. Rule of thumb: if you can't describe the change in 2–3 sentences, split it.

If you want refinements, the agent writes a new revision.

**Output:** `loop-output/[slug]-prd-[rev].json` (e.g., `draft-feedback-prd-0A.json`)

---

## Execution Phase: `loop.sh`

```bash
./loop.sh              # Default: up to 20 iterations, auto-finds prd.json
./loop.sh 10           # Cap at 10 iterations
./loop.sh path/to/prd.json   # Use a specific prd.json
./loop.sh 10 path/to/prd.json # Both
```

### What happens each iteration

1. `loop.sh` invokes Claude Code in headless mode (`claude -p --print --dangerously-skip-permissions`) with the contents of `CLAUDE.md` as the prompt.
2. The agent reads `prd.json` from `loop-output/` (falls back to project root).
3. It reads `progress.txt`, checking the **Codebase Patterns** section first for learnings from prior iterations.
4. It checks out or creates the branch specified in `prd.json`'s `branchName`.
5. It picks the highest-priority story where `passes: false`.
6. It implements that single story.
7. It runs quality checks (typecheck, lint, test — whatever the project requires).
8. If checks pass, it commits with message: `feat: [Story ID] - [Story Title]`.
9. It sets `passes: true` on the completed story in `prd.json`.
10. It appends a progress entry to `progress.txt` with what was built, files changed, and learnings.
11. If all stories now pass → outputs `<promise>COMPLETE</promise>` and the loop exits.
12. If stories remain → the iteration ends, `loop.sh` sleeps 2 seconds, and starts the next iteration.

### How agents learn across iterations

Each iteration is a **fresh Claude Code instance** with no memory of prior work. Knowledge transfer happens through files:

- **`progress.txt`** — Every iteration appends what it did and what it learned. The next iteration reads this before starting.
- **Codebase Patterns section** — A consolidated section at the top of `progress.txt` with the most important reusable patterns (e.g., "always use `IF NOT EXISTS` for migrations"). Agents promote learnings here when they're generally useful.
- **`CLAUDE.md`** — Agents update this with genuinely reusable knowledge that would help future work beyond the current run.

### Archiving previous runs

When `loop.sh` detects a different `branchName` than the last run, it archives the previous run's `prd.json` and `progress.txt` to `archive/YYYY-MM-DD-feature-name/` and resets `progress.txt`.

---

## Post-Execution: Skill 6 — `loop-evaluate`

| # | Skill | Mode | Input | Output |
|---|-------|------|-------|--------|
| 6 | `loop-evaluate` | Autonomous | spec + prd.json + progress.txt + codebase | `loop-output/[slug]-closing-report-[rev].md` + optional `[slug]-finalspec-[rev].md` |

An optional skill you run after all stories are complete. It performs three phases:

### Phase 1: Closing Report

Reads `progress.txt`, `prd.json`, the git log, and the actual source files to produce a performance report: iteration-by-iteration breakdown, what was built, lines of code, test coverage, agent behavioral observations, and a final scorecard.

### Phase 2: Spec Audit

Exhaustive point-by-point comparison of the spec against the actual implementation. Every source file is read (not sampled). Each spec section is scored on a 100-point scale:

| Section | Points |
|---------|--------|
| Module Layout | 10 |
| Data Model | 15 |
| Database Schema | 15 |
| Component Structure | 25 |
| API Contracts | 15 |
| Error Handling | 10 |
| Testing Strategy | 5 |
| Security | 5 |

Deviations are classified: **Match**, **Enhancement** (agent added beyond spec — no penalty), **Spec Bug Fixed** (agent corrected a flawed spec — no penalty), **Alternate Approach** (different implementation, same behavior — no penalty), **Minor Deviation**, **Major Deviation**, or **Missing**.

### Phase 3: Promotion Decision

- **100/100** — Spec is auto-promoted to `[slug]-finalspec-0A.md` with status changed to Final.
- **90–99** — Deductions are presented as code bugs (not spec bugs). User decides whether to promote.
- **< 90** — Recommendation to fix deviations in a follow-up Loop cycle before promoting.

---

## Revision Naming System

All artifacts use the same naming convention: `[slug]-[artifact]-[major][minor].[ext]`

| Component | Description | Example |
|-----------|-------------|---------|
| **Slug** | 2–4 kebab-case words describing the idea. Created once during `loop-idea`, inherited by every downstream artifact. | `draft-feedback`, `task-status` |
| **Artifact** | The document type. | `concept`, `prd`, `spec`, `closing-report`, `finalspec` |
| **Major** | A number starting at `0`. Only bumped on explicit user request. | `0`, `1` |
| **Minor** | An uppercase letter starting at `A`. Incremented each time a skill refines the document. | `A`, `B`, `C` |

### How revisions work

Each design skill can **iterate on itself**. When you ask for changes, the skill writes a new file with the next minor revision — it never overwrites the previous one. All revisions are kept in `loop-output/`:

```
loop-output/
├── draft-feedback-concept-0A.md    # First concept draft
├── draft-feedback-concept-0B.md    # Refined after feedback
├── draft-feedback-prd-0A.md        # PRD from concept 0B
├── draft-feedback-prd-0B.md        # PRD refined once
├── draft-feedback-spec-0A.md       # Spec from PRD 0B
├── draft-feedback-prd-0A.json      # Executable tasks from spec 0A
```

**Key rules:**
- **Immutable files** — once a revision is written, it is never modified. Refinements produce a new file.
- **Minor revisions** go A → B → C → ... → Z within a major version.
- **Major revision bumps** (e.g., `0Z` → `1A`) only happen when the user explicitly requests one.
- **Input chaining** — each skill finds the highest-revision prior artifact automatically. If `concept-0B` and `concept-0A` both exist, the PRD skill picks `0B`.
- **Slug inheritance** — the slug is set once at the concept stage and carried through the entire pipeline. Downstream skills never create new slugs.

## Artifact Chain

A complete Loop run produces these artifacts in `loop-output/`:

```
loop-output/
├── [slug]-concept-[rev].md              # Skill 1: idea → concept
├── [slug]-prd-[rev].md                  # Skill 2: concept → PRD
├── [slug]-spec-[rev].md                 # Skill 3: PRD → spec
├── [slug]-prd-[rev].json               # Skill 4: spec → executable tasks
├── [slug]-closing-report-[rev].md       # Skill 6: post-execution report
└── [slug]-finalspec-[rev].md            # Skill 6: promoted spec (if audit passes)
```

Each artifact is immutable once created. The revision suffix preserves the order and traceability. Multiple revisions of the same artifact type can coexist — each is a snapshot of the document at that stage of refinement.

---

## prd.json Schema

```json
{
  "project": "MyApp",
  "branchName": "loop/feature-name",
  "description": "Feature description",
  "userStories": [
    {
      "id": "US-001",
      "title": "Story title",
      "description": "As a [user], I want [feature] so that [benefit]",
      "acceptanceCriteria": ["Criterion 1", "Typecheck passes"],
      "references": ["loop-output/task-status-spec-0A.md#section"],
      "notes": "Implementation hints from spec.",
      "priority": 1,
      "passes": false
    }
  ]
}
```

| Field | Purpose |
|-------|---------|
| `branchName` | Git branch for execution. Auto-created from `main` if it doesn't exist. |
| `priority` | Execution order. Lower = first. Must respect dependencies (schema before UI). |
| `passes` | Set to `true` by the executing agent after the story is implemented and quality checks pass. |
| `references` | Pointers into the spec so the agent knows where to look for technical details. |
| `notes` | Implementation hints pre-populated from the spec. Agents may append learnings. |

---

## Directory Structure

### Tracked (in repo)

```
loop/
├── .claude/skills/                    # Skill definitions (the design pipeline)
│   ├── loop-idea/SKILL.md            # Skill 1: brainstorm → concept
│   ├── loop-prd/SKILL.md             # Skill 2: concept → PRD
│   ├── loop-spec/SKILL.md            # Skill 3: PRD → spec (with research)
│   ├── loop-task/SKILL.md            # Skill 4: PRD + spec → prd.json
│   └── loop-evaluate/SKILL.md        # Skill 6: post-execution audit
├── CLAUDE.md              # Agent instructions — the prompt each iteration receives
├── README.md              # This file
├── loop.sh                # Execution loop — spawns Claude Code per iteration
├── loop-bash-prompt.md    # Agent instruction template for Skill 5 (sandbox execution)
├── prd.json.example       # Example prd.json for reference
├── loop-flowchart.md      # Mermaid flowchart of the full pipeline
└── .gitignore
```

> **Note on skill numbering:** Skill 5 (`loop-bash`) is a planned feature for spawning multiple Claude Code instances in parallel sandboxes (Proxmox-based). It's in development on a separate branch. Skills are numbered to preserve ordering once it ships.

