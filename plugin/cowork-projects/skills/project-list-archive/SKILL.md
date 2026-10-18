---
name: project-list-archive
description: >-
  Alias "pla". Show the cached list of ARCHIVED projects from the master list file (Project Setup
  folder), in the same format as project-list. Fast single-file read. Use when the user says "list
  archived projects", "show archived projects", "archived list", "pla", or invokes /project-list-archive.
---

# Project List - Archived (fast, cached)

Show the ARCHIVED project index - the counterpart to `/project-list` (`pl`), which shows only active
projects. Same fast single-file read; same table format.

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

Master list file: `Project Setup/Project Setup - Master List.md`. The archived projects live under its
`## Archived Projects` heading as a table `Project | Open Qs | Latest change | Goal`.

## Steps
1. Fetch `Project Setup/Project Setup - Master List.md` from `<Repo>` (`<Branch>`) via the GitHub connector.
2. **If it exists:** display ONLY the `## Archived Projects` table, in the same format `/project-list` uses,
   and state the `Last Refresh` date. If there are zero archived projects, say so plainly.
3. **If it does NOT exist (or has no archived section):** tell the user the index is missing or has never
   been rebuilt, and suggest running `/project-list-update` (plu) to build it.

## Rules
- READ-ONLY. Never write from this command.
- Show exactly what the master list's archived section contains - do not invent projects or details.
- The "Latest change" column is a cached snapshot from the last refresh; mention `/project-list-update`
  (plu) refreshes it.
- The cached numbers are as of the `**Last Refresh:**` date. State that date every time
  (`../../docs/CONVENTIONS.md` section 7) - an undated count reads as a claim about right now.
  That bites harder here than in `/project-list`: an archived project is not being updated, so
  its cached row is the staler of the two lists by construction.
  *(That path resolves from THIS skill's own directory: the conventions file sits at the plugin
  root, two levels above this file. A bare `docs/CONVENTIONS.md` does NOT resolve from a skill
  directory - it names a file that is not there.)*
