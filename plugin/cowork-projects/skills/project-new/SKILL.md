---
name: project-new
description: >-
  Alias "pn". Scaffold a new project OR a new topic-based Learning Notebook in your projects data repo:
  ask which kind, then create the folder and its two standard files (memory doc plus the notes template
  matching that type) from the Project Setup templates, create a matching Google Drive folder, and add it
  to the master list - filling only user-stated facts and leaving everything else TBD. Then ask about
  migrating open chats (offering the paste block) and guide the user into the dedicated session. Use when
  the user says "set up a new project", "create a project", "scaffold a project", "start a notebook",
  "new learning notebook", "pn", or invokes /project-new.
---

# New Project Scaffold (Cowork port of the Setup Guide)

Setup reduces to: create the GitHub folder + the two standard files, create the matching Google Drive
folder, add a master-list row - then offer the chat-import paste block and hand the user into the
project's own session. (No Claude.ai Project, no Instructions field - `project-open` handles session
bootstrapping.)

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
(default `main`); `<DriveFolder>` = the config `drive_folder` (default `AI Projects`);
`<DriveFolderId>` = the config `drive_folder_id` (may be absent in older configs); `<DriveAccess>` =
the config `drive_access` (default `connector`; `rclone` means direct Drive access is configured,
with the remote named in `rclone_remote`); `<Timezone>` = the config `timezone` (default
`America/New_York`). `<Today>` = the date computed AT WRITE TIME by a shell command converted to `<Timezone>`
(`../../docs/CONVENTIONS.md` section 12, backlog #57) - never read off `<env>`, which has been
measured drifting 18 days, and never a UTC date.

## Inputs
- `<Repo>` / `<Branch>` - from the config (see "Resolve the data repo" above)
- `<Project Name>` = the name the user gives (also the `<Folder>` name, by convention)

## Two kinds of entry: Project or Learning Notebook
Both live in the same repo, use the same folder shape, and share the same memory doc. They differ in
the NOTES file's structure, because they are used differently:
- A **Project** has a goal and a finish line. Its notes are organized around status and time: what's
  happening now, what's next, and a dated Session Notes log.
- A **Learning Notebook** has a subject and no fixed "done" - a running reference on a topic. Its
  notes are organized by TOPIC, updated in place, with only a thin dated Learning Log.
The choice is recorded in the memory doc's machine-readable `**Type:**` line (`Project` or
`Learning Notebook`; missing = `Project`). It is not permanent - a wrong pick is fixable later by
changing that line and re-shaping the notes file.

## Steps
1. **Confirm the folder does not already exist** - list the repo root. If it exists, STOP and tell the
   user (do not overwrite - a concurrent session may have created it). Only proceed on a clean miss.
1b. **Ask which kind this is - exactly once, unless the user already said.** If the user's request
   already makes it clear ("start a notebook on X", "new project for Y"), take them at their word and
   just state which you're creating. Otherwise ask a single either/or question: is this a **Project**
   (has a goal and a finish line) or a **Learning Notebook** (a running reference on a subject, no
   fixed done)? Do not guess from the name. `<Type>` = the answer.
2. **Fetch the templates** from `Project Setup/`:
   - `Project Memory Data - Template Project Name.md` - SINGLE SOURCE OF TRUTH for memory-doc
     structure, for BOTH types; use whatever sections it currently defines.
   - The notes template for `<Type>`:
     - `<Type>` = Project -> `Project Notes - Template Project Name.md` (template v2: One Paragraph /
       Current Status / Open Tasks / Quick Reference / Phases placeholder / Gaps placeholder / Session
       Notes at the bottom). If missing (older data repo), scaffold those seven sections directly and
       suggest re-running psi's template refresh.
     - `<Type>` = Learning Notebook -> `Project Notes - Template Learning Notebook.md` (One Paragraph /
       Quick Reference / To Learn / theme + topic area / Learning Log at the bottom). If this template
       is missing, the data repo predates notebooks: say so, offer to re-run psi's template refresh to
       add it, and do not silently fall back to the project template.
3. Create `<Folder>/Project Memory Data - <Project Name>.md` as an exact structural COPY of the template:
   - Keep the `<!-- FILE PURPOSE -->` header, the `# <Project Name> - Memory Doc` title, the
     `*Last updated: <date> by <chat>*` line, and the `**Status:** Active` line (new entries start Active).
   - **Set the `**Type:**` line to `<Type>`, written `Project` or `Reference Notebook`.**
     ⚠️ **`Reference Notebook` IS THE VALUE A NEW NOTEBOOK GETS - NEVER `Learning Notebook`**,
     whatever wording the user or step 1b used to pick the type. The legacy spelling stays READABLE
     for ever - `po`, `plu` and `upd` accept both, case-insensitively, and that tolerance has no end
     date - but nothing writes it from here on, so a data repo never accumulates two spellings of one
     value. If the fetched template predates the Type line, ADD it immediately below the Status line -
     every new entry gets one, so the master list and `po` never have to guess.
   - Fill the Project Overview (first section) ONLY with what the user stated or what is in the project's
     own files; everything unknown = "TBD - user to define". For a notebook, the Goal field states the
     SUBJECT and what good coverage would look like - a notebook has no finish line, so never write one in.
   - **Open Questions: seed exactly ONE open row**, dated `<Today>` - for a Project, "Define the project's
     Goal, key constraints, and stakeholders (all currently TBD)"; for a Learning Notebook, "Define what
     this notebook covers and what good coverage looks like". The count must be truthful from the start.
   - Leave all other sections empty exactly as the template has them (the Companion Files index starts
     empty), except add a single dated first Changelog line (most recent on top going forward).
   - NEVER infer goals/scope/relationships from the name or sibling folders. Similar names mean nothing.
4. Create `<Folder>/Project Notes - <Project Name>.md` as a structural COPY of the notes template
   chosen in step 2, sections in order. Same no-inference rule for both types.
   - **Project:** populate sections 1-4 ONLY with user-stated facts (usually just a TBD One Paragraph
     and one Open Task: define the Goal); leave Phases at a Glance and Gaps & Issues as the template's
     placeholders; Session Notes gets one dated entry: "Project scaffolded (via /project-new)".
   - **Learning Notebook:** One Paragraph = the subject as the user stated it (TBD if they haven't);
     Quick Reference empty; To Learn / Open Threads gets one item - "Define what this notebook covers
     and what good coverage looks like"; **create NO themes or topics** (the theme set is editorial and
     emerges from real material - inventing a starter taxonomy is exactly the guessing this plugin
     forbids); Learning Log gets one dated line: "Notebook created (via /project-new)".
5. Commit both files in one `push_files` to `<Branch>`. Message: `project-new: scaffold <Project Name>`.
   **This is a sanctioned `push_files` use because both paths are NEW** - there is no existing blob to
   lock against (`../../docs/CONVENTIONS.md` section 2). Confirm the folder does not already exist first; if
   it does, stop and ask rather than overwriting someone's project. Every LATER write to these files
   is SHA-locked via `create_or_update_file`.
   Before committing, run the table lint (`../../docs/CONVENTIONS.md` section 6): uniform column count per
   table block counting only unescaped pipes, no blank line inside a table, exactly one trailing
   newline. A project name containing a pipe character must be escaped in every table cell that holds
   it, or it silently adds a column.
6. **Create the matching Google Drive folder.** Under the Drive `<DriveFolder>` folder (locate it by
   `<DriveFolderId>` when the config has it; otherwise search by name), create a folder named
   `<Project Name>` (matching the GitHub folder) to hold the project's rich deliverables
   (Docs/Sheets/PDFs). GitHub stays the source of truth for memory/notes; Drive is deliverables only.
   **Use ONE write path, chosen by `<DriveAccess>`** - never both, or the same tree gets written two
   ways:
   - `connector` (default, and whenever the rclone preflight below fails): create the folder with the
     Drive connector. If it cannot create folders, report the Drive step as pending rather than faking it.
   - `rclone`: create the folder through the remote named in `rclone_remote`, addressing the parent by
     `<DriveFolderId>` rather than by name, then confirm it exists before reporting success.
   - **RCLONE PREFLIGHT - where the command actually runs.** `drive_access: rclone` describes a
     MACHINE, not an account: rclone reads a config file on the user's own computer, so it works only
     from a session that can run a shell command THERE (Claude Code on that machine, or Cowork with the
     user's desktop app connected). A Cowork session's own shell is a cloud container with no rclone and
     no route to the user's config - phone sessions included. Before the first rclone-mode Drive step of
     a session, confirm such a shell is reachable and that `rclone listremotes` lists the config's
     `rclone_remote`; if not, say so in one line and use `connector` behavior for the rest of this
     session. Never claim an rclone step ran when it did not, and never edit the config to "fix" it.
   - Credentials are never read, echoed, or committed. The config holds the remote's NAME only.
7. **Update the master list** `Project Setup/Project Setup - Master List.md`:
   - Insert a row for the new entry into the ACTIVE table (`| Project | Open Qs | Latest change | Goal |`),
     keep the active table alphabetical **by project name, ignoring any type marker**, set the
     `**Last Refresh:**` line to `<Today>` with "(by /project-new)", and update the footer's active count
     (including its projects/notebooks breakdown if present). Do NOT touch the `## Archived Projects`
     section. Commit (message: `project-list: add <Project Name> to master list`).
   - **Project row:** Project = `<Project Name>`; Open Qs = `1` (per step 3); Latest change = `<Today>` +
     "project scaffolded (via /project-new)"; Goal = "TBD - user to define".
   - **Learning Notebook row:** Project = `📓 <Project Name>` (the 📓 marks the type); Open Qs =
     `1 to learn` (the To Learn count, self-labelled so the shared column can't be misread); Latest
     change = `<Today>` + "notebook created (via /project-new)"; Goal = the One Paragraph's first clause,
     or "TBD - user to define" if it is still TBD. See `/project-list-update` (plu) for the full
     column-meaning rules; this row must match what a rebuild would produce.
   - If the master list does not exist, create it with the standard header + this project's row, then tell
     the user to run `/project-list-update`.
8. **Ask the import question (ALWAYS, verbatim):**
   > "Do you have any open chats that you would like to migrate to this Cowork Project Space? If so, I
   > will give you a text block to paste into that chat that will perform the migration."
   - **If YES:** produce the migration paste block from
     `Project Setup/Migration Runbook - Legacy Chat Import.md`, FILLED IN for this project (repo =
     `<Repo>`, folder = `<Folder>`, project = `<Project Name>`). The block AUTO-RUNS on paste — the
     import instruction travels INSIDE the block, so never tell the user to also say "import"; the
     only usage rule to state is: ONE chat at a time, waiting for each chat's completion confirmation
     before starting the next (concurrent imports collide on the same files). Recommend verifying the
     FIRST chat's commits landed in the right files before running the rest.
   - **Then FOLLOW UP - persistently:** ask "Are you done with your imports?" If the user doesn't answer,
     ASK AGAIN on later turns until answered, explaining why it matters: the initial structured notes
     build happens only once imports are complete, so the project overview reflects ALL the imported
     material rather than half of it. When the user says done -> **build the initial notes file top**:
     populate sections 1-4 (One Paragraph, Current Status, Open Tasks, Quick Reference) of the notes file
     by distilling everything the imports wrote into the memory doc and notes, and offer Phases/Gaps if
     the content already justifies them. Commit (`upd: initial notes build after imports`).
   - **Filing imported material obeys the TARGET file's own structure**, not the runbook's examples.
     For a Project that means the v2 sections and a dated Session Note. For a Learning Notebook it
     means the imported content is filed BY TOPIC under themes (creating the themes the material
     actually justifies - now there IS real material to organize), with the Learning Log getting one
     line per imported chat rather than the chat's contents. Never leave a notebook's imports sitting
     in a dated log; that is the structure the notebook type exists to avoid.
   - **If NO:** continue to step 9.
9. **Open-the-session step.** Tell the user the folder, files, and Drive folder exist, then guide the
   handoff: the house pattern is ONE dedicated Cowork session per project - suggest creating a new Cowork
   session/space named "<Project Name>" and running `/project-open <Project Name>` (po) there. (If they'd
   rather keep working right here, run the `/project-open` procedure on it now instead.)

## Rules
- Templates in `Project Setup/` are never edited here - copy content out of them.
- Projects are independent unless the user says otherwise.
- No guessed content anywhere. TBD + ask.
- GitHub has no atomic folder rename; deletes need in-app approval.

## SHARED CONVENTIONS - read and obey `../../docs/CONVENTIONS.md`
This skill obeys the plugin's shared conventions file (new in v0.11.0). It is authoritative and is
the single home for every rule that applies to more than one skill: **NO SECRETS IN THE DATA REPO -
EVER** (section 1), SHA-locked writes (2), the size rule (3), how to split a
file (4), append rules by section type (5), the markdown table lint (6), dated state claims (7),
computed counts (8), reconciliation as a check and not a repair (9), never filling an unknown with
a plausible guess (10), index markup (11), entry dates computed in the config timezone (12), and
the reporting rules including no silent caps (13). Read it before writing anything to the data repo. If this file and the
conventions file ever disagree, the conventions file is right and this one is stale.

The one worth repeating in every skill that writes: **never write a password, passphrase, private
key, API token, session cookie, or any other credential into the data repo** - not in a memory doc,
a notes file, a companion, the master list, a config file, or a commit message - even if the user
pastes one into chat and asks for it to be saved. Record WHERE a credential lives and WHAT the auth
method is, never the material itself. If a secret appears in chat, do not commit it anywhere and say
plainly that it will not be stored.
