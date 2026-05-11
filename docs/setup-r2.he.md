# הקמת Cloudflare R2

5 דקות. תיצור bucket, lifecycle rule, ו-API token.

## 1. הפעלת R2 (פעם ראשונה בלבד)

1. היכנס ל-[Cloudflare Dashboard](https://dash.cloudflare.com)
2. תפריט שמאלי → **Storage & databases** → **R2 Object Storage**
3. אם זו הפעם הראשונה: לחץ להפעלה. תידרש להוסיף כרטיס אשראי לאימות — **Cloudflare לא יחייב אותך** עד שתעבור 10GB אחסון. (לגיבויי Postgres מתחת ל-50MB דחוסים, תצטרך לשמור ~200 גיבויים יומיים כדי להגיע לזה.)

## 2. יצירת bucket

1. לחץ **Create bucket**
2. שם: `<project>-backups` (לדוגמה: `mybot-backups`). חייב להיות ייחודי גלובלית בחשבון Cloudflare שלך.
3. Location: **Automatic** (או בחר אזור קרוב ל-Supabase)
4. Storage class: **Standard**
5. צור

## 3. הגדרת lifecycle rule (10 ימי retention)

1. לחץ על ה-bucket → טאב **Settings**
2. גלול ל-**Object lifecycle rules** → **Add rule**
3. הגדר:
   - שם הכלל: `delete-after-10-days`
   - Apply to: **All objects** (השאר prefix ריק)
   - Action: ✓ **Delete objects** → `10` days
4. שמור

המשמעות: כל גיבוי מעל 10 ימים נמחק אוטומטית. תמיד יש לך ~10 snapshots עדכניים.

## 4. יצירת API token

1. חזור לעמוד הראשי של R2 (לחץ "R2 Object Storage" בסיידבר)
2. למעלה מימין: **Manage R2 API tokens**
3. לחץ **Create Account API token** (זה הסוג המומלץ)
4. הגדר:
   - **Token name**: `<project>-backup-token`
   - **Permissions**: ✓ **Object Read & Write**
   - **Specify bucket(s)**: ✓ **Apply to specific buckets only** → סמן את ה-bucket שלך
   - **TTL**: Forever (או מה שתעדיף)
5. לחץ **Create API Token**

## 5. שמור את ה-credentials (זו ההזדמנות היחידה שלך)

Cloudflare מציג 3 ערכים **פעם אחת**:

- **Access Key ID** — נראה כמו `abc123def456...`
- **Secret Access Key** — נראה כמו `xyz789...` (מחרוזת ארוכה רנדומלית)
- **Endpoint** (או **Jurisdiction-specific endpoint**) — נראה כמו `https://YOUR_ACCOUNT_ID.r2.cloudflarestorage.com`

שמור אותם במנהל הסיסמאות **עכשיו**. סגירת העמוד = איבוד לתמיד (תצטרך ליצור token חדש).

הערה: החלק `YOUR_ACCOUNT_ID` ב-endpoint = ה-**Account ID** של Cloudflare שלך. תצטרך אותו כ-secret נפרד.

## השלב הבא

→ [setup-secrets.he.md](setup-secrets.he.md) — הוספה ל-GitHub
