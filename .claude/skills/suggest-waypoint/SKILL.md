---
name: "suggest-waypoint"
description: "Help the user record recent work as waypoints: find what's worth explaining later (decisions first), ask questions so the user writes the understanding, and give the fact fields to enter in the app. Reads the site's /api/data; never writes to the map."
user-invocable: true
disable-model-invocation: false
---

# Suggest Waypoint Skill

Purpose: help the user notice what recent work deserves a waypoint, and
help them write it. The user writes the understanding; this skill asks
the questions, checks the facts and suggests the fact fields. It never
writes to the map: the user creates every waypoint in the app.

Run it at the end of a session, or any time the user suspects work went
unrecorded.

Usage: `/suggest-waypoint`, optionally `since YYYY-MM-DD` and/or
`site <url>`.

## Setup

1. **Which site.** Use the `site` argument if given. Otherwise, if
   `CLAUDE.md` lists the user's own site, use that
   (`https://cognitive-tours.vercel.app` in the Cognitive Tours repo);
   if it doesn't, ask for the site's URL and wait.
2. **Read the map.** Fetch `<site>/api/data` with a plain GET
   (`curl -sS <site>/api/data`) into a scratch file and work from it
   with `jq`. It holds `waypoints` (`id`, `label`, `type`,
   `description`, `notes`, `filePath`), `edges` (`source`, `target`,
   `label`) and `tours`. GET only; never POST, PUT or DELETE.
3. **Since when.** Use the `since` argument if given. Otherwise use the
   date of the newest `backups/backup-*.json` if there is one, or else
   today. Say which date you chose and why.

## 1. What happened

- `git log --since='<date> 00:00' --name-only --format='%h %s'` (the `00:00` matters: a bare date means that day at the current time): the commits
  and the files they changed.
- If `resources/worklist-history/` exists (a Bram project), read the
  `.md` entries in that window: each committed item's before/after text
  is the best record of **why** a change was made.
- The map: which changed files already have a waypoint, and which
  ADR waypoints and `docs/adr/` files exist.

## 2. Candidates (one block)

Not one waypoint per changed file. The user doesn't want to record
every detail; the goal is what future-them, or a newcomer, would need
explained. Rank in this order:

1. **Decisions** (ADR-worthy): a choice between alternatives with a
   reason. Put first any that **contradict an existing ADR** or ADR
   waypoint, and name it.
2. **New subsystems or capabilities** (Feature, Code Component).
3. **Risks** the work introduced.
4. **Everything else** (small UI tweaks, demos, reverts) in one line:
   "probably not worth a waypoint".

At most about five candidates. Files that belong to a bigger candidate
are listed under it, not separately. For each:

```
1. [ADR] One site and database per team  ← contradicts ADR 007
   file: docs/adr/008-…  (none yet)
   from: 46f4f82 Let team sites start empty…, a0d73f3 …
```

Also list, in one line, existing waypoints this work probably made
stale, and point to `/audit-waypoints` for those.

Then ask which one to start with (or none), and wait.

## 3. One waypoint at a time, questions first

For the candidate picked, ask two or three questions, one message,
then wait. For example:

- What does this do, or what was decided, in your words?
- Why was it needed? What would break or confuse someone without it?
- (Decision) What else did you consider, and why not that?
- Which existing waypoints does it connect to?

When the user answers, reflect back briefly: what's clear, what's
missing, and anything that's factually off, citing the code
(file:line). Don't rewrite their answer.

## 4. Fields to enter in the app

Give these directly, as fact fields:

- **Label**, **type**, **file path** (repo-relative).
- **Edges** to existing waypoints, each with its direction and a label
  from the ontology in `CLAUDE.md` where one fits (motivates, supported
  by, implemented by, leads to, validates, threatens, contains).

**Description and notes come from the user's answers.** Point out
which of their sentences would make a good description. If the user
asks the skill to write them, nudge once, briefly, to try first ("Try
one sentence: what would you want to read here in three months? I'll
react to it"). If they still want it, give a short draft, marked as a
draft to edit. Never offer drafts unprompted.

Waypoint content must make sense on its own, whatever tour brings a
reader to it: no "next we'll see…" in a waypoint.

Then wait. On "next", move to the next candidate. When all picked
candidates are done, say so, and suggest `/suggest-tour` to place the
new waypoints in tours.

## Never

- Write to the map in any way: no POST, PUT or DELETE to the site, no
  edits to `prisma/*.db`, no scripts that do either.
- Offer to create the waypoint for the user. Every waypoint is entered
  by the user in the app.
