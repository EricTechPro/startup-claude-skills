---
name: fix-sentry
description: "Scan all unresolved Sentry errors, group by root cause, fix in batch with TDD, and resolve in Sentry after deploy. Use when the user says 'fix sentry errors', 'scan sentry', 'resolve sentry issues', 'fix all errors', 'sentry cleanup', or wants to proactively fix production errors from Sentry monitoring."
---

# Fix Sentry Errors

Proactively scan Sentry for all unresolved errors, group them by root cause, create a unified fix plan, implement with TDD, review, deploy, and resolve the issues in Sentry.

## Prerequisites

**Read `CONFIG.md` at the start of every run.** This file contains project-specific Sentry settings (org slug, project slug, region URL, deploy provider). If `CONFIG.md` doesn't exist, warn the user and point them to `CONFIG.template.md`.

## When to Use

- User wants to proactively fix production errors from Sentry
- User says "fix sentry errors", "scan sentry", "sentry cleanup"
- Periodic maintenance to clear unresolved Sentry issues

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `branch` | `ask` | Branch strategy: `ask` (prompt user), `worktree` (isolated worktree), `main` (commit directly to main) |

## Pipeline Overview

```
Phase 1: Fetch all unresolved errors from Sentry
Phase 2: Triage & group by root cause → ⛔ USER GATE
Phase 3: Diagnose each group (parallel agents) → ⛔ USER GATE
Phase 4: Create unified fix plan (writing-plans)
Phase 5: Implement with TDD (executing-plans)
Phase 6: Review (layered dispatch)
Phase 6.1: Compatibility check (parallel agents)
Phase 6.2: Simplify (/simplify)
Phase 7: Commit & push
Phase 8: Deploy monitor (Vercel MCP polling)
Phase 9: Resolve in Sentry (with verification gate)
Phase 10: Summary
```

---

### Phase 1: Fetch All Unresolved Errors

**Goal:** Retrieve all unresolved Sentry issues for the project.

Read Sentry config from `CONFIG.md`:
- `sentry.org` — org slug
- `sentry.project` — project slug
- `sentry.region_url` — region URL (e.g., `https://us.sentry.io`)

1. **Search for unresolved issues** using Sentry MCP:
   ```
   mcp__claude_ai_Sentry__search_issues
   - org: {sentry.org from CONFIG.md}
   - project: {sentry.project from CONFIG.md}
   - query: "is:unresolved"
   - sort: "events" (most events first)
   ```

2. **Cap at 25 issues.** If more than 25 unresolved issues exist, fetch only the top 25 by event count and warn:
   ```
   ⚠️ Found {N} unresolved issues — fetching top 25 by event count.
   Consider running fix-sentry again for the remainder.
   ```

3. **For each issue (up to 25)**, fetch details using `mcp__claude_ai_Sentry__get_sentry_resource`:
   - Exception type + message
   - Stack trace (top 5 frames)
   - Event count + user count
   - First seen / last seen
   - Tags: browser, environment, URL, release

4. **Compile structured error list** — one entry per issue with all fields above.

**Error handling:**
- If Sentry MCP auth fails: prompt user to authenticate, retry once, abort if still fails
- If no unresolved issues found: report "No unresolved Sentry errors — nothing to fix" and exit

---

### Phase 2: Triage & Group ⛔ USER GATE

**Goal:** Cluster errors by root cause, rank by impact, and get user's scope decision.

1. **Cluster errors by root cause** using these rules:
   - Same file + same function → same group
   - Related stack traces (shared caller chain) → same group
   - Same error message with different URLs → same group

2. **Rank groups by impact:** `event_count × user_count` (highest first)

3. **Present to user:**
   ```
   Found {N} unresolved errors in {M} groups:

   | # | Group | Errors | Events | Users | Root File |
   |---|-------|--------|--------|-------|-----------|
   | 1 | {description} | 5 | 120 | 45 | lib/services/foo.ts |
   | 2 | {description} | 3 | 80 | 20 | app/api/bar/route.ts |
   ...

   Fix all / Pick specific groups / Cap at top N?
   ```

4. **If >10 groups:** suggest splitting across sessions:
   ```
   That's a lot of groups. Recommend picking top 5 for this session
   and running fix-sentry again for the rest.
   ```

**⛔ STOP — wait for user's scope decision before proceeding.**

Store: `selected_groups` for downstream phases.

---

### Phase 3: Diagnose ⛔ USER GATE

**Goal:** Determine root cause for each selected error group using parallel research agents.

For each selected error group, dispatch parallel research agents using `dispatching-parallel-agents`:

**Per group (2 agents in parallel):**

**Agent A — Superpowers Systematic Debugging:**

Spawn a sub-agent with this prompt:
```
Investigate this Sentry error group using the systematic-debugging methodology.

ERROR GROUP: {group_description}
ERRORS IN GROUP: {list of error titles and Sentry IDs}
STACK TRACES: {top 5 frames from each error}
AFFECTED FILES: {files appearing in stack traces}
EVENT COUNT: {total events across group}
USER COUNT: {total affected users}

Follow the 4-phase process:
1. Reproduce: confirm the bug exists in code (read affected files)
2. Find working examples: find similar code that DOES work correctly
3. Compare differences: what's different between working and broken?
4. Form hypothesis: single root cause with evidence

Do NOT fix anything. Do NOT edit any files.

Return a structured summary:
- ROOT_CAUSE: 1-2 sentences
- EVIDENCE: specific code references (file:line)
- AFFECTED_FILES: list with roles
- HYPOTHESIS: what the fix should be
```

**Agent B — gstack /investigate:**

Spawn a sub-agent with this prompt:
```
Debug this Sentry error group using the /investigate methodology (4-phase iron law).

ERROR GROUP: {group_description}
ERRORS IN GROUP: {list of error titles and Sentry IDs}
STACK TRACES: {top 5 frames from each error}
AFFECTED FILES: {files appearing in stack traces}
EVENT COUNT: {total events across group}
USER COUNT: {total affected users}

Follow the iron law: no fixes without root cause.
Phase 1: Investigate — read errors, check recent changes, gather evidence
Phase 2: Analyze — find patterns, compare against references
Phase 3: Hypothesize — form single hypothesis with evidence

Do NOT implement fixes. Do NOT edit any files.

Return a structured summary:
- ROOT_CAUSE: 1-2 sentences
- EVIDENCE: specific code references (file:line)
- AFFECTED_FILES: list with roles
- HYPOTHESIS: what the fix should be
```

**Both agents run in parallel** (`run_in_background: true`). Wait for both to complete.

#### Aggregation (main orchestrator)

After both agents return for each group:

1. **Compare root causes**: Do both agents agree?
2. **If they agree** (high confidence): Merge into a unified finding
3. **If they disagree**: Present both with evidence to the user:
   ```
   Two research agents investigated Group {N} and reached different conclusions:

   **Agent A (systematic-debugging):**
   Root cause: {A's root cause}
   Evidence: {A's evidence}

   **Agent B (/investigate):**
   Root cause: {B's root cause}
   Evidence: {B's evidence}

   Which analysis looks more accurate? (A / B / neither — let me dig deeper)
   ```
   The user's choice becomes the root cause for that group.
4. **Merge affected file lists** (union from both agents)

After all groups diagnosed, present unified diagnosis:
- Per-group root cause + evidence
- Cross-group patterns (shared root causes across groups)
- Total affected files (deduplicated)

**⛔ STOP: "Does this diagnosis look right?"**

Wait for user confirmation before proceeding.

---

### Phase 4: Plan

**Goal:** Create a unified implementation plan for all selected error groups.

Delegate planning to `superpowers writing-plans`:

Spawn a sub-agent with this prompt:
```
Create a unified implementation plan for these Sentry error fixes using the writing-plans methodology.

ERROR GROUPS:
{for each group: group_description, root_cause, affected_files, hypothesis}

CROSS-GROUP PATTERNS:
{shared root causes if any, shared affected files}

PROJECT: Read CLAUDE.md for conventions, patterns, and project structure.

Requirements:
- TDD: every task starts with a failing test
- Group related fixes into the same task when they share files
- Exact file paths, complete code in every step
- Bite-sized tasks (2-5 minutes each)
- Save plan to: docs/superpowers/plans/fix-sentry-{YYYY-MM-DD}.md

Do NOT implement — only plan.
```

Store the returned `plan_file_path` for Phase 5.

---

### Phase 5: Implement

**Goal:** Implement all fixes using TDD methodology.

**MANDATORY: Write a failing test BEFORE implementing each fix.** This proves the error exists and prevents false-positive "fix verified" claims.

Spawn a sub-agent for the implementation:

```
Implement the Sentry error fixes using TDD.

PLAN: {plan_file_path from Phase 4}
ERROR GROUPS: {group summaries with root causes}
PROJECT: Read CLAUDE.md for conventions, patterns, and project structure.

For each task in the plan, follow this sequence strictly:

STEP 1 — RED (prove the error):
Use the test-driven-development methodology.
Write a minimal failing test that demonstrates the error.
The test should assert the CORRECT behavior — it will fail because the bug exists.
Run the test and confirm it FAILS.
If it passes, your test doesn't capture the bug — rewrite it.

STEP 2 — GREEN (fix the error):
Follow the plan from {plan_file_path} using executing-plans.
Make the minimal code change to make the test pass.
Run the test and confirm it PASSES.

STEP 3 — VERIFY:
npm run build 2>&1 | tail -5
npm run lint 2>&1 | tail -5
npm test -- --testPathPattern="{test_file}" 2>&1

Return:
- FILES_CHANGED: list of modified files
- TEST_FILES: paths to new/modified tests
- TEST_RESULT: PASS/FAIL with output
- BUILD_STATUS: PASS/FAIL
- LINT_STATUS: PASS/FAIL
```

**If BUILD or LINT fails:** review the error, fix, re-verify. Max 2 attempts before escalating to user.

**If TEST still FAILS after fix:** the fix is wrong — loop back. Re-analyze using test failure output as new evidence. Max 2 implementation attempts before escalating to user.

---

### Phase 6: Review

**Goal:** Multi-agent code review of all changes.

**MANDATORY. Skip only if Phase 5 used executing-plans with a plan that included its own review cycle.**

#### Layered Review Architecture

Review uses `dispatching-parallel-agents` with two layers:

**Layer 1 — Always run (2 agents):**

| Agent | Skill | Focus |
|-------|-------|-------|
| Plan Compliance Reviewer | `superpowers requesting-code-review` | Do the fixes match the root causes and plan? |
| Structural Reviewer | `gstack /review` | SQL safety, LLM trust boundaries, conditional side effects |

**Layer 2 — Based on diff analysis (1-3 agents):**

Run `git diff --stat` and classify changed files:

| If diff touches... | Add agent |
|-------------------|-----------|
| `catch`, `error`, `throw`, `.catch(` | `pr-review-toolkit:silent-failure-hunter` |
| `type `, `interface `, `z.object`, `z.string` | `pr-review-toolkit:type-design-analyzer` |
| Lines are verbose or deeply nested | `pr-review-toolkit:code-simplifier` |

**Minimum total: 3 agents. Maximum: 5 agents.**

#### Steps

1. Run `git diff --stat` to see the scope of changes
2. Capture BASE_SHA (`git rev-parse HEAD~1`) and HEAD_SHA (`git rev-parse HEAD`) — if no commit yet, use unstaged diff
3. **Select Layer 2 agents** based on the diff classification above
4. **Announce the team**:
   ```
   Spawning {N} review agents: {Agent1} (Layer 1), {Agent2} (Layer 1), {Agent3} (Layer 2), ... — reviewing {N} files, {M} lines changed
   ```
5. **Dispatch all agents in parallel** (`run_in_background: true`):

   **Layer 1 — Plan Compliance** (superpowers requesting-code-review):
   ```
   Review these Sentry error fixes against the requirements.
   WHAT_WAS_IMPLEMENTED: Fixes for {N} Sentry error groups
   PLAN_OR_REQUIREMENTS: {plan_file_path}
   BASE_SHA: {base_sha}
   HEAD_SHA: {head_sha}
   DESCRIPTION: Batch fix for {N} Sentry error groups. Root causes: {summary}.
   ```

   **Layer 1 — Structural** (gstack /review):
   ```
   Pre-landing review of the diff on this branch.
   Run git diff against the base branch.
   Focus on: SQL safety, LLM trust boundaries, conditional side effects, structural issues.
   ```

   **Layer 2 agents** (PR Review Toolkit):
   Each agent receives: `Run git diff to see the changes. This is a read-only review — no edits. Return findings with severity (critical/high/medium/low), file:line, and specific fix suggestion. Max 10 findings.`

6. **Report completion** as each agent finishes:
   ```
   Agent 1 (Plan Compliance): Done — 2 findings (0 critical, 1 high, 1 medium)
   Agent 2 (Structural): Done — 0 findings
   Agent 3 (Silent Failure Hunter): Done — 1 finding (0 critical, 0 high, 1 medium)
   ```

7. **Merge and deduplicate findings**:
   - Group by file:line
   - Keep highest severity when duplicated across agents
   - Cap at 15 findings total
   - Sort by severity (critical → high → medium → low)

**After review completes:**
- **Critical or high issues**: Fix them, re-run build/lint, create a new commit
- **Medium/low issues**: Note them, fix if quick
- **No issues**: Note "No issues found" in summary

**Iteration limit**: Max 2 fix-then-re-review cycles.

---

### Phase 6.1: Compatibility Check

**Goal:** Verify backward compatibility and pattern consistency.

Dispatch 3 parallel sub-agents using `dispatching-parallel-agents`:

**Agent 1 — Backward Compatibility Checker:**
```
Check backward compatibility of the changes on this branch.

Review all modified files. For each, check:
- Exported function signatures: any parameter changes?
- Exported types/interfaces: any removed or renamed fields?
- Server actions: any changed return types?
- API routes: any changed response shapes?

Return findings as:
- SEVERITY: critical/important/minor
- FILE: path
- ISSUE: what broke
- FIX: how to restore compatibility
```

**Agent 2 — Reusability Auditor:**
```
Audit the changes for code reuse opportunities.

Check:
- Any logic in components that should be in lib/?
- Any duplicated code across the changed files?
- Any utility functions that already exist in lib/ being reimplemented?

Return findings as:
- SEVERITY: important/minor
- FILE: path
- ISSUE: what's duplicated or misplaced
- FIX: where to move it
```

**Agent 3 — Pattern Consistency Checker:**
```
Check the changes follow existing project patterns.

Read CLAUDE.md for project conventions. Then review:
- Do new functions follow naming conventions of sibling files?
- Do error handlers follow the catch block rules?
- Do API calls use apiClient?
- Do date operations use parseISO/format?

Return findings as:
- SEVERITY: important/minor
- FILE: path
- ISSUE: what's inconsistent
- FIX: the correct pattern
```

**Fix critical/important findings.** Minor findings: note for future.

---

### Phase 6.2: Simplify

**Goal:** Remove verbosity and ensure DRY compliance.

Invoke `/simplify scope=session`:
- Auto-fix DRY violations
- Remove dead code introduced by the fixes
- Consolidate duplicated logic into lib/ utilities
- Re-run `npm run build` and `npm run lint` if changes are made

---

### Phase 7: Commit & Push

**Goal:** Create a clean commit with all Sentry issue references and push.

1. **Stage all changes:**
   ```bash
   git add -A
   ```

2. **Create commit** with this message format:
   ```
   NO-TICKET: fix: resolve {N} Sentry errors

   Fixes:
   - {SENTRY-ID-1}: {title}
   - {SENTRY-ID-2}: {title}
   - {SENTRY-ID-3}: {title}
   ...

   Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
   ```

3. **Push to remote:**
   ```bash
   git push
   ```

Store: `commit_hash`, `push_status` for Phase 10.

---

### Phase 8: Deploy Monitor

**Goal:** Wait for deployment to complete.

Read deploy provider from `CONFIG.md` (`deploy.provider`). Default: Vercel.

**For Vercel:**
1. Use `mcp__claude_ai_Vercel__list_deployments` to find the latest deployment
2. Poll on this schedule: 45s, 45s, 60s, 60s, 60s, 60s (max 6 polls)
3. Check deployment state each poll:
   - **READY**: deployment succeeded — proceed to Phase 9
   - **ERROR**: read build logs via `mcp__claude_ai_Vercel__get_deployment_build_logs`, diagnose, fix, re-commit, re-push (max 2 retries)
   - **BUILDING/QUEUED**: continue polling

Store: `deploy_status`, `deploy_url` for Phase 10.

**If deploy fails after 2 retries:** report failure and skip Phase 9 (cannot resolve Sentry issues without successful deploy).

---

### Phase 9: Resolve in Sentry ⛔ VERIFICATION GATE

**Goal:** Mark fixed issues as resolved in Sentry, but only after verifying they stopped receiving events.

**Apply `verification-before-completion` discipline:**

1. **Wait 2 minutes** after deploy is READY (allow time for new errors to surface)

2. **For each fixed Sentry issue:**
   a. Check: has the issue received new events since the deploy?
      - Use `mcp__claude_ai_Sentry__search_events` to look for events after the deploy timestamp
   b. **If no new events since deploy:**
      - Mark as resolved via `mcp__claude_ai_Sentry__update_issue` with `status: "resolved"`
   c. **If still receiving events:**
      - Skip resolution
      - Report as "fix may be incomplete — still receiving events"

3. **Report resolution status per issue:**
   ```
   | Issue | Status |
   |-------|--------|
   | {ID-1} | ✅ Resolved |
   | {ID-2} | ✅ Resolved |
   | {ID-3} | ⚠️ Skipped — still receiving events |
   ```

Store: `resolved_count`, `skipped_count`, `skipped_issues` for Phase 10.

---

### Phase 10: Summary

**Goal:** Present a comprehensive summary of all work done.

```
## fix-sentry Complete

**Errors found**: {N} in {M} groups
**Scope**: {all / user-selected groups}
**Commit**: `{hash}` — {subject}
**Deployment**: {Ready at URL / Failed}

### Fixes Applied
| Group | Errors Fixed | Root Cause | Files Changed |
|-------|-------------|------------|---------------|
| 1 | 5 | {cause} | {files} |
| 2 | 3 | {cause} | {files} |

### Sentry Resolution
| Issue | Status |
|-------|--------|
| {ID-1} | ✅ Resolved |
| {ID-2} | ✅ Resolved |
| {ID-3} | ⚠️ Skipped — still receiving events |

### Review Findings
{summary of review findings — issues found and fixed, suggestions deferred}

### Test Coverage
{list of new test files and what they verify}
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| Sentry MCP auth fails | Prompt user to authenticate, retry once, abort if still fails |
| No unresolved issues found | Report "No unresolved Sentry errors — nothing to fix" and exit |
| Research agents disagree on root cause | Present both to user, user picks |
| Build fails after fix | Diagnose, fix, retry (max 2 attempts), escalate to user |
| Deploy fails | Read logs, fix, re-commit, re-push (max 2 retries) |
| Sentry issue still receiving events after deploy | Skip resolution, report as "fix may be incomplete" |
| >25 unresolved issues | Cap fetch at 25, warn user, suggest multiple sessions |
| >10 error groups | Suggest user picks top N groups for this session |

## Evaluation Criteria

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| 1 | **Sentry fetch** | Phase 1 fetches all unresolved issues with details (stack trace, event count, user count) | Issues fetched without details, or cap not applied |
| 2 | **Grouping quality** | Phase 2 clusters errors by shared file/function/caller chain, not just error message | Each error treated as separate group, or unrelated errors merged |
| 3 | **User gate (scope)** | Phase 2 stops and waits for user to choose scope before proceeding | Proceeds to Phase 3 without user confirmation |
| 4 | **Parallel diagnosis** | Phase 3 dispatches two agents per group (systematic-debugging + /investigate) | Only one research methodology used |
| 5 | **Disagreement handling** | When agents disagree, both findings presented to user for decision | Disagreement silently resolved or ignored |
| 6 | **User gate (diagnosis)** | Phase 3 stops and waits for user to confirm diagnosis | Proceeds to Phase 4 without user confirmation |
| 7 | **Plan delegation** | Phase 4 delegates to writing-plans with TDD requirement | Plan created inline without TDD structure |
| 8 | **TDD gate** | Phase 5 writes failing test before each fix (RED), then fix (GREEN) | Code changed without test, or test passes before fix |
| 9 | **Review enforcement** | Phase 6 dispatches Layer 1 (code-reviewer + /review) plus 1-3 Layer 2 agents | Review skipped or only 1 agent used |
| 10 | **Compat check** | Phase 6.1 checks backward compatibility, reusability, and pattern consistency | Compat check skipped |
| 11 | **Commit format** | Phase 7 commit lists all Sentry IDs in the message body | Generic commit message without Sentry references |
| 12 | **Deploy monitoring** | Phase 8 polls deployment and handles failures | Deploy status not checked |
| 13 | **Verification gate** | Phase 9 checks for new events before resolving each issue | Issues resolved without event check |
| 14 | **Incomplete fix handling** | Issues still receiving events are skipped, not force-resolved | All issues marked resolved regardless of event status |
| 15 | **Summary completeness** | Phase 10 includes fixes, resolution status, review findings, and test coverage | Summary missing sections or only partial |
| 16 | **Context preservation** | Heavy phases (1, 3, 4, 5, 6, 6.1) run as sub-agents. Main orchestrator stays under 40k tokens. | All phases run inline, burning context |

### Mandatory Checklist (before reporting summary)

- [ ] **Sentry issues fetched**: All unresolved issues retrieved with full details
- [ ] **User approved scope**: User explicitly chose which groups to fix
- [ ] **User approved diagnosis**: User confirmed root causes before planning
- [ ] **TDD tests written**: Failing tests written before fixes for each group
- [ ] **All tests pass**: `npm test` passes after all fixes
- [ ] **Build passes**: `npm run build` succeeds
- [ ] **Lint passes**: `npm run lint` has no errors
- [ ] **Review complete**: At least 3 review agents ran and findings addressed
- [ ] **Deployed**: Deployment succeeded (or failure reported)
- [ ] **Sentry resolved**: Fixed issues marked resolved (or skipped with reason)
