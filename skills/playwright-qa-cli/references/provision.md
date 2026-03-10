# Test User Provisioning

Run SQL via your Supabase SQL execution tool (e.g., `mcp__plugin_supabase_supabase__execute_sql`) with the project ID from CONFIG.md (`SUPABASE_PROJECT_ID`).

## Prerequisites

Read service role key and Supabase URL from `.env.local`:

```bash
grep SUPABASE_SERVICE_ROLE_KEY .env.local | cut -d= -f2-
grep NEXT_PUBLIC_SUPABASE_URL .env.local | cut -d= -f2-
```

Store as `{SERVICE_ROLE_KEY}` and `{SUPABASE_URL}`.

Also read the following from CONFIG.md:
- `{TEST_USER_EMAIL}` — the test user's email address
- `{TEST_USER_PASSWORD}` — the test user's password
- `{TEST_USER_DISPLAY_NAME}` — display name for the test user (e.g., "Playwright QA")

## Step 1: Check if User Exists

```sql
SELECT id FROM auth.users WHERE email = '{TEST_USER_EMAIL}';
```

## If User EXISTS → Reset Data

Customize the reset SQL below to match your application's tables. Delete user-owned data from all tables that reference `user_id`, respecting foreign key order (break FK references first, then delete).

```sql
DO $$
DECLARE
  v_user_id UUID;
BEGIN
  SELECT id INTO v_user_id FROM auth.users WHERE email = '{TEST_USER_EMAIL}';
  IF v_user_id IS NULL THEN RETURN; END IF;

  -- Break FK references first (customize to your schema)
  -- Example: UPDATE public.table_a SET linked_id = NULL WHERE user_id = v_user_id AND linked_id IS NOT NULL;

  -- Delete from child tables first, then parent tables (customize to your schema)
  -- Example: DELETE FROM public.child_table WHERE parent_id IN (SELECT id FROM public.parent_table WHERE user_id = v_user_id);
  -- Example: DELETE FROM public.parent_table WHERE user_id = v_user_id;

  -- Reset usage/credits tracking (customize to your schema)
  -- Example: UPDATE public.usage_tracking SET credits_limit = 1000, credits_used = 0 WHERE user_id = v_user_id;
END $$;
```

## If User Does NOT Exist → Create via Admin API

### A: Create auth user

**Use Admin API — NOT raw SQL.** `crypt()` in raw SQL doesn't hash passwords in the format GoTrue expects, causing "Invalid email or password" errors.

```bash
curl -s -X POST "{SUPABASE_URL}/auth/v1/admin/users" \
  -H "Authorization: Bearer {SERVICE_ROLE_KEY}" \
  -H "apikey: {SERVICE_ROLE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"email":"{TEST_USER_EMAIL}","password":"{TEST_USER_PASSWORD}","email_confirm":true,"user_metadata":{"full_name":"{TEST_USER_DISPLAY_NAME}","email_verified_at":"2026-01-01T00:00:00Z"}}'
```

Capture returned `id` as `{USER_ID}`.

**Password note:** The test password should avoid `!`, `\`, `$` and other shell-special characters. The `playwright-cli fill` command escapes these, and they also cause `bad_json` errors in curl. Use only alphanumeric characters if possible.

### B: Create required profile/onboarding records

DB triggers may auto-create some records (e.g., `profiles`, `subscriptions`, `usage_tracking`) — check your schema. Only INSERT records that aren't auto-created, and UPDATE auto-created ones.

Customize the SQL below to match your application's onboarding requirements:

```sql
-- Wait ~2s after user creation for triggers to fire
-- Example: Create business/organization profile
-- INSERT INTO public.business_profiles (user_id, business_name, business_type)
-- VALUES ('{USER_ID}', 'Test Organization', 'default_type')
-- ON CONFLICT (user_id) DO NOTHING;
```

### C: Update credits/usage (if applicable)

Customize to match your application's usage tracking schema:

```sql
-- Example: Reset usage tracking
-- UPDATE public.usage_tracking SET
--   credits_limit = 1000, credits_used = 0
-- WHERE user_id = '{USER_ID}';
```

### D: Verify email verification metadata

Many apps check `user.user_metadata.email_verified_at` in middleware. If missing, users get stuck on a verification page.

The Admin API call above includes it, but verify:

```sql
SELECT raw_user_meta_data->>'email_verified_at' as verified_at
FROM auth.users WHERE email = '{TEST_USER_EMAIL}';
```

If NULL:
```sql
UPDATE auth.users
SET raw_user_meta_data = raw_user_meta_data || '{"email_verified_at": "2026-01-01T00:00:00Z"}'::jsonb
WHERE email = '{TEST_USER_EMAIL}';
```

## Verify Provisioning

Customize the verification query to check all required records exist for your app:

```sql
SELECT u.id, u.email,
  u.raw_user_meta_data->>'email_verified_at' as verified_at
FROM auth.users u
WHERE u.email = '{TEST_USER_EMAIL}';
```

Expected: email_verified_at set, all required profile/onboarding records exist, usage/credits properly configured.
