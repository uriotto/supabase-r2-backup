[English](README.md) · [עברית](README.he.md)

# Supabase → Cloudflare R2 Backup

> Free, automated daily backups for Supabase Postgres. Zero infrastructure. Zero monthly cost. Built and battle-tested in production.

**Why this exists**: Supabase Free has **no automatic backups**. If your data matters, you need this. Supabase Pro has 7-day backups, but a second backup location is still best practice.

## What you get

- ✅ Daily `pg_dump` of your Supabase database (custom format, compressed)
- ✅ Uploaded to Cloudflare R2 — 10GB free, no egress fees
- ✅ Auto-deleted after 10 days (configurable)
- ✅ **Integrity verification** after every backup (`pg_restore --list`)
- ✅ Email alert on failure (GitHub's built-in)
- ✅ **$0/month** total cost
- ✅ Setup in ~15 minutes

## What this is NOT

- ❌ A managed service. You own the workflow, you own the data.
- ❌ Point-in-time recovery (use Supabase Pro for that — this complements, doesn't replace).
- ❌ Continuous replication. Daily snapshots only.

## Stack

| Component        | Why                                                        |
| ---------------- | ---------------------------------------------------------- |
| GitHub Actions   | Free 2000 min/month (private repos), reliable cron         |
| `pg_dump` (PG17) | Standard, well-supported, compressed `--format=custom`     |
| Cloudflare R2    | 10GB free, **zero egress fees** (unlike S3), S3-compatible |
| `rclone`         | Best-in-class S3 uploader                                  |

## Quickstart (15 minutes)

### 1. Create Cloudflare R2 bucket

See [docs/setup-r2.md](docs/setup-r2.md). You'll need: bucket name, Account ID, Access Key ID, Secret Access Key.

### 2. Get your Supabase DB credentials

Supabase Dashboard → Connect → **Session pooler** → URI tab. You'll need: host, user, password.

⚠️ Supabase **doesn't show** your DB password in the UI. Find it in your project's `.env.local`, Vercel env vars, or your password manager. If lost, you can Reset (but it'll break existing connections).

### 3. Add 7 GitHub Secrets

In `Settings → Secrets and variables → Actions` add:

| Name                   | Example                                  |
| ---------------------- | ---------------------------------------- |
| `SUPABASE_DB_HOST`     | `aws-1-eu-central-1.pooler.supabase.com` |
| `SUPABASE_DB_USER`     | `postgres.YOUR_PROJECT_REF`              |
| `SUPABASE_DB_PASSWORD` | Raw password (no URL encoding)           |
| `R2_ACCOUNT_ID`        | From Cloudflare dashboard                |
| `R2_ACCESS_KEY_ID`     | From R2 API token                        |
| `R2_SECRET_ACCESS_KEY` | From R2 API token                        |
| `R2_BUCKET`            | Your bucket name                         |

See [docs/setup-secrets.md](docs/setup-secrets.md) for screenshots.

### 4. Copy the workflow

Copy [`.github/workflows/db-backup.yml`](.github/workflows/db-backup.yml) to your repo's `.github/workflows/` directory.

### 5. Push and run

```bash
git add .github/workflows/db-backup.yml
git commit -m "feat: add daily DB backup"
git push

# Trigger manually first time
gh workflow run "Daily DB Backup"
gh run watch
```

That's it. Daily backups will run at 03:00 UTC. Change the cron in the workflow if you prefer a different time.

## Restoring a backup

See [docs/restore.md](docs/restore.md).

TL;DR:

```bash
# Download from R2 (via Cloudflare dashboard, or rclone)
rclone copy r2:YOUR_BUCKET/your-repo-2026-01-15.dump .

# Restore (CAREFUL — wipes existing data)
PGPASSWORD=... pg_restore --clean --no-owner --no-acl \
  -h ... -U postgres.PROJECT_REF -d postgres \
  your-repo-2026-01-15.dump
```

## Troubleshooting

We hit 4 distinct bugs building this. They're all documented in [docs/troubleshooting.md](docs/troubleshooting.md) — most common ones:

- "could not translate host name" → password has special chars; we already handle this by using `PGPASSWORD` env var instead of URL.
- "password authentication failed" → you're guessing the password. Find it in `.env.local` or env vars.
- "AccessDenied" on R2 upload → token scoped to bucket. We use `--s3-no-check-bucket` flag.
- "unsupported version (1.16)" → pg_restore mismatch. We force PG17 binaries in PATH.

## With Claude Code

If you use [Claude Code](https://claude.com/claude-code), there's a matching skill that walks you through setup interactively. Install it:

```bash
git clone https://github.com/uriotto/supabase-r2-backup.git
cp -r supabase-r2-backup/.claude/skills/supabase-r2-backup ~/.claude/skills/
```

Then in any Claude Code session, just ask: "set up a Supabase backup to R2" and the skill will guide you through the whole flow.

## Contributing

PRs welcome, especially for:

- Other storage backends (S3, GCS, B2)
- Other DB providers (Neon, Railway, RDS)
- Internationalized docs

See [README.he.md](README.he.md) for the Hebrew guide.

## License

MIT. Copy freely, adapt freely.

## Credits

Inspired by [this Hebrew LinkedIn post on independent Supabase backups](https://www.linkedin.com/posts/shramr_supabase-db-backup-%D7%92%D7%99%D7%91%D7%95%D7%99-%D7%93%D7%90%D7%98%D7%94-ugcPost-7459467479203893248-ady1/). This implementation was built and tested in production by [@uriotto](https://github.com/uriotto) on a real Supabase project (179 clients, 540 messages, 13MB DB) with hands-on debugging of the 4 issues documented above.
