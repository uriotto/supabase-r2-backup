# Restoring a Backup

The whole point of backups is restore. Here's how.

## When you'd restore

1. **Something corrupted production data** (bad migration, mass delete, bug) — restore yesterday's snapshot
2. **Supabase account locked/banned/deleted** — restore to a fresh Supabase project
3. **You want to test the backup is actually valid** — restore to a throwaway DB

## Step 1: Get the backup file

### Option A — Cloudflare web UI

1. [Cloudflare dashboard](https://dash.cloudflare.com) → R2 → your bucket
2. Find the backup you want (`YOUR_REPO-YYYY-MM-DD.dump`)
3. Click → **Download**

### Option B — rclone (faster, scriptable)

First-time rclone setup:

```bash
rclone config
# n (new remote)
# name: r2
# type: s3
# provider: Cloudflare
# access_key_id: <from secrets>
# secret_access_key: <from secrets>
# endpoint: https://ACCOUNT_ID.r2.cloudflarestorage.com
# region: auto
```

Then:

```bash
rclone copy r2:YOUR_BUCKET/YOUR_REPO-2026-01-15.dump .
```

## Step 2: Verify the file (optional but recommended)

```bash
pg_restore --list YOUR_REPO-2026-01-15.dump | head -30
```

You should see lines like:

```
4306; 0 17361 TABLE DATA public clients postgres
4307; 0 17372 TABLE DATA public conversations postgres
...
```

If you see `unsupported version (1.16)`, you need pg_restore 17. Install:

```bash
# macOS
brew install postgresql@17

# Ubuntu
sudo apt-get install postgresql-client-17
```

## Step 3: Restore

### To the same Supabase project (overwrites existing data)

⚠️ **DANGEROUS**. Make absolutely sure you want to wipe what's there.

```bash
PGPASSWORD='your_password' pg_restore \
  --clean --no-owner --no-acl \
  -h aws-1-eu-central-1.pooler.supabase.com \
  -p 5432 \
  -U postgres.YOUR_PROJECT_REF \
  -d postgres \
  YOUR_REPO-2026-01-15.dump
```

The `--clean` flag drops existing objects before restoring.

### To a fresh Supabase project (safest)

1. Create a new Supabase project (free tier is fine for testing)
2. Get its connection details (same as setup)
3. Run the same command but with the new project's credentials

This is the recommended approach if you're not 100% sure — you don't risk overwriting your live data.

### To a local Postgres (testing)

```bash
# Start a local Postgres
docker run -d --name pg-restore-test -e POSTGRES_PASSWORD=test -p 5432:5432 postgres:17

# Restore
PGPASSWORD=test pg_restore \
  --no-owner --no-acl \
  -h localhost -U postgres -d postgres \
  YOUR_REPO-2026-01-15.dump

# Verify
PGPASSWORD=test psql -h localhost -U postgres -c "\dt"
```

## Step 4: Verify restored data

After restore, run some sanity checks:

```sql
SELECT count(*) FROM clients;
SELECT count(*) FROM messages;
SELECT max(created_at) FROM messages;  -- should match the backup date
```

## Common errors

### `pg_restore: error: connection to server ... failed`

- Wrong host/user/password
- Network issue (try from another machine)

### `permission denied for schema public`

- Use `--no-owner --no-acl` flags
- Supabase Free has different role permissions than self-hosted Postgres

### `relation "X" already exists`

- Add `--clean` to drop existing objects before restore
- Or use `--if-exists` for safer cleanup

### Tables exist but are empty

- Ensure you used `--format=custom` in the original `pg_dump`
- Check that the dump file size > 1KB (`ls -lh <file>.dump`)

## Practice run

It's a good idea to do a full restore test **once per quarter** to a throwaway Supabase project, just to confirm the workflow works end-to-end. Schedule a reminder.
