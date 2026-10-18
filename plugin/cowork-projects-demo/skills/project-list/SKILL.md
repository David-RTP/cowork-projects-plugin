---
name: project-list
description: >-
  DEMO, read-only, built in to the public example repo. Alias "pl". Show the cached list of ACTIVE projects from the master list file (Project Setup
  folder). Fast single-file read - does not scan every project, and does not list archived projects
  (use project-list-archive for those). Use when the user says "list projects", "show my projects",
  "what projects do I have", "project list", "pl", or invokes /project-list.
---

# Project List (fast, cached, active only)

Show the ACTIVE project index from the master list file. This is a fast single-file read - it does NOT
scan every project's memory doc (that is what `/project-list-update` is for), and it does NOT list
archived projects (that is what `/project-list-archive` / `pla` is for).

## DEMO DATA REPO - BUILT IN, READ-ONLY (replaces config discovery)
This is the DEMO build of the cowork-projects plugin. Its data repo is fixed: `David-RTP/cowork-projects-examples`,
branch `main` - a public repo of example projects.
- Do NOT call `get_me`, do NOT search, and do NOT ask which repo. Fetch
  `Project Setup/cowork-projects.config.json` from that repo and use its `owner`/`repo`/`branch`/
  `timezone` as defined below. If that fetch fails, say so in one line and stop - never fall back
  to any other repo.
- **READ-ONLY.** Never call a GitHub write tool (`create_or_update_file`, `push_files`,
  `create_branch`, `delete_file`, or any other) against any repo, even where a rule below says to
  write. The visitor has no write access to the example repo, and the demo never writes to the
  visitor's own repos either.
- If the user asks for anything that writes - a new project, archive, unarchive, rebuilding the
  master list, setup - say in one line that this is the read-only demo and that the full
  `cowork-projects` plugin does it in the user's own repo. `upd` is available as a PREVIEW: it
  shows what would be written and commits nothing.

Definitions used below: `<Repo>` = `owner/repo` from the config; `<Branch>` = the config `branch`
(default `main`); `<DriveFolder>` = the config `drive_folder` (default `AI Projects`).

Master list file: `Project Setup/Project Setup - Master List.md`. It contains, in order:
1. A `**Last Refresh:**` date line.
2. An ACTIVE table: `Project | Open Qs | Latest change | Goal`.
3. A one-line archived count: `**Archived:** N project(s) not listed above ...`.
4. An `## Archived Projects` section with the archived table (read by `/project-list-archive`, NOT here).

Rows prefixed `📓` are **Learning Notebooks** rather than projects - topic-based reference notebooks
with no finish line. They share the table and the columns, but two columns read differently for them:
Open Qs shows open To Learn threads (written `N to learn`, so the cell says what it is) and Goal shows
the notebook's subject instead of a goal. Render them as they are written; the meaning is already in
the cell. `/project-list-update` (plu, full plugin only) owns these rules - never recompute a row here.

## Steps
1. Fetch `Project Setup/Project Setup - Master List.md` from `<Repo>` (`<Branch>`) via the GitHub connector.
2. **If it exists:** display ONLY the active table, then state its `Last Refresh` date, then show the
   archived-count line verbatim (e.g. "🗄️ 1 Archived project not listed - run /project-list-archive (pla)
   to see it"). Do NOT print the `## Archived Projects` table here. NOTE: the "Latest change" column is a
   cached snapshot from that last refresh - a project may have a newer changelog entry since. Mention they
   can run `/project-list-archive` (pla) to see archived projects,
   or `/project-open <name>` to open one.
3. **If it does NOT exist (or is empty/malformed):** tell the user the example index is missing and
   STOP. The demo never rebuilds it.
   *(Full-plugin text, not used by the demo:)* **THIS BRANCH IS
   WHERE `pl` STOPS BEING PROVABLY SMALL, SO THE CLONE PREFLIGHT FIRES HERE - before the rebuild's first
   read, not at the top of this command.** Steps 1-2 are a one-file cached read and are exempt by being
   small; this branch scans every project folder and cannot know its file count until it looks. **Before
   this branch reads anything further from the repo - and REGARDLESS of what this session has already
   read - run the clone preflight defined in `../../docs/CONVENTIONS.md` section 14. Reaching this branch
   IS the condition; a repo you believe is small does not discharge it.** If this session cannot make a git
   clone, STOP, say so, name the work, say it wants a session that can, and let the caller decide - there
   is no fallback, and "it is only a list" is not one. Only once the caller has ruled, BUILD it now
   by running the `/project-list-update` procedure (scan every project folder, write the file), and show the
   result. This first build is the slow one; every later `/project-list` is a fast read.

A project's memory doc may be **single-file** or **hub-and-spoke** (the doc is a hub and the decisions
and open questions live in indexed companion files). That distinction changes where `plu` counts open
questions from - it does NOT change this table, which has the same four columns either way. There is
no layout marker in the rows and none is wanted. An `Open Qs` cell of `?` means `plu` could not
compute that count and said so at rebuild time; render it as written and do not substitute a zero.

## Rules
- READ-ONLY. This demo never writes, not even a missing index (step 3).
- Show exactly what the master list contains - do not invent projects or details.
- Never list archived projects here; only surface the count and point to `/project-list-archive`.
- **Never recompute a cell here, for any reason** - not the open-question count, not the latest
  change, not the goal. This command reads a cache. If a value looks wrong or stale, say so and
  point at `/project-list-update` (plu, full plugin only); recomputing one cell from a different layout's rules is how
  a wrong number gets laundered into looking authoritative.
- The cached numbers are as of the `**Last Refresh:**` date. State that date every time
  (`../../docs/CONVENTIONS.md` section 7) - an undated count reads as a claim about right now.
  *(That path resolves from THIS skill's own directory: the conventions file sits at the plugin
  root, two levels above this file. A bare `docs/CONVENTIONS.md` does NOT resolve from a skill
  directory - it names a file that is not there.)*
