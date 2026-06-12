<div dir=rtl>

הקבצים ששונו

1. diarization.py

פוצל המתודה PyannoteDiarizationEngine.diarize לשלושה חלקים:


run_pipeline(audio, ...) — מריץ רק את pyannote pipeline על
אודיו גולמי (path או waveform), מחזיר diarization_df (טבלת
speaker turns). אין תלות ב-transcription_segments — ניתן
להריץ בכל זמן, כולל במקביל לתמלול.
assign_speakers(diarization_df, transcription_segments, ...)
— wrapper פומבי ל-_assign_speakers הקיים. שלב מיזוג זול
(pandas), מריץ אחרי ששני התהליכים סיימו.
diarize(...) — נשאר כ-wrapper לתאימות לאחור: קורא
ל-run_pipeline ואז assign_speakers, בדיוק כמו ההתנהגות
המקורית.


המנוע ivrit (clustering) לא שונה — הוא תלוי ב-transcription_segments
כקלט לתהליך ה-embedding שלו, ולכן חייב לרוץ ברצף אחרי התמלול.

2. audio.py

ב-StableWhisperModel.transcribe_core, בענף diarize=True:


נוסף import concurrent.futures.
אם diarization_args.engine == "pyannote" (ה-default הנוכחי):

האודיו נטען פעם אחת ל-waveform (utils.load_audio).
נפתח ThreadPoolExecutor בעל worker יחיד, ובו מורץ
PyannoteDiarizationEngine.run_pipeline במקביל ל-
model_object.transcribe (Whisper) שרץ ב-thread הראשי.
בסיום שני התהליכים, מתבצע assign_speakers למיזוג התוצאות.



אם engine == "ivrit" — ההתנהגות נשארה רציפה, כמו לפני השינוי
(קריאה ל-diarize_func הישן אחרי שכל הסגמנטים נאספו).
ה-executor נסגר תמיד (finally: executor.shutdown(wait=True)),
כך ששגיאות בכל אחד מהתהליכים נזרקות כצפוי ולא נשארים threads
תלויים.
