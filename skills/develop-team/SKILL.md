---
name: develop-team
description: This skill should be used when the user asks to "develop a feature", "implement a ticket", "build PROJ-123", "run the development pipeline", "develop this ticket end to end", or wants fully autonomous feature implementation with parallel research agents, planning, phased implementation, review, and PR creation. Zero checkpoints; pauses only on blockers.
---

# Develop Team

Orchestrate a full feature development cycle using parallel sub-agents. Takes a ticket reference or freeform task and drives it through deep research, planning (with design/test/migration specs), implementation with per-phase test generation, review, and automatic PR creation — fully autonomous with no checkpoints unless a genuine blocker is detected.

## When to Use This Skill

- **Starting a new feature** - Hand off a ticket and let the pipeline research, plan, and implement
- **Exploring before implementing** - Use `plan-only=true` to get a researched plan without writing code
- **Structured development** - Parallel research + review instead of ad-hoc implementation

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `type` | `auto` | Feature type: `auto\|ui-heavy\|full-stack\|backend-only\|async-heavy` |
| `auto-commit` | `true` | Create atomic commits after each implementation phase |
| `skip-review` | `false` | Skip refactor and review phases |
| `skip-tests` | `false` | Skip per-phase test generation |
| `skip-pr` | `false` | Skip automatic PR creation |
| `skip-migrations` | `false` | Skip DB migration analysis and application |
| `plan-only` | `false` | Stop after planning (no implementation) |

## Invocation

```
/develop-team PROJ-123
/develop-team PROJ-123 plan-only=true
/develop-team "Add currency preferences to user settings"
/develop-team PROJ-45 auto-commit=false skip-review=true
/develop-team PROJ-67 type=ui-heavy
/develop-team PROJ-89 type=backend-only skip-tests=true skip-pr=true
```

## Prerequisites

This skill expects the following project conventions:

1. **CLAUDE.md** at project root — defines coding standards, patterns, folder structure, and conventions
2. **Ticket system** (optional) — Jira MCP or similar for fetching ticket details
3. **Spec/backlog location** — configurable; defaults to searching common locations (`/backlog/`, `/specs/`, `/docs/specs/`)
4. **Companion skills** (optional) — `/refactor` and `/review-fix` for Phases 5-6. If not available, those phases are skipped gracefully.

## Architecture

```
ORCHESTRATOR (main session - manages state, cross-references outputs)
|
+-- Phase 0: INIT — parse input, create state, determine ticket vs freeform
|
+-- Phase 1: CONTEXT + CLASSIFY + DB PRE-SCAN
|   Ticket (MCP or manual) + spec file + git state
|   Past-ticket context (git log, gh pr list)
|   DB schema pre-scan (if DB tools available, scan related tables)
|   Classify: ui-heavy | full-stack | backend-only | async-heavy
|
+-- Phase 2: RESEARCH (4-6 parallel agents, run_in_background: true)
|   Agent 1: Deep Codebase Explorer    (system-architect)          [always]
|   Agent 2: CLAUDE.md Compliance Mapper (code-refactorer)         [always]
|   Agent 3: Requirements Mapper       (product-strategy-advisor)  [always]
|   Agent 4: Design System Integrator  (premium-ux-designer)       [if ui-heavy|full-stack]
|   Agent 5: Async Flow Analyzer       (system-architect)          [if async-heavy]
|   Agent 6: DB Migration Analyzer     (system-architect)          [if DB changes needed]
|   (auto-proceed — pause only on blockers)
|
+-- Phase 3: PLANNING — synthesize into plan + design spec + test plan + migration plan
|   (auto-proceed — plan-only=true pauses here)
|
+-- Phase 4: IMPLEMENTATION — per-phase: agent spawn → build/lint → tests → commit
|   UI phases → premium-ux-designer | Backend → general-purpose
|
+-- Phase 4.5: DB MIGRATIONS — apply via migration tooling, save locally
+-- Phase 5: REFACTOR → /refactor scope=session max-iterations=1 auto-fix=true
+-- Phase 6: REVIEW → /review-fix max-iterations=2 auto-commit={inherited}
+-- Phase 7: PR CREATION → gh pr create with full summary
+-- Phase 8: SYNTHESIS — report with commits, tests, migrations, PR link
|
+-- STATE: .develop-team/state.json
```

---

## Workflow

### Phase 0: Initialize

1. Parse `$ARGUMENTS`: ticket reference (pattern: `{PROJECT_KEY}-\d+`, e.g. `PROJ-123`) or freeform task description
2. Parse parameters: `type`, `auto-commit`, `skip-review`, `skip-tests`, `skip-pr`, `skip-migrations`, `plan-only`
3. Create `.develop-team/` directory if needed
4. Check for existing `state.json` — if found, offer to resume from last completed phase
5. Initialize state file

**State file at `.develop-team/state.json`:** See `references/state-schema.md` for full schema and backward compatibility notes.

### Phase 1: Gather Context + Classify Feature Type

#### If ticket reference:
Fetch ticket details using the project's configured ticket system (Jira MCP, GitHub Issues, Linear, etc.):
```
Extract: key, summary, description, acceptance criteria,
         status, issue type, priority, labels
```

#### Find spec file:
Search common spec/backlog locations for files matching ticket number or feature name. Typical locations:
- `/backlog/` (organized by version or priority)
- `/specs/`
- `/docs/specs/`
- Custom location defined in CLAUDE.md

Read spec for requirements, user stories, success criteria.

#### Past-Ticket Context
Load context from previous related work to avoid re-discovering solved problems:
```bash
git log --oneline --all --grep="{ticketKey}" -20
gh pr list --state all --search "{ticketKey}" --json number,title,state,mergedAt --limit 5
```
Store summary of past PRs and patterns learned in `state.json.pastTicketContext`.

#### DB Schema Pre-Scan
If ticket mentions database tables/schema/data model, query the project's database tooling for schema information: table names, columns, policies, indexes. Feeds into Agent 6.

#### Git state:
```bash
git branch --show-current && git status --short && git log --oneline -5
```

#### Branch Creation (ticket references only)
When input is a ticket reference, create a feature branch off the current branch:
```bash
# Extract ticket number and kebab-case summary
# PROJ-111 + "Add currency preferences" → 111-add-currency-preferences
git checkout -b {ticketNumber}-{kebab-case-summary}
```
Store branch name in `state.json.branchName`. Skip for freeform tasks (user is already on their working branch).

#### Classify Feature Type

If `type` != `auto`, use directly. Otherwise, scan combined ticket text for keyword signals:

| Feature Type | Keyword Signals |
|---|---|
| `ui-heavy` | `component`, `page`, `dialog`, `form`, `modal`, `sheet`, `table`, `dashboard`, `responsive`, `layout`, `sidebar`, `nav`, `tab`, `wizard` |
| `backend-only` | `migration`, `RLS`, `API route`, `cron`, `webhook`, `edge function`, `schema`, `index`, `policy`, `trigger` (without UI keywords) |
| `async-heavy` | `queue`, `cron`, `background`, `batch`, `import`, `export`, `process`, `job`, `polling`, `retry`, `state machine`, `pipeline` |
| `full-stack` | Mix of signals or neither dominates (default fallback) |

**Logic:** UI > 2x backend → `ui-heavy`. Backend > 2x UI and UI=0 → `backend-only`. Async >= 2 → `async-heavy`. Else → `full-stack`.

### Phase 2: Research (4-6 Parallel Sub-Agents)

Spawn all applicable agents with `run_in_background: true`. Each receives ticket context, spec file, and past-ticket context.

| Agent | Sub-Agent Type | Condition | Returns |
|-------|---------------|-----------|---------|
| Deep Codebase Explorer | `system-architect` | Always | File map, data flow chains, exact imports |
| CLAUDE.md Compliance Mapper | `code-refactorer` | Always | Per-file-type rule cards, golden references |
| Requirements Mapper | `product-strategy-advisor` | Always | Phases, tasks, test plan, migration plan |
| Design System Integrator | `premium-ux-designer` | `ui-heavy`/`full-stack` | Component hierarchy, exact design tokens, all states |
| Async Flow Analyzer | `system-architect` | `async-heavy` | State machine, race conditions |
| DB Migration Analyzer | `system-architect` | DB changes needed | Schema diff, ready SQL, policies |

For full agent prompts, read `references/agent-prompts.md`.

#### Post-Research: Auto-Proceed with Blocker Detection

Collect results via `TaskOutput`. Auto-proceed UNLESS a blocker is detected.

**Blockers** (pause and ask user only if):
- Research agents recommend incompatible approaches
- Feature requires tables with no migration plan to create them
- Acceptance criteria are ambiguous or contradictory
- A research agent completely failed (total failure, not just partial)

If no blockers, log summary and proceed directly:
```
Research complete ({N} agents, 0 blockers)
  Codebase Explorer: {N} files, {N} flow chains | Compliance Mapper: {N} rule cards
  Requirements Mapper: {N} phases, {N} tasks | Design/Async/Migration: {status}
Proceeding to planning...
```

### Phase 3: Planning (Inline)

The orchestrator synthesizes all research into a cohesive plan. Done inline (not sub-agent) to cross-reference multiple outputs.

**Cross-reference:** Codebase Explorer's file map + Compliance Mapper's rule cards + Requirements Mapper's phases + Design System Integrator's specs (if ran) + Async Analyzer (if ran) + DB Migration Analyzer (if ran).

**Plan structure:** Save to `.develop-team/plan.md`. See `references/plan-template.md` for the full template. Includes: context, per-phase tasks with convention notes, design specification (if UI), test plan, migration plan, AC coverage, and files summary.

Save design spec to `state.json.designSpec`. If `plan-only=true`, present plan and stop. Otherwise auto-proceed.

### Phase 4: Implementation

Execute plan phase by phase:

**1. Select agent:**

| Phase Content | Agent Type |
|---|---|
| Files match `components/**` or page/view files, type is `ui-heavy`/`full-stack` | `premium-ux-designer` |
| All other phases | `general-purpose` |

**2. Spawn agent** (`run_in_background: false`):

For **UI phases** → `premium-ux-designer`:
```
Implement Phase {N}. Follow the design specification EXACTLY — do not improvise colors/shadows/spacing.

## Phase Tasks: {tasks from plan}
## Design Specification: {full design spec — hierarchy, tokens, states, responsive}
## Convention Rules: {component rule card from Compliance Mapper}
## Anti-Slop Checklist: {from Design System Integrator}

Instructions: Implement tasks in order. Run build and lint commands after. Do NOT implement other phases.
```

For **non-UI phases** → `general-purpose`:
```
Implement Phase {N}.

## Phase Tasks: {tasks from plan}
## Convention Rules: {relevant rule cards — api-route, service, schema, store}
## Async Analysis: {if applicable, from Async Flow Analyzer}

Instructions: Implement tasks in order. Run build and lint commands after. Do NOT implement other phases.
```

**3. Verify:** Run build and lint commands (per CLAUDE.md configuration). If fails: spawn debug agent. If still fails after 2 attempts, stop.

**4. Generate tests** (unless `skip-tests=true`):
Spawn `general-purpose` with test cases from the plan for this phase. Run test command. Update `state.json.tests.perPhase`.

**5. Commit** (if `auto-commit=true`): create atomic commit with phase name and files list.

**6. Update state** and proceed to next phase.

### Phase 4.5: DB Migrations (Conditional)

**Skip if:** `skip-migrations=true` OR no migrations planned.

For each migration in the plan:
1. **Apply:** Use the project's migration tooling (Supabase MCP, Prisma, Drizzle, raw SQL, etc.)
2. **Save locally** (CRITICAL): Save migration file to the project's migrations directory following existing naming conventions
3. **Update related files:** Edit any select/query builder files, ORM schemas, or type definitions as specified in the plan
4. **Commit** if auto-commit enabled

**On migration failure:** Save SQL to `.develop-team/failed-migration.sql`, ask user: skip or stop.

### Phase 5: Refactor (Companion Skill Delegation)

Skip if `skip-review=true` or `/refactor` skill not available. Invoke: `/refactor scope=session max-iterations=1 auto-fix=true`

After: read `.refactor/state.json`, update state, clean up `.refactor/`.

### Phase 6: Review (Companion Skill Delegation)

Skip if `skip-review=true` or `/review-fix` skill not available. Invoke: `/review-fix max-iterations=2 auto-commit={inherited}`

After: read `.review-fix/state.json`, update state, clean up `.review-fix/`.

### Phase 7: PR Creation

**Skip if:** `skip-pr=true`.

```bash
gh pr create --title "{type}({scope}): {summary} ({ticket key})" --body "$(cat <<'EOF'
## Summary
{1-3 bullets from ticket}

## Changes
{Phases implemented with key files}

## Test Plan
| Phase | Tests | Passed | Failed |
{from state.json.tests.perPhase}
- [ ] Build passes
- [ ] Lint passes

## Database Migrations
{List of migrations applied, or "None"}

## Acceptance Criteria
| Criterion | Status |
{from plan}

## Ticket
{ticket key}: {URL if available}

---
Generated with [Claude Code](https://claude.com/claude-code) via /develop-team
EOF
)"
```

Update `state.json.pullRequest`. On failure: save body to `.develop-team/pr-body.md`, note in synthesis.

### Phase 8: Synthesis

```markdown
# Development Complete: {Ticket Key} - {Summary}

## Summary
| Metric | Value |
|--------|-------|
| Feature type | {featureType} |
| Phases implemented | {N} |
| Total commits | {N} |
| Tests | {N} generated ({N} passed, {N} failed) |
| Migrations | {N} applied |
| Refactor | {N} fixed, {N} strategic |
| Review | {N} iterations, {N} fixed |
| Pull request | {PR URL or "Skipped"} |

## Commits — {all with hashes}
## Acceptance Criteria — {criterion → status}
## Test Summary — {per-phase results}
## Database Migrations — {migration → status → local file}
## Files Changed — {action → path}
## Next Steps — {merge PR, remaining items, manual testing}
```

Update `state.json` with `status: "completed"`.

---

## State File Reference

**Location:** `.develop-team/state.json` — enables resumability.

If existing state found, ask user: "Resume from Phase {N+1}" or "Start fresh".

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Ticket fetch failure | Ask user for details manually, don't block |
| Research agent failure | Continue with other agents, note gap |
| Build/lint failure | Debug agent x2, then stop and report |
| Implementation agent failure | Save to state, report, offer resume |
| Test failure | Log but don't block — may resolve in later phases |
| Migration failure | Save SQL to `.develop-team/failed-migration.sql`, ask skip/stop |
| PR creation failure | Save body to `.develop-team/pr-body.md`, continue to synthesis |
| /refactor not available | Log, skip to Phase 6 |
| /review-fix not available | Log, skip to Phase 7 |

---

## Integration Points

- **Ticket system**: Jira MCP, GitHub Issues, Linear, or manual input
- **Database tooling**: Supabase MCP, Prisma, Drizzle, or raw SQL — auto-detected from project setup
- **GitHub CLI**: `gh pr create` for PRs, `gh pr list` for past-ticket context
- **Spec/backlog files**: Searched in configurable locations (see Prerequisites)
- **CLAUDE.md**: Per-file-type rule cards extracted and injected into all agents
- **Design system**: Theme config, global styles, and design docs read by Agent 4 (paths auto-detected from project)
- **Companion skills**: `/refactor` and `/review-fix` delegated to in Phases 5-6 (gracefully skipped if unavailable)

---

## Tips

1. **`plan-only=true`** for complex features — review before committing to implementation
2. **Provide a ticket reference** when possible — acceptance criteria drive better research and validation
3. **Override with `type=`** to specify the feature type directly — ensures correct agent routing
4. **Check `.develop-team/state.json`** if interrupted — resumable from last completed phase
5. **`skip-review=true`** for quick iterations — run `/refactor` and `/review-fix` manually later
6. **`skip-tests=true`** for prototypes — skip per-phase test generation
7. **`skip-pr=true`** when not ready — create PR manually from synthesis report
8. **`skip-migrations=true`** for code-only changes
9. **Zero checkpoints** — fully autonomous, pauses only on genuine blockers
