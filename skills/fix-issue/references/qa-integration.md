# QA Integration for Fix-Ticket

Headless browser verification using `playwright-cli` (CLI tool, NOT MCP tools). Runs at two points in the pipeline: pre-fix (verify bug exists) and post-fix (confirm fix works).

**IMPORTANT: Always use `playwright-cli` commands via Bash — NEVER use Playwright MCP tools (`mcp__plugin_playwright_playwright__*`).** The CLI provides consistent behavior, screenshot capture to disk, and works for both local and production testing.

## Prerequisites

### PATH Setup — REQUIRED for every command

If `playwright-cli` is installed globally via npm:

```bash
export PATH="$HOME/.npm-global/bin:$PATH" && playwright-cli ...
```

Adjust the PATH if your global npm bin directory differs.

### Dev Server

Check if the dev server is running (port from CONFIG.md):

```bash
lsof -ti:{PORT} 2>/dev/null && echo "RUNNING" || echo "NOT_RUNNING"
```

If NOT_RUNNING, start it and wait:
```bash
{DEV_COMMAND} > /tmp/dev-server.log 2>&1 &
DEV_PID=$!
# Wait for server to be ready
for i in $(seq 1 30); do lsof -ti:{PORT} >/dev/null 2>&1 && echo "READY" && break; sleep 2; done
```

**Worktree note:** If in a git worktree, you MUST symlink `.env.local` and `node_modules` from the main repo root BEFORE starting the dev server.

Set `WE_STARTED_SERVER=true` if you started it (for cleanup later).

### Test User Provisioning

**Read CONFIG.md** for the test user email, password, and auth provider.

#### Supabase Auth Provider

**Run provisioning SQL** via Supabase MCP (`mcp__plugin_supabase_supabase__execute_sql` or `mcp__supabase__execute_sql`):

```sql
DO $$
DECLARE
  v_user_id UUID;
BEGIN
  SELECT id INTO v_user_id FROM auth.users WHERE email = '{TEST_USER_EMAIL}';
  IF v_user_id IS NULL THEN
    RAISE NOTICE 'USER_NOT_FOUND';
    RETURN;
  END IF;

  -- Reset test data: customize these DELETE/UPDATE statements for your schema
  -- DELETE FROM public.your_table WHERE user_id = v_user_id;

  RAISE NOTICE 'USER_RESET_COMPLETE:%', v_user_id;
END $$;
```

**If result contains `USER_NOT_FOUND`** -> create user via Admin API:

```bash
SUPABASE_URL=$(grep NEXT_PUBLIC_SUPABASE_URL .env.local | cut -d= -f2-)
SERVICE_ROLE_KEY=$(grep SUPABASE_SERVICE_ROLE_KEY .env.local | cut -d= -f2-)

curl -s -X POST "${SUPABASE_URL}/auth/v1/admin/users" \
  -H "Authorization: Bearer ${SERVICE_ROLE_KEY}" \
  -H "apikey: ${SERVICE_ROLE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"email":"{TEST_USER_EMAIL}","password":"{TEST_USER_PASSWORD}","email_confirm":true,"user_metadata":{"full_name":"QA Test User","email_verified_at":"2025-01-01T00:00:00Z"}}'
```

**IMPORTANT: Use Supabase Admin API** (`POST /auth/v1/admin/users`) — NOT raw SQL `crypt()`.

#### Other Auth Providers

For Firebase, Auth0, or custom auth, adapt the user provisioning to your provider's admin API. The key requirement: create a test user that can log in programmatically without email verification.

## Login Flow — API Auth (2 tool calls)

**CRITICAL: Use API auth, NOT form fill.** Form-fill login is unreliable and wastes tool calls on retries.

### Step 1: Get auth tokens

#### Supabase

```bash
SUPABASE_URL=$(grep NEXT_PUBLIC_SUPABASE_URL .env.local | cut -d= -f2-)
SERVICE_ROLE_KEY=$(grep SUPABASE_SERVICE_ROLE_KEY .env.local | cut -d= -f2-)
AUTH_RESPONSE=$(curl -s -X POST "${SUPABASE_URL}/auth/v1/token?grant_type=password" \
  -H "apikey: ${SERVICE_ROLE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"email":"{TEST_USER_EMAIL}","password":"{TEST_USER_PASSWORD}"}')
echo "$AUTH_RESPONSE" | python3 -c "import sys,json; d=json.load(sys.stdin); print('ACCESS_TOKEN=' + d['access_token']); print('REFRESH_TOKEN=' + d['refresh_token'])"
```

Capture `ACCESS_TOKEN` and `REFRESH_TOKEN` from the output.

#### Other Providers

Adapt to your auth provider's token endpoint. The goal: obtain an access token and refresh token via API call.

### Step 2: Open browser and inject session

```bash
playwright-cli open "http://localhost:{PORT}/sign-in"
playwright-cli resize 1440 900
```

Then inject the session via eval:

```bash
playwright-cli eval "async () => {
  // Adapt this to your auth framework
  // For Supabase SSR:
  const { createBrowserClient } = await import('@supabase/ssr');
  const supabase = createBrowserClient('{SUPABASE_URL}', '{ANON_KEY}');
  const { error } = await supabase.auth.setSession({
    access_token: '${ACCESS_TOKEN}',
    refresh_token: '${REFRESH_TOKEN}'
  });
  if (error) throw error;
  return 'session set';
}"
```

**If the eval approach fails**, use the cookie approach instead:

```bash
# For Supabase SSR (chunked cookies)
playwright-cli eval "() => {
  const name = 'sb-{SUPABASE_REF}-auth-token';
  const encoded = btoa(JSON.stringify({
    access_token: '${ACCESS_TOKEN}',
    refresh_token: '${REFRESH_TOKEN}'
  }));
  document.cookie = name + '.0=' + encodeURIComponent(encoded) + '; path=/; SameSite=Lax';
  // Clear stale chunks
  for (let i = 1; i <= 5; i++) {
    document.cookie = name + '.' + i + '=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT';
  }
  return 'cookies set';
}"
```

### Step 3: Navigate to target page

```bash
playwright-cli goto "http://localhost:{PORT}/dashboard"
playwright-cli eval "() => new Promise(r => setTimeout(r, 2000))"
playwright-cli snapshot
```

Verify: URL should contain the expected route. If it redirected to sign-in, fall back to form login.

### Fallback: Form Login (max 2 attempts)

**Only use if API auth fails.** Do NOT try form login first.

```bash
playwright-cli goto "http://localhost:{PORT}/sign-in"
playwright-cli snapshot
```

Read snapshot for element refs, then:

```bash
playwright-cli fill {email-ref} "{TEST_USER_EMAIL}"
playwright-cli fill {password-ref} "{TEST_USER_PASSWORD}"
playwright-cli click {submit-ref}
playwright-cli eval "() => new Promise(r => setTimeout(r, 3000))"
playwright-cli snapshot
```

**Max 2 attempts.** If still on sign-in after 2 tries, recreate user via Admin API and retry API auth.

## Route Detection

Map changed files to routes (customize in CONFIG.md):

| File Pattern | Route |
|-------------|-------|
| `dashboard/**` | `/dashboard` |
| `settings/**` | `/settings` |
| Default | `/` |

## Verify Mode (Pre-Fix)

Goal: Reproduce the bug described in the Jira ticket using the browser.

1. Navigate to the affected page
2. Follow the reproduction steps from the ticket description
3. Screenshot each step: `playwright-cli screenshot --filename=playwright-qa-screenshots/{ticket-id}-verify/NN-description.png`
4. Check for JS errors: `playwright-cli console error`
5. Look for the specific bug symptom described in the ticket

**Verdict outcomes:**
- **Bug confirmed**: Proceed with the fix
- **Not reproducible**: Report to user — ask whether to continue
- **Different bug found**: Report the actual behavior

## Check Mode (Post-Fix)

Goal: Confirm the fix resolves the issue without regressions.

1. Navigate to the affected page
2. Follow the same reproduction steps — the bug should no longer occur
3. Screenshot each step: `playwright-qa-screenshots/{ticket-id}-check/NN-description.png`
4. Check for JS errors and regressions
5. Verify expected behavior matches the ticket's acceptance criteria

**Verdict outcomes:**
- **Fix verified**: Proceed to commit
- **Fix incomplete**: Report what still fails -> loop back to Phase 4 (max 1 retry)
- **Regression found**: Report the regression -> loop back to Phase 4

## Cleanup

After the final QA phase (check mode), clean up:

1. Close the browser:
   ```bash
   playwright-cli close
   ```

2. Clean up test data (run cleanup SQL appropriate for your schema via Supabase MCP or equivalent).

3. If `WE_STARTED_SERVER`, kill the dev server:
   ```bash
   lsof -ti:{PORT} | xargs kill 2>/dev/null
   ```

## Error Handling

| Scenario | Action |
|----------|--------|
| `playwright-cli` not found | Warn user, skip QA phases, continue pipeline |
| Dev server won't start | Warn user, skip QA phases, continue pipeline |
| API auth fails | Fall back to form login (max 2 attempts) |
| Form login fails | Recreate user via Admin API, retry API auth once |
| Browser crashes | `playwright-cli kill-all`, retry once |
| Auth provider MCP unavailable | Skip provisioning, warn user, skip QA phases |
| Bug not reproducible in verify | Report to user, ask whether to continue fixing |
| Fix not verified in check | Loop back to Phase 4 (max 1 retry), then ask user |
