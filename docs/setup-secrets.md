# GitHub Secrets

You need 7 secrets total. Add them at:
`https://github.com/YOUR_USERNAME/YOUR_REPO/settings/secrets/actions`

Click **New repository secret** for each.

## Supabase secrets (3)

### Where to find them

1. [Supabase Dashboard](https://supabase.com/dashboard) → your project
2. Click **Connect** at the top
3. **Direct** tab → **Session pooler** mode

You'll see a URI like:

```
postgresql://postgres.PROJECT_REF:[YOUR-PASSWORD]@aws-1-eu-central-1.pooler.supabase.com:5432/postgres
```

Parse it:

- `postgres.PROJECT_REF` → user
- `aws-1-eu-central-1.pooler.supabase.com` → host
- `[YOUR-PASSWORD]` → see below

### About the password

Supabase **doesn't store** the DB password anywhere visible. You set it once when creating the project, and from then on the dashboard only shows `[YOUR-PASSWORD]` as a placeholder.

**Where to find your actual password:**

- `.env.local` of your project — look for `# DB password:` comment or `DATABASE_URL=...:PASSWORD@...`
- Vercel/Render env vars — `DATABASE_URL` or `POSTGRES_URL` (password is between `:` and `@`)
- Your password manager
- Last resort: **Reset database password** in Supabase Dashboard → Database settings (this will break existing connections — update Vercel/etc too)

### The secrets

| Name                   | Value                                    | Notes                             |
| ---------------------- | ---------------------------------------- | --------------------------------- |
| `SUPABASE_DB_HOST`     | `aws-1-eu-central-1.pooler.supabase.com` | Your region's pooler              |
| `SUPABASE_DB_USER`     | `postgres.YOUR_PROJECT_REF`              | Includes project ref              |
| `SUPABASE_DB_PASSWORD` | Raw password                             | **No URL encoding** — paste as-is |

## Cloudflare R2 secrets (4)

From step 5 of [setup-r2.md](setup-r2.md):

| Name                   | Value             | Notes                                                                              |
| ---------------------- | ----------------- | ---------------------------------------------------------------------------------- |
| `R2_ACCOUNT_ID`        | `abc123def456...` | From the endpoint URL `https://ABC.r2.cloudflarestorage.com` — only the `ABC` part |
| `R2_ACCESS_KEY_ID`     | From API token    |                                                                                    |
| `R2_SECRET_ACCESS_KEY` | From API token    |                                                                                    |
| `R2_BUCKET`            | `mybot-backups`   | Your bucket name                                                                   |

## Tip: Don't share secrets in chat with Claude (or anyone)

If you accidentally paste a connection string with the real password into a chat — including this one — you should:

1. Finish the setup (so you have a backup before rotating)
2. Reset the password in Supabase
3. Update Vercel + GitHub Secret + .env.local
4. Update your password manager

## Next step

→ Back to [README](../README.md) section "Copy the workflow"
