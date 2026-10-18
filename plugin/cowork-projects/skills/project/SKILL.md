---
name: project
description: >-
  Alias "p". Session hub for your projects data repo. Run /project (or "p") with no name to pick or switch projects; with a
  name to view that project's notes. With no project open this session it lists all ACTIVE projects (in
  project-list format) to open one; with a project open it shows the current one and offers to switch or
  create a new project. Use when the user says "/project", "p", "which project is open", "switch project",
  "project menu", or "open a project".
---

# Project — session hub / switcher

A single entry point for choosing, switching, viewing, or creating projects. It DELEGATES to the other
commands (project-open, project-new, project-list) rather than duplicating them. This command itself never
writes to the repo — only the action it hands off to (open or create) does.

Definitions: `<Repo>` / `<Branch>` come from the config - see "Resolve the data repo" below. GitHub connector.

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

**"A project is OPEN"** means one was loaded earlier THIS session (via `/project-open` or a prior `/project`
switch). If none was loaded this session, treat it as no project open. NEVER guess that a project is open —
if genuinely unsure, ask.

## Behavior

### A. A name was given — `/project <Project Name>`
Show that project's NOTES (a read-only peek; this does NOT change the open/working project):
1. Fetch `<Folder>/Project Notes - <Project Name>.md` (if the plain folder misses, try
   `zArchived - <Project Name>` — the project may be archived).
2. Display the notes file. If it is long, show the most recent dated sections and offer the rest.
3. Mention they can run `/project-open <Project Name>` (po) to load it for a working session.

### B. No name AND no project is open this session
List the ACTIVE projects so the user can open one:
1. Read `Project Setup/Project Setup - Master List.md` and show the ACTIVE table EXACTLY as `/project-list`
   (pl) does — the active table plus the "N archived not listed" line. Do NOT list archived projects.
2. Ask: "Which would you like to open?"
3. When the user names one, run the `/project-open` procedure on it (fetch its memory doc; report the latest
   changelog entry + Status).

### C. No name AND a project IS open this session
1. Show the CURRENT project:
   - Its **Name** and **Status** (from the memory doc's `**Status:**` line), then
   - Below that, the **latest Session Notes**: the most recent dated section's TITLE and its paragraph, read
     from `Project Notes - <current>.md` (most recent on top).
2. Ask: "Would you like to switch to a different project, or create a new one?"
   - **Switch** → FIRST note, in one sentence, that the house pattern is one dedicated Cowork session per
     project (a separate session keeps each project's context clean and its updates uncolliding), and
     suggest opening/returning to that project's own session instead. If the user still wants to switch
     here, comply: show all ACTIVE projects in the same `/project-list` (pl) format, ask which one, then
     run `/project-open` on the chosen project. Before switching, if this session has uncommitted work,
     offer `upd` first.
   - **Create a new one** → run the `/project-new` (pn) procedure to scaffold a new project.

## Rules
- READ-ONLY itself; the only writes come from the delegated `/project-open` (none) or `/project-new` (creates).
- Always use the `/project-list` (pl) format for project lists, and NEVER list archived projects here (that
  is `/project-list-archive`).
- Write project names out in full. Do not guess whether a project is open.
