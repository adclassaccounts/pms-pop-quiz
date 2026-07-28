# PMS Pop Quiz

Companion to the PMA quiz app, for the **PMS Training Plan** Notion database. Two files, no build step, no framework — same shape as the PMA app.

```
index.html            the whole app          -> GitHub repo, served by Vercel
quizzes.json          the question bank      -> GitHub repo, alongside index.html
apps-script-Code.gs   the backend            -> pasted into Apps Script by hand
```

Live at `pms-pop-quiz.vercel.app`. Backend is an Apps Script bound to the **PMS Quiz Results** sheet.

## Three traps that cost a whole afternoon

Read these before editing anything.

1. **Saving the Apps Script is not deploying it.** After any code change: **Deploy → Manage deployments → pencil on the existing deployment → Version: New version → Deploy.** Using "New deployment" instead issues a *different* `/exec` URL and leaves the app talking to the old code. `doGet` returns `SCRIPT_VERSION` for exactly this reason — hit the endpoint in a browser and read it back rather than assuming.
2. **Cmd+F in the Apps Script editor lies.** The editor only renders the visible portion of the file, so the browser's find silently misses anything off screen. Check code by scrolling to it, not by searching.
3. **The browser caches `index.html`.** After uploading to GitHub, load the app with a throwaway query string (`?v=7`) rather than a hard refresh. A cached page will keep posting to whatever endpoint it was built with, and the symptoms look identical to the backend being broken.

A fourth, smaller one: Google Sheets rewrites values it thinks are percentages. Writing the string `"88%"` becomes the number `0.88`; writing `100` into a column already formatted as a percentage renders `10000%`. The script stores a fraction and pins the cell format to `0%`.

## If you're rebuilding from scratch

1. New GitHub repo. `index.html` and `quizzes.json` in the root.
2. Import on Vercel. Framework preset **Other**, no build command, no output directory.
3. `CONFIG.googleClientId` in `index.html` is shared with the PMA app. Any new domain has to be added to that OAuth client's **Authorised JavaScript origins** in Google Cloud Console, or sign-in fails with `origin_mismatch`.
4. `CONFIG.endpoint` is the `/exec` URL of the deployment.

## The backend

PMS has **its own Apps Script and its own spreadsheet** (`apps-script-Code.gs`). The PMA script and sheet are untouched and the two share nothing, so neither can affect the other. Both can post to the same Slack channel if you want results landing together — that's just the webhook you give each one.

Setup is in the comment block at the top of `apps-script-Code.gs`. Short version: new Sheet → Extensions → Apps Script → paste → add `SLACK_WEBHOOK_URL` in Script Properties → Deploy as Web app, execute as Me, access Anyone → paste the `/exec` URL into `CONFIG.endpoint`.

The Slack webhook lives in **Script Properties**, never in the file, so it survives code updates and stays out of GitHub. Leave it unset and the app logs to the sheet and stays quiet.

Progress reads go via JSONP rather than `fetch`, because Apps Script blocks cross-origin browser GETs. Writes are a `POST` with `Content-Type: text/plain`, which avoids the CORS preflight Apps Script rejects. `record()` and `fetchProgress()` in `index.html` are the only two functions that talk to the backend.

`writeEnabled: false` runs the app browser-only — scoring, unlocking and retakes all work, nothing is sent or received. Useful for reviewing questions before a cohort starts.

Server progress wins over `localStorage` when it's available, so a trainee can't unlock days by editing devtools. The script returns each day's **best** attempt and never lets a later failed retake erase an earned pass.

### Why a separate script

Sharing PMA's script would have meant editing working code. Its rows are keyed on email + day, and **both plans have a Day 20** — PMA's written exam, PMS's Client Comms II — so a PMS submission could have overwritten a trainee's PMA exam record. Adding a `plan` column to fix that meant changing a live system for no real gain. A second script costs one spreadsheet.

### The contract

What the app sends on submit (`POST`, `Content-Type: text/plain` to dodge the CORS preflight Apps Script rejects):

```json
{
  "action": "submit",
  "plan": "PMS",
  "email": "...", "name": "...",
  "day": 17, "title": "Day 17 — Leadership I: ...",
  "score": 14, "total": 17, "pct": 82, "passed": true,
  "attempts": 2,
  "missed": [{ "n": 3, "tag": "Day 17 · Trust & Team" }],
  "podLeader": "Ryan", "podLeaderSlackId": "U03LBD9G39T",
  "at": "2026-07-28T16:00:00.000Z"
}
```

And on load: `GET {endpoint}?action=get&plan=PMS&email=…&callback=…`, replying `callback({ "progress": { "17": { "passed": true, "pct": 88, … } } })`.

### Sheet layout

Identical to the PMA sheet: `Timestamp, Email, Trainee, Day, Quiz, Score, Total, Percent, Passed, Pod Leader`, then one column per question. A correct answer writes `✓`; a wrong one writes `✗  chose: "…"   correct: "…"`. HTML in the option text is stripped before it reaches the sheet.

The header row extends itself with `Q12`, `Q13`, … the first time a longer quiz is submitted, so Day 22's 37 questions need no setup.

`Pod Leader` is always Matthew — PMS hires report to him, so there's no picker. His Slack ID lives in `CONFIG.manager` in `index.html`; `MENTION_POD_LEADER` at the top of the script switches between an @-mention and plain text.

`Percent` is stored as a fraction with the cell format pinned to `0%`. `doGet` derives the percentage from Score and Total regardless, so the column is for humans only.

Slack messages are for leadership, not the trainee — green bar on a pass, amber and "Below 80%" when short, missed questions listed by number and topic.

## How it behaves

Same as PMA: Google Sign-In, pod-leader picker, dashboard with Passed / Days / Up next, sequential unlock, locked screen, review screen with green/red and explanations, retakes. Pass mark is a single global `0.8`. Options are **not** shuffled, matching PMA.

Server progress wins over `localStorage`, so a trainee can't unlock days by editing devtools. Passing a day is sticky — a failed retake never takes an earned pass away.

Question types, all supported: `mcq` (`answer` = index), `multi` (`answer` = array of indices, exact match required), `fill` (`accept` = array of acceptable strings, matched case- and punctuation-insensitively). No `fill` questions are currently in use — the review pass cut or elevated all of them.

**Option order is shuffled on every attempt** (`CONFIG.shuffleOptions`). Question order never changes, so `Q7` in the sheet is always the same question on that day. Answers are stored and scored by canonical index and the sheet logs answer *text*, so nothing downstream depends on the order a trainee saw. The review screen redraws the order they actually got, and it's saved with the attempt so it survives a reload.

Because of the shuffle, the letters in the review spreadsheet and the leadership answer key won't match a trainee's screen. Reconcile by question number and answer text, not by A/B/C/D.

## Adding or changing questions

Edit `quizzes.json` and redeploy. Nothing else references the questions.

Each day has a `status`:

| status | On the dashboard | Gates progression |
|---|---|---|
| `live` | Openable | Yes |
| `pending` | "Coming soon", not openable | No |
| `noquiz` | "No quiz" | No |

Flip a day to `live` once it has enough questions. **Watch the pass mark when you do** — at 80%, a 1-question day means one wrong answer locks the trainee out, and a 2-question day means the same. Five or more is the sensible floor.

Right now only Days 17–20 are `live`, so the unlock chain runs 17 → 18 → 19 → 20 and a trainee starts at Day 17. That resolves itself as the earlier days get filled in.

`explain` renders as HTML, so `<b>` and `<i>` work. It shows on the review screen after submission.

## Where the current questions came from

- **Days 17–18** (29 questions) — the toggles on the Leadership I / Leadership II Notion pages, with answers and reasoning from the Leadership Quiz Answer Key. Option order is unchanged from Notion so Matthew's review notes still line up.
- **Days 19–22 and the scattered singles** — the "Suggested PMS Questions" tab of the quiz review sheet. That tab was written before the two leadership days were inserted at 17–18, so its Day 17 → 19, Day 18 → 20, Day 19 → 21, and "Day 20 (Exam)" → 22.
- **Days 1, 4, 6, 7, 12, 13** — empty. See below.

Two deliberate departures from the source material:

- Every question on that sheet tab had its correct answer as option A. Options were reordered once in `quizzes.json` so answers spread across A–D. Option sets and correct answers are otherwise untouched.
- Q29 was a written question. It's now multiple choice, testing whether the trainee recognises what a real answer looks like, and it points them at the written version in the debrief. The written one can't be auto-graded and is worth keeping.

The sheet's three "Leadership (future)" rows were not carried over — they're recall questions about the reading, and Days 17–18 now have 29 purpose-built ones.

## Still outstanding

**The 127 questions on the "Current Trainings" tab aren't in here.** That tab is a dump of the live PMA app, and it has no explanations column — the explanations only exist in the PMA app's source. Importing without them would strip the review screen of the thing that makes it useful. Fixing that needs the PMA repo.

Also worth knowing before that import: `Matthew's notes` is empty in all 127 rows, and your own `My read` column already marks a number of them `Cut/PMA-only` or `Reframe->client`.
