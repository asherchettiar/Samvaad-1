# Samvaad — Society Complaint Triage

**ZEPHYR ’26 · Vibe Coding · Statement 02 — Housing Societies & Civic Tech**

Samvaad turns the messy flood of resident complaints into a small, calm, prioritised board.
Residents type (or speak) one message in English, Hindi or Hinglish — **no forms, no categories to pick**.
The AI reads it, decides **what** it is and **how urgent** it is, writes a one-line summary, and
**merges it with the issue it belongs to** so the committee sees *one* problem instead of fifteen messages.

Built for the real constraint in the statement: **~100 flats, mixed English/Hindi, and committee volunteers
who have only a few minutes a day.**

---

## Table of contents

1. [What it does](#what-it-does)
2. [Quick start (2 minutes, no build step)](#quick-start)
3. [Getting a free Groq API key](#getting-a-free-groq-api-key)
4. [Optional: switch on Firebase realtime sync](#optional-switch-on-firebase-realtime-sync)
5. [Feature tour](#feature-tour)
6. [How the AI works](#how-the-ai-works)
7. [Data model & storage modes](#data-model--storage-modes)
8. [Project structure](#project-structure)
9. [Tips](#tips)
10. [Tests](#tests)
11. [Troubleshooting](#troubleshooting)
12. [Design decisions & edge cases](#design-decisions--edge-cases)
13. [How this maps to the judging criteria](#how-this-maps-to-the-judging-criteria)

---

## What it does

| Screen | Who it is for | What happens |
|---|---|---|
| `index.html` | Residents | One message box (type or tap the mic). The complaint is AI-triaged, merged with duplicates, and the resident gets a live status tracker. |
| `dashboard.html` | Committee volunteers | A 4-column triage board (**New → Assigned → In Progress → Resolved**) where every card is an AI-clustered issue, sorted by urgency and age, with AI-drafted replies ready to send on WhatsApp. |

Everything runs in the browser. There is **no backend to deploy, no build step and nothing to install**.

---

## Quick start

**Prerequisites:** any modern browser (Chrome, Edge, Firefox). Nothing else is required — not even Node.js.

### Option A — just open it (works offline)

1. Open the folder.
2. Double-click **`index.html`**.
3. Click **Committee** in the top bar, enter the committee password (**`zephyr26`** by default, set in
   `js/config.js` → `committee.password`), then **Load demo data** to see a fully populated board immediately.

That's it. Without an API key the app runs in **rules mode**: everything works, but triage uses the built-in
keyword/urgency engine instead of the AI.

### Option B — with the AI switched on (recommended)

1. Get a free Groq key (30 seconds, no card) — see the next section.
2. Copy **`js/config.example.js`** to **`js/config.js`** and paste the key into `groq.apiKey`.
   `js/config.js` is already listed in `.gitignore`, so it can never be committed by accident.
   Visitors are never asked for a key — the app has no key screen; the key lives only in this one file.
3. Send a complaint from `index.html` in another tab — it appears on the board, already triaged, ifn about a second.

### Option C — serve it locally (needed for the mic + two-tab realtime demo)

Any static server works. For example, with the VS Code **Live Server** extension: right-click `index.html` → *Open with Live Server*.
Or with Python: `python -m http.server 5500` in this folder, then open `http://localhost:5500`.

---

## Getting a free Groq API key

1. Go to **https://console.groq.com/keys** and sign in (free, no credit card).
2. Click **Create API key**, copy the `gsk_...` value.
3. Paste it into `js/config.js` → `groq.apiKey` and reload the page.

The project uses **`openai/gpt-oss-20b`** — a production model on Groq's free tier that returned
**~950 ms** per triage during testing. If a model is unavailable on your account, the client automatically
retries down a chain (`openai/gpt-oss-20b` → `qwen/qwen3.8-27b` → `openai/gpt-oss-120b`) before falling
back to the rules engine, so the demo never breaks.


---

## Optional: switch on Firebase realtime sync

Out of the box the board stores data in **localStorage** (per browser, survives refresh, and syncs live
between tabs of the same browser). No account needed.

To make it truly multi-device — *resident submits on a phone, the committee board updates instantly* — add a
free Firebase project:

1. **Firebase Console → Create project** (Spark / free plan, no card).
2. **Build → Firestore Database → Create database** → start in **test mode** → pick a region.
3. **Project settings → Your apps → Web (`</>`)** → register an app → copy the `firebaseConfig` object.
4. Paste those 6 values into `js/config.js` → `firebase: { ... }`.
5. Reload. The badge in the toolbar changes from *Local storage* to **Firebase live sync**.

**Firestore security rules for a demo** (Firestore → Rules):

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /samvaad_complaints/{doc} { allow read, write: if true; }
    match /samvaad_issues/{doc}      { allow read, write: if true; }
  }
}
```

> These rules are deliberately open so a judge can run the demo in one click. For real deployment you would add
> Firebase Auth and restrict writes to committee members. Collections used: `samvaad_complaints`, `samvaad_issues`.

If Firebase is unreachable or not configured, the app silently falls back to localStorage — it never breaks.

---

## Feature tour

### Resident portal (`index.html`)

* **One box, no forms.** Type it the way you speak: *"3 din se paani nahi aa raha, tap bilkul sookha hai"*.
* **Voice input** via the browser's free Web Speech API (`hi-IN`) — for residents who type slowly or not at all.
* **Example chips** that fill the box in one tap, so nobody stares at an empty field.
* **Instant receipt:** the AI's category, urgency and one-line summary, a **tracking id**, and a live
  **New → Assigned → In Progress → Resolved** progress bar.
* **"N residents reported this"** — the resident immediately sees they are not alone, which is exactly the
  reassurance that usually never comes.
* **My complaints:** enter a flat number and see every message you sent plus its current status.
* Identity (flat / name / phone) is remembered locally, so the next complaint takes two taps.

### Committee board (`dashboard.html`)

* **Kanban triage board** — one card per **issue** (not per message), sorted by urgency and age.
* **Drag a card between columns** to change status, or use the one-tap button
  (`Assign` → `Start work` → `Mark resolved`), with **Reopen** for mistakes.
* **Live stat tiles:** open issues, critical count, resolved today, oldest open issue, and
  **AI accuracy** computed from the corrections the committee makes.
* **Duplicate merging:** a card shows `4 residents`, the flats involved, and the merged messages inside.
* **"Needs clarity" flag:** when a message is too vague to act on (`"problem hai"`), the AI proposes the exact
  question to ask — one tap to copy it or open WhatsApp.
* **AI reply drafting:** choose Hinglish / हिन्दी / English, regenerate, edit, **copy**, or **Send on WhatsApp**
  (opens `wa.me` with the message pre-filled and the resident's number attached), then mark it sent — which
  resolves the issue.
* **Paste a complaint in:** complaints that still arrive on WhatsApp can be pasted straight onto the board and
  go through the same triage and duplicate check.
* **Filters and search:** status / category / urgency chips, a "needs clarity" filter, full-text search across
  message text and flat numbers, and sorting by urgency, age or how many residents are affected.
* **Committee corrections:** the details drawer lets a volunteer fix the AI's category/urgency in one click.
  Corrections feed the accuracy number and mark the message `you corrected this` instead of hiding the mistake.
* **Demo controls:** `Load demo data` (a realistic pre-triaged society), `Export` (JSON), `Reset`.
* **Activity log per issue:** every status change is timestamped with who did it, so the board is auditable.

### Cross-cutting

* **Works with bad internet:** reads and writes are local-first; Firestore writes that fail while offline are
  queued and flushed automatically when the connection returns, with an offline banner explaining what happened.
* **Dark mode**, responsive down to a small phone, `prefers-reduced-motion` respected, 
  ARIA live regions, focus rings and toasts for every action.
* **Zero external dependencies at runtime:** the CSS is hand-written and the Firebase SDK is vendored into
  `vendor/`, so the whole demo runs with the venue Wi-Fi switched off.

---

## How the AI works

All AI work happens in the browser through **one free Groq key**. No server, no proxy, no secrets in a repo
(Groq sends `Access-Control-Allow-Origin: *`, which is what makes a pure-frontend AI app possible).

### 1. Triage + duplicate clustering — one call per complaint

`AI.processComplaint(text, openIssues)` sends the raw message **together with a compact list of the open
issues** (id, title, category, urgency, summary, complaint count) and asks for a strict JSON contract:

```json
{
  "category": "Water | Lift | Parking | Cleaning | Security | Other",
  "urgency": "Critical | High | Medium | Low",
  "summary": "one clear line in English, max 14 words",
  "escalate": true,
  "needsClarity": false,
  "clarifyQuestion": "",
  "issueId": "existing issue id or null",
  "issueTitle": "short title only when issueId is null"
}
```

The system prompt encodes the society's reality — Hinglish/Hindi/typos, and an explicit urgency policy
(safety risk, stuck lift, outage over 24 h, elderly or children affected → *Critical*). The duplicate rules are
deliberately strict: *reuse an id only when the root cause matches, never invent an id*.

Because triage and clustering happen in the **same** call, a submission needs ~1 second and **one** API call
(measured: **906–1800 ms**).

### 2. Reply drafting — only when the volunteer asks

`AI.draftReply(issue, complaints, lang)` is a separate, non-JSON call that gives the model the issue summary,
the residents' own words and the intended next action, and asks for a WhatsApp reply of at most four sentences
in Hinglish, Hindi or English. It is explicitly forbidden from promising money or compensation.

### 3. Wrong or unusable AI output is handled, not hidden

| Failure | What the app does |
|---|---|
| Model replies with prose instead of JSON | JSON is extracted from the first `{` to the last `}` (including fenced code blocks); if parsing still fails, the complaint is triaged by the rules engine and marked `rules-fallback` |
| Invented `issueId` | Discarded — an id that is not in the open-issue list is treated as `null` |
| Enum outside the allowed values (`"Water supply"`, `"VERY HIGH"`) | Normalised to `Other` / `Medium` |
| Model unavailable / HTTP 400 / 429 / 5xx / timeout | Retried down a model chain (`gpt-oss-20b` → `qwen3.8-27b` → `gpt-oss-120b`) with one automatic retry per model |
| No key, no internet, or all models down | The **rules engine** takes over: keyword scoring for the 6 categories, an urgency policy, Hindi + Hinglish + Devanagari vocabulary, and token-level duplicate matching with a synonym map (`paani`/`water`/`पानी` → same bucket) |
| AI output that a human disagrees with | The volunteer corrects category/urgency in the details drawer; the correction is recorded, the card is marked, and the **AI accuracy** tile on the board updates |
| Complaint too vague to act on (`"problem hai"`) | `needsClarity` is set, a clarifying question is drafted, and the card is flagged so it can be filtered |

### 4. Why this is "AI at the core", not a chat widget

The AI output **is** the data model: category, urgency, summary, issue grouping and the reply all come from the
model. Remove the AI and the board loses its sorting, its grouping and its replies — it is not a decoration
bolted onto a form.

---

## Data model & storage modes

```js
// complaint (one resident message)
{ id, createdAt, flat, residentName, phone, text,
  category, urgency, summary, escalate, needsClarity, clarifyQuestion,
  source: "ai" | "rules" | "rules-fallback", model, latencyMs, aiVerified, corrected, issueId }

// issue (an AI-clustered problem, the unit the committee works on)
{ id, createdAt, updatedAt, title, category, urgency, summary,
  status: "New" | "Assigned" | "In Progress" | "Resolved",
  assignee, complaintIds[], replyDraft, replyLang, replySentAt,
  firstActionAt, resolvedAt, escalated, needsClarity, clarifyQuestion, timeline[] }
```

`Store` exposes one API with two interchangeable backends:

* **local** — localStorage (`samvaad.v1`) plus `BroadcastChannel`/`storage` events, so two tabs of the same
  browser update each other live. This is the default and it needs nothing.
* **firebase** — Firestore `onSnapshot` for realtime multi-device sync, with a persistent write queue for
  offline edits. Write failures are never lost and never block the UI.

Metrics are derived, not stored: open issues, critical count, resolved today, oldest open age,
**average hours to first action**, unclustered messages, and AI accuracy (`1 − corrections / triages`).

---

## Project structure

```
index.html            Resident portal (submit + track)
dashboard.html        Committee triage board
css/styles.css        Hand-written design system: tokens, dark mode, responsive, animations
js/config.example.js  Committed template for your keys (safe to share)
js/config.js          Your real Groq + Firebase keys — GITIGNORED, never committed
js/ui.js              DOM + formatting helpers, toasts, modals, theme, shared helpers
js/store.js           Data layer: localStorage ⇄ Firestore, offline write queue, stats, demo seed
js/ai.js              Groq client: triage + clustering, reply drafting, validation, rules fallback
js/resident.js        Resident portal behaviour
js/dashboard.js       Board behaviour: filters, drag & drop, details drawer, replies
vendor/               Firebase compat SDK, downloaded locally so the demo works offline
test/run-tests.js     66 automated tests (logic + wiring + hygiene), no network
test/live-check.js    Optional: 3 real Groq calls to prove the AI path works
```

---

## Tips

**Both pages:** the theme toggle and help (?) button live in the top bar; <kbd>Esc</kbd> closes any dialog.

**Resident:** <kbd>Ctrl</kbd>+<kbd>Enter</kbd> sends from the message box.

**Committee:** <kbd>Enter</kbd> opens the focused card, <kbd>Space</kbd> advances its status.

---

## Tests

```bash
node test/run-tests.js      # 66 checks, no network, no key needed
node test/live-check.js     # 3 real Groq calls: triage, duplicate merge, reply draft (needs your key)
```

`run-tests.js` loads the real `ui.js`, `store.js` and `ai.js` inside a sandbox with a stubbed DOM and a
**stubbed Groq API**, so it verifies the interesting behaviour without spending a single API call:

* rules engine: Hindi, Hinglish and Devanagari input, urgency policy, vague-input detection, duplicate matching
* store: seed counts, derived stats, status workflow (`firstActionAt`, `resolvedAt`, reopen), merging, deletion
* AI layer: fenced JSON parsing, invented `issueId` rejection, enum normalisation, model fallback chain on
  HTTP 400, rules fallback on junk output, reply drafting, and the offline template reply
* wiring + hygiene: every `#id` the JS looks for exists in the HTML (or is rendered by it), script load order,
  **no ES-module syntax** (so `index.html` still works from `file://`), and **no API key present in any file
  that gets committed**

Expected output:

```
PASSED: 66    FAILED: 0
```

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Board says **Rules mode (no key)** | No key yet. Copy `js/config.example.js` to `js/config.js` and paste your free `gsk_...` key. Everything else still works. |
| "Could not triage" toast | Wrong or expired key, rate limit, or no internet. The app falls back to the rules engine and the complaint is still saved. |
| Nothing appears on the board | You are in a different browser profile or on a different origin (`file://` vs `http://localhost:5500`). Use the same tab/origin, or add the Firebase config for cross-device sync. |
| Firebase badge stays **Local storage** | `firebase.apiKey` or `projectId` is empty, or the SDK files in `vendor/` are missing. |
| Firestore permission errors | Paste the rules from the Firebase section above (test mode). |
| Mic button does nothing | Speech recognition needs Chrome/Edge, microphone permission, and an `http://` origin — use Live Server or any static host. |
| `js/config.js` 404 in the console | Expected for anyone who did not create it. The file is optional; without it the app runs in rules mode. |

---

## Design decisions & edge cases

* **One message box instead of a form.** The persona is a resident with a phone, patience for one sentence, and
  no interest in picking a category. Category is the committee's problem, so the AI owns it.
* **A board of issues, not messages.** Ten water complaints should cost the volunteer one decision, not ten.
* **The committee keeps the last word.** Every AI output is editable, corrections are recorded and measured —
  honest about the model, useful to the volunteer.
* **Slow internet is a normal case, not an error.** Local-first writes, a retry queue, an offline banner and a
  rules fallback mean the demo (and the residents) keep working with no signal.
* **WhatsApp is the real channel.** Replies and clarifying questions are one tap from `wa.me`, and complaints
  that still arrive on WhatsApp can be pasted onto the board and triaged the same way.
* **No money and no signup wall.** A judge needs one free key and one double-click; demo data, storage and the
  libraries are already local.
* **Visibility for the resident.** A tracking id and a live status bar, because "we will look into it" with no
  visibility is exactly what erodes trust in a society committee.

---

## How this maps to the judging criteria

| Criterion | Where it shows up |
|---|---|
| **Working product (25)** | Two complete flows: resident submit → AI triage → live tracking, and committee board → status workflow → AI reply → WhatsApp → resolved. `Load demo data` fills a realistic society in one click so a judge can evaluate without typing anything. Runs from `file://` or any static host, with or without a key, with or without internet. |
| **AI at the core (20)** | Category + urgency + one-line summary, cross-language duplicate clustering into issues, clarifying questions for unusable input, and drafted replies — all produced by the model, with JSON validation, enum normalisation, a model fallback chain, a measured accuracy score, and a rules engine so failure degrades instead of crashing. |
| **Fit to problem & constraint (15)** | Built for mixed English/Hindi/Hinglish and for a volunteer with only minutes a day: no forms, urgency-sorted cards, drag and drop,  one-line summaries, and WhatsApp-based replies. |
| **Scope & difficulty (10)** | A 4-stage workflow with an audit timeline, a deduplication engine, realtime sync (localStorage ⇄ Firestore) with an offline write queue, derived metrics (average time to first action, AI accuracy), drag and drop, JSON export, and a 66-test suite. |
| **UI & UX (10)** | A hand-written design system with dark mode, an urgency colour language, skeletons, empty states, toasts, confirmations, focus rings, ARIA live regions, reduced-motion support and a phone layout. |
| **Code quality & README (10)** | Modular files with JSDoc comments, no dead code, a documented data model, install/run instructions, Firebase rules, a troubleshooting table, and a real test suite with a live-API verifier. |
| **Idea & thoughtfulness (10)** | The cases the statement hints at are handled: unclear input, mixed languages, duplicates, wrong AI output, slow internet, mis-taps (reopen, edit before sending) and residents who cannot type (voice, example chips, remembered flat number). |

---

## Credits

* **AI:** Groq free tier (`openai/gpt-oss-20b`) — https://console.groq.com
* **Realtime storage:** Firebase Firestore free (Spark) tier — optional
* **Everything else:** hand-written HTML, CSS and JavaScript. No framework, no build step, no paid service.



