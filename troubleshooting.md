# Troubleshooting

8 הבאגים שנתקלנו בהם במהלך הקמת המערכת ב-production, ואיך לפתור אותם.

## בעיה 1: `could not translate host name "X@host..." to address`

**שגיאה מלאה (דוגמה)**:
```
pg_dump: error: could not translate host name "4050@aws-1-eu-central-1.pooler.supabase.com" to address: Name or service not known
```

**סיבה**: סיסמת ה-DB מכילה תו `@` (או תווים מיוחדים אחרים). כשמשתמשים ב-connection string URL, ה-parser מבולבל ומפצל לא נכון על ה-`@`.

**פתרון**: השתמש ב-`PGPASSWORD` env var ולא ב-URL. בתבנית שלנו זה כבר מוטמע:
```yaml
env:
  PGHOST: ${{ secrets.SUPABASE_DB_HOST }}
  PGUSER: ${{ secrets.SUPABASE_DB_USER }}
  PGPASSWORD: ${{ secrets.SUPABASE_DB_PASSWORD }}
```

⚠️ **תקלה לא לנסות לפתור**: URL encoding (`@` → `%40`). זה אמור לעבוד, אבל בשטח מצאנו ש-libpq לא תמיד מפענח נכון.

## בעיה 2: `password authentication failed for user "postgres"` למרות שהסיסמה "נכונה"

**סיבה**: המשתמש מנסה להמציא או לנחש את הסיסמה. Supabase **לעולם לא מציג** את הסיסמה ב-dashboard — רק `[YOUR-PASSWORD]` placeholder.

**איך למצוא את הסיסמה האמיתית**:
1. בקובץ `.env.local` של הפרויקט — חפש שורה שמתחילה ב-`# DB password:` (או דומה)
2. ב-Vercel/Render env vars — חפש `DATABASE_URL` או `POSTGRES_URL` — הסיסמה היא החלק בין `:` ל-`@` ב-URL
3. ב-1Password / מנהל סיסמאות אחר
4. אם לא נמצאת — **Reset database password** ב-Supabase (יחייב לעדכן בכל המקומות שמשתמשים בה)

⚠️ **תקלה לא לנסות לפתור**: לבקש מהמשתמש "מה הסיסמה" בלי לחפש קודם בקבצים. רוב המשתמשים יזרקו ניחוש.

## בעיה 3: `S3: CreateBucket, StatusCode: 403, AccessDenied` בהעלאה ל-R2

**שגיאה מלאה**:
```
ERROR : Failed to copy: failed to prepare upload: operation error S3: CreateBucket, https response error StatusCode: 403, api error AccessDenied: Access Denied
```

**סיבה**: ברירת המחדל של rclone היא לבדוק שה-bucket קיים לפני העלאה — באמצעות `CreateBucket` (idempotent operation). אם ה-API token שלך מוגבל רק לכתיבה ב-bucket ספציפי (כפי שמומלץ ב-security), אין לו הרשאה ל-`CreateBucket`.

**פתרון**: הוסף `--s3-no-check-bucket` ל-rclone:
```bash
rclone copy "${FILENAME}" "r2:bucket-name/" --s3-no-check-bucket
```

זה כבר מוטמע בתבנית שלנו.

## בעיה 4: `pg_restore: error: unsupported version (1.16) in file header`

**שגיאה מלאה**:
```
pg_restore: error: unsupported version (1.16) in file header
```

**סיבה**: ב-GitHub Actions runner של Ubuntu, יש PostgreSQL client ישן מותקן כברירת מחדל (לרוב גרסה 14 או 15). ה-PATH מצביע עליו, אבל ה-`pg_dump` החדש (גרסה 17 שהתקנו) כותב פורמט שגרסאות ישנות לא יודעות לקרוא.

**פתרון**: לכפות את ה-PATH של PostgreSQL 17 ראשון:
```yaml
- name: Install PostgreSQL 17 client
  run: |
    sudo apt-get install -y postgresql-client-17
    echo "/usr/lib/postgresql/17/bin" >> $GITHUB_PATH
```

זה כבר מוטמע בתבנית שלנו.

## בעיה 5: Secrets מופיעים ריקים בלוג (לא `***`)

**תסמין**: בלוג ה-workflow רואים:
```
PGHOST: 
PGUSER: 
PGPASSWORD: 
```
ולא `***` כמצופה. pg_dump מנסה להתחבר לשרת מקומי.

**סיבה**: ה-secrets הוספו תחת **Environment secrets** (או Organization secrets) במקום **Repository secrets**. Workflow ללא הגדרת `environment:` לא מקבל Environment secrets.

**פתרון**:
1. היכנס ל-`https://github.com/USER/REPO/settings/secrets/actions`
2. וודא שאתה רואה **"Repository secrets"** בכותרת — לא "Environments"
3. מחק את ה-secrets שהוספת בטעות ב-Environments
4. הוסף מחדש תחת Repository secrets

---

## בעיה 6: `Network is unreachable` או `connection to server ... failed: Network is unreachable`

**שגיאה מלאה**:
```
pg_dump: error: connection to server at "***" (2a05:d014:...), port 5432 failed: Network is unreachable
```

**סיבה**: שימוש ב-**Direct connection** של Supabase שהוא IPv6 בלבד. GitHub Actions רץ על רשת IPv4 בלבד ולא יכול להגיע לכתובות IPv6.

**פתרון**: עדכן את `SUPABASE_DB_HOST` ו-`SUPABASE_DB_USER` לפרטי **Session Pooler**:
- Supabase Dashboard → Connect → **Session pooler**
- Host: `aws-X-REGION.pooler.supabase.com`
- User: `postgres.PROJECT_REF`

---

## בעיה 7: `NoSuchBucket: The specified bucket does not exist`

**שגיאה מלאה**:
```
ERROR : Failed to copy: operation error S3: PutObject, StatusCode: 404, api error NoSuchBucket: The specified bucket does not exist.
```

**סיבה**: ה-R2 API token נוצר לפני שה-bucket נוצר, או שם ה-bucket ב-secret שגוי.

**פתרון**:
1. בדוק ב-Cloudflare R2 dashboard שה-bucket קיים ושמו תואם בדיוק ל-`R2_BUCKET` secret (רגיש לאותיות גדולות/קטנות)
2. אם ה-bucket לא קיים — צור אותו ואז הרץ שוב

---

## בעיה 8 (אזהרה): שיתוף סיסמת ה-DB בצ׳אט

**הקשר**: בזמן הצבת ה-secret ב-GitHub, יש פיתוי להדביק את ה-connection string המלא בצ'אט עם Claude לבדיקה.

**הסיכון**: היסטוריית הצ'אט נשמרת. אם הצ'אט אי פעם ידלוף — הסיסמה דלפה.

**מה לעשות אם זה קרה**:
1. השלם את ההגדרה (שיהיה גיבוי לפני סיבוב)
2. ב-Supabase: **Reset database password**
3. עדכן את הסיסמה החדשה בכל המקומות:
   - GitHub Secret `SUPABASE_DB_PASSWORD`
   - Vercel / Render env vars
   - קובץ `.env.local` מקומי
   - מנהל סיסמאות

**איך למנוע**: כשמדריכים את המשתמש, אמור במפורש "אל תשלח את הסיסמה אליי בצ'אט — רק תאשר שהיא אצלך."
