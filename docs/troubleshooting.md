# Troubleshooting

The 4 bugs we hit setting this up in production, and how to fix them.

## Bug 1: `could not translate host name "X@host..." to address`

**Full error (example)**:

```
pg_dump: error: could not translate host name "4050@aws-1-eu-central-1.pooler.supabase.com" to address: Name or service not known
```

**Cause**: Your DB password contains an `@` character (or other special characters). When using a connection-string URL, the parser gets confused and splits incorrectly on the `@`.

**Fix**: Use the `PGPASSWORD` env var instead of embedding the password in a URL. Our template already does this:

```yaml
env:
  PGHOST: ${{ secrets.SUPABASE_DB_HOST }}
  PGUSER: ${{ secrets.SUPABASE_DB_USER }}
  PGPASSWORD: ${{ secrets.SUPABASE_DB_PASSWORD }}
```

⚠️ **Don't try to fix this with URL encoding** (`@` → `%40`). In theory it should work, but in practice we found libpq doesn't always decode properly.

## Bug 2: `password authentication failed for user "postgres"` despite "correct" password

**Cause**: You're guessing or inventing the password. Supabase **never displays** the password in the dashboard — only `[YOUR-PASSWORD]` as a placeholder.

**How to find the real password**:

1. Project's `.env.local` — look for a `# DB password:` comment or `DATABASE_URL=...:PASSWORD@...`
2. Vercel/Render env vars — `DATABASE_URL` or `POSTGRES_URL` (password is between `:` and `@` in the URL)
3. Password manager (1Password, etc.)
4. If lost — **Reset database password** in Supabase (will break existing connections — you'll need to update everywhere)

⚠️ **Don't** just ask the user "what's the password" without searching files first. Most users will throw out a guess.

## Bug 3: `S3: CreateBucket, StatusCode: 403, AccessDenied` on R2 upload

**Full error**:

```
ERROR : Failed to copy: failed to prepare upload: operation error S3: CreateBucket, https response error StatusCode: 403, api error AccessDenied: Access Denied
```

**Cause**: rclone's default behavior is to check the bucket exists before uploading — via `CreateBucket` (an idempotent op). If your API token is scoped to a specific bucket only (as recommended for security), it doesn't have permission to call `CreateBucket`.

**Fix**: Add `--s3-no-check-bucket` to rclone:

```bash
rclone copy "${FILENAME}" "r2:bucket-name/" --s3-no-check-bucket
```

Already in our template.

## Bug 4: `pg_restore: error: unsupported version (1.16) in file header`

**Full error**:

```
pg_restore: error: unsupported version (1.16) in file header
```

**Cause**: GitHub Actions' Ubuntu runner has an older PostgreSQL client preinstalled (typically version 14 or 15). PATH points to it, but the new `pg_dump` (version 17 you just installed) writes a format that older versions can't read.

**Fix**: Force PostgreSQL 17 first in PATH:

```yaml
- name: Install PostgreSQL 17 client
  run: |
    sudo apt-get install -y postgresql-client-17
    echo "/usr/lib/postgresql/17/bin" >> $GITHUB_PATH
```

Already in our template.

## Bug 5 (warning): Sharing DB password in chat

**Context**: While setting up the GitHub Secret, there's a temptation to paste the full connection string into a chat (with Claude or anyone) for verification.

**Risk**: Chat history is retained. If the chat ever leaks — your password leaked too.

**What to do if this happened**:

1. Complete the setup first (so you have a backup before rotating)
2. In Supabase: **Reset database password**
3. Update the new password everywhere:
   - GitHub Secret `SUPABASE_DB_PASSWORD`
   - Vercel / Render env vars
   - Local `.env.local`
   - Password manager

**How to prevent**: When guiding users, explicitly say "Don't paste the password to me — just confirm you have it."
