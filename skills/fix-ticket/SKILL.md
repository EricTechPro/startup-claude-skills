---
name: fix-ticket
description: "Automate the full Jira bug-fix pipeline end-to-end. Use when the user says 'fix ticket', 'fix PROJ-123', 'fix this bug', 'resolve PROJ-XXX', 'fix and ship this ticket', or passes a Jira ticket ID for autonomous bug resolution. Reads the Jira ticket, implements the fix, reviews it, commits, moves the ticket to QA, assigns to QA engineer, and comments with a summary. Also use when the user wants to automate the fix-review-commit-handoff cycle for any Jira bug ticket."
---

# Fix Ticket

Automate the full lifecycle of a Jira bug ticket: read the ticket, implement the fix, review changes, commit, transition to QA, assign to QA engineer, and comment with a summary. Zero-touch pipeline from ticket to QA handoff.

## Prerequisites

**Read `CONFIG.md` at the start of every run.** This file contains project-specific settings (Jira project key, team members, transition IDs, deployment config, QA credentials). If `CONFIG.md` doesn't exist, warn the user and point them to `CONFIG.template.md`.

## When to Use

- User passes a Jira ticket ID (e.g., `PROJ-123`) and wants it fixed end-to-end
- User says "fix this ticket", "resolve this bug", "fix and ship PROJ-XXX"
- Any bug ticket that needs the full fix-review-commit-handoff cycle

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| (first arg) | required | Jira ticket ID (e.g., `PROJ-123`) |
| `branch` | `ask` | Branch strategy: `ask` (default — prompt user to choose), `worktree` (isolated worktree, auto-merges to main), `main` (commit directly to main), `new` (create fix branch) |
| `skip-review` | `false` | Skip the review-team phase |
| `skip-jira` | `false` | Skip Jira transition/assign/comment (just fix and commit) |
| `assignee` | (from CONFIG.md) | QA assignee: alias from CONFIG.md or a Jira `accountId` |
| `auto-commit` | `true` | Auto-commit after fix is verified |
| `skip-deploy` | `false` | Skip deployment monitoring |
| `skip-verify` | `false` | Skip pre-fix QA verification (headless browser bug reproduction) |
| `skip-qa-check` | `false` | Skip post-fix QA verification (headless browser fix confirmation) |

## Invocation

```
/fix-ticket PROJ-123                       # asks branch strategy first (default)
/fix-ticket PROJ-123 branch=main          # urgent fix, commit directly to main
/fix-ticket PROJ-123 branch=new           # create fix/PROJ-123 branch
/fix-ticket PROJ-123 branch=worktree      # isolated worktree (great for parallel runs)
/fix-ticket PROJ-123 skip-review=true
/fix-ticket PROJ-123 skip-jira=true
/fix-ticket PROJ-123 skip-deploy=true
/fix-ticket PROJ-123 skip-verify=true
/fix-ticket PROJ-123 skip-qa-check=true
```

## Architecture

```
ORCHESTRATOR (main session)
|
+-- Phase 0: BRANCH STRATEGY (runs FIRST — before any code work)
|   If branch=ask (default): prompt user with 3 options:
|     1. Main branch — commit directly (fast, for simple/urgent fixes)
|     2. Separate branch — create fix/{TICKET-ID} branch (for PR workflow)
|     3. Worktree — isolated worktree (for running parallel fix-tickets)
|   If branch=worktree: call EnterWorktree, symlink .env.local + node_modules
|   If branch=new: create fix/{TICKET-ID} branch
|   If branch=main: stay on current branch
|
+-- Phase 1: READ TICKET
|   Jira MCP -> ticket details, comments, acceptance criteria
|
+-- Phase 1.5: QA VERIFY <- headless browser bug reproduction (unless skip-verify=true)
|   Start dev server if needed, provision test user, login via browser
|   Reproduce the bug steps from the ticket in headless Chrome
|   Screenshot evidence, generate inline verdict
|   If not reproducible -> ask user whether to continue
|   -> Read references/qa-integration.md for full steps
|
+-- Phase 2: RESEARCH & UNDERSTAND <- MANDATORY + USER GATE
|   Research the bug (Explore agent or direct search)
|   Budget: ~10 tool calls, target root cause in <60s
|   Present findings to user (include QA verify evidence if available)
|   STOP — ask user "Does this look right?" — do NOT continue until they respond
|
+-- Phase 3: ANALYZE & PLAN
|   Classify complexity using thresholds below
|   For complex fixes: use delegation skill from CONFIG.md (if configured)
|   For simple fixes: plan and proceed
|
+-- Phase 4: IMPLEMENT
|   Apply the fix (direct edit or delegated)
|   Run build + lint to verify
|
+-- Phase 5: REVIEW <- MANDATORY (unless skip-review=true or delegation used)
|   Pick 3-5 agents from pool based on what the diff touches
|   Large/risky fixes (5+ files, DB, auth): escalate to full review team
|   If critical/high findings: fix -> re-review (max 2 cycles)
|   NEVER skip review because "change looks trivial"
|
+-- Phase 5.5: QA CHECK <- headless browser fix verification (unless skip-qa-check=true)
|   Re-provision test user, re-run same steps as Phase 1.5
|   Verify the bug is resolved — expected behavior now works
|   If fix incomplete -> loop back to Phase 4 (max 1 retry)
|   -> Read references/qa-integration.md for full steps
|
+-- Phase 6: COMMIT & PUSH <- BOTH REQUIRED
|   Branch strategy: ask user, main, or new fix branch
|   Handle remote-ahead: stash -> pull --rebase -> stash pop -> push
|   git push to remote — local-only commit = incomplete
|
+-- Phase 6.5: WORKTREE MERGE (if branch=worktree, runs AFTER Phase 7)
|   Merge worktree branch into main, push, clean up worktree + branch
|   Must use absolute paths (Bash tool resets cwd after cd)
|
+-- Phase 7: DEPLOYMENT MONITOR
|   Poll deployment platform until build succeeds
|   If Ready: proceed to Phase 6.5 merge (if worktree), then Phase 8
|   If Error: read logs, fix, re-commit, re-push, re-monitor (max 2 retries)
|
+-- Phase 8: JIRA HANDOFF (unless skip-jira=true)
|   Smart transition: QA (clean) / In Review (unresolved) / Done (trivial+skip-review)
|   Assign to QA engineer
|   Post comment: what was fixed, commit hash, QA steps, review findings
|   If ticket has existing comments: respond contextually
|
+-- Phase 9: SUMMARY
|   Report to user: what was done, commit hash, deployment status, Jira link
```

### Phase Execution Order — STRICT SEQUENTIAL

**You MUST execute phases 0 -> 1 -> 1.5 -> 2 -> 3 -> 4 -> 5 -> 5.5 -> 6 -> 7 -> 6.5 -> 8 -> 9 in order. Do NOT skip or reorder phases.** (Phase 0 always runs — it asks the user their branch preference when `branch=ask` (default), or sets up worktree when `branch=worktree`. Phase 1.5 skips when `skip-verify=true`. Phase 5.5 skips when `skip-qa-check=true`. Phase 6.5 only runs when worktree was chosen — it merges the worktree branch into main AFTER deployment succeeds.)

### Parameter Gate — How to Check Skip Flags

Before each skippable phase, perform this literal check on the invocation arguments:

```
Phase 1.5: Was `skip-verify=true` in the user's invocation? YES -> skip. NO -> run it.
Phase 5:   Was `skip-review=true` in the user's invocation? YES -> skip. NO -> run it.
Phase 5.5: Was `skip-qa-check=true` in the user's invocation? YES -> skip. NO -> run it.
Phase 7:   Was `skip-deploy=true` in the user's invocation? YES -> skip. NO -> run it.
```

Do NOT invent reasons to skip. The parameter either exists or it doesn't. If it doesn't exist, the phase is mandatory regardless of what the bug is about.

### Phases That Get Skipped (and Why That's Wrong)

Every rationalization below has been observed in real runs and is INVALID:

- **Phase 1.5 (QA Verify)**: "I'll research first then verify" — NO. "I'll verify in Phase 5.5 instead" — NO. "This is a navigation/routing bug" — navigation bugs are EXACTLY what browser testing catches. "Requires specific test data" — provision the data.
- **Phase 5.5 (QA Check)**: "This is a straightforward URL param change verified by build + review" — NO. Build + review verify code correctness; browser testing verifies user-facing behavior. Different things. "skip-qa-check" — check the invocation: was this parameter actually passed?
- **Phase 5 (Review)**: "Change is too small" — small changes with subtle bugs are the hardest to catch yourself.
- **Phase 7 (Deploy Monitor)**: A broken build = QA can't test.

When announcing each phase, use the exact header: `Phase {N}: {Name}` so the user can track progress.

### Context Budget Awareness

This pipeline typically uses 120-180k tokens across all phases. The biggest budget risks:

| Phase | Risk | Mitigation |
|-------|------|-----------|
| Phase 2 (Research) | Explore agent can burn 100k+ tokens | Use direct Glob/Grep/Read when ticket names specific files. Reserve Explore for vague bugs. |
| Phase 5 (Review) | 3 agents x ~15k tokens each | Keep agent prompts focused on the diff, not the whole file |
| Phase 7 (Deploy) | Each poll returns verbose JSON | After first poll finds deployment ID, extract only `state` from responses |
| Build output | `tail -20` returns too many lines | Use `tail -5` for build/lint verification |

If context usage exceeds 60% before Phase 6, switch to minimal output mode: shorter build output, no verbose commentary, brief review summaries.

### Mandatory Checklist (before reporting summary)

Before moving to Phase 9, verify ALL of these are done:

- [ ] **Branch strategy resolved**: User was asked (or explicit param used) — strategy stored for Phase 6
- [ ] **QA verify ran**: Bug reproduced in headless browser (or `skip-verify=true`)
- [ ] **Research done**: Bug was investigated AND user acknowledged findings with diagram
- [ ] **Review ran**: 3 agents were spawned (or `skip-review=true` was explicitly set)
- [ ] **Critical/high findings fixed**: All critical and high review findings addressed
- [ ] **QA check ran**: Fix verified in headless browser (or `skip-qa-check=true`)
- [ ] **Committed**: Changes committed with conventional commit message
- [ ] **Pushed**: `git push` executed and confirmed — local-only commits are NOT complete
- [ ] **Deployed**: Deployment platform checked and status confirmed
- [ ] **Worktree merged**: If `branch=worktree`, worktree merged to main, pushed, and cleaned up
- [ ] **Jira updated**: Ticket transitioned, assigned, and commented (or `skip-jira=true`)

**If any item is missing, go back and complete it before proceeding to Phase 9.**

---

## Workflow

### Phase 0: Branch Strategy

**This phase ALWAYS runs first — before reading the ticket or doing any work.**

**First: Read `CONFIG.md`** to load project-specific settings (team members, Jira transitions, deployment config, QA credentials). If not found, warn user and point to `CONFIG.template.md`.

If the `branch` parameter was explicitly set (`main`, `new`, or `worktree`), use that value. Otherwise (default `ask`), prompt the user:

```
Before I start, how would you like to handle the branch for this fix?

1. **Main branch** — commit directly to main (fast, for simple/urgent fixes)
2. **Separate branch** — create a `fix/{TICKET-ID}` branch (for PR workflow)
3. **Worktree** — isolated worktree (for running parallel fix-tickets)
```

**STOP — wait for the user's response before proceeding.**

Once the branch strategy is determined:

| Choice | Action |
|--------|--------|
| **Main** | Stay on current branch. No setup needed. |
| **Separate branch** | `git checkout -b fix/{TICKET-ID}` — create the branch now so all work happens on it. |
| **Worktree** | Call `EnterWorktree` with `name` = `fix-{TICKET-ID}`. Then immediately symlink `.env.local` and `node_modules` from the main repo root. All subsequent phases run inside the worktree. |

Store the resolved branch strategy — Phase 6 and Phase 6.5 need it for commit/push/merge logic.

### Phase 1: Read Ticket

Fetch the Jira ticket details using MCP:

```
mcp__jira__jira_get:
  path: /rest/api/3/issue/{ticketId}
  jq: "{key: key, summary: fields.summary, description: fields.description, status: fields.status.name, assignee: fields.assignee.displayName, priority: fields.priority.name}"
```

Also fetch comments to understand any ongoing conversation:

```
mcp__jira__jira_get:
  path: /rest/api/3/issue/{ticketId}/comment
  jq: "comments[*].{author: author.displayName, body: body, created: created}"
```

Fetch acceptance criteria if available (field ID from CONFIG.md):

```
mcp__jira__jira_get:
  path: /rest/api/3/issue/{ticketId}
  jq: "fields.{acceptance_criteria_field}"
```

Extract:
- **Summary**: What the bug is
- **Description**: Full context, repro steps, expected behavior
- **Acceptance criteria**: What "fixed" looks like
- **Comments**: Any discussion, additional context, or previous attempts
- **Priority**: Helps determine fix urgency

If the Jira fetch fails, ask the user to describe the bug manually and continue.

### Phase 1.5: QA Verify (Headless Browser Bug Reproduction)

**Skip if `skip-verify=true`.** Also skip if the QA tool (from CONFIG.md) is not installed — warn user and continue.

**Phase 1.5 runs BEFORE Phase 2 research. Do NOT skip it to "research first".** The browser evidence from Phase 1.5 directly informs Phase 2 research — seeing the actual bug behavior in a browser is more valuable than reading code first.

**Read `references/qa-integration.md` for the full provisioning, login, and interaction steps.**

#### Steps

1. **Start dev server** if not running (command and port from CONFIG.md):
   ```bash
   lsof -ti:{PORT} || ({DEV_COMMAND} &)
   ```
   Wait up to 30s for it to be ready. Set `WE_STARTED_SERVER=true` if you started it.

2. **Provision test user** — follow the provisioning steps in `references/qa-integration.md` for your auth provider (from CONFIG.md).

3. **Open browser, login, navigate** to the affected page — follow the login flow in `references/qa-integration.md`.

4. **Reproduce the bug** — follow the reproduction steps from the Jira ticket description. Screenshot each step:
   ```bash
   playwright-cli screenshot --filename=playwright-qa-screenshots/{ticket-id}-verify/NN-step.png
   ```

5. **Report verdict** to the user:

   | Verdict | Action |
   |---------|--------|
   | **Bug confirmed** | Show screenshots, continue to Phase 2 |
   | **Not reproducible** | Show screenshots, ask user whether to continue |
   | **Different behavior found** | Report what actually happened, ask user how to proceed |

6. **Keep browser open** — Phase 5.5 will reuse the session.

#### Error Handling

| Scenario | Action |
|----------|--------|
| QA tool not found | Warn user, skip QA phases, continue pipeline |
| Dev server won't start | Warn user, skip QA phases, continue pipeline |
| Auth fails | Retry with alternative auth method (max 2 attempts) |
| Browser crash | Kill all browser instances, retry once |

### Phase 2: Research & Understand

**CHECKPOINT: Did Phase 1.5 run?** If not, and `skip-verify=true` was NOT in the invocation, STOP and go back to Phase 1.5 NOW.

**MANDATORY: Before writing any code, you MUST research the affected area thoroughly.**

**Research budget**: Keep research focused. Start with the lightest strategy:

| Ticket specificity | Strategy | Token budget |
|-------------------|----------|-------------|
| Names specific component/file/function | Direct search: Glob -> Read -> Grep | ~5 tool calls, <10k tokens |
| Names a feature area but not specific files | Explore agent with focused prompt | ~15 tool calls, <50k tokens |
| Vague, no code pointers | Explore agent with broad prompt | ~25 tool calls, <80k tokens |

**How to research**: Use an `Explore` agent (preferred for thorough investigation) or search directly with Glob/Grep/Read. Either way, you must:

1. **Search the codebase** for files related to the bug
2. **Map the affected code paths**: all files involved, how they interact, where the bug originates
3. **Understand the current behavior**: what the code does vs. what it should do
4. **Check for existing patterns**: look in shared libs for utilities that could help

**Output to the user**: Present findings using this structure:

1. **Root cause** (1-2 sentences)

2. **Bug flow diagram** (MANDATORY — always include a visual):

   Draw an ASCII data-flow diagram showing how data/control moves through the bug. Use arrows with annotations marking the bug point:

   ```
   User clicks [Action] -> ComponentA
       -> handlerFunction()
       -> someCall('/api/endpoint')  <- BUG: incorrect parameter
       -> wrong behavior in UI
   ```

   Keep it under 10 lines. Mark the bug with `<- BUG:` or `X` and the fix with `OK`.

3. **Affected files** and their roles (table or bullet list)

4. **Proposed approach** (1-3 sentences)

Then ask: **"Does this look right, or would you like a different approach?"**

**STOP HERE. Do NOT proceed to Phase 3 until the user responds.** This is a hard gate — even if the fix seems obvious, wait for the user to confirm.

### Phase 3: Analyze & Plan

1. **Classify complexity** using these thresholds:

| Threshold | Simple | Complex |
|-----------|--------|---------|
| Files changed | 1-2 files | 3+ files with different concerns |
| Lines changed | < ~100 lines total | 100+ lines across multiple files |
| Scope | Single concern | Cross-cutting (API + UI + service + DB) |
| DB migration | No | Yes |
| New files | No | Yes |

   **Edge case**: 3+ files but same mechanical concern (renaming a constant, updating imports) -> still Simple.

   - **Simple fix**: Proceed to Phase 4 directly
   - **Complex fix**: If a delegation skill is configured in CONFIG.md, invoke it. Otherwise, plan the implementation in detail and proceed.

2. **For simple fixes**, create a brief plan:
   - Which files to change
   - What the change is
   - How to verify it works

### Phase 4: Implement

For **simple fixes**:

1. Make the code changes using Edit tool
2. Verify the fix compiles and passes lint:

```bash
npm run build 2>&1 | tail -5
npm run lint 2>&1 | tail -5
```

3. If build/lint fails, fix the issues and re-verify
4. Run relevant tests if they exist

For **complex fixes** delegated to another skill: wait for completion, then continue from Phase 5.

### Phase 5: Review

**Skip if `skip-review=true`.** Also skip if a delegation skill was used that includes its own review.

**MANDATORY: You MUST spawn review agents. There are ZERO exceptions other than `skip-review=true`.**

If you catch yourself thinking "this change is too small to review" — that is exactly the signal to RUN the review. Small changes with subtle bugs are the hardest to catch yourself.

**HARD GATE: Do NOT proceed to Phase 6 without spawning agents.**

#### Agent Pool

Pick 3-5 agents from this pool based on what the fix actually touches:

**Tier 1 — Always include (mandatory):**

| Agent | Source | Focus |
|-------|--------|-------|
| Code Reviewer | PR Review Toolkit | Bugs, logic errors, style violations |
| Silent Failure Hunter | PR Review Toolkit | Swallowed errors, empty catches, missing logging |

**Tier 2 — Pick based on what the diff touches (add 1-3):**

| Agent | Source | Focus | When to pick |
|-------|--------|-------|-------------|
| Security & Architecture | `system-architect` | Auth checks, injection risks, XSS, SQL injection | API routes, auth, user input, DB queries |
| Type Design Analyzer | PR Review Toolkit | Type encapsulation, invariant expression | New/modified types, interfaces, schemas |
| Test Analyzer | PR Review Toolkit | Behavioral coverage gaps, missing edge cases | Fix adds/modifies tests |
| Code Simplifier | PR Review Toolkit | Unnecessary complexity, redundant abstractions | Fix is verbose or over-engineered |
| Comment Analyzer | PR Review Toolkit | Comment accuracy vs code, documentation rot | Fix modifies comments/docs |
| State & Data Flow | `code-reviewer` | Race conditions, stale state, missing cleanup | Hooks, stores, context, async flows |
| UI/UX Reviewer | `code-reviewer` | Accessibility, responsive breakpoints, error/loading states | Components, layouts, user-facing UI |
| Database Reviewer | `system-architect` | Query performance, missing indexes, migration safety | DB queries, migrations |
| API Contract Reviewer | `code-refactorer` | Request/response shape, error codes, backwards compatibility | API routes, server actions |

**Selection rules:**
- **Minimum 3 agents**, maximum 5
- **Code Reviewer + Silent Failure Hunter** are mandatory in every review
- Pick the remaining 1-3 based on which areas the diff touches

#### Steps

1. Run `git diff --stat` to see the scope of changes
2. **Select agents** based on the diff
3. **Announce the team** before spawning:
   ```
   Spawning {N} review agents: {Agent1} ({type}), {Agent2} ({type}), ... — reviewing {N} files, {M} lines changed
   ```
4. **Spawn agents in parallel** (`run_in_background: true`), each receiving the `git diff` output
5. **Report completion** as each agent finishes
6. Merge and deduplicate findings

**After review completes:**
- Critical or high issues: Fix them, re-run build/lint, create a new commit
- Medium/low issues: Note them in the Jira comment, fix if quick
- No issues: Note "No issues found" in summary

**Iteration limit**: Max 2 fix-then-re-review cycles.

### Phase 5.5: QA Check (Headless Browser Fix Verification)

**PARAMETER CHECK: Read back the user's invocation arguments. Is `skip-qa-check=true` literally present?**
- **YES** -> skip this phase.
- **NO** -> this phase is MANDATORY.

Do NOT infer `skip-qa-check` from the nature of the bug. Build + review verify code correctness; browser testing verifies user-facing behavior. They test different things.

**Read `references/qa-integration.md` for the full interaction steps.**

#### Steps

1. **Re-provision test user** (reset test data)
2. **Restore browser session** or re-login
3. **Navigate to the same page** tested in Phase 1.5
4. **Re-run the reproduction steps** — the bug should no longer occur. Screenshot each step.
5. **Check for regressions**: look for new console errors
6. **Report verdict**:

   | Verdict | Action |
   |---------|--------|
   | **Fix verified** | Show before/after screenshots, proceed to Phase 6 |
   | **Fix incomplete** | Report what still fails -> loop back to Phase 4 (max 1 retry) |
   | **Regression found** | Report the regression -> loop back to Phase 4 |

7. **Cleanup** — close browser, clean up test user, kill dev server if you started it

#### Escalation After 2 Failed QA Attempts

If still failing after 2 attempts, present the user with a structured refinement menu:

```
QA check failed twice. Here's what's still broken:
- <specific failure with screenshot references>
- <browser console errors if any>

How would you like to proceed?

1. **Manual test** — I'll keep the browser open so you can inspect it yourself.
2. **Different approach** — I'll re-research the bug from scratch and try an alternative fix.
3. **Commit with known issue** — Ship what we have and note the remaining issue in Jira.
4. **Pair debug** — Walk me through what you expect step by step.
```

**STOP — wait for the user's response.**

### Phase 6: Commit & Push

**MANDATORY: You MUST both commit AND push. A local-only commit is not a completed fix.**

#### Branch strategy

Use the strategy resolved in Phase 0:

| Resolved strategy | Phase 6 action |
|-------------------|---------------|
| **main** | Commit directly to the current branch |
| **separate branch** | Already on `fix/{TICKET-ID}` from Phase 0 — just commit |
| **worktree** | Already inside worktree from Phase 0 — commit in worktree branch |

#### Worktree flow

**Phase 6.5: Worktree Merge & Cleanup (after Phase 7 succeeds):**

After deployment succeeds, merge the fix into main and clean up. Use a single chained Bash command (Bash tool resets cwd between calls):

```bash
REPO_ROOT=$(git worktree list | head -1 | awk '{print $1}') && BRANCH_NAME=$(git branch --show-current) && cd "$REPO_ROOT" && git checkout main && git pull origin main && git merge "$BRANCH_NAME" --no-edit && git push origin main && echo "Merged $BRANCH_NAME into main"
```

Then clean up:
```bash
REPO_ROOT=$(git worktree list | head -1 | awk '{print $1}') && WORKTREE_PATH=$(pwd) && BRANCH_NAME=$(git branch --show-current) && cd "$REPO_ROOT" && git worktree remove --force "$WORKTREE_PATH" && git branch -d "$BRANCH_NAME" 2>/dev/null && git push origin --delete "$BRANCH_NAME" 2>/dev/null && echo "Worktree cleaned up"
```

**If merge conflicts occur:** Stop and ask the user. Do NOT force-merge.

#### Commit

Create a conventional commit referencing the ticket:

```bash
git add <specific-files-changed>
git commit -m "$(cat <<'EOF'
fix: <concise description of what was fixed>

<Optional body: 1-2 sentences explaining the root cause and fix>

Resolves {TICKET-ID}

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

Then push to remote:

```bash
git push origin <current-branch>
```

**If push is rejected** (remote has newer commits):

```bash
git stash && git pull --rebase origin <current-branch> && git stash pop
git push origin <current-branch>
```

If pushed to a feature branch, offer to create a PR.

Capture the commit hash: `git rev-parse --short HEAD`

### Phase 7: Deployment Monitor

**Skip if `skip-deploy=true`.**

**After pushing, monitor the deployment to confirm the build succeeds before handing off to QA.**

The deployment platform is configured in CONFIG.md. Currently supported: **Vercel** (via MCP tools).

#### Vercel

Read `.vercel/project.json` for the project ID and team ID.

1. **Find the latest deployment**: Use `mcp__claude_ai_Vercel__list_deployments`. Find the one matching your commit SHA with `state: "BUILDING"`.

2. **Poll**: Use `mcp__claude_ai_Vercel__get_deployment` with the deployment ID.

   | State | Action |
   |-------|--------|
   | `BUILDING` | Wait, poll again |
   | `READY` | Report success, proceed |
   | `ERROR` | Read logs, fix, re-commit, re-push (max 2 retries) |
   | `CANCELED` | Warn user, proceed |

   **Polling schedule**: 45s, 45s, 60s, 60s, 60s, 60s (max 6 polls).

3. **On failure**: Read build logs via `mcp__claude_ai_Vercel__get_deployment_build_logs`, fix the code, create a new commit, push, and re-monitor.

#### Other Platforms

If `platform` in CONFIG.md is `none`, skip this phase entirely.
For unsupported platforms, warn the user and suggest they verify the deployment manually.

### Phase 8: Jira Handoff

**Skip if `skip-jira=true`.**

Resolve the assignee from CONFIG.md using the `assignee` parameter (alias lookup).

**IMPORTANT: Jira MCP `body` parameter must be a JSON object, NOT a JSON string.**

Execute these three Jira operations in parallel:

**1. Transition ticket** (transition IDs from CONFIG.md):

| Condition | Target |
|-----------|--------|
| Review had unresolved medium+ findings | In Review |
| Normal fix (review clean or all findings resolved) | QA |
| Trivial fix + `skip-review=true` + build passes | Done |

```
mcp__jira__jira_post:
  path: /rest/api/3/issue/{ticketId}/transitions
  body: {"transition": {"id": "<transition-id-from-config>"}}
```

**2. Assign to QA engineer:**

```
mcp__jira__jira_put:
  path: /rest/api/3/issue/{ticketId}
  body: {"fields": {"assignee": {"accountId": "<account-id-from-config>"}}}
```

**3. Post comment** (Jira ADF format):

Include:
- What was fixed (root cause + solution)
- Commit reference (short hash)
- QA verification steps
- Review findings summary

If the ticket has existing comments with questions, acknowledge them.

```json
{
  "body": {
    "type": "doc",
    "version": 1,
    "content": [
      {
        "type": "paragraph",
        "content": [
          {"type": "text", "text": "Fix committed: <description>."}
        ]
      },
      {
        "type": "paragraph",
        "content": [
          {"type": "text", "text": "Commit: <short-hash> — <commit subject line>"}
        ]
      },
      {
        "type": "paragraph",
        "content": [
          {"type": "text", "text": "QA verification steps:", "marks": [{"type": "strong"}]}
        ]
      },
      {
        "type": "orderedList",
        "attrs": {"order": 1},
        "content": [
          {
            "type": "listItem",
            "content": [
              {"type": "paragraph", "content": [{"type": "text", "text": "<step 1>"}]}
            ]
          }
        ]
      }
    ]
  }
}
```

### Phase 9: Summary

Report to the user:

```
## Fix Complete: {TICKET-ID}

**Ticket**: {summary}
**Status**: Moved to {QA|In Review|Done} -> Assigned to {assignee name}
**Commit**: `{short-hash}` — {commit subject}
**Deployment**: {Ready at <url> / Failed / Skipped}

### What was fixed
{1-2 sentence description of root cause and fix}

### Files changed
{list of files}

### Review findings
{summary or "Skipped" or "No issues found"}

### Deployment
{Build succeeded / Failed after N retries / Monitoring skipped}

### QA steps posted to Jira
{numbered list}

### Phase tracker
| Phase | Status |
|-------|--------|
| 0. Branch Strategy | {main / separate branch / worktree} |
| 1. Read Ticket | {Done/Failed} |
| 1.5. QA Verify | {Bug confirmed / Not reproducible / Skipped} |
| 2. Research | {Done — user acknowledged} |
| 3. Plan | {Simple fix / Complex -> delegated} |
| 4. Implement | {Done — build+lint passed} |
| 5. Review | {Done — N findings / Skipped} |
| 5.5. QA Check | {Fix verified / Fix incomplete / Skipped} |
| 6. Commit & Push | {Done — hash} |
| 7. Deploy | {Ready / Failed / Skipped} |
| 6.5. Worktree Merge | {Merged + cleaned up / N/A} |
| 8. Jira Handoff | {Done / Skipped} |
| 9. Summary | Done |
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Jira MCP fails to read ticket | Ask user to describe the bug, continue without Jira context |
| Jira MCP "expected record" error | Body was passed as a string — pass as JSON object instead |
| Build fails after fix | Diagnose and fix, retry up to 2 times, then pause and ask user |
| Lint fails after fix | Auto-fix with formatter, retry commit |
| Review finds critical issues | Fix them before committing |
| Jira transition fails | Log warning, continue with assign + comment, tell user to move manually |
| Push rejected (remote ahead) | `git stash && git pull --rebase origin <branch> && git stash pop && git push` |
| Commit fails (pre-commit hook) | Fix issues, create new commit (never amend) |
| Delegation skill fails | Fall back to direct implementation |
| Deployment build fails | Read logs, fix code, new commit, re-push, re-monitor (max 2 retries) |
| Deployment platform not configured | Warn user, skip deploy monitoring |
| Worktree merge conflict | Stop, show conflicting files, ask user how to proceed |
| QA tool not found | Warn user, skip QA phases (1.5 and 5.5), continue pipeline |
| Dev server won't start | Warn user, skip QA phases, continue pipeline |
| QA verify: bug not reproducible | Report to user with screenshots, ask whether to continue |
| QA check: fix not verified | Loop back to Phase 4 (max 1 retry), then present 4-option refinement menu |

---

## Tips

1. **Simple bugs are the sweet spot** — This skill shines for 1-2 file fixes where the overhead of manual Jira management exceeds the coding effort
2. **Skip review for trivial fixes** — Use `skip-review=true` for obvious one-liner fixes
3. **Add more QA assignees** — Update CONFIG.md as your team grows
4. **Existing comments matter** — The skill reads ticket comments and responds contextually
5. **Parallel fixes with worktrees** — Use `branch=worktree` to run multiple `/fix-ticket` in separate terminals simultaneously

---

## Evaluation Criteria

After each run, check whether the skill passed on these specific behaviors. A run passes if ALL criteria are met:

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| 1 | **Phase 0 branch question** | User prompted for branch strategy before Phase 1 (when `branch=ask` or default) | Skipped branch question on default |
| 2 | **Diagram format** | Phase 2 includes a data-flow ASCII diagram (<=10 lines, `<- BUG:` marker) | No diagram |
| 3 | **User gate** | Phase 2 stops and waits for user response before proceeding | Skips straight to Phase 3 |
| 4 | **Review enforcement** | Phase 5 spawns 3-5 agents including both mandatory agents (unless `skip-review=true`) | Review skipped or fewer than 3 agents |
| 5 | **Agent selection** | Agents chosen match the areas the diff touches | Fixed agents regardless of diff content |
| 6 | **Push after commit** | `git push` executed — not just local commit | Local commit only |
| 7 | **Deploy monitoring** | Deployment checked after push (unless `skip-deploy=true`) | Skipped without parameter |
| 8 | **Jira handoff** | Ticket transitioned + assigned + commented (unless `skip-jira=true`) | Any Jira op missing |
| 9 | **Phase headers** | Each phase announced as `Phase {N}: {Name}` | Inconsistent naming |
| 10 | **Phase tracker** | Summary includes the phase tracker table | Missing phase tracker |
| 11 | **QA verify ran** | Phase 1.5 runs with screenshots (unless `skip-verify=true`) | Skipped without parameter |
| 12 | **QA check ran** | Phase 5.5 re-runs reproduction steps (unless `skip-qa-check=true`) | Skipped without parameter |
| 13 | **QA retry loop** | If QA fails, loops back to Phase 4, then presents 4-option menu | Proceeds despite failed QA |
| 14 | **Research budget** | Uses appropriate strategy for ticket specificity | Explore agent used for specific bugs |
| 15 | **Deploy polling efficiency** | First poll at 45s+. Max 6 polls. | Polling starts at 10s |
| 16 | **Worktree cleanup** | Worktree removed, branch deleted locally and remotely | Cleanup errors left unresolved |
