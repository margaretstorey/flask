---
name: "audit-waypoints"
description: "Check Code Component, ADR, and Test waypoint descriptions against the actual current contents of their linked files, and report factual drift as a list of manual edits — never writes to the waypoint data itself."
user-invocable: true
disable-model-invocation: false
---

# Audit Waypoints Skill

Purpose: check Code Component, ADR, and Test waypoint descriptions
against the actual current contents of their linked files, and flag
factual drift. This skill is read-only with respect to waypoint data:
it reads the map from the site's `/api/data` and never writes to it or
edits anything on the user's behalf. All corrections are applied by
the user, by hand, through the app's own Edit Waypoint UI.

Usage: `/audit-waypoints`, optionally `site <url>`.

## Setup

1. **Which site.** Use the `site` argument if given. Otherwise, if
   `CLAUDE.md` lists the user's own site, use that
   (`https://cognitive-tours.vercel.app` in the Cognitive Tours repo);
   if it doesn't, ask for the site's URL and wait.
2. **Read the map.** Fetch `<site>/api/data` with a plain GET
   (`curl -sS <site>/api/data`) into a scratch file and work from it
   with `jq`. Its `waypoints` each have `id`, `label`, `type`,
   `description`, `notes` and `filePath`. Say which site was read and
   how many waypoints it has.

## Procedure

1. From that JSON, take all waypoints of type "Code Component", "ADR",
   or "Test" that have a non-empty filePath.
   A filePath starting with `http://` or `https://` is an external
   link: don't read or flag it; list it once at the end as
   "external, not checked".
2. For each one, read the actual file at that path.
3. Compare the waypoint's description and notes against the file's
   current contents. Flag only concrete, verifiable claims that have
   become false — counts (e.g. "8 types" when there are now 10),
   function/type/component names that changed or were removed, file
   paths that moved, "this is the only place X happens" claims that are
   no longer true.
   - **Code Component**: as above — description should match current
     code.
   - **ADR**: an ADR is a historical record of a decision, not living
     documentation — do NOT flag an ADR just because the code has since
     evolved past it or been superseded (a waypoint like "Status:
     Superseded by Phase 2" is doing its job correctly, not drifting).
     Only flag concrete pointer errors: a filePath that no longer
     exists or was renamed, or a component/function name the ADR
     references by name that no longer exists under that name.
   - **Test**: check the waypoint's claims against the actual test
     file's current assertions/steps — e.g. "verifies all 6 steps" when
     the test now checks a different number, a described scenario or
     selector that no longer appears in the test, or a filePath that no
     longer matches an existing test file.
4. Do NOT flag subjective or interpretive language ("this is the brain
   of the app", "simple, read-focused") — only factual claims.
5. For each flagged waypoint, report:
   - Waypoint label and id
   - The outdated claim (quote it)
   - What's actually true now, with a specific reference to the file/line
   - The exact corrected field value(s), ready to paste
6. Present flagged waypoints one at a time, not as a batch list. For
   the first one, report it in this format, with nothing else asked or
   implied:

   ```
   [id] label
      Field: <field name>
      Change to: <exact new value>
   ```

   Then stop and wait — do not print the next flagged waypoint yet.
7. When the user comes back (e.g. "next", "ok done, what's next",
   "back to do wNN"), present the next flagged waypoint in the same
   format. Repeat until all flagged waypoints have been presented, then
   say so explicitly ("that's all of them").
8. Do not ask which ones to update, and do not offer to apply any of
   them yourself. This skill never edits waypoints — local or
   production. Each fix is applied by the user, by hand, in the app's
   own Edit Waypoint form, at whatever pace they choose.

## Feature filePath existence check (separate from the drift audit above)

Features don't have one canonical implementation file the way Code
Components do, so there's nothing reliable to diff a Feature's
description against. This check is narrower and purely mechanical: does
the referenced file exist, yes or no. It is not a judgment about
whether the description is still accurate, and it runs and reports
independently of the numbered, one-at-a-time drift findings above.

1. Fetch all waypoints of type "Feature".
2. For each one, check whether its filePath is empty/null or points to
   a file that exists on disk. Do not read or evaluate the file's
   contents — existence only.
3. Report all Feature waypoints together in one block (not one at a
   time, and not mixed into the drift-audit list), in this format:

   ```
   Feature filePath existence check:
   - <id> "<label>" → filePath: <path or "(none)"> → <MISSING | OK | OK, no spec file expected>
   ```

   Use `MISSING` when a filePath is set but the file doesn't exist,
   `OK` when a filePath is set and the file exists, and `OK, no spec
   file expected` when filePath is empty/null.
4. As with the rest of this skill: report only. Never write to the
   waypoint data, and never offer to create the missing file or edit
   the waypoint.

## Never

- Write to the map in any way: no POST, PUT or DELETE to the site, no
  edits to `prisma/*.db`, no scripts that do either.
