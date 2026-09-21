# Supabase — Patterns & Reference

| Field          | Value                                                   |
|----------------|---------------------------------------------------------|
| Version        | 1.1                                                     |
| Last Updated   | 2026-09-21                                              |
| Applicability  | Supabase (hosted or self-hosted), PostgreSQL 15+        |
| Dependencies   | supabase-js v2+, Supabase CLI, @supabase/ssr (for SSR frameworks) |

---

## RLS Policy Patterns

### Fundamentals

RLS is enforced on every query made through the Supabase client (which uses the `anon` or `authenticated` role). The `service_role` key **bypasses RLS entirely**.

```sql
-- Enable RLS (mandatory — tables have NO protection without this)
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Force RLS even for table owners (recommended in migrations)
ALTER TABLE documents FORCE ROW LEVEL SECURITY;
```

### Common Policy Patterns

```sql
-- 1. Users can only read their own rows
CREATE POLICY "users_read_own" ON documents
    FOR SELECT
    USING (user_id = auth.uid());

-- 2. Users can insert rows attributed to themselves
CREATE POLICY "users_insert_own" ON documents
    FOR INSERT
    WITH CHECK (user_id = auth.uid());

-- 3. Users can update their own rows
CREATE POLICY "users_update_own" ON documents
    FOR UPDATE
    USING (user_id = auth.uid())       -- which rows can they see to update
    WITH CHECK (user_id = auth.uid()); -- what the row must look like after update

-- 4. Users can delete their own rows
CREATE POLICY "users_delete_own" ON documents
    FOR DELETE
    USING (user_id = auth.uid());

-- 5. Organization-based access (user belongs to org)
CREATE POLICY "org_members_read" ON projects
    FOR SELECT
    USING (
        org_id IN (
            SELECT org_id FROM org_members
            WHERE user_id = auth.uid()
        )
    );

-- 6. Role-based access via JWT claims
CREATE POLICY "admins_full_access" ON documents
    FOR ALL
    USING (
        (auth.jwt() ->> 'role') = 'admin'
    );

-- 7. Public read, authenticated write
CREATE POLICY "public_read" ON posts
    FOR SELECT
    USING (published = true);

CREATE POLICY "auth_write" ON posts
    FOR INSERT
    WITH CHECK (auth.uid() IS NOT NULL);
```

### Performance Considerations for RLS

```sql
-- BAD: Subquery in policy runs per-row
CREATE POLICY "slow" ON items
    FOR SELECT
    USING (
        EXISTS (
            SELECT 1 FROM permissions
            WHERE permissions.user_id = auth.uid()
              AND permissions.item_id = items.id
        )
    );
-- FIX: Add index on permissions(user_id, item_id)

-- BETTER for many-to-many: use a security definer function
CREATE OR REPLACE FUNCTION get_user_accessible_item_ids()
RETURNS SETOF UUID
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
    SELECT item_id FROM permissions WHERE user_id = auth.uid();
$$;

CREATE POLICY "fast" ON items
    FOR SELECT
    USING (id IN (SELECT get_user_accessible_item_ids()));
```

**Rules:**
- Always enable RLS on every table that holds user data.
- Test policies by switching to the `anon`/`authenticated` role in SQL editor.
- Use `SECURITY DEFINER` functions for complex permission logic (they bypass RLS inside the function body — be careful).
- Set `search_path` explicitly in `SECURITY DEFINER` functions to prevent injection.

---

## Auth Integration

### Session Management (Client-Side)

```typescript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY)

// Sign up
const { data, error } = await supabase.auth.signUp({
  email: 'user@example.com',
  password: 'secure-password',
})

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'secure-password',
})

// OAuth
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: { redirectTo: 'http://localhost:3000/auth/callback' },
})

// Listen for auth state changes
supabase.auth.onAuthStateChange((event, session) => {
  // event: 'SIGNED_IN' | 'SIGNED_OUT' | 'TOKEN_REFRESHED' | ...
  // session contains access_token, refresh_token, user
})

// Sign out
await supabase.auth.signOut()
```

### SSR / Server-Side (Next.js, SvelteKit)

```typescript
// Use @supabase/ssr for server-side auth
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'

export async function createSupabaseServer() {
  const cookieStore = await cookies()
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll()
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          )
        },
      },
    }
  )
}
```

### JWT Verification (Custom Backend)

```python
# When calling your own API with the Supabase access token
import jwt  # PyJWT

def verify_supabase_jwt(token: str) -> dict:
    """Verify a Supabase-issued JWT."""
    payload = jwt.decode(
        token,
        key=SUPABASE_JWT_SECRET,       # From project settings
        algorithms=["HS256"],
        audience="authenticated",       # Supabase sets this
    )
    # payload contains: sub (user_id), email, role, exp, etc.
    return payload
```

**Rules:**
- Never store the access token in `localStorage` on security-sensitive apps (use `httpOnly` cookies via `@supabase/ssr`).
- Always verify the `aud` claim when validating JWTs in a custom backend.
- Refresh tokens are handled automatically by the client library — do not manage them manually.
- Use `supabase.auth.getSession()` on the client; use `supabase.auth.getUser()` on the server (it re-validates with the auth server).

---

## Real-Time Subscriptions

```typescript
// Listen to all inserts on a table
const channel = supabase
  .channel('messages-channel')
  .on(
    'postgres_changes',
    { event: 'INSERT', schema: 'public', table: 'messages' },
    (payload) => {
      console.log('New message:', payload.new)
    }
  )
  .subscribe()

// Filter by column value
const channel = supabase
  .channel('room-messages')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'messages',
      filter: 'room_id=eq.abc-123',
    },
    (payload) => { ... }
  )
  .subscribe()

// Presence (track online users)
const channel = supabase.channel('room-1')
channel
  .on('presence', { event: 'sync' }, () => {
    const state = channel.presenceState()
  })
  .subscribe(async (status) => {
    if (status === 'SUBSCRIBED') {
      await channel.track({ user_id: '...', online_at: new Date().toISOString() })
    }
  })

// Cleanup
supabase.removeChannel(channel)
```

### Enabling Real-Time on a Table

```sql
-- Via Supabase dashboard or migration:
ALTER PUBLICATION supabase_realtime ADD TABLE messages;

-- For filtered subscriptions, enable replica identity
ALTER TABLE messages REPLICA IDENTITY FULL;
-- FULL is required if you filter on non-PK columns
```

**Rules:**
- RLS applies to real-time subscriptions. Users only receive changes for rows they can SELECT.
- Always call `removeChannel()` on component unmount to prevent memory leaks.
- Real-time has a default limit of 100 concurrent connections on the free tier.
- Use Broadcast for ephemeral messages (typing indicators, cursor positions) — no database involvement.

---

## Edge Functions

```typescript
// supabase/functions/hello-world/index.ts
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

Deno.serve(async (req: Request) => {
  // CORS handling
  if (req.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Authorization, Content-Type',
      },
    })
  }

  // Access Supabase from within an edge function
  const { createClient } = await import('jsr:@supabase/supabase-js@2')

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,  // Service role for admin access
  )

  // Or use the user's JWT for RLS-scoped access
  const authHeader = req.headers.get('Authorization')!
  const userClient = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_ANON_KEY')!,
    { global: { headers: { Authorization: authHeader } } }
  )

  const { data } = await userClient.from('items').select('*')

  return new Response(JSON.stringify({ items: data }), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

### Calling Edge Functions

```typescript
const { data, error } = await supabase.functions.invoke('hello-world', {
  body: { name: 'test' },
})
```

**Rules:**
- Edge functions run on Deno. Use `jsr:` or `npm:` specifiers for imports.
- `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and `SUPABASE_SERVICE_ROLE_KEY` are auto-injected.
- Set custom secrets via `supabase secrets set MY_KEY=value`.
- Default timeout: 150 seconds (can be extended on Pro plan).
- Edge functions have JWT verification enabled by default (`--no-verify-jwt` to disable for webhooks).

---

## Storage Buckets

```typescript
// Upload
const { data, error } = await supabase.storage
  .from('avatars')
  .upload(`${userId}/avatar.png`, file, {
    contentType: 'image/png',
    upsert: true,
  })

// Get public URL (bucket must be public)
const { data } = supabase.storage
  .from('avatars')
  .getPublicUrl('user-123/avatar.png')

// Get signed URL (for private buckets)
const { data, error } = await supabase.storage
  .from('documents')
  .createSignedUrl('report.pdf', 3600)  // expires in 1 hour

// Delete
await supabase.storage.from('avatars').remove(['user-123/avatar.png'])
```

### Storage RLS Policies

```sql
-- Storage objects are in the storage schema
-- Policy: users can upload to their own folder
CREATE POLICY "user_upload" ON storage.objects
    FOR INSERT
    WITH CHECK (
        bucket_id = 'avatars'
        AND (storage.foldername(name))[1] = auth.uid()::text
    );

-- Policy: public read on a public bucket
CREATE POLICY "public_read" ON storage.objects
    FOR SELECT
    USING (bucket_id = 'avatars');

-- Policy: users can delete their own files
CREATE POLICY "user_delete" ON storage.objects
    FOR DELETE
    USING (
        bucket_id = 'avatars'
        AND (storage.foldername(name))[1] = auth.uid()::text
    );
```

---

## Migration Workflow

### Local Development

```bash
# Initialize Supabase locally
supabase init

# Start local Supabase (Postgres, Auth, Storage, etc.)
supabase start

# Create a new migration
supabase migration new create_users_table
# Edit: supabase/migrations/TIMESTAMP_create_users_table.sql

# Apply migrations locally
supabase db reset  # drops and recreates from migrations

# Diff: generate migration from schema changes made in local DB
supabase db diff -f add_avatar_column

# Push migrations to remote project
supabase db push

# Pull remote migrations (if created via dashboard)
supabase db pull
```

### Migration File Structure

```
supabase/
├── config.toml
├── migrations/
│   ├── 20260301000000_create_users.sql
│   ├── 20260302000000_create_documents.sql
│   └── 20260306000000_add_rls_policies.sql
├── functions/
│   └── hello-world/
│       └── index.ts
└── seed.sql  # Optional seed data for local dev
```

### Linking to Remote

```bash
supabase link --project-ref <project-id>

# Check migration status
supabase migration list

# Apply pending migrations to remote
supabase db push
```

**Rules:**
- Never edit migrations that have already been pushed to a shared environment.
- Use `supabase db diff` to auto-generate migrations from schema changes made via the dashboard or SQL editor.
- Run `supabase db reset` frequently during local development to verify the full migration chain works.
- Include RLS policies in migrations, not just table definitions.

---

### Applying a Migration by Hand (SQL Editor Paste)

Sometimes `supabase db push` is not the path: no CLI credentials on the machine,
a locked-down environment, or an operator who prefers to read every statement
before it runs. Pasting into the SQL Editor works, but two things `db push` does
for you are now your job.

**1. Record the ledger row, in the same transaction as the body.** `db push`
writes a row into `supabase_migrations.schema_migrations`. A pasted migration
writes nothing, so the change is applied but unrecorded, and the next
`supabase migration list` shows it as pending (and the next `db push` tries to
run it again). The ledger's columns are `version` (primary key), `name`, and
`statements`; `statements` stays null for a pasted migration.

```sql
begin;

-- <the migration body, exactly as in the file>

insert into supabase_migrations.schema_migrations (version, name)
values ('20260101120000', 'add_widgets_table')   -- the file name, split at the first underscore
on conflict (version) do nothing;

commit;
```

**2. Supply the transaction yourself.** The SQL Editor does not wrap a paste in
a transaction. Without `begin` / `commit`, a failure halfway leaves the first
half applied.

Habits that make a hand-applied migration safe:

- **Make a data migration one statement.** A single `do $$ ... $$` block is
  atomic by itself: any exception undoes everything the block did, whether or
  not a caller wrapped it. Do not rely on the caller.
- **Guard, then act.** Before a destructive step, check the precondition and
  `raise exception` naming the offending row, so a violated assumption stops
  the whole migration instead of half-applying it.
- **Write it to run twice.** `on conflict do nothing` on inserts; updates that
  only touch rows that differ (`where col is distinct from <value>`), so a
  second run changes nothing, not even `updated_at`.
- **Loop over tenants, never assume one.** A body that iterates organizations
  is a no-op on an empty database, which keeps a from-zero replay clean.
- **Ship the verification with it.** End the file with the read-only `select`s
  to run before and after, as comments, and hand the operator the same queries.
- **Rehearse with `rollback`.** With no local or branch database, the rehearsal
  is the same block with `rollback;` in place of `commit;`, reading the
  verification queries just before the rollback.
- **Never probe with a write.** If the agent's database access is meant to be
  read-only, confirm that by reading its configuration, not by attempting a
  write to see whether it fails.

## Local Development with Supabase CLI

```bash
# Prerequisites
brew install supabase/tap/supabase  # macOS

# Start local stack
supabase start
# Outputs: API URL, anon key, service_role key, Studio URL

# Local Studio (GUI): http://localhost:54323
# Local API: http://localhost:54321
# Local DB: postgresql://postgres:postgres@localhost:54322/postgres

# Stop
supabase stop

# Stop and reset all data
supabase stop --no-backup

# Run tests against local instance
SUPABASE_URL=http://localhost:54321 \
SUPABASE_ANON_KEY=<local-anon-key> \
npm test

# Generate types from local schema
supabase gen types typescript --local > src/types/database.ts
```

### Type Generation

```bash
# From remote project
supabase gen types typescript --project-id <ref> > src/types/database.ts
```

```typescript
// Use generated types with the client
import { Database } from './types/database'

const supabase = createClient<Database>(url, key)

// Now .from('table') is fully typed
const { data } = await supabase.from('users').select('id, email')
// data is typed as { id: string; email: string }[] | null
```

---

## Gotchas & Common Mistakes

### 1. RLS Not Enabled

Tables created via SQL do NOT have RLS enabled by default. Forgetting `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` means the table is fully accessible to anyone with the anon key.

**Fix:** Always include `ENABLE ROW LEVEL SECURITY` in every `CREATE TABLE` migration.

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id),
    content TEXT
);
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
-- Then add policies
```

### 2. Service Role Key Exposure

The `service_role` key bypasses RLS entirely. It must NEVER be exposed to the client.

**Rules:**
- Only use `service_role` in server-side code, edge functions, or backend services.
- Store it in environment variables, never in client-side bundles.
- If leaked, rotate immediately in the Supabase dashboard.

### 3. RLS Policy with No Matching Policy = No Access

If RLS is enabled but no policy grants access, all queries return empty results (not errors). This is a common source of "my query returns nothing" bugs.

**Debug:** Test policies in SQL editor:
```sql
SET ROLE authenticated;
SET request.jwt.claims = '{"sub": "user-uuid-here", "role": "authenticated"}';
SELECT * FROM documents;  -- should return only rows the policy allows
RESET ROLE;
```

### 4. Foreign Key to auth.users

```sql
-- Correct: reference auth.users(id)
CREATE TABLE profiles (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    display_name TEXT
);

-- Set up a trigger to auto-create profile on signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO public.profiles (id, display_name)
    VALUES (NEW.id, NEW.raw_user_meta_data->>'display_name');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
    AFTER INSERT ON auth.users
    FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

### 5. Real-Time Not Working

Checklist:
- Table added to `supabase_realtime` publication? (`ALTER PUBLICATION supabase_realtime ADD TABLE ...`)
- RLS policies allow SELECT for the subscribing user?
- `REPLICA IDENTITY FULL` set if filtering on non-PK columns?
- Not exceeding concurrent connection limits?

### 6. Anon Key vs. Service Role Key

| Key            | Used For                        | RLS     | Expose to Client |
|----------------|--------------------------------|---------|-------------------|
| `anon`         | Client-side queries, auth      | Enforced | Yes               |
| `service_role` | Server-side admin operations   | Bypassed | NEVER             |

### 7. Supabase Client Caching

`supabase.auth.getSession()` reads from local storage/memory — it does NOT validate the token with the server. Use `supabase.auth.getUser()` when you need a server-validated user on the backend.

### 8. `ON DELETE CASCADE` with auth.users

If you `DELETE` a user via the auth admin API, any rows with `REFERENCES auth.users(id) ON DELETE CASCADE` will be deleted. Make sure this is the behavior you want, or use `ON DELETE SET NULL`.

---

### 9. Triggers That Read `auth.uid()` See NULL in the SQL Editor

A trigger that enforces "only an administrator may change this" usually resolves
the acting user from `auth.uid()`. In the SQL Editor, in a seed script, and in a
migration there is no JWT, so `auth.uid()` is null and the trigger treats the
caller as nobody. Consequences worth knowing before they surprise you:

- A seed that **updates** a guarded column (promoting the first user, say) is
  refused by the trigger, even though it runs as the database owner.
- The same seed **inserting** the row is fine when the trigger is
  `before update` only: inserts never reach it.
- An `on conflict ... do update set role = 'x'` fires the update trigger. It
  passes only when the value is unchanged, so such a seed is re-runnable only
  while the stored value already matches.

So: seed privileged rows by insert, state the re-run limit in the seed's header,
and route later changes through the application, where a real actor exists. Do
not "fix" it by disabling the trigger in a seed; that is the guard you wanted.

---

## Quick Reference Checklist

- [ ] RLS enabled on every table with user data
- [ ] RLS policies tested by switching to `authenticated` role in SQL editor
- [ ] `service_role` key only in server-side code, never in client bundles
- [ ] `@supabase/ssr` used for server-rendered frameworks (not vanilla `createClient`)
- [ ] Tables added to `supabase_realtime` publication for real-time
- [ ] `REPLICA IDENTITY FULL` on tables with non-PK real-time filters
- [ ] Profile table with trigger to auto-create on auth signup
- [ ] Migrations include RLS policies, not just table definitions
- [ ] Types generated after schema changes (`supabase gen types typescript`)
- [ ] Local dev uses `supabase start` / `supabase db reset` workflow
- [ ] Edge function secrets managed via `supabase secrets set`
- [ ] Storage policies use `storage.foldername()` for path-based access control
