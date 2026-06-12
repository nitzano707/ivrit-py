<div dir=rtl>

  
# הרצה מקבילית של תמלול ודיאריזציה (pyannote)

## המטרה

מערכת התמלול רצה על RunPod Serverless GPU, עם חיוב לפי שניות עיבוד
(`actual_processing_seconds`). בתצורה המקורית, כאשר `diarize=True` עם
מנוע `pyannote`, התהליך רץ **באופן רציף**:

1. תמלול מלא (Whisper) — מפיק את כל הסגמנטים.
2. רק לאחר מכן — הרצת pyannote pipeline לדיאריזציה.
3. מיזוג (`assign_speakers`) של תוצאות הדיאריזציה עם הסגמנטים.

שלב 1 ושלב 2 הם שני תהליכי GPU **כבדים ובלתי-תלויים זה בזה** — pyannote
לא צריך את פלט ה-Whisper כדי לרוץ (רק שלב המיזוג הסופי, שהוא זול,
תלוי בשניהם). המטרה הייתה לזהות את אי-התלות הזו ולהריץ את שני
השלבים **במקביל**, כדי לקצר את `wall-clock time` ולכן את עלות
החיוב לפי שניה.

## מבנה הריפוזיטוריז

- **`nitzano707/ivrit-py`** (ענף `master`, fork של `ivrit-ai/ivrit-py`)
  — מכיל את הקבצים ששונו, `ivrit/audio.py` ו-`ivrit/diarization.py`.
  בריפו זה קיימים גם snapshots של הגרסה הקודמת (`audio311לפני
  מקביליות.py`, `diarization311לפניהקביליות.py`) לצורך השוואה.

- **`nitzano707/runpod-serverless-ENGLISH-HEBREW`** (ענף `main`) —
  מכיל את ה-`Dockerfile` שמתקין את `ivrit[all]` מ-
  `git+https://github.com/nitzano707/ivrit-py.git` (ראה שורה
  המתקינה את `pip3 install "ivrit[all] @ git+..."`). זהו ה-repo
  שממנו נבנה האימייג' שנפרס ל-RunPod.

## הקבצים ששונו

### 1. `diarization.py`

פוצל המתודה `PyannoteDiarizationEngine.diarize` לשלושה חלקים:

- **`run_pipeline(audio, ...)`** — מריץ רק את pyannote pipeline על
  אודיו גולמי (path או waveform), מחזיר `diarization_df` (טבלת
  speaker turns). **אין תלות ב-`transcription_segments`** — ניתן
  להריץ בכל זמן, כולל במקביל לתמלול.
- **`assign_speakers(diarization_df, transcription_segments, ...)`**
  — wrapper פומבי ל-`_assign_speakers` הקיים. שלב מיזוג זול
  (pandas), מריץ אחרי ששני התהליכים סיימו.
- **`diarize(...)`** — נשאר כ-wrapper לתאימות לאחור: קורא
  ל-`run_pipeline` ואז `assign_speakers`, בדיוק כמו ההתנהגות
  המקורית.

המנוע `ivrit` (clustering) **לא שונה** — הוא תלוי ב-`transcription_segments`
כקלט לתהליך ה-embedding שלו, ולכן חייב לרוץ ברצף אחרי התמלול.

### 2. `audio.py`

ב-`StableWhisperModel.transcribe_core`, בענף `diarize=True`:

- נוסף `import concurrent.futures`.
- אם `diarization_args.engine == "pyannote"` (ה-default הנוכחי):
  - האודיו נטען פעם אחת ל-waveform (`utils.load_audio`).
  - נפתח `ThreadPoolExecutor` בעל worker יחיד, ובו מורץ
    `PyannoteDiarizationEngine.run_pipeline` **במקביל** ל-
    `model_object.transcribe` (Whisper) שרץ ב-thread הראשי.
  - בסיום שני התהליכים, מתבצע `assign_speakers` למיזוג התוצאות.
- אם `engine == "ivrit"` — ההתנהגות נשארה **רציפה**, כמו לפני השינוי
  (קריאה ל-`diarize_func` הישן אחרי שכל הסגמנטים נאספו).
- ה-executor נסגר תמיד (`finally: executor.shutdown(wait=True)`),
  כך ששגיאות בכל אחד מהתהליכים נזרקות כצפוי ולא נשארים threads
  תלויים.

## תוצאות מדידה (לפני / אחרי, אותו GPU — RTX A4500)

| קובץ | אורך אודיו | לפני (s) | אחרי (s) | שיפור |
|---|---|---|---|---|
| 3 ראיונות | 54:21 | 208.69 | 162.68 | ~22% |
| סגל הטרוגני | 1:14:34 | 276.99 | 209.59 | ~24.3% |

בכל הריצות שנמדדו אחרי השינוי, יחס המהירות (audio/processing) נע
בין **×18.8 ל-×22.4**, ועלה ככל שהאודיו ארוך יותר — לעומת
**×15.6–16.2** לפני השינוי. איכות הדיאריזציה (שיעור segments עם
`speaker: null`) נשארה זניחה (0.06%–0.5%) ולא הושפעה לרעה.

## הערה פתוחה — נוסחת ההערכה המקדימה

`estimated_processing_seconds` במערכת עדיין מבוסס על יחס ישן
(`~0.080` שניות עיבוד לשנייית אודיו), שלא מעודכן לאחר השינוי.
היחס בפועל לאחר המקביליות הוא **~0.045–0.062**. מומלץ לעדכן את
נוסחת האומדן כך שתשקף את הביצועים המדודים, כדי שההערכה המוצגת
למשתמש לפני הרצה לא תהיה כפולה מהעלות בפועל.

## ניסוי שבוצע ולא אומץ — חיסכון בדקודינג הכפול (FFMPEG)

ההורדה של קובץ האודיו (מ-URL/blob) מתבצעת **פעם אחת בלבד** —
אין כפילות הורדת רשת. עם זאת, נוסף קריאה ל-`utils.load_audio`
שמריצה ffmpeg subprocess כדי להפיק waveform מהקובץ המקומי בשביל
pyannote, ו-Whisper מבצע פענוח/דקודינג דומה משלו, internally, על
אותו קובץ — כלומר יש **דקודינג כפול ל-waveform**.

**נוסה תיקון**: ב-`audio.py`, העברת ה-`audio_waveform` שכבר נטען
(עבור pyannote) גם ל-`model_object.transcribe`, במקום `audio_path`,
כך ש-Whisper לא יפענח את הקובץ בשנית (`transcribe_input =
audio_waveform if run_pyannote_concurrently else audio_path`).
התיקון נשען על תיעוד stable-whisper, שמקבל
`Union[str, np.ndarray, torch.Tensor, ...]` עם waveform ב-16kHz.

**תוצאות מדידה (אותו GPU — A4500, warm-to-warm):**

| קובץ | אורך אודיו | מקביליות בלבד (s) | +FFMPEG-fix (s) | שינוי |
|---|---|---|---|---|
| 3 ראיונות | 54:21 | 162.68 | 165.28 | **+1.6%** (גרוע יותר) |
| זום ציבורי | 1:45:34 | 283.34 | ~265.4* | **−6.3%** (טוב יותר) |

\* מוערך מהלוג אחרי קיזוז ~16s של cold-start model loading; לא
מספר רשמי מ-RunPod.

**המסקנה**: התוצאה **לא עקבית** — בקובץ אחד שיפור, בשני הרעה,
בטווח קטן (±1.6%–6.3%) שדומה לסדר הגודל של רעש בין-ריצות. בהינתן
שהפוטנציאל התיאורטי (ביטול דקודינג כפול) הוא קטן מיסודו (כ-1-15
שניות לקובץ ארוך), והסיכון (שינוי התנהגות `stable-whisper` עם
input מסוג `np.ndarray` לעומת `str`, כולל שינוי קל במספר
ה-segments שנצפה — 1593→1627, 2155→2162) — **התיקון לא אומץ
לפרודקשן**. הקוד נשאר עם `audio_path` (לא `audio_waveform`)
מועבר ל-`transcribe`, כמתואר בסעיף `audio.py` למעלה.

אם בעתיד יהיה זמן לבדיקות נוספות (כמה ריצות warm חזרתיות, על
קבצים שונים), אפשר לשקול שוב — אך זה לא ה-bottleneck המרכזי; עיקר
החיסכון (~20-25%) מגיע מהמקביליות שכן אומצה.
