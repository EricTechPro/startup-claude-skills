---
name: review-fix
description: Automated review-fix loop that spawns 8 reviewers in parallel, fixes quick-fix items automatically, and accumulates strategic items for user decision. Iterates until no issues remain or max iterations reached.
---

# Review-Fix Loop

Orchestrate an automated review-fix cycle that iterates until your code is clean. Each iteration spawns fresh sub-agents with full 200k context windows, fixes quick-fix items automatically, and accumulates strategic items for user decision at the end.

## When to Use This Skill

- **Before creating a PR** - Get comprehensive feedback AND automatic fixes
- **After major refactoring** - Ensure quality across multiple review cycles
- **When you want hands-off improvement** - Let the loop fix issues automatically
- **For iterative polish** - Run until no quick-fix items remain

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max-iterations` | 3 | Maximum review-fix cycles before stopping |
| `auto-commit` | true | Create a commit after each fix cycle |
| `skip-strategic` | false | Skip strategic item questions at the end |
| `base-commit` | `main` | Commit/branch to compare against (e.g., `abc123`, `HEAD~5`, `develop`) |

## Architecture

```
ORCHESTRATOR (main session - manages state only)
|
+-- REVIEW PHASE (8 parallel sub-agents, each with fresh 200k context)
|   +-- premium-ux-designer
|   +-- product-strategy-advisor
|   +-- general-purpose
|   +-- system-architect
|   +-- code-refactorer
|   +-- code-refactorer
|   +-- Explore (reusable components)
|   +-- Explore (dead code)
|
+-- FIX PHASE (single sub-agent, fresh 200k context)
|   +-- General agent with fix prompt + file context
|
+-- STATE FILE (.review-fix/state.json)
    +-- Tracks iterations, strategic items, fixed items, commits
```

---

## Workflow

### Step 0: Initialize State

Create/update the state file at `.review-fix/state.json`:

```json
{
  "iteration": 1,
  "maxIterations": 3,
  "baseCommit": "main",
  "strategicItems": [],
  "fixedItems": [],
  "commits": [],
  "status": "in_progress",
  "startedAt": "2025-01-18T10:00:00Z",
  "branchName": "feature/example"
}
```

### Step 1: Extract Branch Context

```bash
# Get current branch name
BRANCH=$(git branch --show-current)

# Get base commit (from parameter or default to main)
BASE_COMMIT="${base_commit:-main}"

# Validate base commit exists
if ! git rev-parse "$BASE_COMMIT" >/dev/null 2>&1; then
  # Fallback to master if main doesn't exist and no custom base provided
  if [ "$BASE_COMMIT" = "main" ]; then
    BASE_COMMIT="master"
  fi
fi

# Get diff stats against base commit
git diff --stat "$BASE_COMMIT"..HEAD

# Get list of changed files
git diff --name-only "$BASE_COMMIT"..HEAD

# Get commit messages since base
git log --oneline "$BASE_COMMIT"..HEAD

# Get detailed diff for context
git diff "$BASE_COMMIT"..HEAD --no-color
```

### Step 2: Spawn 8 Reviewer Sub-Agents IN PARALLEL

Use the Task tool to spawn all 8 reviewers simultaneously with `run_in_background: true`.

Each reviewer receives:
1. The branch context (diff, changed files, commits)
2. Their specific review focus
3. Instructions to return JSON findings

#### Reviewer Prompts

All reviewers must return findings in this format:
```json
[{
  "severity": "critical|important|suggestion",
  "label": "blocking|important|nit|suggestion",
  "category": "designer|pm|qa|architect|developer|maintainability|reusable",
  "file": "path/to/file.tsx",
  "line": 123,
  "issue": "Brief description of the problem",
  "suggestion": "Specific fix or improvement",
  "type": "quick-fix|strategic"
}]
```

### Severity Labels
- **blocking** - Must fix before merge
- **important** - Should fix, discuss if disagree
- **nit** - Nice to have, not blocking
- **suggestion** - Alternative approach to consider

**MAX 10 findings per reviewer. Focus on high-impact issues only.**

##### Persona 1: Frontend Designer
```
Sub-agent: premium-ux-designer

Review the following changed files for UI/UX quality:
{list of component/page files}

Look for:
- Missing hover/focus states
- Inconsistent spacing or typography
- Poor loading states
- Accessibility issues (ARIA, keyboard nav)
- Animation opportunities
- Visual hierarchy issues

Review against project CLAUDE.md conventions for UI patterns and components.

Mark items as "quick-fix" if they're CSS/styling changes.
Mark items as "strategic" if they require design decisions or major restructuring.

Return findings as JSON array. MAX 10 findings.
```

##### Persona 2: Product Manager
```
Sub-agent: product-strategy-advisor

Review the overall feature changes for product fit:
{summary of changes and commit messages}

Look for:
- Is this solving the right problem?
- Feature creep or over-engineering
- Missing user-facing documentation
- Edge cases that affect UX
- Should this be built differently?

Most PM feedback should be marked "strategic" as it involves direction.

Return findings as JSON array. MAX 10 findings.
```

##### Persona 3: QA Engineer
```
Sub-agent: general-purpose

Review test coverage for new functionality:
{list of changed files}

Look for:
- Missing unit/integration tests
- Untested edge cases
- Error handling gaps
- Regression risks

Testing Checklist:
- [ ] Happy path tested
- [ ] Edge cases covered (empty arrays, null values, boundaries)
- [ ] Error cases tested (network failures, invalid input)
- [ ] Test names are descriptive
- [ ] Tests are deterministic (no flaky tests)
- [ ] Tests are independent (no shared mutable state)

Mark missing tests as "quick-fix" with specific test case suggestions.
Mark architectural test changes as "strategic".

Return findings as JSON array. MAX 10 findings.
```

##### Persona 4: System Architect
```
Sub-agent: system-architect

Review the architectural impact of changes:
{list of changed files with focus on services, utils, migrations}

Look for:
- Wrong abstraction level
- Scalability concerns
- Pattern violations (review against project CLAUDE.md conventions)
- Database design issues
- Performance bottlenecks

Security Checklist (mark violations as blocking):
- [ ] User input validated and sanitized
- [ ] SQL queries use parameterization
- [ ] Authentication/authorization checked on protected routes
- [ ] Secrets not hardcoded (check for API keys, passwords)
- [ ] Error messages don't leak sensitive info
- [ ] File uploads restricted (size, type)
- [ ] CSRF protection for state-changing operations

Most architecture feedback should be "strategic".
Simple pattern fixes can be "quick-fix".
Security violations should be "quick-fix" with label "blocking".

Return findings as JSON array. MAX 10 findings.
```

##### Persona 5: Senior Developer
```
Sub-agent: code-refactorer

Review code for quality and correctness:
{list of all changed files}

Look for:
- Bugs and logic errors
- React hook issues (deps, memory leaks)
- TypeScript type safety
- Naming conventions per project CLAUDE.md
- Error handling patterns

Performance Checklist:
- [ ] No N+1 queries in database operations
- [ ] Database queries use proper indexes
- [ ] Large lists are paginated
- [ ] Expensive operations are cached or memoized
- [ ] No blocking I/O in hot paths
- [ ] No unnecessary re-renders in React components

Most code fixes should be "quick-fix".

Return findings as JSON array. MAX 10 findings.
```

##### Persona 6: Code Maintainability
```
Sub-agent: code-refactorer

Review code for maintainability:
{list of changed files}

Look for:
- Overly complex code
- Code duplication
- Inconsistent patterns
- Missing comments on complex logic
- Dead code

Mark simplifications as "quick-fix".
Mark major refactors as "strategic".

Return findings as JSON array. MAX 10 findings.
```

##### Persona 7: Reusable Components
```
Sub-agent: Explore

Research the codebase for reuse opportunities:
{list of new components/utilities}

Look for:
- Existing components that could be reused
- New components that should be extracted
- Duplicate logic across files
- Missing shared utilities

Mark simple extractions as "quick-fix".
Mark architectural component changes as "strategic".

Return findings as JSON array. MAX 10 findings.
```

##### Persona 8: Dead Code Hunter
```
Sub-agent: Explore

Scan the codebase for dead code patterns:
{list of new/modified files}

Look for:
- Exported functions never imported elsewhere
- Unused imports at top of files
- Variables declared but never read
- Orphan files (never imported)
- Commented-out code blocks
- Unreachable code paths
- Unused dependencies

Mark as "quick-fix" for:
- Unused imports
- Unused variables
- Small unused functions

Mark as "strategic" for:
- Large unused modules
- Potentially unused dependencies
- Architectural dead code

Return findings as JSON array. MAX 10 findings.
```

### Step 3: Collect and Aggregate Results

After all agents complete:

1. **Use TaskOutput** to retrieve results from each agent
2. **Parse JSON findings** from each response
3. **Handle errors** - If an agent fails, log it and continue with others
4. **Deduplicate** - Remove similar findings across personas
5. **Separate by type**:
   - `quick-fix` items -> Generate fix prompt for this iteration
   - `strategic` items -> Append to `state.strategicItems`

### Step 4: Check Exit Conditions

**Exit the loop if ANY of these are true:**
- No quick-fix items found in this iteration
- `iteration >= maxIterations`
- Critical build/test failure that blocks progress

**Update state.json:**
```json
{
  "iteration": 2,
  "status": "in_progress|completed|blocked",
  "lastIterationFindings": {
    "quickFix": 12,
    "strategic": 3
  }
}
```

### Step 5: Generate Fix Prompt

Create a comprehensive fix prompt for quick-fix items only:

```markdown
# Code Review Fixes - Iteration {N}

You are fixing issues identified by automated code review. Work through each section systematically.

## Instructions
1. Read each issue carefully before making changes
2. Make the changes specified - do not expand scope
3. After all fixes, run the project's build and lint commands
4. Do NOT modify or implement strategic items

## Changed Files Context
{List files that will need changes}

---

## Section 1: Critical Issues

### Issue 1.1: {issue title}
**File:** {file}:{line}
**Problem:** {description}
**Fix:**
```
{specific code change}
```

{... more critical issues}

---

## Section 2: Important Issues

{... important issues with same format}

---

## Section 3: Suggestions

{... suggestion issues with same format}

---

## Verification
After completing all fixes, run the project's build and lint commands to verify.

Report any errors encountered.
```

### Step 6: Fix Phase (Fresh Sub-Agent)

Spawn a single fix agent with:
- The fix prompt from Step 5
- `run_in_background: false` (wait for completion)
- Fresh 200k context window

```
Sub-agent: general-purpose (or appropriate fixer type)

{The fix prompt generated in Step 5}

After fixing all issues:
1. Run the project's build and lint commands
2. Report any failures
3. If successful, report completion
```

### Step 7: Commit Changes (if auto-commit enabled)

If `auto-commit: true` and fixes were made:

```bash
git add -A
git commit -m "fix: address code review findings (iteration {N})

Co-Authored-By: Claude Code <noreply@anthropic.com>"
```

Update state.json with commit hash:
```json
{
  "commits": ["abc1234", "def5678"]
}
```

### Step 8: Loop or Complete

**If more iterations needed:**
- Increment `state.iteration`
- Go back to Step 1

**If complete:**
- Update `state.status: "completed"`
- Proceed to Strategic Items Review

---

## Step 9: Strategic Items Review

After all iterations complete, present accumulated strategic items to the user.

### Grouping Strategy
- Group related items (max 4 per AskUserQuestion)
- Present one question at a time if many items

### Question Format
```
AskUserQuestion with options:
- "Add to backlog" - Creates a backlog item
- "Implement now" - Queues for immediate implementation
- "Ignore" - Mark as dismissed
```

### Backlog File Template

For items marked "Add to backlog", create:

```markdown
# {Issue Title}

**Status**: Backlog
**Priority**: {Medium|High based on severity}
**Added**: {today's date}
**Source**: Code review (review-fix iteration {N})

## Overview
{Brief description of the issue}

## Context
{File and line references}

## Suggested Approach
{The suggestion from the review}

## Future Considerations
{Any related strategic items}
```

---

## State File Reference

**Location:** `.review-fix/state.json`

```json
{
  "iteration": 1,
  "maxIterations": 3,
  "baseCommit": "main",
  "status": "in_progress",
  "startedAt": "2025-01-18T10:00:00Z",
  "branchName": "feature/example",
  "strategicItems": [
    {
      "severity": "important",
      "category": "architect",
      "file": "lib/services/auth.ts",
      "line": 45,
      "issue": "Authentication should use middleware pattern",
      "suggestion": "Refactor to use middleware for auth checks",
      "iteration": 1
    }
  ],
  "fixedItems": [
    {
      "issue": "Missing error boundary",
      "file": "components/Dashboard.tsx",
      "iteration": 1
    }
  ],
  "commits": [
    "abc1234"
  ],
  "iterationHistory": [
    {
      "iteration": 1,
      "quickFixCount": 8,
      "strategicCount": 2,
      "commitHash": "abc1234"
    }
  ]
}
```

---

## Iteration Logs

Each iteration creates a log file:

**Location:** `.review-fix/iteration-{N}.json`

```json
{
  "iteration": 1,
  "timestamp": "2025-01-18T10:05:00Z",
  "reviewers": {
    "designer": { "findings": 3, "quickFix": 2, "strategic": 1 },
    "pm": { "findings": 2, "quickFix": 0, "strategic": 2 },
    "qa": { "findings": 4, "quickFix": 4, "strategic": 0 },
    "architect": { "findings": 2, "quickFix": 0, "strategic": 2 },
    "developer": { "findings": 5, "quickFix": 5, "strategic": 0 },
    "maintainability": { "findings": 3, "quickFix": 2, "strategic": 1 },
    "reusable": { "findings": 1, "quickFix": 1, "strategic": 0 },
    "deadcode": { "findings": 2, "quickFix": 2, "strategic": 0 }
  },
  "totalFindings": 20,
  "quickFixItems": [],
  "strategicItems": [],
  "fixPrompt": "...",
  "fixResult": "success|failure",
  "buildResult": "pass|fail",
  "commitHash": "abc1234"
}
```

---

## Execution Instructions

When this skill is invoked:

### 1. Parse Parameters
```
/review-fix                              # defaults: max-iterations=3, auto-commit=true, base-commit=main
/review-fix max-iterations=5             # 5 iterations max
/review-fix auto-commit=false            # don't auto-commit
/review-fix skip-strategic=true          # skip strategic questions
/review-fix base-commit=abc123           # compare against specific commit
/review-fix base-commit=HEAD~3           # compare last 3 commits
/review-fix base-commit=develop          # compare against develop branch
```

### 2. Initialize
- Create `.review-fix/` directory if needed
- Initialize or load `state.json`
- Check if resuming a previous run

### 3. Run Loop
```
for iteration in 1..maxIterations:
  1. Extract branch context
  2. Spawn 8 reviewers IN PARALLEL (run_in_background: true)
  3. Wait for all reviewers (TaskOutput)
  4. Aggregate findings
  5. If no quick-fix items -> exit loop
  6. Generate fix prompt
  7. Spawn fix agent (run_in_background: false)
  8. Run build/lint
  9. Commit if auto-commit enabled
  10. Update state
```

### 4. Strategic Review
- Present accumulated strategic items
- Create backlog files as needed

### 5. Final Report
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

## Files Modified
{list of files touched during fix phases}
```

---

## Error Handling

### Reviewer Agent Failure
- Log the failure
- Continue with results from other reviewers
- Note in iteration log

### Fix Agent Failure
- Save fix prompt to `.review-fix/failed-fix-prompt-{N}.md`
- Increment iteration
- Try again with fresh context
- After 2 consecutive failures, exit and report to user

### Build/Lint Failure
- Do NOT commit
- Spawn debug sub-agent to analyze failure
- Include failure context in next fix prompt
- If build keeps failing, exit loop and report

### Max Iterations Reached
- Normal exit condition
- Report remaining quick-fix items (if any) to user
- Suggest manual review

---

## Persona-to-SubAgent Mapping

| Persona | Sub-Agent Type | Focus |
|---------|---------------|-------|
| Frontend Designer | `premium-ux-designer` | Visual polish, UX |
| Product Manager | `product-strategy-advisor` | Feature direction |
| QA Engineer | `general-purpose` | Test coverage |
| System Architect | `system-architect` | Architecture |
| Senior Developer | `code-refactorer` | Code quality |
| Code Maintainability | `code-refactorer` | Simplification |
| Reusable Components | `Explore` | Reuse opportunities |
| Dead Code Hunter | `Explore` | Unused code detection |

---

## Tips

1. **Start with default settings** - 3 iterations is usually enough
2. **Review strategic items thoughtfully** - They represent real decisions, not just code fixes
3. **Check the iteration logs** - Useful for understanding what was found and fixed
4. **Use skip-strategic for quick runs** - If you just want automated fixes
5. **Run after major features** - Gets your code clean before PR review
6. **Add `.review-fix/` to `.gitignore`** - State files are temporary and should not be committed
