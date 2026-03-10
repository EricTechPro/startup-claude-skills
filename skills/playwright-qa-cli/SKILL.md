---
name: playwright-qa-cli
description: "Automated QA testing using playwright-cli (CLI tool, NOT MCP tools). Provisions a test user, opens a headless browser, logs in, navigates to target pages, takes screenshots, and generates a REPORT.md. Two modes: VERIFY (reproduce a bug before fixing) and CHECK (confirm a fix works). Use this skill whenever the user says 'QA this', 'test this fix', 'verify this bug', 'smoke test', 'check if this works', 'reproduce this issue', 'run playwright QA', or 'playwright-qa-cli'. Also triggered from /fix-ticket at pre-fix and post-fix stages. Works for any page or user flow — not limited to specific features."
allowed-tools: Bash(playwright-cli:*), Bash(npx:playwright-cli:*)
---

# Playwright QA (CLI Variant)

Automated headless-browser QA using `playwright-cli` bash commands. Provisions a test user, logs in, navigates to target pages, screenshots each step, generates REPORT.md, cleans up.

## CRITICAL: No MCP Tools

Do NOT use any `mcp__plugin_playwright_playwright__*` tools. Use ONLY `playwright-cli` bash commands via the Bash tool. If `playwright-cli` is not found, STOP and ask user to install — do NOT fall back to MCP tools.

## PATH Setup — REQUIRED

**Prefix EVERY `playwright-cli` bash command** with:
```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli ...
```

## Parameters

| Parameter | Default | Example | Description |
|-----------|---------|---------|-------------|
| `base` | `prod` | `local` | `prod` = CONFIG.md `PROD_URL`, `local` = `http://localhost:3000` |
| `mode` | `check` | `verify` | `verify` = reproduce bug, `check` = confirm fix |
| `url` | auto-detect | `/dashboard/receipts` | Page to test |
| `change_description` | from git diff | `"Fixed button overflow"` | What to verify |
| `ticket_id` | none | `PROJ-123` | Issue tracker ticket (folder naming) |
| `skip_cleanup` | `false` | `true` | Keep test user alive |

## Configuration

All project-specific settings are defined in `CONFIG.md` at the root of this skill directory. Before first use, copy `CONFIG.example.md` to `CONFIG.md` and fill in your values.

| Setting | Source |
|---------|--------|
| **Test Email** | `CONFIG.md` → `TEST_USER_EMAIL` |
| **Test Password** | `CONFIG.md` → `TEST_USER_PASSWORD` |
| **Supabase Project ID** | `CONFIG.md` → `SUPABASE_PROJECT_ID` |
| **Production URL** | `CONFIG.md` → `PROD_URL` |
| **Viewport** | `1440x900` |

**Password note:** Avoid `!`, `\`, `$` in test passwords — `playwright-cli fill` escapes shell-special characters.

## Execution Flow

```
Phase 1:   Setup (mkdir screenshots, detect route from git diff)
Phase 1.5: Dev server (local only — check port 3000, start if needed)
Phase 2:   Provision test user → read references/provision.md
Phase 3:   Login via CLI → use state-save for auth persistence
Phase 4:   Navigate & test target page
Phase 5:   Cleanup → read references/cleanup.md + playwright-cli close
Phase 6:   Generate REPORT.md → read references/report-templates.md
```

## Phase 1: Setup

```bash
mkdir -p playwright-qa-screenshots/{test-name}-{mode}
```

`{test-name}` = kebab-case from ticket ID or change description. `{mode}` = `-verify` or `-check`.

**Route detection** (if no `url` param): run `git diff --name-only HEAD`, map files to routes.

Customize the route detection table in CONFIG.md. Example format:

| File Pattern | Route |
|-------------|-------|
| `app/dashboard/page-a/**`, `components/page-a/**` | `/dashboard/page-a` |
| `app/dashboard/page-b/**`, `components/page-b/**` | `/dashboard/page-b` |
| `app/dashboard/settings/**` | `/dashboard/settings` |
| Default | `/dashboard` |

## Phase 1.5: Dev Server (Local Only)

Skip if `base=prod`. Check `lsof -ti:3000`. If no server, start `npm run dev` in background, poll until ready (max 30s). Set `WE_STARTED_SERVER` flag for cleanup.

## Phase 2: Provision Test User

**Read `references/provision.md` and follow it exactly.** It contains the complete Admin API flow, reset SQL, business profile creation, credits setup, and email verification.

## Phase 3: Login via Playwright CLI

### Open browser and navigate

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli open "{base_url}/sign-in"
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli resize 1440 900
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli screenshot --filename=playwright-qa-screenshots/{test-name}/01-sign-in-page.png
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli snapshot
```

### Find element refs and fill login form

Read the snapshot output to find element refs (e.g., `e93` for Email, `e99` for Password, `e104` for Sign In). Refs change between pages — always snapshot before interacting.

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli fill {email-ref} "{TEST_USER_EMAIL}"
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli fill {password-ref} "{TEST_USER_PASSWORD}"
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli click {submit-ref}
```

### Wait for navigation and verify

After clicking login, wait for the page to settle before taking a snapshot. Use `eval` to check readiness instead of a blind sleep:

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli eval "() => new Promise(r => setTimeout(r, 3000))"
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli snapshot
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli screenshot --filename=playwright-qa-screenshots/{test-name}/02-after-login.png
```

**Why 3s here:** Auth redirects involve a server round-trip + client-side redirect chain. This is one of the few places a timed wait is appropriate because there's no single DOM element that reliably signals "auth complete + redirect finished."

### Verify redirect target

Customize these redirect targets in CONFIG.md to match your app's auth flow:

| URL Contains | Meaning | Action |
|-------------|---------|--------|
| `/dashboard` | Login succeeded | Continue to Phase 4 |
| `/verify-email` | Missing email verification | Fix via provision.md, retry |
| `/setup` or `/onboarding` | Missing profile/onboarding | Re-run provision setup, retry |
| `/sign-in` | Login failed | Recreate user via Admin API |

### Save auth state (optional but recommended)

If you'll navigate multiple pages, save auth state to skip re-login on session loss:

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli state-save playwright-qa-screenshots/{test-name}/auth-state.json
```

Restore later with `playwright-cli state-load` if the session drops.

## Phase 4: Navigate & Test

**Read `references/cli-patterns.md` for detailed interaction patterns** (waiting, forms, dialogs, file uploads, scrolling, error checking). The patterns below are a summary.

### Navigate to target page

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli goto "{base_url}{target_route}"
```

### Wait for page to load

Use `eval` to wait for a meaningful signal rather than a blind sleep:

```bash
# Wait for network idle (no pending fetches for 500ms)
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli eval "() => new Promise(resolve => { let timer; const check = () => { timer = setTimeout(resolve, 500); }; const observer = new PerformanceObserver(() => { clearTimeout(timer); check(); }); observer.observe({entryTypes: ['resource']}); check(); })"
```

If that's too complex for the situation, a simple 2-3s wait is acceptable — just document why:

```bash
# Simple wait — acceptable when page has heavy async rendering
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli eval "() => new Promise(r => setTimeout(r, 2000))"
```

### Snapshot + screenshot

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli snapshot
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli screenshot --filename=playwright-qa-screenshots/{test-name}/03-target-page.png
```

### Mode-specific testing

**verify mode**: Reproduce bug steps from ticket using CLI commands. Screenshot each step. Check `playwright-cli console error` for JS errors. Verdict: "Bug confirmed" or "Not reproducible".

**check mode**: Inspect snapshot for error indicators. Verify expected UI elements are present. Run change-specific checks. Check console for errors.

### Expected elements by route

Customize this table in CONFIG.md to match your application's routes and expected elements:

| Route | Expected Elements |
|-------|-------------------|
| `/dashboard` | Main dashboard content, navigation |
| `/dashboard/page-a` | Page A content or empty state |
| `/dashboard/page-b` | Page B content or empty state |
| `/dashboard/settings` | Settings form |

### Key interaction commands

| Action | Command |
|--------|---------|
| Click element | `playwright-cli click {ref}` |
| Fill input | `playwright-cli fill {ref} "text"` |
| Type text (keystroke) | `playwright-cli type "text"` |
| Select dropdown | `playwright-cli select {ref} "value"` |
| Upload file | `playwright-cli upload "/absolute/path/to/file"` |
| Check checkbox | `playwright-cli check {ref}` |
| Press key | `playwright-cli press Enter` / `press Tab` / `press Escape` |
| Scroll down | `playwright-cli mousewheel 0 500` |
| Full-page screenshot | `playwright-cli screenshot --full-page --filename=...` |
| Check JS errors | `playwright-cli console error` |
| Check network | `playwright-cli network` |
| Accept dialog | `playwright-cli dialog-accept` |
| Dismiss dialog | `playwright-cli dialog-dismiss` |
| Evaluate JS | `playwright-cli eval "() => document.title"` |

**Golden rule:** Always `snapshot` before interacting with a new page. Element refs are page-specific and change after navigation or significant DOM updates.

## Phase 5: Cleanup

**Read `references/cleanup.md` and execute the SQL** via your Supabase SQL execution tool. Then:

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli close
```

If `WE_STARTED_SERVER`, kill port 3000: `lsof -ti:3000 | xargs kill 2>/dev/null`

If skip_cleanup=true, skip the SQL cleanup but still close the browser.

## Phase 6: Report

**Read `references/report-templates.md`** and generate `REPORT.md` in the screenshots folder using the Write tool.

## Error Handling

| Scenario | Action |
|----------|--------|
| `playwright-cli` not found | STOP. Tell user: `npm install -g playwright-cli`. Do NOT use MCP. |
| Stale browser session | `playwright-cli kill-all` then retry `open` |
| Login redirect to verify-email | Fix email verification (see provision.md) |
| Login redirect to setup/onboarding | Re-run profile creation (provision.md) |
| Login redirect back to sign-in | Recreate user via Admin API |
| Element ref not found | Re-run `snapshot` — refs may have changed after DOM update |
| Supabase MCP unavailable | Skip provisioning, ask user to log in manually |
| Cleanup fails | Log warning, continue |
| Page shows loading spinner | Wait with `eval`, then re-snapshot |
