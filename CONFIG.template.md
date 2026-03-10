# Project Configuration

Copy this file to `CONFIG.md` and fill in your project-specific values.
**Do NOT commit CONFIG.md** — it may contain sensitive information.

---

## Jira

| Setting | Value | Notes |
|---------|-------|-------|
| Project key | `PROJ` | e.g., `KAN`, `ENG`, `PROD` — the prefix before ticket numbers |
| Acceptance criteria field | `customfield_10014` | Custom field ID for acceptance criteria (check your Jira admin) |

## Team

### QA Assignees

| Name | Account ID | Alias |
|------|-----------|-------|
| Jane Doe | `abc123def456` | `jane` |

To find Jira account IDs: Jira → User Profile → the ID in the URL, or use `mcp__jira__jira_get` with `/rest/api/3/user/search?query=<name>`.

### Jira Transitions

Map your Jira workflow columns to transition IDs. Find yours via:
```
mcp__jira__jira_get:
  path: /rest/api/3/issue/{ticketId}/transitions
```

| Column | Transition ID | Used When |
|--------|--------------|-----------|
| QA | `41` | Default: fix complete, ready for testing |
| In Review | `31` | When fix needs code review before QA |
| Done | `51` | When fix is trivial and verified |

## Deployment

| Setting | Value | Notes |
|---------|-------|-------|
| Platform | `vercel` | Currently supported: `vercel`. Set to `none` to skip deployment monitoring. |
| Production URL | `https://yourapp.com` | Used for "test in prod" QA mode |

### Vercel (if applicable)

Read from `.vercel/project.json` automatically. No config needed here unless the file doesn't exist.

## QA Testing (Optional)

These settings are only needed if you want headless browser QA verification.

| Setting | Value | Notes |
|---------|-------|-------|
| QA tool | `playwright-cli` | CLI tool for headless browser testing |
| Dev server command | `npm run dev` | Command to start local dev server |
| Dev server port | `3000` | Port the dev server runs on |
| Test user email | `qa-test@yourapp.test` | Dedicated QA test user |
| Test user password | `YourSecurePassword` | Test user password |

### Auth Provider

| Setting | Value | Notes |
|---------|-------|-------|
| Provider | `supabase` | Auth provider: `supabase`, `firebase`, `auth0`, `custom` |
| Supabase project ID | `your-project-id` | Only if using Supabase |

### Test Data Schema

If your QA tests need to seed test data, document your table schemas here:

```sql
-- Example: seed a test record
INSERT INTO public.your_table (id, user_id, created_at, name)
VALUES (gen_random_uuid(), '{USER_ID}', now(), 'Test Record');
```

### Route Detection

Map file paths to routes for QA navigation:

| File Pattern | Route |
|-------------|-------|
| `dashboard/**` | `/dashboard` |
| `settings/**` | `/settings` |
| Default | `/` |

## Complex Fix Delegation (Optional)

| Setting | Value | Notes |
|---------|-------|-------|
| Delegation skill | `/develop-team` | Skill to invoke for complex (3+ file) fixes. Set to `none` to handle all fixes inline. |
