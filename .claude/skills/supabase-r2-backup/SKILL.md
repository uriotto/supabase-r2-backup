---
name: supabase-r2-backup
description: הגדרת גיבוי יומי אוטומטי של Supabase Postgres ל-Cloudflare R2 דרך GitHub Actions. חינמי לחלוטין, מאמת תקינות, מוחק אוטומטית אחרי 10 ימים, שולח התראת מייל בכישלון. הפעל כשהמשתמש צריך גיבוי DB ל-Supabase (במיוחד ב-Free tier שאין בו גיבוי אוטומטי).
---

# Supabase → Cloudflare R2 Backup

מערכת גיבוי יומי חינמית ומוכחת ל-Supabase, מבוססת על GitHub Actions + Cloudflare R2 + rclone.

## מתי להפעיל את הסקיל

- המשתמש שואל איך לגבות Supabase
- המשתמש ב-Supabase Free (אין גיבוי מובנה!) ויש לו נתוני production
- המשתמש רוצה רובד הגנה נוסף מעבר לגיבוי של Supabase Pro
- המשתמש שואל על data backup, disaster recovery, DB safety

## דרישות מקדימות

- פרויקט Supabase קיים
- פרויקט מנוהל ב-GitHub repo (אפילו פרטי — Actions עובד גם בפרטי, 2000 דקות בחינם בחודש)
- כרטיס אשראי לאימות חשבון Cloudflare (לא יחויב — 10GB ראשונים ב-R2 בחינם)

## תהליך — 7 שלבים

### שלב 1 — אבחון ראשוני

שאל את המשתמש:
- איזו תוכנית Supabase? (אם Pro — הסבר שכבר יש גיבוי 7 ימים, אבל שכבת ביטוח שנייה זה best practice)
- מה ה-project ref של Supabase? (אפשר לראות ב-URL של ה-dashboard: `supabase.com/dashboard/project/XXXX`)
- באיזה GitHub repo הפרויקט?

אם אין repo ב-GitHub — צריך ליצור אחד ראשון. הסקיל הזה לא רץ בלי GitHub Actions.

### שלב 2 — הכנת Cloudflare R2 (המשתמש עושה ידנית)

הדרך את המשתמש:

1. היכנס ל-[Cloudflare Dashboard](https://dash.cloudflare.com) → תפריט שמאלי → **Storage & databases** → **R2 Object Storage**
2. אם זו הפעם הראשונה: לחץ להפעיל R2 (חינם, יבקש כרטיס אשראי לאימות בלבד)
3. לחץ **Create bucket**:
   - Name: `<project-name>-backups` (לדוגמה: `mybot-backups`)
   - Location: Automatic
4. אחרי יצירה → טאב **Settings** → **Object lifecycle rules** → **Add rule**:
   - Rule name: `delete-after-10-days`
   - Action: Delete objects after **10 days**
   - Apply to: All objects
5. חזור לעמוד R2 הראשי → **Manage R2 API tokens** → **Create Account API token**:
   - Token name: `<project>-backup-token`
   - Permissions: **Object Read & Write**
   - Scope: Apply to specific buckets only → `<project>-backups`
   - TTL: Forever
6. **שמור מיד את 3 הערכים** (מוצגים פעם אחת בלבד):
   - Access Key ID
   - Secret Access Key
   - Account ID (מופיע ב-Endpoint כמו `https://XXXXX.r2.cloudflarestorage.com` — ה-XXXXX הוא ה-Account ID)

### שלב 3 — שליפת פרטי חיבור Supabase (המשתמש עושה ידנית)

1. Supabase Dashboard → לחץ **Connect** למעלה
2. טאב **Direct** → **Connection method: Session pooler**
3. רשום את שלושת הערכים:
   - **PGHOST** — לדוגמה: `aws-1-eu-central-1.pooler.supabase.com`
   - **PGUSER** — לדוגמה: `postgres.PROJECT_REF` (PROJECT_REF הוא ה-ID של הפרויקט)
   - **PGPORT** — `5432`

4. **DB password** — זו הסיסמה שנקבעה בעת יצירת הפרויקט. ⚠️ Supabase **לא מציג** את הסיסמה — רק `[YOUR-PASSWORD]` placeholder. אם המשתמש שכח:
   - אופציה א': חפש בקובץ `.env.local` או ב-Vercel/Render env vars שורה שמתחילה ב-`# DB password:` או `DATABASE_URL=postgresql://...:PASSWORD@...`
   - אופציה ב': **Reset database password** (יחייב עדכון הסיסמה בכל המקומות שמשתמשים בה — Vercel, .env.local וכו')

### שלב 4 — הוספת GitHub Secrets

ב-`https://github.com/USER/REPO/settings/secrets/actions` הוסף 5 secrets:

| Name | Value |
|---|---|
| `SUPABASE_DB_PASSWORD` | סיסמת ה-DB (raw, **בלי URL encoding**) |
| `SUPABASE_DB_HOST` | למשל `aws-1-eu-central-1.pooler.supabase.com` |
| `SUPABASE_DB_USER` | למשל `postgres.PROJECT_REF` |
| `R2_ACCOUNT_ID` | Account ID מ-Cloudflare |
| `R2_ACCESS_KEY_ID` | מ-R2 API token |
| `R2_SECRET_ACCESS_KEY` | מ-R2 API token |
| `R2_BUCKET` | שם ה-bucket שיצרת (לדוגמה `mybot-backups`) |

(זה 7, לא 5 — שיניתי לשליפת פרמטרי DB כ-secrets כדי שהסקיל יעבוד לכל אזור/פרויקט).

### שלב 5 — העתקת ה-workflow

העתק את הקובץ `templates/db-backup.yml.template` (יחד עם הסקיל הזה) ל-`.github/workflows/db-backup.yml` בפרויקט של המשתמש.

הקובץ generic — לא צריך לערוך אותו. כל הערכים נטענים מ-secrets.

### שלב 6 — Commit, Push, הרצה ידנית

```bash
git add .github/workflows/db-backup.yml
git commit -m "feat: add daily DB backup to Cloudflare R2"
git push
```

הפעלה ידנית:
```bash
gh workflow run "Daily DB Backup"
gh run watch  # עקוב אחרי הריצה
```

### שלב 7 — אימות ותיעוד

1. אחרי הרצה מוצלחת — בדוק שהקובץ הופיע ב-R2 bucket (יש Object אחד בשם `<project>-YYYY-MM-DD.dump`)
2. הוסף סעיף "Database Backups" לקובץ `CLAUDE.md` של הפרויקט עם:
   - מיקום ה-workflow
   - שעת הריצה
   - איך משחזרים (פקודת pg_restore)

## באגים נפוצים

ראה `troubleshooting.md` ליד הסקיל. 4 הבאגים המתועדים:
1. שגיאת URL parsing על סיסמה עם תווים מיוחדים
2. סיסמה שגויה (המשתמש מנסה להמציא במקום למצוא את האמיתית)
3. `AccessDenied` ב-upload ל-R2 (חסר flag ל-rclone)
4. `unsupported version (1.16) in file header` (גרסת pg_restore לא תואמת)

## עקרונות

- **אל תקודד URL** את הסיסמה. השתמש ב-`PGPASSWORD` env var ישירות.
- **אל תניח** ערכים — תמיד שאל את המשתמש או חפש בקבצים הקיימים.
- **תאמת תקינות** של הגיבוי אחרי כל הרצה (`pg_restore --list`).
- **שמור על "$0 setup"** — אל תוסיף תלות בכלים בתשלום (Resend וכו'). GitHub שולח מייל אוטומטי בכישלון.

## למה לא pg_cron לניקוי טבלאות?

המדריך המקורי כולל שלב 5 (pg_cron לניקוי טבלאות logs/cache). דילגנו עליו כברירת מחדל כי רוב הפרויקטים הקטנים אין להם טבלאות נפוחות. הוסף רק אם ה-DB > 100MB ויש logs.

## רישוי

MIT — חופשי לעתק, להתאים, להפיץ.

## מקור הקוד

ה-template ב-`templates/db-backup.yml.template` הוא הקוד הסופי שנבדק על Supabase production (פרויקט פעיל, 13MB DB, 540 הודעות). נתקלנו ב-4 באגים תוך כדי, ואת כולם פתרנו — הפתרונות מטמועים בתבנית.
