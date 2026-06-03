---
type: guide
purpose: shareable setup instructions for a cloud briefing assistant
audience: humans + AI agents
contains_secrets: false
---

# Daily & Weekly Briefing Assistant — Setup Guide

A reusable recipe for a **cloud-based personal briefing assistant** that, on a schedule,
reads your **calendar(s)** and your **notes/tasks repository**, composes a short brief,
and **pushes it to your phone** — with **no computer needing to be on**.

It was designed around three goals: **token efficiency** (works on a consumer
Claude subscription), **reliability** (runs unattended in the cloud), and **privacy**
(read-only toward your accounts; never sends mail or responds to invites).

This document is safe to share: it contains **no secrets, IDs, or personal data** —
only placeholders in `<ANGLE_BRACKETS>` that you fill in.

---

## 1. What you get

Two scheduled briefs delivered to a chat/notification channel on your phone:

- **☀️ Daily Rundown** (e.g. weekday mornings): today's meetings across all your
  calendars, urgent/overdue tasks, and prep flags for upcoming meetings.
- **🗓️ Weekly Prep** (e.g. Sunday afternoon): the week ahead, action items tied to
  meetings, tasks due this week, and open follow-ups.

Each brief is also **archived as a file in your notes repo**, so you have a durable record.

---

## 2. How it works (architecture)

```
┌────────────────────────────────────────────────────────────────┐
│  CLOUD (runs on schedule, no personal computer required)         │
│                                                                  │
│  Scheduled AI routine, in an isolated cloud session, with:       │
│    • a git checkout of your notes repo (tasks, people, projects) │
│    • Calendar tool access (one or more calendars)                │
│    • outbound network access to your notification webhook        │
│                                                                  │
│   1. Read events from ALL your calendars → merge & de-duplicate  │
│   2. Read tasks/notes from the repo checkout                     │
│   3. Compose a short markdown brief                              │
│   4. POST the brief to your chat webhook  ──────► 📱 phone push  │
│   5. Write the brief to a file + commit/push ──► 🗄️ repo archive │
└────────────────────────────────────────────────────────────────┘
```

**Key design choices:**

- **Calendars** are read through a calendar connector/tool. The brief reads **every**
  calendar you specify and merges them, because most people's true schedule is split
  across several (work + personal + shared).
- **Tasks/notes** reach the cloud **through the git repository**, not through your
  local disk. The cloud session clones your repo; it only sees what you have **pushed**.
- **Delivery** is a plain HTTPS POST to a **chat webhook** (e.g. Discord, Slack, or any
  service that offers an incoming-webhook URL). This is what makes phone delivery work
  from the cloud without your computer.

---

## 3. Prerequisites

| Need | Notes |
|------|-------|
| A Claude subscription that supports **scheduled cloud routines** | The routines run server-side on a schedule. |
| A **calendar connector** authorized in your AI account | e.g. Google Calendar, Microsoft 365 — any connectable calendar. Authorize it for **read** access. |
| A **notes/tasks repository on GitHub** (or similar) | Markdown notes work well. A private repo is fine. |
| The AI provider's **GitHub App** installed on that repo | ⚠️ This is separate from "connecting GitHub" — see §4.2. |
| A **chat webhook URL** for notifications | Discord/Slack/etc. incoming webhook. Treat it like a password. |
| A way to **keep the repo current** | A scheduled `git push` of your notes (see §6). |

---

## 4. One-time setup

### 4.1 Authorize the calendar connector(s)

In your AI provider's connector/settings page, connect each calendar account you use.
Grant **read** access.

**Identify each wanted calendar by its exact ID, not its name.** Most calendar tools
let you list calendars to get a stable **calendar ID** (for Google Calendar these look
like an email address such as `you@gmail.com`, or `…@group.calendar.google.com`; your
personal calendar's ID is usually your account email and may *display* as your name).
Record the ID of **each** calendar you want merged.

> ⚠️ **Pin IDs; never match by name (learned the hard way).** If your prompt tells the
> agent to "find my calendars by name" or "include all my calendars," it may enumerate
> calendars and sweep in **shared/other calendars you don't want** (e.g. a family
> member's calendar you have access to). The fix is to **hard-code the exact IDs** of
> the calendars to include and explicitly forbid querying any others (and forbid the
> "list all calendars" call). See the prompt templates in §5.

> 💡 If two calendars contain overlapping events, the brief de-duplicates them by
> matching start time + title.

### 4.2 Install the GitHub App on your notes repo (the easy thing to miss)

There are **two different GitHub authorizations**, and you need the second one:

1. **OAuth "connect GitHub"** — lets the assistant act as your identity. *Not enough on
   its own* for a cloud routine to clone a repo.
2. **GitHub App installation** — grants repo-level read/clone access. **This is the one
   the cloud routine uses.**

To install: go to your AI provider's GitHub App install flow (or
`github.com/apps/<provider-app>/installations/new`), choose your account, and under
**Repository access** select **All repositories** or **Only select repositories** with
your notes repo added. Confirm it appears under `github.com/settings/installations`.

> ❌ Symptom if you skip this: cloud runs fail with a *"repository access denied / re-authorize
> GitHub"* error, even though "GitHub is connected." Fix = install the App on the repo.

### 4.3 Format your to-dos so the brief can find them (Obsidian + Tasks plugin)

The brief finds action items by **grepping your notes for task lines**, so it depends on a
consistent, machine-readable to-do format. This recipe was built around
[Obsidian](https://obsidian.md/) with the **[Tasks community plugin](https://publish.obsidian.md/tasks/)**,
which is a popular way to manage to-dos *inside* your notes — but if you don't already use
it, the syntax below looks cryptic, so here's what it means.

**The basics.** A task is a standard Markdown checkbox line. Tasks live **inline in
whatever note they belong to** (a project note, a meeting note, a daily note) — you do *not*
keep a separate master to-do file. The Tasks plugin then aggregates them into live query
views across the whole vault.

```markdown
- [ ] An open (incomplete) task
- [x] A completed task
```

**Metadata via emoji.** The Tasks plugin attaches due dates, priorities, etc. using a set
of **emoji "signifiers"** appended to the line. The brief keys off the **due date** in
particular:

```markdown
- [ ] Email the contractor about the quote 📅 2026-06-15 🔼
- [ ] Submit the grant application 📅 2026-06-09 ⏫ ➕ 2026-06-01
```

| Signifier | Meaning |
|-----------|---------|
| `📅 YYYY-MM-DD` | **Due date** (what the brief reads to decide "overdue"/"due this week") |
| `➕ YYYY-MM-DD` | Created date |
| `🛫 YYYY-MM-DD` | Start date · `⏳ YYYY-MM-DD` Scheduled date |
| `🔺 / ⏫ / 🔼 / 🔽` | Priority: highest / high / medium / low |
| `🔁 <rule>` | Recurrence (e.g. `🔁 every week`) |
| `#tag` | Plain Markdown tag, for grouping/filtering |

> 💡 **You don't have to type the emoji by hand.** In Obsidian, the Tasks plugin's
> *"Create or edit task"* command gives you a dialog with date pickers and priority
> dropdowns, and inserts the correct signifiers for you. The emoji are just how the data is
> stored on the line.

**What the briefing prompts assume:** open tasks are lines beginning with `- [ ]`, and a
task's due date is the value after `📅`. If you use a **different** task system, change two
things in the prompt templates (§5): the **task marker** (e.g. `TODO:` or `* [ ]`) and the
**due-date pattern** the agent should parse. Everything else stays the same.

> ℹ️ The same approach works with other Markdown task conventions (e.g.
> [Logseq](https://logseq.com/), `TODO`/`DONE` keywords, or Dataview-style `[due:: ...]`
> fields) — just describe your format in the prompt so the agent's grep matches it.

### 4.4 Create the notification webhook

In your chat app, create an **incoming webhook** for the channel you want briefs in, and
copy its URL. Examples:

- **Discord:** Channel → Edit Channel → Integrations → Webhooks → New Webhook → Copy URL.
- **Slack:** create an app → Incoming Webhooks → add to a channel → copy URL.

⚠️ **Keep this URL out of your repository.** Anyone with it can post to that channel.
Store it **inside the routine's prompt/config** (which lives in your AI account, not in
git), **not** in a committed file. Rotation is rare — only if it leaks.

### 4.5 Create (or designate) a cloud environment that allows the webhook host

Cloud routines often run in a **network-restricted sandbox**. By default, outbound POSTs
to your chat webhook may be **blocked** (you'll see HTTP `403`). To fix:

- Use/create a routine **environment** whose **network allowlist** permits your webhook
  host (e.g. `discord.com` / `discordapp.com`, or `hooks.slack.com`).
- Assign your routines to **that** environment.

> ⚠️ **Critical gotcha:** if you later edit a routine **via API** and don't re-specify the
> environment, it may **reset to the default environment** (no allowlist) and delivery
> silently breaks (`403`). **Always pass the correct environment on every edit.** If you
> edit routines only in the web UI, this is less of a risk.

---

## 5. Create the two routines

Create two scheduled routines (daily + weekly). For each, configure:

- **Schedule (cron):** in the provider's required timezone (often **UTC**). Convert from
  your local time and **double-check around Daylight Saving** — when your offset changes,
  the local fire-time shifts unless you update the cron. Tip: avoid `:00`/`:30` exact
  minutes to dodge scheduler congestion (e.g. use `:03`, `:07`).
- **Repository source:** your notes repo.
- **Environment:** the allowlisted one from §4.5.
- **Tools allowed:** read/search/file tools + a shell for git, and write only if you want
  the routine to archive the brief into the repo. Keep it **read-only toward your accounts**
  (no calendar-write, no email).
- **Prompt:** use the templates in §5.1 / §5.2, filling placeholders.

### 5.1 Daily Rundown — prompt template

```
You are my morning briefing agent running in the cloud. Be concise and token-efficient.
READ-ONLY toward my external accounts: do NOT send email, create or edit calendar events,
or RSVP. You MAY write a briefing file into the git repo and POST it to a chat webhook as
instructed in the final steps.

Context: you have a git checkout of my notes repo and calendar tools. My complete schedule
spans MULTIPLE calendars that must be COMBINED: <CALENDAR_NAME_1>, <CALENDAR_NAME_2>
(add more as needed). Always query ALL of them and merge into one timeline; de-duplicate
any event appearing in more than one (same start time + title). My active notes live under
<NOTES_SUBPATH>/ — people files in <PEOPLE_PATH>/, projects in <PROJECTS_PATH>/.

Produce a SHORT rundown for TODAY:
1. SCHEDULE: list today's events from ALL calendars, merged chronologically (time, title,
   location). De-duplicate. If none, say 'No meetings today.'
2. URGENT TASKS: grep the repo for open tasks (your task marker, e.g. '- [ ]') with a
   due-date on/before today. Use grep; do not read whole files. List up to 8, most overdue
   first, each with its due date. EXCLUDE <QUERY_VIEWS_FOLDER> (auto-generated views).
3. BEFORE-MEETING FLAGS: for each meeting, if a person/project file clearly matches the
   attendees or title, note its open tasks (1–2 lines each; skip non-matches).

Compose the brief as tight markdown titled '☀️ Morning Rundown — <weekday, Month D>'.
Lead with the first meeting time. Keep under ~250 words. End with the single most important
thing to do before the first meeting. If repo or calendar can't be read, say so at the top.

=== DELIVERY (in order) ===
FIRST write the brief to /tmp/brief.md.

STEP A — POST TO CHAT WEBHOOK at <WEBHOOK_URL>. Build valid JSON and POST it. Example bash
(python3 if present, else jq), capturing the HTTP status:
  WEBHOOK='<WEBHOOK_URL>'
  if command -v python3 >/dev/null 2>&1; then
    PAYLOAD=$(python3 -c 'import json;t=open("/tmp/brief.md").read();t=(t[:1900]+"\n(truncated; full brief in repo)") if len(t)>1900 else t;print(json.dumps({"content":t}))')
  else
    PAYLOAD=$(jq -Rs '{content: .}' < /tmp/brief.md)
  fi
  HTTP=$(printf '%s' "$PAYLOAD" | curl -s -o /dev/null -w '%{http_code}' -X POST -H 'Content-Type: application/json' --data-binary @- "$WEBHOOK")
  echo "WEBHOOK_HTTP=$HTTP"
(Adjust the JSON field name to your chat service: Discord/Slack use "content"/"text".
A 2xx status = success; 403 = the webhook host is blocked by the environment allowlist.)

STEP B — ARCHIVE TO REPO: write the brief to <ARCHIVE_PATH>/<YYYY-MM-DD>_Morning.md
(create the folder if needed). Append at the very end: <!-- webhook_http: $HTTP, generated:
<UTC timestamp> -->. Then:
  git config user.email 'routine@local'; git config user.name 'Briefing Routine';
  git add <ARCHIVE_PATH>/;
  git commit -m 'briefing: morning rundown <YYYY-MM-DD>';
  git pull --rebase origin <DEFAULT_BRANCH>;
  git push origin <DEFAULT_BRANCH>
If push is rejected, run 'git pull --rebase' once more and push again.

SECURITY: NEVER print, echo, or commit the webhook URL. The HTML comment must contain ONLY
the status code, never the URL.

FINAL REPORT: (a) WEBHOOK_HTTP status, (b) whether the file committed and pushed, (c) the
full brief text (for the run log).
```

### 5.2 Weekly Prep — prompt template

Same as §5.1, but the body covers the next 7 days:

```
... (same context + calendar-merge instructions) ...

Produce a WEEK-AHEAD brief for the next 7 days:
1. MEETINGS: events from ALL calendars for 7 days, merged and grouped by day. De-duplicate.
   Flag scheduling friction: double-bookings, or back-to-back meetings in different
   locations.
2. ACTION ITEMS BY MEETING: for meetings matching a person/project file, list that file's
   open tasks. Skip non-matches. Use grep.
3. TASKS DUE THIS WEEK: open tasks with a due-date in the next 7 days, grouped by date.
   EXCLUDE <QUERY_VIEWS_FOLDER>.
4. FOLLOW-UPS: read <OPEN_LOOPS_FILE> and surface unresolved items needing action this week.

Title: '🗓️ Week Ahead — <Month D–D>'. Under ~400 words. End with a 3-item
'Top priorities this week' list.

=== DELIVERY === (identical to the Daily template; archive file name <YYYY-MM-DD>_Weekly.md)
```

### 5.3 Placeholder reference

| Placeholder | Meaning | Example |
|-------------|---------|---------|
| `<CALENDAR_NAME_n>` | Exact display name of each calendar to merge | `Work`, `Personal` |
| `<NOTES_SUBPATH>` | Root of your active notes in the repo | `notes/` |
| `<PEOPLE_PATH>` / `<PROJECTS_PATH>` | Where person/project files live | `notes/People`, `notes/Projects` |
| `<QUERY_VIEWS_FOLDER>` | Folder of auto-generated task views to skip | `00_Tasks/` |
| `<OPEN_LOOPS_FILE>` | File listing unresolved follow-ups | `agent/_OpenLoops.md` |
| `<ARCHIVE_PATH>` | Where briefs are saved | `agent/briefings` |
| `<WEBHOOK_URL>` | Your chat incoming-webhook URL | *(kept in routine config only)* |
| `<DEFAULT_BRANCH>` | Repo default branch | `main` |

---

## 6. Keep the repo fresh (so the brief sees current tasks)

The cloud only sees what's **pushed**. If you edit notes locally, schedule an automatic
push so the cloud has today's tasks:

- A small **scheduled job on the machine where your notes live** that runs, e.g. nightly:
  `git add -A && git commit -m "notes: nightly auto-commit" && git pull --rebase && git push`
- Consequence to accept: the morning brief reflects your notes **as of the last push**.
  A nightly push means edits made after that won't appear until the next day.

Optionally, a **pull job** on your machine brings the cloud-written brief files back into
your local notes for reading in your editor. (Delivery to your phone does not depend on
this — it's just for local archival convenience.)

---

## 7. Test & verify

1. **Manually run** the daily routine (most providers have a "run now" button/endpoint).
2. Confirm the **chat message arrives on your phone** (push works).
3. Confirm the **status stamp** in the archived file is a success code (e.g. `204`),
   not `403` (= webhook host blocked → fix the environment allowlist, §4.5).
4. Confirm events from **every** calendar appear and there are **no duplicates**.
5. Verify the **first scheduled run** fires at the right local time (mind UTC/DST).

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|--------|--------------|-----|
| `repository access denied` on cloud run | GitHub **App** not installed on the repo (only OAuth connected) | Install the App on the repo (§4.2) |
| Webhook POST returns **403** | Environment network allowlist blocks the webhook host | Add the host to the environment allowlist; assign routine to it (§4.5) |
| Delivery broke after an **edit** | Editing via API reset the routine to the default environment | Re-specify the allowlisted environment on every edit (§4.5) |
| Brief missing some meetings | Only one calendar queried | List **all** calendar names in the prompt and merge (§5) |
| Brief shows **stale** tasks | Repo not pushed recently | Set up the auto-push (§6) |
| Brief fires at the **wrong hour** | Cron is UTC and/or DST shifted | Recompute cron; adjust at DST changeovers (§5) |
| Webhook URL leaked | It was committed or shared | Delete/rotate the webhook in your chat app; update the routine config |

---

## 9. Privacy & safety defaults

- **Read-only** toward your accounts: the routines never send email, create events, or RSVP.
  Enforce this by **not granting** write/send tools and by **not attaching** an email
  connector.
- The **webhook URL is the only secret**; keep it in the routine config, never in the repo.
- Briefs are archived in **your** repo; nothing leaves your control except the chat message
  to **your** channel.
- Start small: get the daily brief working end-to-end before adding the weekly one or any
  extra data sources.

---

## 10. Ideas to extend (optional)

- "Waiting on" detection: surface threads/tasks where someone owes you a reply (requires
  an email connector — adds cost and privacy surface; opt in deliberately).
- Post-meeting capture nudges; relationship-maintenance reminders (people not contacted in
  a long time); travel/location-conflict flags from event locations.
- Populate a daily note template from the brief instead of a standalone archive file.
```
