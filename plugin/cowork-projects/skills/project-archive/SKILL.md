---
name: project-archive
description: >-
  Alias "pa". Archive a project in your projects data repo: set its memory doc Status to Archived with a banner,
  rename its GitHub folder (and its Google Drive folder) with a "zArchived - " prefix, then ALWAYS
  rebuild the master list. Use when the user says "archive a project", "archive this project", "pa", or
  invokes /project-archive.
---

# Project Archive (pa)

Retire a project without deleting anything: mark it Archived, prefix its folders with `zArchived - `, and
rebuild the master list. History is preserved - never delete content, only mark and relocate. (Reverse
operation: `/project-unarchive` / pua.)

## CLONE PREFLIGHT - run this BEFORE THIS COMMAND'S FIRST REPO READ
**Before this command reads anything from the repo - the config read below included, and
REGARDLESS of what this session has already read - run the clone preflight defined in
`../../docs/CONVENTIONS.md` section 14. It is unconditional, and a repo you believe is small
does not discharge it.** If this session cannot make a git clone, STOP, say so, name the work,
say it wants a session that can, and let the caller decide. There is no fallback.

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

## Determine the project
- Use the project the user names (or the one loaded this session). If ambiguous, ask exactly once:
  "Which project should I archive?" Do not guess. Write the name out in full.
- `<Repo>` / `<Branch>` - from the config (see "Resolve the data repo" above). `<Project Name>` == the original (un-prefixed) name.

## HARD EXEMPTION - never archive-rename "Project Setup"
`Project Setup` is the meta/system folder: it holds the templates, the Setup Guide, the cached master list,
AND this plugin's own source. The plugin hardcodes those paths. NEVER prepend `zArchived - ` to the
`Project Setup` folder - doing so breaks every command. If asked to archive Project Setup, STOP and explain
this. (Project Setup should stay Active.)

## Steps
1. **Confirm intent.** Restate what archiving does (Status -> Archived, folder(s) renamed `zArchived - <Name>`,
   row moved to the archived list) and confirm the user wants it. This is a structural change; don't do it silently.
2. **Mark the memory doc Archived (in place, before moving).** Re-fetch `<Folder>/Project Memory Data -
   <Project Name>.md`, then:
   - Set the `**Status:**` line to `Archived`.
   - Add a short ARCHIVED banner near the top (a blockquote) stating the project is archived, the date, and
     where its work moved if the user said so.
   - Append a dated Changelog entry (most recent on top) and a Decisions Log row recording the archive.
     Never delete prior text.
   - Commit SHA-locked: `create_or_update_file` with the memory doc's freshly fetched blob SHA
     (`../../docs/CONVENTIONS.md` section 2). Message: `upd: archive <Project Name>`. Never `push_files` a
     file that already exists.
3. **Rename the GitHub folder to `zArchived - <Project Name>`.** GitHub has no atomic folder rename:
   - Recreate every file in the folder at the new path `zArchived - <Project Name>/...` (same filenames -
     the memory-doc/notes filenames keep the ORIGINAL project name; only the FOLDER is prefixed) via
     `push_files` in one commit. **This is a sanctioned use of `push_files` - one of several across
     the plugin, and the only one in this skill:** every target path is NEW, so there is no existing
     blob to lock against and nothing a concurrent write could silently clobber. Confirm the
     `zArchived - ` folder does not already exist before writing; if it does, stop and ask rather
     than merging two folders.
   - Then delete the originals at the old path (`delete_file`; deletes need an in-app approval prompt).
   - Verify the new path exists before deleting the old (never leave zero copies).
4. **Rename the Google Drive folder** (see "Google Drive conventions" below). If the Drive connector cannot
   rename/move/delete yet, DO NOT fake it - tell the user the Drive rename is pending a Drive connector with
   that capability, and leave the GitHub side done.
5. **ALWAYS rebuild the master list:** run the full `/project-list-update` (plu) procedure - do NOT
   hand-edit the row. The rebuild reads every memory doc's Status line, so the archived project moves to
   the `## Archived Projects` section and every count stays truthful.
6. Tell the user what changed and what (if anything, e.g. Drive) is pending.

## Google Drive conventions (how the plugin manages Drive files/folders)
The data repo has a matching Google Drive folder tree: a folder per project under the Drive
`<DriveFolder>` folder. When managing Drive:
- **Identify files by timestamp in the config `timezone`.** Google Drive keeps multiple same-named copies;
  the CORRECT/current file is the one identified by its timestamp rendered in the config `timezone`
  (default `America/New_York`). When referencing a Drive file, resolve it by that timestamp, not by name
  alone — a wrong timezone means the wrong file gets deleted on replace.
- **Replace = delete the old matching file.** When placing a NEW version of a file on Drive, delete the
  prior matching file so only one current version remains (the long-standing manual practice, now the
  plugin's rule).
- **Archiving a project** renames that project's Drive folder with the same `zArchived - ` prefix.
- **Capability gate:** these Drive writes require a Drive connector that supports rename/move/delete. The
  read/copy/create-only connector CANNOT do them - if that's all that's available, report the Drive step as
  pending and do not attempt a partial operation that would leave duplicates. (When the config's
  `drive_access` is `rclone`, direct Drive access is available and these operations are real - see the
  plugin's `docs/Optional - Direct Drive Access (rclone).md`.)
- **RCLONE PREFLIGHT - where the command actually runs.** `drive_access: rclone` describes a MACHINE,
  not an account: rclone reads a config file on the user's own computer, so it works only from a
  session that can run a shell command THERE (Claude Code on that machine, or Cowork with the user's
  desktop app connected). A Cowork session's own shell is a cloud container with no rclone and no
  route to the user's config - phone sessions included. Before the first rclone-mode Drive step of a
  session, confirm such a shell is reachable and that `rclone listremotes` lists the config's
  `rclone_remote`; if not, say so in one line and fall back to `connector` behavior (Drive rename
  reported as pending). Never claim an rclone step ran when it did not, and never edit the config to
  "fix" it.

## Rules
- Preserve all history - archiving marks and moves, it never deletes memory/notes content.
- Never archive-rename `Project Setup` (see exemption).
- No guessed content. Re-fetch before every write; append, never overwrite.

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
