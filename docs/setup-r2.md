# Setting up Cloudflare R2

5 minutes. You'll create a bucket, a lifecycle rule, and an API token.

## 1. Activate R2 (first time only)

1. Sign in to [Cloudflare dashboard](https://dash.cloudflare.com)
2. Left sidebar → **Storage & databases** → **R2 Object Storage**
3. If first time: click to activate. You'll need to add a credit card for verification — **Cloudflare won't charge you** until you exceed 10GB of storage. (For Postgres backups under ~50MB compressed, you'd need to keep ~200 daily backups to hit that.)

## 2. Create a bucket

1. Click **Create bucket**
2. Name: `<project>-backups` (e.g. `mybot-backups`). Must be globally unique within your Cloudflare account.
3. Location: **Automatic** (or pick a region near your Supabase)
4. Storage class: **Standard**
5. Create

## 3. Set lifecycle rule (10-day retention)

1. Click your bucket → **Settings** tab
2. Scroll to **Object lifecycle rules** → **Add rule**
3. Configure:
   - Rule name: `delete-after-10-days`
   - Apply to: **All objects** (leave prefix empty)
   - Action: ✓ **Delete objects** → `10` days
4. Save

This means: every backup older than 10 days gets auto-deleted. You always have ~10 recent snapshots.

## 4. Create an API token

1. Go back to R2 main page (click "R2 Object Storage" in sidebar)
2. Top right: **Manage R2 API tokens**
3. Click **Create Account API token** (the recommended kind)
4. Configure:
   - **Token name**: `<project>-backup-token`
   - **Permissions**: ✓ **Object Read & Write**
   - **Specify bucket(s)**: ✓ **Apply to specific buckets only** → check your bucket
   - **TTL**: Forever (or whatever you prefer)
5. Click **Create API Token**

## 5. Save the credentials (THIS IS YOUR ONLY CHANCE)

Cloudflare shows you 3 values **once**:

- **Access Key ID** — looks like `abc123def456...`
- **Secret Access Key** — looks like `xyz789...` (long random string)
- **Endpoint** (or **Jurisdiction-specific endpoint**) — looks like `https://YOUR_ACCOUNT_ID.r2.cloudflarestorage.com`

Save these in your password manager **right now**. Closing the page = lost forever (you'd need to make a new token).

Note: the `YOUR_ACCOUNT_ID` part of the endpoint = your Cloudflare **Account ID**. You'll need it as a separate secret.

## Next step

→ [setup-secrets.md](setup-secrets.md) — adding these to GitHub
