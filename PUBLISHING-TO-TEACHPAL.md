# העלאת קורס ל-TeachPal — מדריך מלא

> משלים את `CLAUDE.md` שבתיקייה הזו. `CLAUDE.md` = **איך בונים** קורס (עקרונות למידה, עיצוב, stack).
> הקובץ הזה = **איך מחברים אותו למערכת** (campus.z-school.co.il).
>
> נכתב אחרי העלאת CapCut ב-16.7.2026 — כולל כל מה שנשבר בדרך, כדי שלא יישבר שוב.

---

## 1. המודל: "חבילת תוכן" (content pack)

הלומדה **לא** נבנית מחדש בתוך TeachPal. היא נשארת HTML עצמאי, ורצה בתוך `<iframe>` בתוך המערכת.

```
TeachPal (React)                    החבילה (HTML רגיל)
┌──────────────────────────┐
│ דף הקורס = מסך הבית      │
│  ├── יחידה 0             │        public/capcut-course/
│  ├── יחידה 1  ──────────────────► units/unit-01-install.html
│  └── ...                 │        assets/shared.css, shared.js
│                          │
│  מאזין ל-postMessage ◄──────────  "סיימתי" שולח הודעת סיום
│  שומר התקדמות ב-DB       │
└──────────────────────────┘
```

**חלוקת האחריות:**
- **TeachPal** = מסך הבית, רשימת היחידות, ההתקדמות (בדאטהבייס), הרשאות, תשלום.
- **החבילה** = התוכן של היחידה עצמה בלבד.

למה זה טוב: ההתקדמות נשמרת בדאטהבייס (שורדת החלפת דפדפן/מכשיר, והמורים רואים אותה), במקום ב-localStorage של הלומד.

**קורסים קיימים:** `cyber` (CyberPath), `capcut`. שניהם באותו מנגנון.

---

## 2. חוזה החבילה — מה הלומדה חייבת לעשות

אלה הדרישות שבלעדיהן החיבור לא יעבוד. **תבנו ככה מההתחלה** — לתקן בסוף זה כאב.

### 2.1 כל יחידה מסומנת במספר

```html
<body data-unit="1">
```

### 2.2 כפתור הסיום שולח הודעה ל-LMS

```html
<a class="btn" data-complete href="unit-02-basic-edit.html">סיימתי</a>
```

וב-`shared.js`:

```js
function isEmbedded() {
  try { return window.parent && window.parent !== window; } catch (e) { return true; }
}

const doneBtn = document.querySelector("[data-complete]");
if (doneBtn && isEmbedded()) doneBtn.textContent = "סיימתי ✓";
if (doneBtn) doneBtn.addEventListener("click", function (ev) {
  markComplete(n);
  if (isEmbedded()) {
    ev.preventDefault();                      // ← קריטי, ראה 2.3
    window.parent.postMessage(
      { type: "capcut-unit-complete", unitId: n }, "*"   // ← שם ייחודי לקורס
    );
  }
});
```

שם ההודעה הוא `<slug>-unit-complete` (למשל `capcut-unit-complete`). חייב להיות ייחודי לקורס.

### 2.3 מצב מוטמע (embedded) — למה זה חובה

בלי זה נוצרים **שני "בתים" מתחרים**: הלומד לוחץ "סיימתי", ה-iframe קופץ ליחידה הבאה בעצמו, ובמקביל ה-LMS מנסה לחזור לדף הקורס. התוצאה מבלבלת וההתקדמות בורחת מהמערכת.

```js
if (isEmbedded()) document.body.classList.add("is-embedded");
```

```css
/* מסך הבית והניווט שייכים ל-LMS, לא לקורס */
body.is-embedded .topbar a.home { display: none; }
```

**הרצה עצמאית (file:// או לינק ישיר) לא מושפעת בכלל** — הכל נשאר עובד כרגיל.

### 2.4 יחידה 0 חייבת להיות ניתנת לסיום ⚠️

היחידות נפתחות **בשרשרת**: יחידה 1 נפתחת רק אחרי שיחידה 0 הושלמה. אם ליחידת האוריינטציה אין כפתור `data-complete` — **כל הקורס נעול לנצח**. זה נבדק בפועל.

### 2.5 נתיבים יחסיים

```html
<link rel="stylesheet" href="../assets/shared.css">
```
מ-`units/` זה נפתר ל-`/capcut-course/assets/shared.css`. עובד גם עצמאית וגם במערכת.

### 2.6 וידאו — ביוטיוב, לא בגיט

הסרטונים של CapCut שוקלים **840MB**. אסור לדחוף אותם.

- `assets/videos/` נמצא ב-`.gitignore`
- ביחידות: הטמעת YouTube **Unlisted**:
  ```html
  <iframe src="https://www.youtube-nocookie.com/embed/VIDEO_ID" allowfullscreen loading="lazy"></iframe>
  ```
- המקור נשאר ב-Google Drive / מקומי.

---

## 3. סדר ההעלאה — ⚠️ קודם דאטהבייס, אחר כך קוד

> **זה הלקח הכי חשוב במסמך הזה.**
>
> ב-16.7 דחפתי קוד ו-SQL יחד. Lovable העלה את הקוד — אבל **לא הריץ את ה-SQL**.
> הקוד ביקש עמודה שלא קיימת, וכל דף קורס באתר (כולל CyberPath עם תלמידים משלמים)
> החזיר "הקורס לא נמצא". **הקוד והסכימה עולים במסלולים נפרדים.**

**הסדר הבטוח:**

```
1. מריצים את ה-SQL ב-Lovable  ✅ ומאמתים שעבר
2. רק אחר כך דוחפים את הקוד ל-GitHub
```

ככה אין רגע שבו הקוד מחפש משהו שלא קיים. הפוך = השבתה.

---

## 4. שלבי ההעלאה בפועל

נניח קורס חדש עם slug בשם `myslug`.

### שלב א' — העתקת הקבצים

```bash
# מהתיקייה של הקורס, אל תוך הריפו של TeachPal
cp -r units   "C:/Users/shani/teachpal-zone/public/myslug-course/"
cp -r assets  "C:/Users/shani/teachpal-zone/public/myslug-course/"   # בלי videos!
```

בדקו שלא נכנסו סרטונים:
```bash
find "C:/Users/shani/teachpal-zone/public/myslug-course" -iname '*.mp4' | wc -l   # חייב 0
```

### שלב ב' — רישום הקורס ברג'יסטרי

`src/lib/self-paced-courses.ts` — מוסיפים ערך:

```ts
const MYSLUG: SelfPacedCourseDef = {
  slug: "myslug",
  completeMessage: "myslug-unit-complete",
  unitPath: (u) => `/myslug-course/units/${u.file}.html`,
  marketingUrl: "https://...",   // אופציונלי — בלי זה הכפתור "מידע נוסף" מוסתר
  // levels: [...]               // רק אם הקורס מגומיפיי (XP). CapCut בכוונה בלי.
  units: [
    { id: 0, file: "unit-00-orientation", xp: 0,   mins: 3, title: "...", desc: "..." },
    { id: 1, file: "unit-01-intro",       xp: 100, mins: 4, title: "...", desc: "..." },
  ],
};

export const SELF_PACED_COURSES = { cyber: CYBER, capcut: CAPCUT, myslug: MYSLUG };
```

- `file` = שם הקובץ בלי `.html`, וחייב להתאים ל-`data-unit` שבו.
- `xp` כאן הוא **מקור האמת** — מה שה-iframe מצהיר לא נסמך עליו (אבטחה).

### שלב ג' — קובץ ראוט

`src/routes/learn.myslug.$unitId.tsx` (העתק של הקיים, משנים רק את ה-slug):

```tsx
import { createFileRoute, useSearch } from "@tanstack/react-router";
import { AuthGate } from "@/components/AuthGate";
import { SelfPacedUnitPlayer } from "@/components/SelfPacedUnitPlayer";
import { z } from "zod";

export const Route = createFileRoute("/learn/myslug/$unitId")({
  head: () => ({ meta: [{ title: "יחידה — Z-School" }] }),
  validateSearch: z.object({ courseId: z.string().optional() }),
  component: () => (
    <AuthGate allow={["student", "teacher", "admin"]}>
      <Page />
    </AuthGate>
  ),
});

function Page() {
  const { unitId } = Route.useParams();
  const { courseId } = useSearch({ from: "/learn/myslug/$unitId" });
  return <SelfPacedUnitPlayer slug="myslug" unitId={unitId} courseId={courseId} />;
}
```

> למה קובץ נפרד לכל קורס ולא ראוט גנרי אחד? כי `/learn/$courseSlug/$unitId` היה מתנגש
> עם `/learn/$teacherId` הקיים. הקובץ הזה הוא קליפה בת 20 שורות — כל הלוגיקה משותפת
> ב-`SelfPacedUnitPlayer`.

### שלב ד' — ה-SQL (מריצים **לפני** הדחיפה)

הקורס צריך שורה בטבלת `courses` עם `content_slug`:

```sql
insert into public.courses (name, description, is_self_paced, content_slug, teacher_id)
select
  convert_from(decode('<base64 של השם>','base64'),'UTF8'),
  convert_from(decode('<base64 של התיאור>','base64'),'UTF8'),
  true,
  'myslug',
  (select p.id from public.profiles p where p.role = 'admin' order by p.created_at limit 1)
where not exists (select 1 from public.courses where content_slug = 'myslug');
```

**למה base64?** ראו סעיף 5.2 — עברית ב-SQL מתהפכת בהעתקה.

קישור תשלום (אם מוכרים אותו):
```sql
insert into public.course_payment_links (morning_link_id, course_id, price_shekels, active, note)
select '<המזהה מ-Morning>', c.id, null, true, 'myslug self-paced'
from public.courses c
where c.content_slug = 'myslug'
  and not exists (
    select 1 from public.course_payment_links l
    where l.course_id = c.id or l.morning_link_id = '<המזהה מ-Morning>'
  );
```

### שלב ה' — דחיפה

```bash
cd "C:/Users/shani/teachpal-zone"
git pull --rebase origin main    # ⚠️ Lovable דוחף ל-main כל הזמן — תמיד למשוך קודם
git add -A && git commit -m "..." && git push origin main
```

Lovable מזהה את הדחיפה ומעלה את הקוד. **אין צורך ללחוץ Publish** בשביל קוד שהגיע מגיט.

---

## 5. המלכודות — כולן נתקלנו בהן באמת

### 5.1 Lovable לא מריץ מיגרציות מגיט 🔴

קבצים ב-`supabase/migrations/` שנדחפים ב-GitHub **לא רצים**. Lovable מריץ רק מיגרציות שנוצרו דרך הממשק שלו.

**מה עושים:** מריצים את ה-SQL ידנית בכלי ה-SQL query של Lovable, ומאמתים.
**עדיין שומרים** את הקובץ ב-`supabase/migrations/` — כדי שהריפו ישקף את המציאות.

### 5.2 עברית ב-SQL מתהפכת בהעתקה 🔴

`'קראש קורס CapCut — עריכת וידאו',` הגיע לדאטהבייס בתור `,'ואדיו תכירע — CapCut סרוק שארק'` — הפוך, עם הפסיק בהתחלה. השגיאה שמתקבלת מטעה:

```
ERROR: 42601: syntax error at or near ","
```

**הסיבה:** ההעתקה תופסת את הסדר **החזותי** (RTL), לא הלוגי.
**הפתרון:** SQL שכולו ASCII. כל מחרוזת עברית מקודדת:
```sql
convert_from(decode('<base64>','base64'),'UTF8')
```
עברית **בתוך קובץ** זה בסדר גמור — הבעיה היא רק העתקה דרך צ'אט/טרמינל.

### 5.3 `CREATE OR REPLACE FUNCTION` לא יכול לשנות סוג החזרה

```
ERROR: 42P13: cannot change return type of existing function
```
**הפתרון:** `drop function if exists public.get_self_paced_courses();` לפני היצירה.

### 5.4 שגיאת שאילתה = "הקורס לא נמצא"

`course.$courseId.tsx` עושה:
```ts
const { data: c } = await supabase.from("courses").select("...");
if (!c) { setLoading(false); return; }   // ← כל שגיאה נראית כמו "אין קורס"
```
לכן **עמודה חסרה אחת מפילה את כל דפי הקורסים באתר**, לא רק את החדש. זה מה שקרה ב-16.7.

### 5.5 Lovable מריץ הכל בטרנזקציה אחת

הכל-או-כלום. שגיאה באמצע = הכל מתבטל, לא נשאר חצי מצב. זה **טוב** — אבל אומר שהסקריפט חייב לעבור במלואו בפעם אחת. אל תשימו `begin;/commit;` משלכם.

### 5.6 המזהה של Morning הוא **לא** הקוד הקצר של הקישור 🔴

הקוד ב-webhook משווה למזהה ש-Morning שולח (`data.paymentLink.id`). זה **UUID**, ולא הקוד הקצר שרואים ב-`mrng.to/XXXX`. הקוד הקצר הוא רק מקצר כתובות.

**איך משיגים את המזהה האמיתי — בלי רכישת טסט:** עוקבים אחרי ההפניה.

```bash
curl -sI https://mrng.to/iQMFOwRrY1
# → 302 Location: https://pages.greeninvoice.co.il/payments/links/bce01fe3-9007-49f8-aa76-8eac866a0588
#                                                                └── זה המזהה ל-DB ──┘
```

הטריק הזה חסך רכישת טסט ותפס טעות שהייתה **דוחה כל רכישה** של CapCut.

**בונוס — לרשום את שניהם:** הטבלה מרשה כמה קישורים לאותו קורס, אז אפשר להכניס גם את ה-UUID וגם את הקוד הקצר. זה עולה כלום ומבטל את הניחוש לגמרי.

אם בכל זאת טועים, זה נכשל **בקול** ובלוגים יופיע המזהה הנכון:
```
[morning-webhook] no active course mapping for payment link <המזהה האמיתי>
```
והתיקון:
```sql
update public.course_payment_links set morning_link_id = '<המזהה מהלוג>'
where note like 'myslug%';
```

### 5.7 Morning משבית webhook שצובר כשלונות 🔴

רוכשת שילמה (2.8, קוראים חכמים) — ושום דבר לא קרה: לא משתמש, לא שיוך, לא מייל.
ה-endpoint היה תקין והמיפוי היה קיים, אבל ב-Morning ה-webhook הופיע כ"לא פעיל" —
**Morning מכבה אוטומטית webhook שמחזיר שגיאות שוב ושוב**, ומאותו רגע אף רכישה
לא מדווחת. את הכשלונות ייצרו אירועים לא-רלוונטיים (מסמכים/חשבוניות בלי productId)
שקיבלו מאיתנו 400.

**התיקון (2.8):** כל אירוע לא-רלוונטי מקבל 200 ("התקבל, לא רלוונטי") עם לוג רועש —
רק שגיאות אמת (סוד שגוי, כשל DB) מחזירות לא-2xx. **ובכל זאת לבדוק אחרי כל השקת
קורס שה-webhook ב-Morning עדיין פעיל**, ואחרי טסט רכישה ראשון.

### 5.8 הריפו המקומי מפגר

Lovable דוחף ל-`main` כל הזמן. ב-16.7 הריפו המקומי היה **~100 קומיטים מאחור**. תמיד `git pull --rebase origin main` לפני עבודה.

### 5.9 הרצת האתר מקומית לא עובדת

Node מקומי הוא 20.16, ו-Vite דורש 20.19+. `npx vite build` ו-`npm run dev` נכשלים.
מה כן עובד: `npx tsc --noEmit` (בדיקת טיפוסים). הבנייה האמיתית קורית אצל Lovable.

---

## 6. בדיקות לפני שאומרים "סיימנו"

### 6.1 בדיקת החבילה (לפני העלאה)

מגישים את התיקייה ובודקים כל יחידה בדפדפן מול "מאזין" שמדמה את ה-LMS:

```bash
cd public/myslug-course/.. && python -m http.server 8899
```

לכל יחידה חייב להתקיים:
- [ ] `data-unit` תואם ל-`id` ברג'יסטרי
- [ ] יש כפתור `data-complete`
- [ ] הלחיצה שולחת `{type:"myslug-unit-complete", unitId:N}` עם המספר הנכון
- [ ] ה-iframe **לא** מנווט ליחידה הבאה
- [ ] הקישור "כל היחידות" מוסתר
- [ ] **גם יחידה 0** (אחרת הכל נעול)

### 6.2 אימות הדאטהבייס (אחרי ה-SQL)

השאילתה האחרונה בסקריפט צריכה להחזיר שורה לכל קורס:
```sql
select name, content_slug, is_self_paced from public.courses where content_slug is not null;
```

בדיקה חיצונית בלי סיסמאות (עם המפתח הציבורי מ-`.env`):
```
GET {SUPABASE_URL}/rest/v1/courses?select=content_slug&limit=1
```
- `200 []` → העמודה **קיימת** (רק RLS מסתיר שורות) ✅
- `400 42703 column does not exist` → **לא קיימת** ❌

### 6.3 אחרי העלייה לאוויר

- [ ] `https://campus.z-school.co.il/myslug-course/units/unit-01-....html` מחזיר 200
- [ ] הקורס מופיע ב-`/self-paced`
- [ ] **דף קורס קיים (CyberPath) עדיין נפתח** ← בדיקת הרגרסיה החשובה
- [ ] סיום יחידה מעדכן התקדמות ורושם XP

---

## 7. דברים שקל לשכוח

| מה | איפה |
|----|------|
| תמונת שער לקורס | `courses.cover_image_url` — בלעדיה הכרטיס מציג פס גרדיאנט |
| הקבצים ב-`public/` הם **עותק** | עריכה בתיקיית הקורס לא מסתנכרנת — צריך להעתיק שוב |
| כפתור "מידע נוסף" | מוצג רק אם יש `marketingUrl` ברג'יסטרי |
| XP או זמן | קורס עם `levels` מציג XP; בלי — מציג `mins` (דקות) |

---

## 8. קישורים

- **הריפו:** `C:\Users\shani\teachpal-zone` · github.com/ZSchool-contact/teachpal-zone
- **האתר:** https://campus.z-school.co.il
- **Supabase project:** `wulbsbyzkguulqppqwua`
- **קבצים מרכזיים:**
  - `src/lib/self-paced-courses.ts` — הרג'יסטרי (מוסיפים כאן קורס)
  - `src/components/SelfPacedUnitPlayer.tsx` — הנגן המשותף
  - `src/routes/course.$courseId.tsx` — דף הקורס (רשימת היחידות)
  - `src/routes/self-paced.tsx` — הקטלוג
  - `src/routes/api/public/morning-webhook.ts` — התשלומים

<!-- flowpad:capsule identity
version: 1
data:
  id: 42bb3e17-9478-4a01-a1f1-ea5fbd372b3f
flowpad:endcapsule identity -->
