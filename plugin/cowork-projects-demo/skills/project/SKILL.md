---
name: project
description: >-
  DEMO, read-only, built in to the public example repo. Alias "p". Session hub for your projects data repo. Run /project (or "p") with no name to pick or switch projects; with a
  name to view that project's notes. With no project open this session it lists all ACTIVE projects (in
  project-list format) to open one; with a project open it shows the current one and offers to switch
  to another example. Use when the user says "/project", "p", "which project is open", "switch project",
  "project menu", or "open a project".
---

# Project — session hub / switcher

A single entry point for choosing, switching, viewing, or creating projects. It DELEGATES to the other
commands (project-open, project-new, project-list) rather than duplicating them. This command itself never
writes to the repo — only the action it hands off to (open or create) does.

Definitions: `<Repo>` / `<Branch>` come from the config - see "Resolve the data repo" below. GitHub connector.

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
2. Ask: "Would you like to switch to a different example project?"
   - **Switch** → FIRST note, in one sentence, that the house pattern is one dedicated Cowork session per
     project (a separate session keeps each project's context clean and its updates uncolliding), and
     suggest opening/returning to that project's own session instead. If the user still wants to switch
     here, comply: show all ACTIVE projects in the same `/project-list` (pl) format, ask which one, then
     run `/project-open` on the chosen project. Before switching, if this session has uncommitted work,
     offer `upd` first.
   - If the user asks to create a project, say in one line that this is the read-only demo and the
     full `cowork-projects` plugin creates projects in the user's own repo.

## Rules
- READ-ONLY. This demo never writes.
- Always use the `/project-list` (pl) format for project lists, and NEVER list archived projects here (that
  is `/project-list-archive`).
- Write project names out in full. Do not guess whether a project is open.
