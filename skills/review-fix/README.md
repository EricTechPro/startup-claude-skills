# /review-fix Skill

Automated review-fix loop that iterates until your code is clean. Spawns 8 reviewers in parallel, fixes quick-fix items automatically, and accumulates strategic items for your decision at the end.

## Quick Start

```
/review-fix
```

That's it. The skill will:
1. Extract your branch context
2. Spawn 8 reviewer personas in parallel
3. Aggregate findings and separate quick-fix vs strategic
4. Auto-fix quick-fix items
5. Commit changes
6. Repeat until no issues remain (max 3 iterations)
7. Ask about strategic items at the end

## Parameters

```
/review-fix                         # Default: 3 iterations, auto-commit on, base-commit=main
/review-fix max-iterations=5        # More iterations
/review-fix auto-commit=false       # Don't commit automatically
/review-fix skip-strategic=true     # Skip strategic item questions
/review-fix base-commit=abc123      # Compare against specific commit
/review-fix base-commit=HEAD~3      # Compare last 3 commits only
/review-fix base-commit=develop     # Compare against develop branch
```

## The 8 Reviewers

| Persona | Focus | Quick-Fix Examples | Strategic Examples |
|---------|-------|-------------------|-------------------|
| Frontend Designer | UI/UX polish | Missing hover states, CSS fixes | Major design changes |
| Product Manager | Feature direction | - | Scope questions, direction |
| QA Engineer | Test coverage | Add specific test cases | Test architecture |
| System Architect | Architecture | Pattern fixes | Structural refactors |
| Senior Developer | Code quality | Bugs, type fixes, logging | - |
| Code Maintainability | Simplicity | Remove duplication | Major refactors |
| Reusable Components | DRY | Simple extractions | Component architecture |
| Dead Code Hunter | Unused code | Unused imports, variables | Large unused modules |

## How It Works

```
+----------------------------------------------------------+
|                  ITERATION LOOP                           |
+----------------------------------------------------------+
|                                                           |
|  REVIEW PHASE                                             |
|  | 8 agents spawn in parallel (fresh 200k context each)   |
|  | Each returns JSON findings array                       |
|           |                                               |
|  AGGREGATE                                                |
|  | Deduplicate findings                                   |
|  | Separate: quick-fix -> fix prompt                      |
|  |           strategic -> accumulate in state              |
|           |                                               |
|  FIX PHASE                                                |
|  | Single agent with fresh context                        |
|  | Implements all quick-fix items                         |
|  | Runs build and lint commands                           |
|           |                                               |
|  COMMIT                                                   |
|  | git commit -m "fix: address code review findings"      |
|           |                                               |
|  CHECK                                                    |
|  | No quick-fix items? -> Exit loop                       |
|  | Max iterations? -> Exit loop                           |
|  | Otherwise -> Next iteration                            |
|                                                           |
+-----------------------------------------------------------+
           |
+-----------------------------------------------------------+
|  STRATEGIC ITEMS REVIEW                                   |
|  | Present accumulated items via AskUserQuestion          |
|  | Options: [Add to backlog] [Implement now] [Ignore]     |
|  | Create backlog files for deferred items                |
+-----------------------------------------------------------+
```

## State Management

State persists in `.review-fix/state.json`:

```json
{
  "iteration": 2,
  "maxIterations": 3,
  "baseCommit": "main",
  "status": "in_progress",
  "strategicItems": [],
  "fixedItems": [],
  "commits": ["abc1234", "def5678"]
}
```

Iteration logs in `.review-fix/iteration-{N}.json`.

## Example Output

```markdown
# Review-Fix Complete

## Summary
| Metric | Value |
|--------|-------|
| Iterations | 2 |
| Quick-fix items resolved | 15 |
| Strategic items accumulated | 4 |
| Commits created | 2 |

## Commits
- `abc1234` - fix: address code review findings (iteration 1)
- `def5678` - fix: address code review findings (iteration 2)

## Strategic Items Handled
- Added to backlog: 2
- Implemented now: 1
- Ignored: 1
```

## When to Use

| Scenario | Recommended |
|----------|-------------|
| Want automated fixes before PR | Yes |
| Comprehensive cleanup after refactoring | Yes |
| Iterative polish over multiple cycles | Yes |
| Just want to see issues without fixing | No (use a review-only skill) |
| Quick one-time feedback | No (use a review-only skill) |

## Troubleshooting

### Loop exits immediately
No quick-fix items were found. Either your code is clean or all issues are strategic.

### Build keeps failing
The skill will try to debug build failures. After 2 consecutive failures, it exits and reports the issue. Check `.review-fix/failed-fix-prompt-{N}.md` for the attempted fixes.

### Too many iterations
Reduce with `max-iterations=2`. Some codebases converge slower than others.

### State files cluttering the repo
Add `.review-fix/` to your `.gitignore`. State files are temporary and should not be committed.
