# QA Report Templates

Write to `playwright-qa-screenshots/{test-name}-{mode}/REPORT.md`.

## check mode

```markdown
# QA Report: {test-name}-check

**Date**: {YYYY-MM-DD}
**Mode**: check
**Page**: {target_route}
**Base URL**: {base_url}
**Change**: {change_description}

## Result: {PASS | FAIL}

## Checks

| # | Check | Result | Notes |
|---|-------|--------|-------|
| 1 | Test user provisioned | {pass/fail} | {credits, tier} |
| 2 | Login succeeded | {pass/fail} | {redirect target} |
| 3 | Page loaded without errors | {pass/fail} | {any error indicators} |
| 4 | Expected UI elements present | {pass/fail} | {what was found/missing} |
| 5 | Change-specific verification | {pass/fail/skipped} | {details} |

## Screenshots

| Step | File | Description |
|------|------|-------------|
| 1 | [01-sign-in-page.png](01-sign-in-page.png) | Sign-in page |
| 2 | [02-after-login.png](02-after-login.png) | Dashboard after login |
| 3 | [03-target-page.png](03-target-page.png) | {target page description} |

## Issues Found

{List any issues, or "None — all checks passed."}
```

## verify mode

```markdown
# Bug Verification: {test-name}-verify

**Date**: {YYYY-MM-DD}
**Mode**: verify
**Ticket**: {ticket_id}
**Page**: {target_route}
**Base URL**: {base_url}
**Bug Description**: {from issue tracker or user input}

## Result: {BUG CONFIRMED | NOT REPRODUCIBLE}

## Reproduction Steps

| # | Step | Result | Notes |
|---|------|--------|-------|
| 1 | Navigate to {page} | {done/failed} | {notes} |
| 2 | {action from ticket} | {done/failed} | {notes} |
| 3 | Check for {expected bug behavior} | {bug seen/not seen} | {notes} |

## Screenshots

| Step | File | Description |
|------|------|-------------|
| 1 | [01-sign-in-page.png](01-sign-in-page.png) | Sign-in page |
| 2 | [02-after-login.png](02-after-login.png) | Dashboard after login |
| 3 | [03-target-page.png](03-target-page.png) | {target page} |

## Verdict

{Bug confirmed — matches ticket description. Proceeding with fix.}
{OR: Not reproducible — tried X, Y, Z. Suggest adding comment to ticket.}
```
