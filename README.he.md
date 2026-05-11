[English](README.md) · [עברית](README.he.md)

# Supabase → Cloudflare R2 Backup

> גיבוי יומי אוטומטי וחינמי ל-Supabase Postgres. אפס תשתית, אפס עלות חודשית. נבנה ונבדק ב-production.

**למה זה קיים**: ל-Supabase Free **אין גיבוי אוטומטי**. אם הנתונים שלך חשובים — אתה צריך את זה. ל-Supabase Pro יש גיבוי 7 ימים, אבל גיבוי במיקום שני זה best practice.

## מה אתה מקבל

- ✅ גיבוי יומי של ה-DB (פורמט custom דחוס)
- ✅ העלאה ל-Cloudflare R2 — 10GB חינם, בלי דמי egress
- ✅ מחיקה אוטומטית אחרי 10 ימים (ניתן להגדרה)
- ✅ **אימות תקינות** אחרי כל גיבוי (`pg_restore --list`)
- ✅ התראת מייל בכישלון (אוטומטית מ-GitHub)
- ✅ **$0 לחודש** עלות כוללת
- ✅ הקמה ב-~15 דקות

## מה זה **לא**

- ❌ שירות מנוהל. אתה הבעלים של ה-workflow ושל הנתונים.
- ❌ Point-in-time recovery (לזה צריך Supabase Pro — זה משלים, לא מחליף).
- ❌ Replication רציף. רק snapshots יומיים.

## ה-Stack

| רכיב             | למה                                                 |
| ---------------- | --------------------------------------------------- |
| GitHub Actions   | חינם 2000 דק'/חודש (אפילו ב-private), cron אמין     |
| `pg_dump` (PG17) | סטנדרטי, נתמך היטב, דחוס `--format=custom`          |
| Cloudflare R2    | 10GB חינם, **אפס דמי egress** (שלא כמו S3), תואם S3 |
| `rclone`         | הכלי הכי טוב להעלאה ל-S3                            |

## התחלה מהירה (15 דקות)

### 1. צור bucket ב-Cloudflare R2

ראה [docs/setup-r2.he.md](docs/setup-r2.he.md). תצטרך: שם bucket, Account ID, Access Key ID, Secret Access Key.

### 2. שלוף את פרטי החיבור של Supabase

Supabase Dashboard → Connect → **Session pooler** → טאב URI. תצטרך: host, user, password.

⚠️ Supabase **לא מציג** את סיסמת ה-DB ב-UI. מצא אותה ב-`.env.local` של הפרויקט, ב-Vercel env vars, או במנהל הסיסמאות. אם איבדת — אפשר Reset (אבל זה ישבור חיבורים קיימים).

### 3. הוסף 7 GitHub Secrets

ב-`Settings → Secrets and variables → Actions` הוסף:

| Name                   | דוגמה                                    |
| ---------------------- | ---------------------------------------- |
| `SUPABASE_DB_HOST`     | `aws-1-eu-central-1.pooler.supabase.com` |
| `SUPABASE_DB_USER`     | `postgres.YOUR_PROJECT_REF`              |
| `SUPABASE_DB_PASSWORD` | סיסמה גולמית (בלי URL encoding)          |
| `R2_ACCOUNT_ID`        | מ-Cloudflare dashboard                   |
| `R2_ACCESS_KEY_ID`     | מ-R2 API token                           |
| `R2_SECRET_ACCESS_KEY` | מ-R2 API token                           |
| `R2_BUCKET`            | שם ה-bucket שלך                          |

### 4. העתק את ה-workflow

העתק את [`.github/workflows/db-backup.yml`](.github/workflows/db-backup.yml) לתיקיית `.github/workflows/` של הפרויקט שלך.

### 5. Push והרצה

```bash
git add .github/workflows/db-backup.yml
git commit -m "feat: add daily DB backup"
git push

# הפעלה ידנית בפעם הראשונה
gh workflow run "Daily DB Backup"
gh run watch
```

זהו. גיבויים יומיים ירוצו ב-03:00 UTC (06:00 שעון ישראל). אם רוצה שעה אחרת — שנה את ה-cron ב-workflow.

## שחזור גיבוי

ראה [docs/restore.he.md](docs/restore.he.md).

בקצרה:

```bash
# הורד מ-R2 (דרך הממשק של Cloudflare או דרך rclone)
rclone copy r2:YOUR_BUCKET/your-repo-2026-01-15.dump .

# שחזר (זהיר — מוחק נתונים קיימים)
PGPASSWORD=... pg_restore --clean --no-owner --no-acl \
  -h ... -U postgres.PROJECT_REF -d postgres \
  your-repo-2026-01-15.dump
```

## תקלות נפוצות

נתקלנו ב-4 באגים שונים בעת הבנייה. כולם מתועדים ב-[docs/troubleshooting.he.md](docs/troubleshooting.he.md):

1. "could not translate host name" → סיסמה עם תווים מיוחדים. הפתרון כבר מובנה: שימוש ב-`PGPASSWORD` env var במקום URL.
2. "password authentication failed" → אתה מנחש את הסיסמה. תמצא אותה ב-`.env.local` או ב-env vars.
3. "AccessDenied" בהעלאה ל-R2 → token מוגבל ל-bucket. אנחנו משתמשים בדגל `--s3-no-check-bucket`.
4. "unsupported version (1.16)" → אי-התאמת גרסה של pg_restore. אנחנו מכריחים PG17 ב-PATH.

## רישיון

MIT. חופשי לעתק, להתאים.

## קרדיטים

נוצר בהשראת [הפוסט הזה בלינקדאין על גיבוי עצמאי של Supabase](https://www.linkedin.com/posts/shramr_supabase-db-backup-%D7%92%D7%99%D7%91%D7%95%D7%99-%D7%93%D7%90%D7%98%D7%94-ugcPost-7459467479203893248-ady1/). היישום הזה נבנה ונבדק ב-production על ידי [@uriotto](https://github.com/uriotto) על פרויקט Supabase אמיתי (179 לקוחות, 540 הודעות, 13MB DB) — כולל debug ידני של 4 הבאגים שמתועדים למעלה.
