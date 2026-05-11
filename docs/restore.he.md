# שחזור גיבוי

המטרה היחידה של גיבוי היא שחזור. הנה איך.

## מתי משחזרים

1. **משהו השחית נתוני production** (מיגרציה רעה, מחיקה מאסיבית, באג) — שחזר snapshot של אתמול
2. **חשבון Supabase נחסם/נמחק** — שחזר לפרויקט Supabase חדש
3. **רוצה לבדוק שהגיבוי באמת תקין** — שחזר ל-DB זמני

## שלב 1: השג את קובץ הגיבוי

### אופציה א — ממשק Cloudflare

1. [Cloudflare dashboard](https://dash.cloudflare.com) → R2 → ה-bucket שלך
2. מצא את הגיבוי שאתה רוצה (`YOUR_REPO-YYYY-MM-DD.dump`)
3. לחץ → **Download**

### אופציה ב — rclone (מהיר יותר, ניתן לסקריפט)

הגדרה ראשונית של rclone:

```bash
rclone config
# n (new remote)
# name: r2
# type: s3
# provider: Cloudflare
# access_key_id: <מה-secrets>
# secret_access_key: <מה-secrets>
# endpoint: https://ACCOUNT_ID.r2.cloudflarestorage.com
# region: auto
```

אחר כך:

```bash
rclone copy r2:YOUR_BUCKET/YOUR_REPO-2026-01-15.dump .
```

## שלב 2: ודא תקינות הקובץ (אופציונלי אבל מומלץ)

```bash
pg_restore --list YOUR_REPO-2026-01-15.dump | head -30
```

אמורות להופיע שורות כמו:

```
4306; 0 17361 TABLE DATA public clients postgres
4307; 0 17372 TABLE DATA public conversations postgres
...
```

אם אתה רואה `unsupported version (1.16)`, צריך pg_restore 17:

```bash
# macOS
brew install postgresql@17

# Ubuntu
sudo apt-get install postgresql-client-17
```

## שלב 3: שחזר

### לאותו פרויקט Supabase (דורס נתונים קיימים)

⚠️ **מסוכן**. וודא במאת אחוז שאתה רוצה למחוק את מה שיש.

```bash
PGPASSWORD='your_password' pg_restore \
  --clean --no-owner --no-acl \
  -h aws-1-eu-central-1.pooler.supabase.com \
  -p 5432 \
  -U postgres.YOUR_PROJECT_REF \
  -d postgres \
  YOUR_REPO-2026-01-15.dump
```

הדגל `--clean` מוחק אובייקטים קיימים לפני שחזור.

### לפרויקט Supabase חדש (הכי בטוח)

1. צור פרויקט Supabase חדש (Free זה בסדר לבדיקה)
2. השג את פרטי החיבור שלו (אותו תהליך כמו בהקמה)
3. הרץ את אותה פקודה רק עם ה-credentials של הפרויקט החדש

זו הגישה המומלצת אם אתה לא 100% בטוח — לא מסכן את הנתונים החיים.

### ל-Postgres מקומי (בדיקה)

```bash
# הפעל Postgres מקומי
docker run -d --name pg-restore-test -e POSTGRES_PASSWORD=test -p 5432:5432 postgres:17

# שחזר
PGPASSWORD=test pg_restore \
  --no-owner --no-acl \
  -h localhost -U postgres -d postgres \
  YOUR_REPO-2026-01-15.dump

# וודא
PGPASSWORD=test psql -h localhost -U postgres -c "\dt"
```

## שלב 4: ודא נתונים משוחזרים

אחרי השחזור, הרץ בדיקות שפיות:

```sql
SELECT count(*) FROM clients;
SELECT count(*) FROM messages;
SELECT max(created_at) FROM messages;  -- אמור להתאים לתאריך הגיבוי
```

## שגיאות נפוצות

### `pg_restore: error: connection to server ... failed`

- host/user/password שגויים
- בעיית רשת (נסה ממכונה אחרת)

### `permission denied for schema public`

- השתמש בדגלים `--no-owner --no-acl`
- ל-Supabase Free יש הרשאות role שונות מ-Postgres self-hosted

### `relation "X" already exists`

- הוסף `--clean` כדי למחוק אובייקטים קיימים לפני שחזור
- או השתמש ב-`--if-exists` לניקוי בטוח יותר

### טבלאות קיימות אבל ריקות

- וודא שהשתמשת ב-`--format=custom` ב-`pg_dump` המקורי
- בדוק שגודל קובץ ה-dump > 1KB (`ls -lh <file>.dump`)

## ריצת תרגול

מומלץ לבצע תרגיל שחזור מלא **פעם ברבעון** לפרויקט Supabase זמני, רק כדי לוודא שה-workflow עובד מקצה לקצה. שים תזכורת ביומן.
