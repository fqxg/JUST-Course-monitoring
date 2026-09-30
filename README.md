# JUST Course Seat Monitor (Telegram bot)

Watches course sections on the JUST Course Schedule and messages you when a section goes from
**0 seats to 1+ seats**. Multi-user, SQLite-backed, no notification spam.

---
## 1. What the JUST website actually does (investigated, not assumed)

Page: `https://services.just.edu.jo/CourseSchedule/Default_En.aspx` (Arabic: `/CourseSchedule/`).
It is a classic **ASP.NET WebForms** page: one `<form id="aspnetForm" method="post" action="./Default_En.aspx">`.
There is **no JSON API**; everything is full-page postbacks. No login, but a session cookie
(`ASP.NET_SessionId`) and the F5 cookie `TS0120d50a` are set on the first GET, so a cookie jar is required.

| Question | Finding |
|---|---|
| Semester | `<select name="ctl00$contentPH$ddlSemester">`, values like `202720261`. **The site pre-selects the current semester** (`selected`), so the bot reads it - nothing is hard-coded. Changing it triggers `__doPostBack`. |
| Faculty | `ctl00$contentPH$ddlFaculty`, internal ids e.g. `70` = Computer & Information Technology, `20` = Engineering, `90` = Science & Arts. `onchange` does a postback which fills the department list. |
| Department | `ctl00$contentPH$ddlDept`, ids e.g. `173` = Computer Science, `176` = Software Engineering, `175` = Network Engineering And Security. Only populated after the faculty postback. |
| Section Status | `ctl00$contentPH$ddlSectionStatus`: `-1` = **All**, `1` = Opened, `2` = Closed. The bot always sends `-1`. |
| Search | POST to `./Default_En.aspx` with `__VIEWSTATE`, `__EVENTVALIDATION`, `__VIEWSTATEGENERATOR`, the four select values and the submit button `ctl00$contentPH$btnSubmit=Submit`. |
| Results | One big table `#ctl00_contentPH_gvSchedule` = **every course of the department**. Each course row holds a nested table `gvSections` (one row per section). |

**Course identifier.** JUST's unique key is the 7-digit **Line Number** (e.g. `1752020`), shown with a
**Course Symbol** (`NES202`). The bot accepts either in `/add`. (The spec's example `907592` matches neither
format on the pages I saw - use whatever the Course Schedule shows for your course.)

**Section columns** (verified against a real page; the meaning of the three numeric columns is *not* obvious):

| Column | Meaning |
|---|---|
| `الشعبة` | section number |
| `Seat Count` | **physical seats of the hall** - NOT enrollment (values like 1200 for online) |
| `Capacity` | enrollment cap of the section |
| `Registered` | students registered |
| `Status` | text is only `Active` / `Cancelled` |
| first-cell colour | Green = Opened, Yellow = Closed, Red = Cancelled (the site's own legend) |

`remaining = capacity - registered` (clamped at 0: sections can be over-booked, e.g. 40 cap / 42 registered).
**Cancelled sections are always 0** - they often show `registered = 0`, which would otherwise look like free seats.
On the 45 real sections checked, this rule agreed 100 % with the website's own Green/Yellow colouring.

### The important limitation: Cloudflare Turnstile
The Submit button is protected by **Cloudflare Turnstile** (`<div class="cf-turnstile" ...>`). Submitting
without a valid token returns *"The user couldn't be verified."* and no results. Turnstile exists to stop
automated clients, and this project deliberately **does not** try to defeat it (no CAPTCHA solvers, no stealth
tricks). Metadata (semester / faculties / departments) is served without it, so that part is live.

### Data source (`RESULTS_SOURCE`)
| Mode | How it works |
|---|---|
| `snapshot` (default) | An **admin** opens the JUST page in their own browser, picks the faculty/department, solves the check normally, presses Submit, saves the page (Ctrl+S → "Webpage, HTML only") and **sends the .html file to the bot**. The bot validates it, stores it and checks everyone's courses against it immediately. Monitoring runs every `CHECK_INTERVAL` against the newest snapshot, so alerts are only as fresh as the last upload (notifications state "Data as of HH:MM"). |
| `live` | Replays the browser's postbacks. It works only if JUST stops enforcing Turnstile or gives you access another way; otherwise checks report "JUST asked for human verification". |
| `auto` | Try live, fall back to snapshots, 30-minute cool-down after a rejection. |

For true 24/7 automatic monitoring you need JUST's permission/an endpoint - ask the university IT / registrar
office for an approved feed or allow-listed access. The client is isolated (`just/sources.py`), so a new source
is one small class implementing `fetch(semester, faculty, department) -> ResultsPage`.

---
## 2. Architecture
```
main.py                  wiring + APScheduler job (python-telegram-bot JobQueue)
config.py                .env settings
just/    errors.py models.py parser.py sources.py client.py     <- knows JUST, knows nothing about Telegram
database/ database.py models.py                                  <- SQLite
scheduler/monitor.py     one fetch per (faculty, department); transition logic; per-course error isolation
bot/     handlers.py conversations.py notifications.py parsing.py
tests/   70 tests, all with mocked JUST pages (no network)
```
The client API (async):
```python
sections = await just_client.get_course_sections(faculty="70", department="173", course_id="CS207")
# -> [Section(course_id, course_symbol, course_name, section, capacity, registered, ... .available)]
```
One fetch serves **all** monitors of a department, so load on JUST does not grow with the number of users.

## 3. Telegram commands
`/start` `/help` · `/add` (guided: Faculty → Department → Course → All/Specific section, with inline buttons) ·
`/add COURSE FACULTY DEPARTMENT [SECTION]` · `/remove [COURSE] [SECTION]` (`all` = the all-sections entry) ·
`/list` · `/check` (your courses only) · `/status` · `/faculties` · `/departments [FACULTY]` · `/cancel`.

One-line `/add` copes with spaces in names (quoted or not), internal ids, and the alias `cit`:
```
/add NES202 "Computer & Information Technology" "Network Engineering And Security"      # all sections
/add 1752020 CIT Network Engineering and Security 1                                       # only section 1
/add NES202 70 175                                                                        # internal ids work too
```
If you type a department where the faculty belongs (`/add X Computer Science Computer Science`), the bot infers
the faculty when the department name is unique university-wide and shows what it picked.

## 4. Notification rules (no spam)
State per section is stored in SQLite (`section_states`: `available`, `previous_available`, …).
* `0 → >0` **always** notifies. `>0 → same`, `>0 → 0`, `0 → 0` never notify.
* `NOTIFY_ON_SEAT_CHANGE=true` additionally reports `2 → 1` style changes.
* `/add` records a baseline, so a course that already has seats does not alert immediately.
* A section that appears later with seats (baseline already exists) counts as `0 → >0`.
* If Telegram delivery fails, the state is **not** advanced, so the alert is retried next cycle.
* Every alert is addressed only to the chat id of the user who owns that monitor; `/check` and `/remove` only touch the caller's rows.

## 5. Local setup
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env         # put your @BotFather token in TELEGRAM_BOT_TOKEN, your id in ADMIN_USER_IDS
python main.py
pytest -q                    # run the tests
```
Find your Telegram id by messaging @userinfobot. Then: send the bot a saved results page (.html) as a file,
and `/add` your courses.

## 6. 24/7 deployment
**systemd (any Linux VPS / Raspberry Pi)**
```bash
sudo useradd -r -m justbot
sudo cp -r . /opt/just-course-bot && sudo chown -R justbot: /opt/just-course-bot
cd /opt/just-course-bot && sudo -u justbot python3 -m venv .venv && sudo -u justbot .venv/bin/pip install -r requirements.txt
sudo -u justbot cp .env.example .env && sudo -u justbot nano .env
sudo cp just-course-bot.service /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now just-course-bot
journalctl -u just-course-bot -f
```
**Docker**
```bash
docker build -t just-course-bot .
docker run -d --name just-bot --restart unless-stopped --env-file .env -v just-bot-data:/app/data just-course-bot
```
Keep `data/` (SQLite + snapshots) on a persistent volume. Back it up by copying `data/bot.sqlite3`.

## 7. Error handling
Invalid course / faculty / department, course with no sections, non-existent section, duplicate monitor,
removing something not monitored, website down or timeout (stale metadata cache is used if the site is
temporarily unreachable), Turnstile rejection, missing snapshot, unexpected page structure (`ParseError`
with the missing column named) - each produces a clear message. A failure in one department or course never
stops the others, and the scheduler job itself is wrapped so it survives any exception.

## 8. Security notes
* Snapshots are shared by all users, so **only `ADMIN_USER_IDS` may upload** them (otherwise anyone could fake seat data). Uploads are parsed, never executed.
* Token comes from the environment only; httpx URL logging is silenced.
* Be polite to JUST: keep `CHECK_INTERVAL` ≥ 120 s when using `live`/`auto`.
