# GitHub Secrets

צריך 7 secrets בסה"כ. הוסף ב-`https://github.com/YOUR_USERNAME/YOUR_REPO/settings/secrets/actions`.

לחץ **New repository secret** עבור כל אחד.

## Supabase secrets (3)

### איפה מוצאים

1. [Supabase Dashboard](https://supabase.com/dashboard) → הפרויקט שלך
2. לחץ **Connect** למעלה
3. טאב **Direct** → מצב **Session pooler**

תראה URI כזה:

```
postgresql://postgres.PROJECT_REF:[YOUR-PASSWORD]@aws-1-eu-central-1.pooler.supabase.com:5432/postgres
```

חלץ:

- `postgres.PROJECT_REF` → user
- `aws-1-eu-central-1.pooler.supabase.com` → host
- `[YOUR-PASSWORD]` → ראה למטה

### על הסיסמה

Supabase **לא שומר** את סיסמת ה-DB במקום נגיש. הגדרת אותה פעם אחת בעת יצירת הפרויקט, ומאז ה-dashboard מציג רק `[YOUR-PASSWORD]` כפלייסהולדר.

**איפה למצוא את הסיסמה האמיתית:**

- `.env.local` של הפרויקט — חפש שורת comment `# DB password:` או `DATABASE_URL=...:PASSWORD@...`
- Vercel/Render env vars — `DATABASE_URL` או `POSTGRES_URL` (הסיסמה בין `:` ל-`@`)
- מנהל הסיסמאות שלך
- מוצא אחרון: **Reset database password** ב-Supabase Dashboard → Database settings (זה ישבור חיבורים קיימים — עדכן את Vercel/וכו' גם)

### ה-Secrets

| Name                   | Value                                    | הערות                           |
| ---------------------- | ---------------------------------------- | ------------------------------- |
| `SUPABASE_DB_HOST`     | `aws-1-eu-central-1.pooler.supabase.com` | ה-pooler של האזור שלך           |
| `SUPABASE_DB_USER`     | `postgres.YOUR_PROJECT_REF`              | כולל project ref                |
| `SUPABASE_DB_PASSWORD` | סיסמה גולמית                             | **בלי URL encoding** — כמו שהיא |

## Cloudflare R2 secrets (4)

משלב 5 של [setup-r2.he.md](setup-r2.he.md):

| Name                   | Value             | הערות                                                                  |
| ---------------------- | ----------------- | ---------------------------------------------------------------------- |
| `R2_ACCOUNT_ID`        | `abc123def456...` | מה-endpoint URL `https://ABC.r2.cloudflarestorage.com` — רק החלק `ABC` |
| `R2_ACCESS_KEY_ID`     | מה-API token      |                                                                        |
| `R2_SECRET_ACCESS_KEY` | מה-API token      |                                                                        |
| `R2_BUCKET`            | `mybot-backups`   | שם ה-bucket                                                            |

## טיפ: אל תשתף secrets בצ'אט עם Claude (או עם אף אחד)

אם בטעות הדבקת connection string עם הסיסמה האמיתית בצ'אט — כולל הצ'אט הזה — צריך:

1. לסיים את ההגדרה (כדי שיהיה גיבוי לפני סיבוב)
2. Reset password ב-Supabase
3. עדכון Vercel + GitHub Secret + .env.local
4. עדכון מנהל הסיסמאות

## השלב הבא

→ חזור ל-[README](../README.he.md) חלק "העתק את ה-workflow"
