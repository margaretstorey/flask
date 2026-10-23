---
name: "suggest-tour"
description: "Help the user fit recent waypoints into tours: suggest stops to add to existing tours or a new tour with a starting order, ask what the tour is for, and have the user write the narratives. Reads the site's /api/data; never writes to the map."
user-invocable: true
disable-model-invocation: false
---

# Suggest Tour Skill

Purpose: help the user keep tours in step with the map after new work,
usually right after `/suggest-waypoint`. The skill suggests structure
(which tour, which stops, what order); the user decides the tour's
shape and writes every narrative. It never writes to the map: the user
makes every change in the app.

Usage: `/suggest-tour`, optionally `since YYYY-MM-DD` and/or
`site <url>`.

## Setup

1. **Which site.** Use the `site` argument if given. Otherwise, if
   `CLAUDE.md` lists the user's own site, use that
   (`https://cognitive-tours.vercel.app` in the Cognitive Tours repo);
   if it doesn't, ask for the site's URL and wait.
2. **Read the map, fresh.** Fetch `<site>/api/data` with a plain GET
   (`curl -sS <site>/api/data`) into a scratch file, even if it was
   read earlier in the conversation: the user may have just added
   waypoints. It holds `waypoints`, `edges` (`source`, `target`,
   `label`) and `tours` (`name`, `description`, `stops` in order, each
   `{waypointId, narrative}`). GET only.
3. **Since when.** Use the `since` argument if given. Otherwise use the
   date of the newest `backups/backup-*.json` if there is one, or else
   today. Use `git log --since='<date> 00:00'` (the `00:00` matters: a bare date means that day at the current time) (and `resources/worklist-history/`
   if present) to know which waypoints relate to recent work: those
   whose `filePath` changed, plus any waypoints the user just created
   in this conversation.

## 1. Suggestions (one block)

- **Existing tours that should change:**
  - a recent waypoint linked by edges to two consecutive stops of a
    tour: suggest inserting it between them;
  - a stop whose waypoint changed a lot since the date: its narrative
    and the next stop's may need a look (point to `/audit-tours`).
- **Whether a new tour is warranted:** several recent waypoints linked
  to each other (or to the same existing waypoint) that no tour covers.
  Suggest a working name and a stop **order** built from the edges
  (e.g. decision → feature → code → test), as a starting point only.

Keep it to the few that matter. Then ask which one to work on (or
none), and wait.

## 2. For the tour picked

1. Ask what the tour is **for** and who will read it (e.g. "a student
   joining a team site", "me in three months"). One message, then
   wait.
2. Confirm the stop list and order with the user. Adjust to what they
   say; the shape is theirs.
3. If it's a new tour, the **tour description** is the user's to write
   (same draft rule as below).

## 3. Narratives, one stop at a time

A stop's narrative is the bridge **from that stop to the next one**
(shown as "Next up"). For each stop, ask: "Why does the reader go from
this stop to the next?" Then wait for the user's sentence and react:
does it lead into the next stop, is anything factually off (cite
file:line)?

If the user asks the skill to write a narrative, nudge once, briefly,
to try first ("Try one sentence: what does the reader need from this
stop to make sense of the next? I'll react to it"). If they still want
it, give a short draft, marked as a draft to edit. Never offer drafts
unprompted. The last stop's narrative can close the tour or point to
another tour.

## 4. Steps to do in the app

When the stops and narratives are settled, list what to do, in plain
text (no wide code blocks):

- which tour, and **Edit tour** or **+ New Tour**;
- which stops to add, and where to move them;
- which narratives to paste (the user's own wording).

Then wait. On "next", move to the next suggestion; when done, say so.

## Never

- Write to the map in any way: no POST, PUT or DELETE to the site, no
  edits to `prisma/*.db`, no scripts that do either.
- Offer to create or edit the tour for the user. Every change is made
  by the user in the app.
