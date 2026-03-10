---
name: review-team
description: Agent Teams PR review with Devil's Advocate. Spawns 5 team members — 4 specialist reviewers + 1 adversarial challenger — to produce confidence-rated findings. Only findings that survive cross-examination make the final report.
---

# PR Review Team with Devil's Advocate

Orchestrate a team-based PR review using Agent Teams (TeamCreate + SendMessage). Four specialist reviewers analyze a GitHub PR, then a Devil's Advocate challenges every finding. Original reviewers must defend, adjust, or withdraw. Only surviving findings — with confidence ratings — appear in the final report.

## When to Use This Skill

- **Before merging a PR** - Get high-confidence, adversarially filtered feedback
- **For critical PRs** - When false positives are costly (large features, infra changes)
- **When you want fewer, better findings** - Devil's Advocate filters noise
- **For PRs with ticket context** - Auto-detects ticket for acceptance criteria validation

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| (first arg) | required | PR URL or PR number |
| `ticket` | auto-detect | Ticket key override (e.g., `ticket=PROJ-94`) |
| `skip-challenge` | `false` | Skip Devil's Advocate phase (degrades to standard parallel review) |
| `verbose` | `false` | Include full challenge/defense transcript in report |

## Invocation

```
/review-team https://github.com/owner/repo/pull/123
/review-team 123
/review-team 123 ticket=PROJ-94
/review-team 123 skip-challenge=true
/review-team 123 verbose=true
```

## Architecture

```
ORCHESTRATOR (main session - team lead, manages state + synthesis)
|
+-- Phase 0: INIT
|   Parse PR URL, fetch metadata + diff, auto-detect ticket
|   Read CLAUDE.md for project conventions
|   TeamCreate("pr-review-{number}")
|
+-- Phase 1: TASK CREATION
|   Create 5 tasks in shared task list (TaskCreate)
|   Assign 4 reviewer tasks to specialist teammates
|
+-- Phase 2: PARALLEL REVIEW (4 agents work simultaneously)
|   +-- staff-engineer      (code-refactorer)
|   +-- qa-engineer          (general-purpose)
|   +-- security-reviewer    (general-purpose)
|   +-- architect            (system-architect)
|   Each sends findings JSON to team lead via SendMessage
|
+-- Phase 3: DEVIL'S ADVOCATE CHALLENGE
|   +-- devils-advocate      (general-purpose)
|   Receives ALL aggregated findings from Phase 2
|   Sends targeted challenges to each original reviewer
|
+-- Phase 4: DEFENSE / REBUTTAL (4 agents respond)
|   Each reviewer: DEFEND / ADJUST / WITHDRAW
|   Sends responses to devils-advocate
|
+-- Phase 5: VERDICT
|   devils-advocate assigns confidence ratings
|   Sends final verdict to team lead
|
+-- Phase 6: SYNTHESIS
|   Build unified report, shutdown teammates, TeamDelete
|   Present report to user
|
+-- STATE: .review-team/state.json
```

---

## Team Members

| Name | Sub-Agent Type | Role | Focus |
|------|---------------|------|-------|
| `staff-engineer` | `code-refactorer` | Staff Engineer | Code quality, bugs, project conventions, framework patterns, TypeScript, performance |
| `qa-engineer` | `general-purpose` | QA Engineer | Test coverage gaps, edge cases, error paths, acceptance criteria |
| `security-reviewer` | `general-purpose` | Security Reviewer | Auth bypasses, silent failures, input validation, data exposure, injection |
| `architect` | `system-architect` | Architect | Scalability, pattern violations, DB design, N+1 queries, migrations |
| `devils-advocate` | `code-refactorer` | Devil's Advocate | Challenges ALL findings — questions assumptions, demands evidence, filters false positives |

**Why `code-refactorer` for Devil's Advocate**: Needs strong code reading ability to open every file reviewers reference and verify claims against actual source. Can trace patterns across the codebase ("Is this really a bug?", "Does this pattern exist elsewhere?", "Is the reviewer quoting this line correctly?").

---

## Workflow

### Phase 0: Initialize

1. Parse `$ARGUMENTS` to extract PR identifier and parameters:
   - **PR URL**: Extract from `https://github.com/owner/repo/pull/123` pattern
   - **PR number**: Plain number like `123` — uses current repo
   - **Parameters**: `ticket=PROJ-94`, `skip-challenge=true`, `verbose=true`

2. Fetch PR metadata and diff:

```bash
# Get PR metadata
gh pr view <PR> --json title,body,additions,deletions,changedFiles,baseRefName,headRefName,labels,number

# Get full diff
gh pr diff <PR>

# Get commit history
gh pr view <PR> --json commits
```

If `gh pr view` fails, ask the user to confirm auth (`gh auth status`) and URL.

3. **Read project conventions**: Read the project's `CLAUDE.md` file (and any referenced convention/pattern docs) to extract the project-specific rules that reviewers should check against. Build a convention context block summarizing:
   - API patterns and required utilities
   - State management rules
   - Error handling conventions
   - Code reuse and centralization rules
   - UI component patterns
   - Database and scalability rules
   - Logging conventions
   - Any other project-specific rules

   This convention context block will be included in every reviewer prompt.

4. **Ticket auto-detection**: Scan PR title and branch name for common ticket patterns (`{PROJECT_KEY}-\d+`, e.g., `PROJ-123`, `JIRA-456`). If found (or `ticket` parameter provided), attempt to fetch ticket details via available Jira MCP tools:
   - Path: `/rest/api/3/issue/{key}`
   - Extract: `key`, `fields.summary`, `fields.description`, `fields.customfield_10014` (acceptance criteria), `fields.status.name`, `fields.priority.name`
   - Non-blocking if Jira MCP is unavailable or no ticket found — continue without ticket context.

5. Create `.review-team/` directory if needed.

6. Initialize state file at `.review-team/state.json`:

```json
{
  "prNumber": 123,
  "prTitle": "PR title",
  "prUrl": "https://github.com/owner/repo/pull/123",
  "teamName": "pr-review-123",
  "ticketKey": "PROJ-94",
  "parameters": {
    "skipChallenge": false,
    "verbose": false
  },
  "currentPhase": 0,
  "status": "in_progress",
  "startedAt": "2026-02-21T10:00:00Z",
  "agents": {
    "staff-engineer": { "status": "pending", "findingsCount": 0 },
    "qa-engineer": { "status": "pending", "findingsCount": 0 },
    "security-reviewer": { "status": "pending", "findingsCount": 0 },
    "architect": { "status": "pending", "findingsCount": 0 },
    "devils-advocate": { "status": "pending" }
  },
  "rawFindings": [],
  "challengedFindings": [],
  "finalFindings": [],
  "verdict": null
}
```

7. Check for existing `state.json` — if found, offer to resume:

```
AskUserQuestion:
  question: "Found existing review state for PR #{number}. Phase {N} was last completed. Resume or start fresh?"
  header: "Resume"
  options:
    - label: "Resume from Phase {N+1}"
      description: "Continue where you left off"
    - label: "Start fresh"
      description: "Delete state and start over"
```

8. Create the team:

```
TeamCreate:
  team_name: "pr-review-{number}"
  description: "PR Review Team for #{number} - {title}"
```

### Phase 1: Create Tasks and Spawn Team

Create 5 tasks in the shared task list using TaskCreate:

1. **Staff Engineer Review** — "Review PR #{number} for code quality, bugs, and project convention compliance"
2. **QA Engineer Review** — "Review PR #{number} for test coverage gaps, edge cases, and error path handling"
3. **Security Review** — "Review PR #{number} for auth bypasses, silent failures, input validation, and data exposure"
4. **Architecture Review** — "Review PR #{number} for scalability, pattern violations, DB design, and N+1 queries"
5. **Devil's Advocate Challenge** — "Challenge ALL review findings — question assumptions, demand evidence, filter false positives"

Spawn 4 reviewer teammates using the Task tool with `team_name` parameter. Devil's Advocate is spawned later in Phase 3.

Each reviewer teammate receives:
1. The PR diff
2. The convention context block (extracted from CLAUDE.md in Phase 0)
3. Ticket context (if available)
4. Their specific review focus
5. Instructions to send findings as JSON via SendMessage to team lead

### Phase 2: Parallel Review (4 Agents)

All 4 reviewers work simultaneously. Each reviewer sends findings to the team lead via SendMessage when complete.

#### Convention Context Block

**Built dynamically in Phase 0** by reading the project's CLAUDE.md. The block should be structured as:

```
PROJECT CONVENTIONS TO REVIEW AGAINST:
======================================

{Extracted convention rules from CLAUDE.md, organized by category:
 - API patterns
 - State management
 - Error handling
 - Code reuse / centralization
 - UI component rules
 - Database / scalability
 - Logging
 - Testing patterns
 - Any other project-specific conventions}
```

If CLAUDE.md references additional guide documents (e.g., pattern guides, architecture docs), read those too and include relevant rules.

#### Reviewer Agent Prompts

All reviewers must return findings in this JSON format via SendMessage to the team lead:

```json
{
  "reviewer": "staff-engineer",
  "findings": [
    {
      "id": "SE-1",
      "severity": "critical|important|suggestion",
      "file": "path/to/file.tsx",
      "line": 123,
      "issue": "Brief description of the problem",
      "suggestion": "Specific fix or improvement",
      "evidence": "Code snippet or reasoning that supports this finding"
    }
  ]
}
```

**MAX 10 findings per reviewer. Focus on high-impact issues only.**

Finding IDs use prefix: `SE-` (staff-engineer), `QA-` (qa-engineer), `SR-` (security-reviewer), `AR-` (architect).

##### Staff Engineer

```
Sub-agent: code-refactorer
Name: staff-engineer

You are a Staff Engineer reviewing PR #{number}: "{title}".

## PR Diff
{full diff}

## Changed Files
{file list}

{ticket context if available}

## Project Conventions
{convention context block extracted from CLAUDE.md}

## Your Review Focus
- Bugs, logic errors, and incorrect behavior
- Project convention violations (check every rule in the conventions block above)
- Framework-specific issues (hook misuse, missing deps, memory leaks, unnecessary re-renders)
- TypeScript type safety (any casts, missing types, unsound assertions)
- Performance issues (N+1 queries, missing memoization, blocking operations)
- Code quality (naming, structure, readability)

## Evidence Requirements
Every finding MUST include:
- Exact file path + line number (e.g., `src/api/route.ts:45`)
- Code snippet from the diff showing the violation
- Specific convention rule name being violated (from the project conventions above)
- User impact: what breaks, degrades, or becomes vulnerable

Findings without code evidence will be dismissed by the Devil's Advocate.

## Devil's Advocate Warning
Your findings WILL be challenged by a DA with full codebase access. They will:
- Open every file you reference to verify claims
- Search for existing handling you might have missed
- Dismiss findings without code evidence
To survive: cite exact lines, quote code, name the convention rule.

## Instructions
1. Review the diff thoroughly against the conventions above
2. For each finding, provide concrete evidence (code snippets, line numbers)
3. Send your findings as JSON via SendMessage to the team lead

MAX 10 findings. Focus on high-impact issues only.
```

##### QA Engineer

```
Sub-agent: general-purpose
Name: qa-engineer

You are a QA Engineer reviewing PR #{number}: "{title}".

## PR Diff
{full diff}

## Changed Files
{file list}

{ticket context if available}
{acceptance criteria if available}

## Project Conventions
{convention context block extracted from CLAUDE.md}

## Your Review Focus
- Missing unit/integration tests for new functionality
- Untested edge cases (empty arrays, null values, boundary conditions)
- Error path coverage gaps (network failures, auth failures, validation errors)
- Regression risks from changed behavior
- Test quality (descriptive names, deterministic, independent)
- Acceptance criteria coverage (if ticket context provided)

## Convention Testing Compliance
Check that tests cover project-specific patterns from the conventions block:
- API client success/error path tests (both success and failure branches)
- State management store tests (state changes, reset, selector behavior)
- Server-side auth flow tests (especially auth failure paths)
- Error boundary behavior tests for data-driven components
- Background operation failure paths

## Devil's Advocate Warning
Your findings WILL be challenged by a DA with full codebase access. They will:
- Open every file you reference to verify claims
- Search for existing handling you might have missed
- Dismiss findings without code evidence
To survive: cite exact lines, quote code, name the convention rule.

## Instructions
1. For each changed file with logic, check if corresponding tests exist and are adequate
2. Identify specific test cases that should be written
3. If ticket acceptance criteria are available, map each to implementation evidence
4. Send your findings as JSON via SendMessage to the team lead

MAX 10 findings. Focus on critical coverage gaps.
```

##### Security Reviewer

```
Sub-agent: general-purpose
Name: security-reviewer

You are a Security Reviewer reviewing PR #{number}: "{title}".

## PR Diff
{full diff}

## Changed Files
{file list}

{ticket context if available}

## Project Conventions
{convention context block extracted from CLAUDE.md}

## Your Review Focus
- Auth bypasses (missing auth checks, trusting client-supplied user identity)
- Silent failures (empty catch blocks, swallowed errors, inappropriate fallbacks)
- Input validation gaps (missing schema validation, SQL injection vectors, XSS)
- Data exposure (error internals in UI responses, PII in logs, secrets in code)
- Missing authorization checks on protected routes
- Error suppression without user feedback
- Background operations with no failure feedback

## Common Security Anti-Patterns
Watch for these high-priority violations:
- **Auth from body**: User identity read from request body instead of server-side session — impersonation vulnerability
- **Error message leakage**: Raw error messages (from DB, APIs, etc.) exposed in UI responses, toast messages, or API responses — leaks internals
- **Bare catch blocks**: `catch {}` or `catch (_e) {}` without logging — errors disappear silently
- **Bare fire-and-forget**: Detached async with `.catch(() => {})` — silently swallows errors with no logging
- **Hardcoded secrets/URLs**: API keys, internal URLs, or service domains hardcoded instead of using environment variables

## Devil's Advocate Warning
Your findings WILL be challenged by a DA with full codebase access. They will:
- Open every file you reference to verify claims
- Search for existing handling you might have missed
- Dismiss findings without code evidence
To survive: cite exact lines, quote code, name the convention rule.

## Instructions
1. Focus on security-critical code paths (auth, data access, mutations)
2. Check every catch block for proper error handling
3. Verify auth checks exist on all protected operations
4. Send your findings as JSON via SendMessage to the team lead

MAX 10 findings. Focus on actual security risks, not theoretical concerns.
```

##### Architect

```
Sub-agent: system-architect
Name: architect

You are a System Architect reviewing PR #{number}: "{title}".

## PR Diff
{full diff}

## Changed Files
{file list}

{ticket context if available}

## Project Conventions
{convention context block extracted from CLAUDE.md}

## Your Review Focus
- Scalability issues (client-side aggregation, missing pagination, unbounded queries)
- Pattern violations (not following established service/repository/mapping patterns)
- Database design (missing indexes, security policy gaps, migration quality)
- N+1 queries and unnecessary database round-trips
- Architectural misalignment (wrong abstraction level, wrong file location)
- Over-engineering (premature abstractions, unnecessary indirection)

## Common Architecture Anti-Patterns
Watch for these pattern violations:
- **Raw DB rows in UI**: Database rows passed directly to UI components instead of using established DTO/mapping patterns
- **Wildcard selects**: `.select('*')` or `SELECT *` instead of named column selects
- **Client-side aggregation**: Fetching all rows and computing totals/counts in JavaScript instead of SQL
- **Missing cascade updates**: New tables storing user data not added to cleanup/deletion flows
- **Duplicated logic**: Business logic repeated across files instead of extracted to shared services

## Devil's Advocate Warning
Your findings WILL be challenged by a DA with full codebase access. They will:
- Open every file you reference to verify claims
- Search for existing handling you might have missed
- Dismiss findings without code evidence
To survive: cite exact lines, quote code, name the convention rule.

## Instructions
1. Evaluate architectural impact of the changes
2. Check database queries for scalability (will this work with 10K rows? 100K?)
3. Verify patterns match existing codebase conventions
4. Send your findings as JSON via SendMessage to the team lead

MAX 10 findings. Focus on architectural concerns that affect maintainability or scale.
```

### Phase 3: Devil's Advocate Challenge

**Skip this phase if `skip-challenge=true`.** Instead, assign all raw findings MEDIUM confidence and proceed to Phase 6.

After all 4 reviewers send their findings:

1. Aggregate all findings into `.review-team/findings-raw.json`
2. Update state: `currentPhase: 3`
3. Spawn the Devil's Advocate teammate:

```
Sub-agent: code-refactorer
Name: devils-advocate

You are a Devil's Advocate on a PR review team. Your job is to CHALLENGE every finding from 4 specialist reviewers. You are skeptical of ALL findings equally — your goal is to filter false positives, calibrate severity, and catch issues reviewers missed.

## PR Context
PR #{number}: "{title}"
{brief PR description}

## PR Diff
{full diff}

## Project Conventions
{convention context block extracted from CLAUDE.md}

## All Review Findings
{JSON array of all findings from all 4 reviewers}

## Phase 1: Independent Verification

Before challenging any finding, perform your own automated pattern scan on the diff for known anti-patterns:

| Pattern | Search For | Convention Category |
|---------|-----------|---------------------|
| Unsafe date parsing | `new Date(` on string args | Date Handling |
| Bare catch blocks | `catch {`, `catch (_` | Error Handling |
| Server-side console | `console.log`, `console.error`, `console.warn` in server code | Logging |
| Wildcard selects | `.select('*')`, `SELECT *` | Database Queries |
| Error message leakage | Raw error messages in UI state, toast, or API responses | Error Handling |
| Auth from body | User identity from request body instead of server session | Authentication |
| Bare fire-and-forget | `.catch(() => {})`, `.catch(() => undefined)` | Error Handling |
| Hardcoded secrets | API keys, tokens, passwords in source code | Security |
| Missing input validation | Unvalidated request body used directly | Input Validation |

Record what you find — these calibrate reviewer accuracy and may reveal false negatives.

## Phase 2: Challenge Protocol

For EACH finding, apply these 6 challenge types:

1. **False Positive Check** — "Is this actually a problem?"
   Open the actual file (not just the diff). Check if the reviewer missed context.
   DISMISS if: the issue is already handled elsewhere in the same file/module.

2. **Severity Calibration** — "Is the severity correct?"
   Blast radius (how many users affected?) x likelihood (how often will this trigger?).
   DISMISS if: severity is "critical" but blast radius is < 1% of users AND requires unlikely conditions.

3. **Fix Quality Check** — "Is the suggestion actually better?"
   Does the suggested fix introduce new issues? Is it overcomplicating things?
   DISMISS if: suggestion is worse than current code or adds unnecessary complexity.

4. **Actionability Test** — "Can someone act on this?"
   Vague findings like "could be improved" with no specific change.
   DISMISS if: no concrete code change is specified.

5. **Context Verification** — "Did the reviewer miss existing handling?"
   Search the codebase for error handling, validation, or patterns the reviewer might not have seen.
   DISMISS if: existing code already handles the concern within 20 lines of the flagged location.

6. **Convention Accuracy** — "Is the cited convention rule correct?"
   Verify the reviewer cited the right convention rule and applied it correctly.
   DISMISS if: the convention doesn't say what the reviewer claims it says.

## Phase 3: Send Challenges

For each finding, you MUST:
1. Open the actual file referenced and read the surrounding context
2. Search the codebase for related handling the reviewer may have missed
3. Write a specific challenge — not generic "are you sure?"
4. Include your `verificationResult` — what you actually found when you checked

Group challenges by reviewer and send via SendMessage:
- Send challenges for staff-engineer's findings TO staff-engineer
- Send challenges for qa-engineer's findings TO qa-engineer
- Send challenges for security-reviewer's findings TO security-reviewer
- Send challenges for architect's findings TO architect

## Challenge Message Format (send to each reviewer)

Your message to each reviewer must be a JSON object:

{
  "challenges": [
    {
      "findingId": "SE-1",
      "challengeType": "false_positive|severity_calibration|fix_quality|actionability|context_verification|convention_accuracy",
      "challenge": "Specific question or counterargument",
      "evidence": "What you found in the codebase that supports your challenge",
      "verificationResult": "I opened {file}:{line} and found: {what you saw}",
      "recommendation": "dismiss|reduce_severity|accept_as_is"
    }
  ]
}

After sending all challenges, wait for responses from all 4 reviewers before proceeding to verdict.
```

4. Save challenges to `.review-team/challenge-log.json`

### Phase 4: Defense / Rebuttal (4 Agents Respond)

Each reviewer receives challenges from the Devil's Advocate and must respond. The reviewers were already spawned in Phase 2 — the Devil's Advocate sends challenges directly to them via SendMessage, and they respond back to the Devil's Advocate.

Each reviewer responds with exactly one of three responses per challenged finding:

```json
{
  "responses": [
    {
      "findingId": "SE-1",
      "action": "DEFEND",
      "evidence": "Specific file paths, line numbers, code snippets that prove this is a real issue",
      "updatedFinding": null
    },
    {
      "findingId": "SE-2",
      "action": "ADJUST",
      "evidence": "The DA raised a fair point about severity",
      "updatedFinding": {
        "severity": "suggestion",
        "issue": "Revised issue description",
        "suggestion": "Revised suggestion"
      }
    },
    {
      "findingId": "SE-3",
      "action": "WITHDRAW",
      "reason": "DA correctly identified this is already handled in middleware"
    }
  ]
}
```

- **DEFEND** — provide specific evidence (file paths, line numbers, code snippets)
- **ADJUST** — modify the finding (change severity, refine suggestion)
- **WITHDRAW** — retract the finding with explanation

#### Evidence Tiers (Confidence Caps)

The quality of evidence in a defense determines the maximum confidence a finding can achieve:

| Tier | Evidence Quality | Max Confidence | Example |
|------|-----------------|----------------|---------|
| **TIER 1** (strongest) | Code snippet + exact line number + convention rule name | HIGH (90-100%) | "Line 45 of `route.ts` uses `catch {}` — violates error handling rules" |
| **TIER 2** | File reference + logical argument | MEDIUM (60-89%) | "`route.ts` doesn't validate auth — likely trusts request body" |
| **TIER 3** (weakest) | General reasoning only, no code reference | LOW (30-59%) | "This pattern could lead to security issues" |

DA uses these tiers to cap confidence during verdict. A finding with only TIER 3 evidence CANNOT reach HIGH confidence, even if the reviewer insists it's critical.

### Phase 5: Verdict

The Devil's Advocate receives all responses and assigns confidence ratings:

| Confidence | Range | Criteria |
|-----------|-------|----------|
| HIGH | 90-100% | Defended with code evidence, DA agrees finding is valid |
| MEDIUM | 60-89% | Defended but DA has minor reservations, or finding was adjusted |
| LOW | 30-59% | Weak defense, DA cannot definitively disprove but remains skeptical |
| Dropped | <30% | DA override — finding removed from final report |

The Devil's Advocate sends the final verdict to the team lead via SendMessage:

```json
{
  "verdict": [
    {
      "findingId": "SE-1",
      "confidence": 95,
      "confidenceLevel": "HIGH",
      "evidenceTier": 1,
      "severityVerified": true,
      "status": "defended",
      "summary": "Finding is valid — reviewer provided clear evidence of missing auth check"
    },
    {
      "findingId": "SE-2",
      "confidence": 70,
      "confidenceLevel": "MEDIUM",
      "evidenceTier": 2,
      "severityVerified": true,
      "status": "adjusted",
      "summary": "Severity reduced from critical to important — not a security issue but a quality concern"
    },
    {
      "findingId": "SE-3",
      "confidence": 0,
      "confidenceLevel": "Dropped",
      "evidenceTier": 3,
      "severityVerified": false,
      "status": "withdrawn",
      "summary": "Reviewer withdrew — pattern already handled in middleware"
    }
  ],
  "falseNegatives": [
    {
      "id": "FN-1",
      "severity": "important",
      "file": "path/to/file.tsx",
      "line": 78,
      "issue": "Bare `catch {}` block — no reviewer flagged this",
      "conventionRule": "Error Handling",
      "foundBy": "pattern_scan"
    }
  ],
  "stats": {
    "totalChallenged": 25,
    "defended": 12,
    "adjusted": 5,
    "withdrawn": 8,
    "falsePositiveRate": 32,
    "falseNegativesFound": 1
  }
}
```

**`falseNegatives`**: Issues the DA found during independent pattern scanning (Phase 1 of DA prompt) that no reviewer caught. These are added to the final report as additional findings with MEDIUM confidence.

**`severityVerified`**: DA independently confirms the severity using blast radius (how many users?) x likelihood (how often?). If `false`, the original reviewer's severity stands but with a note.

### Phase 6: Synthesis

1. Build the final report from verdict data
2. Save report to `.review-team/report.md`
3. Shutdown all teammates via SendMessage `type: "shutdown_request"`
4. TeamDelete to clean up the team
5. Update `state.json` with `status: "completed"`
6. Present the report to the user

---

## Output Report

```markdown
# PR Review: #{number} - {title}

## Verdict Summary
| Metric | Value |
|--------|-------|
| Raw findings (pre-challenge) | N |
| Defended | N |
| Adjusted | N |
| Withdrawn | N |
| **Final findings** | **N** |
| False positive rate | N% |

## Critical Issues (must fix before merge)

### 1. [HIGH 95%] {Issue title}
**Source**: {Reviewer name} | **Defended against challenge**
**File**: `path/to/file.tsx:123`
**Issue**: {Description of the problem}
**Suggestion**: {Specific fix}
**Devil's Advocate**: {What was challenged and how it was defended}

### 2. [MEDIUM 75%] {Issue title}
**Source**: {Reviewer name} | **Adjusted after challenge**
**File**: `path/to/file.tsx:45`
**Issue**: {Revised description}
**Suggestion**: {Revised fix}
**Devil's Advocate**: {What was challenged and how it was adjusted}

## Important Issues

{Same format as Critical, for important-severity findings}

## Suggestions

{Same format, for suggestion-severity findings}

## Withdrawn Findings (filtered by Devil's Advocate)

| # | Finding | Reviewer | Withdrawal Reason |
|---|---------|----------|-------------------|
| 1 | {Brief description} | staff-engineer | {Why it was withdrawn} |
| 2 | {Brief description} | architect | {Why it was withdrawn} |

## Convention Violations Summary

| Rule | Violations | Files |
|------|-----------|-------|
| Error Handling | 2 | `route.ts`, `service.ts` |
| Date Handling | 1 | `utils.ts` |
| API Patterns | 0 | - |

## False Negatives (found by Devil's Advocate)

Issues found by DA pattern scanning that no reviewer caught:

### 1. [MEDIUM 65%] {Issue title}
**Found by**: DA pattern scan
**File**: `path/to/file.tsx:78`
**Issue**: {Description}
**Convention Rule**: {Rule name from CLAUDE.md}

## Acceptance Criteria Coverage (if ticket found)

| Criterion | Status | Evidence |
|-----------|--------|----------|
| {AC text} | Covered / Partial / Missing | {What implements it} |
```

If `verbose=true`, append a "Challenge/Defense Transcript" section with the full back-and-forth for each finding.

---

## Communication Flow

```
Phase 2:  staff-engineer ------> team-lead (findings JSON)
          qa-engineer ---------> team-lead
          security-reviewer ---> team-lead
          architect -----------> team-lead

Phase 3:  team-lead -----------> devils-advocate (aggregated findings)
          devils-advocate -----> staff-engineer (challenges to their findings)
          devils-advocate -----> qa-engineer (challenges)
          devils-advocate -----> security-reviewer (challenges)
          devils-advocate -----> architect (challenges)

Phase 4:  staff-engineer ------> devils-advocate (DEFEND/ADJUST/WITHDRAW)
          qa-engineer ---------> devils-advocate (responses)
          security-reviewer ---> devils-advocate (responses)
          architect -----------> devils-advocate (responses)

Phase 5:  devils-advocate -----> team-lead (verdict + confidence ratings)

Phase 6:  team-lead -----------> all (shutdown_request)
```

---

## State Files

| File | Purpose |
|------|---------|
| `.review-team/state.json` | Phase tracking, agent status, parameters |
| `.review-team/findings-raw.json` | All findings before challenge |
| `.review-team/challenge-log.json` | Challenges + responses |
| `.review-team/report.md` | Final output report |

---

## Error Handling

| Scenario | Action |
|----------|--------|
| `gh pr view` fails | Ask user to confirm auth (`gh auth status`) and verify URL |
| Reviewer agent fails | Continue with remaining agents, note gap in report |
| DA fails before sending challenges | Fall back to standard mode — present raw findings with MEDIUM confidence |
| DA fails mid-challenge | Use challenges sent so far, mark remaining as "unchallenged" MEDIUM confidence |
| Reviewer doesn't respond to challenge | Finding marked "undefended" LOW confidence |
| Jira MCP fails | Continue without ticket context (non-blocking) |
| CLAUDE.md not found | Warn user, proceed with generic best-practice review (no project-specific conventions) |
| Early exit / error | Shutdown all teammates, TeamDelete, update state with `"status": "error"` |

---

## Tips

1. **Start with a real PR** - The skill needs a PR URL or number to fetch diff and metadata
2. **Use `skip-challenge=true` for quick runs** - Gets you parallel review without the adversarial filter
3. **Check `.review-team/report.md`** - The full report is saved for reference
4. **High false positive rate is good** - Means the DA is filtering noise effectively
5. **Use `verbose=true` for learning** - See the full challenge/defense debate to understand confidence ratings
6. **CLAUDE.md drives the review** - The richer your project conventions, the more targeted the review
