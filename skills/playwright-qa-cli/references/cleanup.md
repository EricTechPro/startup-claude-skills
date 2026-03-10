# Test User Cleanup

Run via your Supabase SQL execution tool (e.g., `mcp__plugin_supabase_supabase__execute_sql`) with the project ID from CONFIG.md (`SUPABASE_PROJECT_ID`).

Customize the cleanup SQL below to match your application's schema. Delete from child tables first (respecting foreign key constraints), then parent tables, then auth tables.

```sql
DO $$
DECLARE
  v_user_id UUID;
BEGIN
  SELECT id INTO v_user_id FROM auth.users WHERE email = '{TEST_USER_EMAIL}';
  IF v_user_id IS NULL THEN RETURN; END IF;

  -- Break FK references first (customize to your schema)
  -- Example:
  -- UPDATE public.table_a SET linked_id = NULL
  --   WHERE user_id = v_user_id AND linked_id IS NOT NULL;

  -- Delete from child tables (customize to your schema)
  -- Example:
  -- DELETE FROM public.child_table WHERE parent_id IN
  --   (SELECT id FROM public.parent_table WHERE user_id = v_user_id);

  -- Delete from parent/main data tables (customize to your schema)
  -- Example:
  -- DELETE FROM public.parent_table WHERE user_id = v_user_id;

  -- Delete profile/settings records (customize to your schema)
  -- Example:
  -- DELETE FROM public.business_profiles WHERE user_id = v_user_id;
  -- DELETE FROM public.usage_tracking WHERE user_id = v_user_id;
  -- DELETE FROM public.subscriptions WHERE user_id = v_user_id;
  -- DELETE FROM public.profiles WHERE id = v_user_id;

  -- Delete auth records (these are standard Supabase tables)
  DELETE FROM auth.identities WHERE user_id = v_user_id;
  DELETE FROM auth.users WHERE id = v_user_id;
END $$;
```

Replace `{TEST_USER_EMAIL}` with the value from CONFIG.md.
