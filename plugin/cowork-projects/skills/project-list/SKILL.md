---
name: project-list
description: >-
  Alias "pl". Show the cached list of ACTIVE projects from the master list file (Project Setup
  folder). Fast single-file read - does not scan every project, and does not list archived projects
  (use project-list-archive for those). Use when the user says "list projects", "show my projects",
  "what projects do I have", "project list", "pl", or invokes /project-list.
---

# Project List (fast, cached, active only)

Show the ACTIVE project index from the master list file. This is a fast single-file read - it does NOT
scan every project's memory doc (that is what `/project-list-update` is for), and it does NOT list
archived projects (that is what `/project-list-archive` / `pla` is for).

## Resolve the data repo FIRST (config-driven - never hardcoded)
Every command operates on the user's projects DATA repo - the GitHub repo holding the project folders -
identified by the file `Project Setup/cowork-projects.config.json` inside that repo:
- If the data repo was already resolved earlier THIS session (or the user names one), use it.
- Else discover it: `get_me` for the user's GitHub login, then `search_code` for
  `filename:cowork-projects.config.json user:<login>`. Exactly one hit -> fetch that config and use its
  `owner`/`repo`/`branch` (tell the user which repo was resolved). Multiple hits -> ask which one.
- **ZERO HITS IS NOT PROOF OF ABSENCE — fall back to a repository search** (new in v0.12.0, backlog #18). `search_code` returns nothing for a PRIVATE repo, reproduced six times, most recently by the very session that shipped this rule. Since a data repo is normally private, the empty result is the COMMON case rather than the missing case, and treating it as "no config" sends the skill down the wrong branch: it offers to bootstrap a repo the user already has. So on zero hits, do NOT ask yet — run `search_repositories` for `user:<login>`, then probe `Project Setup/cowork-projects.config.json` with `get_file_contents` in each candidate, most-recently-updated first. Exactly one repo holds the file -> use it and say which. More than one -> ask which. Only after that probe comes back empty is the config genuinely absent.
- Still no hit -> ask ONCE for `owner/repo`. If that repo has no `Project Setup/` folder, offer to run `/project-setup-init` (psi) to bootstrap it. Never guess, and never fall back to a hardcoded repo.

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
the cell. `/project-list-update` (plu) owns these rules - never recompute a row here.

## Steps
1. Fetch `Project Setup/Project Setup - Master List.md` from `<Repo>` (`<Branch>`) via the GitHub connector.
2. **If it exists:** display ONLY the active table, then state its `Last Refresh` date, then show the
   archived-count line verbatim (e.g. "🗄️ 1 Archived project not listed - run /project-list-archive (pla)
   to see it"). Do NOT print the `## Archived Projects` table here. NOTE: the "Latest change" column is a
   cached snapshot from that last refresh - a project may have a newer changelog entry since. Mention they
   can run `/project-list-update` (plu) to refresh, `/project-list-archive` (pla) to see archived projects,
   or `/project-open <name>` to open one.
3. **If it does NOT exist (or is empty/malformed):** tell the user the index is missing. **THIS BRANCH IS
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
- Default behavior is READ-ONLY. Only write when the file is missing and you are building it for the first
  time (step 3).
- Show exactly what the master list contains - do not invent projects or details.
- Never list archived projects here; only surface the count and point to `/project-list-archive`.
- **Never recompute a cell here, for any reason** - not the open-question count, not the latest
  change, not the goal. This command reads a cache. If a value looks wrong or stale, say so and
  point at `/project-list-update` (plu); recomputing one cell from a different layout's rules is how
  a wrong number gets laundered into looking authoritative.
- The cached numbers are as of the `**Last Refresh:**` date. State that date every time
  (`../../docs/CONVENTIONS.md` section 7) - an undated count reads as a claim about right now.
  *(That path resolves from THIS skill's own directory: the conventions file sits at the plugin
  root, two levels above this file. A bare `docs/CONVENTIONS.md` does NOT resolve from a skill
  directory - it names a file that is not there.)*
