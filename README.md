# PMS Pop Quiz

Companion to the PMA quiz app, for the **PMS Training Plan** Notion database. Two files, no build step, no framework — same shape as the PMA app.

```
index.html      the whole app
quizzes.json    the question bank
```

## Deploy

1. New GitHub repo, e.g. `pms-pop-quiz`. Drop both files in the root.
2. Import it on Vercel. Framework preset: **Other**. No build command, no output directory.
3. Fill in the two placeholders at the top of `index.html`:

```js
googleClientId: "PASTE_PMA_GOOGLE_CLIENT_ID_HERE",
endpoint:       "PASTE_PMA_APPS_SCRIPT_EXEC_URL_HERE",
```

Both values are sitting in plain text at the top of the PMA app's `index.html` — copy them across. The client ID also needs the new Vercel domain added to its **Authorised JavaScript origins** in Google Cloud Console, or sign-in will fail with `origin_mismatch`.

## Reusing the PMA backend

The app posts to the same Apps Script the PMA quiz uses, but tags every request with `plan: "PMS"` so the two don't collide. **The Apps Script has to be updated to read that field**, otherwise PMS attempts will overwrite PMA progress for anyone who has done both, and Slack messages will say the wrong thing.

Two things to change in the Apps Script:

- **Progress lookup / write** — key rows on `plan + email + day`, not `email + day`.
- **Slack message** — say "PMS Training" rather than "PMA Training", and confirm the destination channel is the one you want PMS results in.

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

And on load: `GET {endpoint}?action=get&plan=PMS&email=...`, expecting `{ "progress": { "17": { "passed": true, "pct": 82, ... } } }`.

If the PMA script's field names differ, align `record()` and `fetchProgress()` in `index.html` — they're the only two functions that talk to the backend.

## How it behaves

Same as PMA: Google Sign-In, pod-leader picker, dashboard with Passed / Days / Up next, sequential unlock, locked screen, review screen with green/red and explanations, retakes. Pass mark is a single global `0.8`. Options are **not** shuffled, matching PMA.

Server progress wins over `localStorage`, so a trainee can't unlock days by editing devtools. Passing a day is sticky — a failed retake never takes an earned pass away.

Question types, all supported: `mcq` (`answer` = index), `multi` (`answer` = array of indices, exact match required), `fill` (`accept` = array of acceptable strings, matched case- and punctuation-insensitively).

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
