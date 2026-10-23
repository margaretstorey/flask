---
name: "audit-tours"
description: "Check tours against the current code and map: which recently changed files have no waypoint or no tour, stale or non-bridging stop narratives, redundant edges and possible missing stops. Reads the site's /api/data; reports only, never writes to the map."
user-invocable: true
disable-model-invocation: false
---

# Audit Tours Skill

Purpose: the tour-level counterpart to `/audit-waypoints`. It surfaces
drift between the tours and the code, and gaps in coverage after a
stretch of work. It is read-only with respect to the map: it never
writes waypoints, tours or edges, and never sends anything to the
app. Rewriting a narrative and deciding a tour's shape stay with the
user, who makes every change by hand in the app.

Usage: `/audit-tours`, optionally `since YYYY-MM-DD` and/or
`site <url>`, e.g. `/audit-tours since 2026-10-20`.

## Setup

1. **Which site.** Use the `site` argument if given. Otherwise, if
   `CLAUDE.md` lists the user's own site, use that
   (`https://cognitive-tours.vercel.app` in the Cognitive Tours repo);
   if it doesn't, ask for the site's URL and wait.
2. **Read the map.** Fetch `<site>/api/data` with a plain GET
   (`curl -sS <site>/api/data`) into a scratch file and work from it
   with `jq`. It holds `waypoints` (`id`, `label`, `type`,
   `description`, `notes`, `filePath`), `edges` (`source`, `target`,
   `label`) and `tours` (`name`, `description`, `stops` in order, each
   `{waypointId, narrative}`). This is a GET only; never POST, PUT or
   DELETE to the site. Say which site and how many waypoints, edges
   and tours were read.
3. **Since when.** Use the `since` argument if given. Otherwise use the
   date of the newest `backups/backup-*.json` if there is one, or else
   7 days ago. Say which date you chose and why.

## Section A — Coverage (mechanical, one block)

This answers "what did I change that the map doesn't know about yet?"

1. Run `git log --since='<date> 00:00' --name-only --format='%h %s'`
   (the `00:00` matters: a bare date means that day at the current
   time) and collect the changed files under `app/`, `tests/`, `scripts/` and
   `prisma/schema.prisma` that still exist.
2. For each file, list the waypoints whose `filePath` is that file
   (compare ignoring a leading `./`). Mark it **NO WAYPOINT** when
   there are none.
3. For each of those waypoints, list the tours that include it as a
   stop. Mark it **IN NO TOUR** when there are none.
4. Under each file marked NO WAYPOINT or IN NO TOUR, list the subjects
   of the commits that touched it, as raw material for the user's own
   writing.

Report it all together:

```
Coverage since <date>:
- app/lib/sites.ts → NO WAYPOINT
    faf1086 Back up every site nightly to a private Vercel Blob store
- app/components/TourBar.tsx → [w05] TourBar → tours: Architecture…
```

Don't suggest waypoint text here. Coverage is a list of gaps; filling
them is the user's work (see "When the user asks for a draft" below).

## Section B — Tour drift (judgment, one finding at a time)

Walk each tour's stops in order. Remember: a stop's `narrative` is the
bridge **from that stop to the next one** (shown as "Next up").
Flag only these:

- **Stale bridge.** The narrative names a component, file, panel or UI
  location (e.g. "the amber bar in TourBar") that the current code, or
  the named waypoint's current description, places somewhere else, or
  that no longer exists. Cite the file and line that shows what's true
  now.
- **Bridge that doesn't bridge.** The narrative mostly repeats the
  description of its own stop or of the next stop, rather than saying
  why the reader moves from one to the other.
- **Bridge to the wrong place.** The narrative introduces a waypoint
  that isn't the next stop (the tour was reordered, or a stop was
  removed).
- **Redundant path.** Two edges reach the same waypoint where one is
  implied by the other, e.g. A→contains→C when A→contains→B→contains→C
  exists. Report the edge that looks redundant; deleting it is the
  user's call.
- **Missing stop (suggestion only).** A waypoint connected by edges to
  two consecutive stops of a tour, but not itself in the tour. Phrase
  it as "consider", never as an error.

Do not flag style, tone or length, and do not flag a tour for being
incomplete in general.

Present the first finding in this format, with nothing else asked or
implied:

```
[<tour name>] stop <N>: <waypoint label>
   Finding: <stale bridge | doesn't bridge | wrong place | redundant path | missing stop>
   Text: "<the narrative or edge, quoted>"
   True now: <what the code or map shows, with file:line>
```

Then stop and wait. When the user comes back ("next", "done, what's
next"), present the next finding in the same format. When there are no
more, say so explicitly ("that's all of them").

## Section C — External links (one line)

List any waypoint whose `filePath` starts with `http://` or `https://`
as "external, not checked". Don't fetch them.

## When the user asks for a draft

Fact fixes (a name, a count, a path, a stop number) can be given
directly as the corrected text.

New or rewritten **narratives, descriptions and tour descriptions** are
the user's to write: writing them is how they build understanding. If
the user asks for one, first nudge once, briefly, to try it themselves
(e.g. "Try a sentence first: what does the reader need from this stop
to make sense of the next one? I'll react to it"). If they still want
a draft, give one, short, marked as a draft to edit rather than paste.
Never offer drafts unprompted.

## Never

- Write to the map in any way: no POST, PUT or DELETE to the site, no
  edits to `prisma/*.db`, no scripts that do either.
- Offer to apply changes. Every change is made by the user in the app.
