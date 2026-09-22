# תמלול סרטונים ללומדה — כשאין קובץ כתוביות

> משלים את `CLAUDE.md`. הקובץ הזה = **איך משיגים תמלול** מסרטוני קורס כשאין קובץ כתוביות/תסריט.
>
> נכתב אחרי בניית CapCut (07.2026), שבו קיבלנו 9 סרטונים בלבד, בלי תמלול. כולל כל מה שנשבר בדרך.

---

## 0. מתי צריך את זה

יש סרטוני קורס (mp4), אבל **אין תסריט/כתוביות** לבנות מהם את התוכן. במקום לנחש מה נאמר בסרטון, מתמללים אותו מקומית ובונים את היחידה מהתמלול האמיתי (checklist, "שים לב", וכו' תואמים מילה-במילה למה שהלומד רואה).

**קודם בדקו אם יש כתוביות מובנות** — לפעמים חבל להריץ תמלול:
- קובץ נפרד (`.srt`/`.vtt`) ליד הסרטונים? מושלם, השתמשו בו.
- כתוביות מוטמעות **בקובץ** (soft subs)? בדקו:
  ```bash
  ffprobe -v error -select_streams s -show_entries stream=codec_name -of csv=p=0 video.mp4
  # פלט ריק = אין כתוביות מוטמעות → צריך לתמלל
  ```
- כתוביות **שרופות על התמונה** (פיקסלים)? לא ניתנות לחילוץ כטקסט בלי OCR → פשוט תמללו את האודיו.

---

## 1. השיטה: תמלול מקומי עם faster-whisper

speech-to-text מקומי, בעברית, עם מודל Whisper `large-v3`. הכל רץ על המחשב — שום דבר לא עולה לענן. איכות עברית מצוינת.

### 1.1 התקנה חד-פעמית

```bash
# ffmpeg (לחילוץ אודיו) — דרך winget
winget install --id Gyan.FFmpeg -e --accept-source-agreements --accept-package-agreements

# סביבת python ל-whisper (uv כבר מותקן במחשב)
uv venv "C:/whisper-env" --python 3.11
VIRTUAL_ENV="C:/whisper-env" uv pip install faster-whisper
```

המודל (`large-v3`, ~3GB) יורד **אוטומטית בפעם הראשונה** ונשמר במטמון ב-`~/.cache/huggingface`. הרצות הבאות לא מורידות שוב.

### 1.2 סקריפט התמלול

`transcribe_one.py` — מתמלל סרטון אחד ל-`.txt` (טקסט מלא) ול-`.srt` (עם חותמות זמן):

```python
# -*- coding: utf-8 -*-
import os, sys, time
VID = r"...\courses\<course>\assets\videos"     # תיקיית הסרטונים
OUT = r"...\courses\<course>\transcripts"        # תיקיית הפלט
slug = sys.argv[1]                               # למשל "01-install"
os.makedirs(OUT, exist_ok=True)

from faster_whisper import WhisperModel
model = WhisperModel("large-v3", device="cpu", compute_type="int8")

def fmt(s):
    h=int(s//3600); m=int((s%3600)//60); sec=s%60
    return f"{h:02d}:{m:02d}:{sec:06.3f}".replace(".", ",")

f = os.path.join(VID, slug + ".mp4")
segs, _ = model.transcribe(f, language="he", vad_filter=True,
                           beam_size=5, condition_on_previous_text=True)
txt, srt = [], []
for i, s in enumerate(segs, 1):
    t = s.text.strip(); txt.append(t)
    srt.append(f"{i}\n{fmt(s.start)} --> {fmt(s.end)}\n{t}\n")
    if i % 10 == 0: print(f"  ...{i} segments ({s.end:.0f}s)", flush=True)
open(os.path.join(OUT, slug+".txt"), "w", encoding="utf-8").write("\n".join(txt)+"\n")
open(os.path.join(OUT, slug+".srt"), "w", encoding="utf-8").write("\n".join(srt))
print(f"done: {slug}", flush=True)
```

### 1.3 הרצה

```bash
# ⚠️ חובה PYTHONUTF8=1 — אחרת הדפסת עברית מקריסה את הסקריפט על Windows (cp1252)
export PYTHONUTF8=1 PYTHONIOENCODING=utf-8
# ⚠️ ffmpeg חייב להיות ב-PATH (או להוסיף אותו לפני ההרצה)
"C:/whisper-env/Scripts/python.exe" -u transcribe_one.py 01-install
```

מריצים **סרטון-סרטון** (לא באצווה) — ראו מלכודת 3.2.

---

## 2. ⚠️ לוודא שהתוכן תואם לשם הקובץ — לא לדלג על זה

**הלקח הכי חשוב במסמך הזה.** ב-CapCut קובץ בשם "שיעור 3 אפקטים ומעברים" הכיל בפועל את **שיעור הטקסט** — הסרטונים היו ממוספרים לא נכון במקור. בנייה עיוורת לפי השם הייתה יוצרת יחידה שגויה לגמרי.

**לפני שבונים — "סריקת פתיחים":** מתמללים רק את **60 השניות הראשונות** של כל סרטון, כי כל שיעור פותח בהכרזה על הנושא ("עכשיו נלמד על אפקטים..."). ככה מזהים בוודאות מה באמת בכל קובץ.

```python
# intro_scan.py — מזהה נושא לפי הפתיחה
import os, glob, subprocess, tempfile
VID = r"...\assets\videos"; FFMPEG = r"...\ffmpeg.exe"
from faster_whisper import WhisperModel
model = WhisperModel("large-v3", device="cpu", compute_type="int8")
for mp4 in sorted(glob.glob(os.path.join(VID, "*.mp4"))):
    wav = os.path.join(tempfile.gettempdir(), "intro.wav")
    subprocess.run([FFMPEG,"-y","-t","60","-i",mp4,"-ar","16000","-ac","1",wav],
                   stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    segs,_ = model.transcribe(wav, language="he", beam_size=1)
    print(os.path.basename(mp4), "→", " ".join(s.text.strip() for s in segs)[:200])
```

מיפוי תוכן→קובץ, ורק אז ממספרים/משבצים את היחידות. אם סרטון חסר (הובטח בסילבוס ולא קיים) — לסמן ולבקש מהמשתמשת.

---

## 3. מלכודות — נתקלנו בכולן

### 3.1 עברית בפלט מקריסה על Windows 🔴
ברירת המחדל של הקונסולה היא cp1252. `print` של עברית זורק `UnicodeEncodeError`. **הפתרון:** `PYTHONUTF8=1` + `PYTHONIOENCODING=utf-8` לפני כל הרצה. (התמלול עצמו תמיד נכתב ל-UTF-8; הבעיה רק בהדפסת התקדמות.)

### 3.2 משימות רקע נהרגות — מריצים סרטון-סרטון 🔴
הרצת אצווה ארוכה (כל הסרטונים יחד) נהרגה שוב ושוב כשהמחשב נכנס לשינה / הסשן הושהה. **הפתרון:** מתמללים **סרטון אחד בכל פעם** (הרצה קצרה שמספיקה להסתיים), ושומרים כל תמלול מיד. הרצות בודדות שרדו; אצווה של 30+ דקות לא.

### 3.3 להזיז/לשנות שם קבצים באמצע ריצה = קריסה
אם משנים שם או מזיזים סרטון בזמן שהתמלול קורא אותו מהמיקום הישן — הוא נופל (`FileNotFoundError`). לסיים תמלול, ורק אז לסדר קבצים.

### 3.4 מצב מהיר כשצריך
אם התמלול איטי מדי (או ממשיך להיתקע), `beam_size=1` + `condition_on_previous_text=False` → פי ~3 מהיר, בירידת איכות זניחה לדיבור ברור. שימושי גם לסריקת פתיחים.

### 3.5 גדלים
9 סרטוני CapCut = ~840MB, בודדים עד 190MB. **הסרטונים לא נכנסים לגיט** (מגבלת 100MB של GitHub). מחריגים `assets/videos/` ב-`.gitignore`, ולהפצה מטמיעים ביוטיוב Unlisted (ראו `PUBLISHING-TO-TEACHPAL.md` §2.6).

---

## 4. מהתמלול ליחידה

התמלול הוא **חומר הגלם** של היחידה:
- ה-checklist בתרגול, ה"שים לב", והיעדים — נגזרים ישירות ממה שנאמר בסרטון (מונחים, סדר פעולות, טיפים).
- המסלול השני (מובייל, אם הסרטון בדסקטופ) הוא **שחזור** ולא מגובה בסרטון — לסמן זאת למשתמשת.
- שומרים את ה-`.txt` וה-`.srt` תחת `transcripts/` בתיקיית הקורס (טקסט קטן, כן נכנס לגיט — מתעד את המקור).

---

## 5. תקציר להרצה מהירה

```
1. בדוק אם יש .srt / כתוביות מוטמעות — אם כן, דלג על התמלול.
2. התקן (חד-פעמי): ffmpeg + faster-whisper (uv venv).
3. סריקת פתיחים — ודא שכל קובץ מכיל את מה ששמו מבטיח.
4. תמלל סרטון-סרטון (PYTHONUTF8=1, למקרה הצורך beam_size=1).
5. בנה כל יחידה מהתמלול; שמור transcripts/ בגיט, videos/ בהחרגה.
```
