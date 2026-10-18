---
name: project-list-archive
description: >-
  DEMO, read-only, built in to the public example repo. Alias "pla". Show the cached list of ARCHIVED projects from the master list file (Project Setup
  folder), in the same format as project-list. Fast single-file read. Use when the user says "list
  archived projects", "show archived projects", "archived list", "pla", or invokes /project-list-archive.
---

# Project List - Archived (fast, cached)

Show the ARCHIVED project index - the counterpart to `/project-list` (`pl`), which shows only active
projects. Same fast single-file read; same table format.

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

Master list file: `Project Setup/Project Setup - Master List.md`. The archived projects live under its
`## Archived Projects` heading as a table `Project | Open Qs | Latest change | Goal`.

## Steps
1. Fetch `Project Setup/Project Setup - Master List.md` from `<Repo>` (`<Branch>`) via the GitHub connector.
2. **If it exists:** display ONLY the `## Archived Projects` table, in the same format `/project-list` uses,
   and state the `Last Refresh` date. If there are zero archived projects, say so plainly.
3. **If it does NOT exist (or has no archived section):** tell the user the index is missing or has never
   been rebuilt. In this demo, stop there.

## Rules
- READ-ONLY. Never write from this command.
- Show exactly what the master list's archived section contains - do not invent projects or details.
- The "Latest change" column is a cached snapshot from the last refresh; the full plugin's `plu` refreshes it.
- The cached numbers are as of the `**Last Refresh:**` date. State that date every time
  (`../../docs/CONVENTIONS.md` section 7) - an undated count reads as a claim about right now.
  That bites harder here than in `/project-list`: an archived project is not being updated, so
  its cached row is the staler of the two lists by construction.
  *(That path resolves from THIS skill's own directory: the conventions file sits at the plugin
  root, two levels above this file. A bare `docs/CONVENTIONS.md` does NOT resolve from a skill
  directory - it names a file that is not there.)*
