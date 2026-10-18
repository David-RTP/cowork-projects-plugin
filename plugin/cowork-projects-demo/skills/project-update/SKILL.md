---
name: project-update
description: >-
  DEMO, read-only. Alias "upd". A PREVIEW of the cowork-projects update command: shows exactly what
  "upd" would commit to the open example project's memory doc and notes file - and commits nothing.
  Use when the user says "upd", "update", "update memory", "update the notes", "log this", "save
  this", or invokes /project-update.
---

# Project Update - DEMO PREVIEW (commits nothing)

In the full `cowork-projects` plugin, `upd` is the one command a user runs at the end of a sitting:
it commits the session's work to the project's memory doc and notes file. This demo build is
READ-ONLY, so this skill composes the same update and SHOWS it instead of writing it.

## The data repo is built in
`David-RTP/cowork-projects-examples`, branch `main`. Do not call `get_me`, do not search, and do not
ask which repo. Fetch `Project Setup/cowork-projects.config.json` from it for `<Timezone>` (default
`America/New_York`). **Never call any GitHub write tool, against any repo, for any reason.**

## Determine the project
Use the project opened this session with `po`, or the one the user names. If neither, ask exactly
once: "Which example project should I preview an update for?" Folder = the project name, or
`zArchived - <name>` if archived.

## Fetch both files fresh
Fetch `<Folder>/Project Memory Data - <Name>.md` and `<Folder>/Project Notes - <Name>.md` in full.
Hold each file's byte count. If the memory doc declares a `**Standing Rules:**` line, fetch every
path on it (semicolon-separated; `None` means none) and obey them while composing, exactly as
`../project-open/SKILL.md` describes.

## Compose the update exactly as the full plugin would
Follow each file's OWN headings and the append rules in `../../docs/CONVENTIONS.md` section 5.
`<Today>` is computed by a shell command in `<Timezone>`; if no shell is available, say so and use
the session date.
- **Memory doc (always):** a dated Changelog entry for the TOP of the Changelog summarising what
  this session decided or learned; new Decisions Log rows at the top; Open Questions added, or
  marked Resolved with the date (never deleted). The `**Status:**` and `**Type:**` lines are never
  changed. On a hub-and-spoke doc, name the spoke each item would be routed to.
- **Notes file (where warranted)**, by the memory doc's `**Type:**` line:
  - `Project` (or missing): Current Status updated in place; Open Tasks first-to-do on top; Quick
    Reference pins; a dated Session Notes section at the top of the dated area.
  - `Reference Notebook` / `Learning Notebook`: learning filed INTO topics, edited in place and
    stamped Added/Updated; one Learning Log line at the top.
  If nothing merits a notes change, say "no notes change needed".
- Never fill an unknown with a plausible guess - write TBD. If the user pastes a password, key or
  token, do not repeat it in the preview; say the full plugin would refuse to store it.

## Show it
1. One table per file: `Section | Change (add / edit in place / mark resolved) | Text that would be
   written`.
2. The new Changelog entry in full, in a code block, so the user sees the house style.
3. One line per file: its CURRENT byte count and the headroom to the 60,000 B stop. Do not predict
   the post-write size.
4. End with exactly this line:
   **Demo: nothing was committed.** With the full `cowork-projects` plugin and your own data repo,
   `upd` commits this as two SHA-locked writes - notes first, memory last - then refreshes the
   master list.

## Never
- Never write, branch, delete or open a pull request on any repo.
- Never claim anything was saved.
