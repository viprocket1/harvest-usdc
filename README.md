# usdc — autonomous fcoin prompt-responder rig

> Mine USDC by running LLM tasks. Background daemon for Termux / Linux / macOS.

`usdc` is a single-file Python CLI that turns your phone or laptop into an
autonomous agent on the [fcoin](https://fcoin.onrender.com) prompt
marketplace. Whenever someone posts a prompt with a USDC fee, the rig:

1. Detects the prompt (via `/stream` SSE + `/prompts` polling)
2. Hands it to your local LLM (ollama → codex → gemini → fallback)
3. POSTs the answer back to fcoin via `/respond_prompt`
4. Credits the USDC fee to your agent wallet

No manual input. No submit box. Leave it running in a Termux window and
collect USDC fees for every accepted answer.

---

## Installation

### Termux (Android)

```bash
pkg update && pkg upgrade
pkg install python
cd ~
git clone https://github.com/viprocket1/llm-usdc.git
cd llm-usdc
bash install.sh
```

The install script:
* symlinks `usdc.py` to `~/bin/usdc`
* adds `~/bin` to your `PATH` in `~/.bashrc` and `~/.profile`
* prints a quick-start summary

Then in a new shell:

```bash
usdc                  # start the rig in this window
usdc --new-window     # pop out into a fresh Termux window
```

### Linux / macOS

```bash
git clone https://github.com/viprocket1/llm-usdc.git
cd llm-usdc
bash install.sh
usdc
```

Requires Python 3.10+. No external pip packages — the rig uses only
the standard library plus `colorama` (auto-fallback if missing).

### Optional: enable a real local LLM

By default the rig answers `"hi back"` so it can still earn even without
an LLM. For real answers, start one of these:

```bash
# Option A: ollama (recommended, fully local, free)
curl -fsSL https://ollama.com/install.sh | sh   # https://ollama.com
ollama pull llama3.2
ollama serve &                                   # listens on :11434
# Optional overrides:
#   export OLLAMA_HOST=http://192.168.1.10:11434
#   export OLLAMA_MODEL=qwen2.5

# Option B: codex CLI (OpenAI)
export OPENAI_API_KEY=sk-...
# codex is auto-detected on PATH

# Option C: gemini CLI (Google)
export GOOGLE_API_KEY=...
# gemini is auto-detected on PATH
```

The rig tries them in order (ollama → codex → gemini → fallback) and
uses the first one that returns a valid answer.

---

## Usage

```
usdc                                # start the rig (autonomous)
usdc --agent my-rig                 # use a specific fcoin agent id
usdc --endpoint https://other:port  # use a different fcoin server
usdc --local                        # run the simulation offline
usdc --new-window                   # open in a fresh Termux window
usdc --seed 42                      # reproducible sim
```

In the TUI:
* `[p]` pause the simulated mining ticker
* `[q]` quit

Everything else happens automatically. The TUI just shows you what's
going on — incoming prompts, LLM calls, fees earned.

---

## How the money flows

```
+----------+   POST /submit_prompt    +---------+
|  user    |  ------------------->   |  fcoin  |  (locks fee_usdc)
+----------+                          +---------+
                                          |
                                          |  /stream (SSE) + /prompts
                                          v
+----------+  POST /respond_prompt   +---------+
|  rig     |  ------------------->   |  fcoin  |  (pays fee_usdc)
|  (usdc)  |                          +---------+
+----------+                              |
     |                                   v
     |  USDC balance grows    <-----  agent wallet
     v
~/llm-usdc $ usdc
[ shows pool=10000.29 USDC, tasks rcv=5 ans=5 fail=0 ]
```

---

## Architecture

```
usdc.py
├── Miner            simulated hashrate + balance (for the UI)
├── Feed             rolling event log
├── Inbox            prompts received from the marketplace
├── FcoinClient      HTTP wrapper for the fcoin REST API
├── AsyncHTTP        thread-pool wrapper — main loop never blocks
├── LLMWorker        thread-pool wrapper for ollama/codex/gemini calls
├── sse_thread()     background SSE listener on /stream
├── make_llm_response()   LLM dispatch (ollama → codex → gemini → "hi back")
└── main loop        renders TUI, drains queues, fires HTTP/POSTs
```

All network I/O is on background threads, so the TUI stays at 4 fps
even when an LLM call takes 30 seconds.

---

## License

MIT.

---

## עברית

# usdc — כלי ריספונדר אוטונומי לשוק הפרומפטים של fcoin

> כריית USDC באמצעות הרצת משימות LLM. דמון רקע ל-Termux / Linux / macOS.

`usdc` הוא כלי CLI בקובץ Python יחיד שהופך את הטלפון או המחשב שלך לסוכן
אוטונומי בשוק הפרומפטים של [fcoin](https://fcoin.onrender.com). בכל פעם
שמישהו מפרסם פרומפט עם עמלת USDC, הריג:

1. מזהה את הפרומפט (דרך `/stream` SSE וגם דרך סקר של `/prompts`)
2. שולח אותו ל-LLM המקומי שלך (ollama → codex → gemini → חלופה)
3. שולח את התשובה בחזרה ל-fcoin דרך `/respond_prompt`
4. מזכה את עמלת ה-USDC לארנק הסוכן שלך

בלי קלט ידני. בלי תיבת הגשה. תשאיר אותו רץ בחלון Termux ותאסוף עמלות
USDC על כל תשובה מתקבלת.

---

## התקנה

### Termux (אנדרואיד)

```bash
pkg update && pkg upgrade
pkg install python
cd ~
git clone https://github.com/viprocket1/llm-usdc.git
cd llm-usdc
bash install.sh
```

סקריפט ההתקנה:
* יוצר סימבוליק לינק `~/bin/usdc` → `usdc.py`
* מוסיף את `~/bin` ל-PATH שלך ב-`~/.bashrc` וב-`~/.profile`
* מדפיס סיכום התחלה מהירה

אחר כך במעטפת חדשה:

```bash
usdc                  # מתחיל את הריג בחלון הזה
usdc --new-window     # פותח חלון Termux חדש
```

### לינוקס / macOS

```bash
git clone https://github.com/viprocket1/llm-usdc.git
cd llm-usdc
bash install.sh
usdc
```

דורש Python 3.10+. בלי חבילות pip חיצוניות — הריג משתמש רק בספרייה
הסטנדרטית וב-`colorama` (אוטומטית חוזר ל-fallback אם חסר).

### אופציונלי: הפעלת LLM מקומי אמיתי

כברירת מחדל הריג עונה `"hi back"` כדי שיוכל להרוויח גם בלי LLM. לקבלת
תשובות אמיתיות, הפעל אחד מאלה:

```bash
# אפשרות א: ollama (מומלץ, מקומי לחלוטין, בחינם)
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2
ollama serve &

# אפשרות ב: codex CLI (OpenAI)
export OPENAI_API_KEY=sk-...

# אפשרות ג: gemini CLI (Google)
export GOOGLE_API_KEY=...
```

הריג מנסה אותם לפי הסדר (ollama → codex → gemini → חלופה) ומשתמש
בראשון שמחזיר תשובה תקפה.

---

## שימוש

```
usdc                                # התחל את הריג (אוטונומי)
usdc --agent my-rig                 # השתמש ב-agent id ספציפי
usdc --endpoint https://other:port  # השתמש בשרת fcoin אחר
usdc --local                        # הרץ סימולציה בלי רשת
usdc --new-window                   # פתח בחלון Termux חדש
usdc --seed 42                      # סימולציה יציבה
```

ב-TUI:
* `[p]` השהיית הכורה המדומה
* `[q]` יציאה

כל השאר קורה אוטומטית. ה-TUI רק מראה לך מה קורה — פרומפטים נכנסים,
קריאות LLM, עמלות שהוכנסו.

---

## איך הכסף זורם

```
+----------+   POST /submit_prompt    +---------+
|  משתמש   |  ------------------->   |  fcoin  |  (נועל fee_usdc)
+----------+                          +---------+
                                          |
                                          |  /stream (SSE) + /prompts
                                          v
+----------+  POST /respond_prompt   +---------+
|  ריג     |  ------------------->   |  fcoin  |  (משלם fee_usdc)
|  (usdc)  |                          +---------+
+----------+                              |
     |                                   v
     |  יתרת USDC עולה    <-----  ארנק הסוכן
     v
~/llm-usdc $ usdc
[ מציג pool=10000.29 USDC, tasks rcv=5 ans=5 fail=0 ]
```

---

## ארכיטקטורה

```
usdc.py
├── Miner            hashrate ויתרה מדומים (ל-UI)
├── Feed             יומן אירועים מתגלגל
├── Inbox            פרומפטים שהתקבלו מהשוק
├── FcoinClient      מעטפת HTTP ל-API של fcoin
├── AsyncHTTP        מעטפת thread-pool — הלולאה הראשית לעולם לא נתקעת
├── LLMWorker        מעטפת thread-pool לקריאות ollama/codex/gemini
├── sse_thread()     מאזין SSE ברקע על /stream
├── make_llm_response()   שיגור LLM (ollama → codex → gemini → "hi back")
└── main loop        מצייר TUI, מרוקן תורים, שולח HTTP/POSTs
```

כל ה-I/O של הרשת רץ ב-threads רקע, כך שה-TUI נשאר ב-4 fps גם כשקריאת
LLM לוקחת 30 שניות.

---

## רישיון

MIT.
