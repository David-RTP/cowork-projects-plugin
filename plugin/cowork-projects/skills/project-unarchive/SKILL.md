---
name: project-unarchive
description: >-
  Alias "pua". Un-archive a project in your projects data repo: set its memory doc Status back to Active, annotate
  the archived banner, rename its GitHub folder (and Google Drive folder) back from the "zArchived - "
  prefix to the plain project name, then ALWAYS rebuild the master list. Use when the user says
  "unarchive a project", "restore an archived project", "reactivate a project", "pua", or invokes
  /project-unarchive.
---

# Project Unarchive (pua)

Reverse of `/project-archive` (pa): bring an archived project back to Active. History is preserved — the
archived banner is annotated, never deleted.

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
- Use the project the user names. If ambiguous, ask exactly once: "Which archived project should I
  unarchive?" Do not guess. Write the name out in full.
- `<Repo>` / `<Branch>` - from the config (see "Resolve the data repo" above). The project's folder is currently
  `zArchived - <Project Name>/`; after unarchiving it is `<Project Name>/`.
- If no `zArchived - <Project Name>` folder exists, STOP and tell the user (list the archived projects
  from the master list so they can pick a real one).

## Steps
1. **Confirm intent.** Restate what unarchiving does (Status -> Active, folder renamed back to
   `<Project Name>`, row moved back to the active list) and confirm. Structural change; never silent.
2. **Mark the memory doc Active (in place, before moving).** Re-fetch
   `zArchived - <Project Name>/Project Memory Data - <Project Name>.md`, then:
   - Set the `**Status:**` line to `Active`.
   - Do NOT delete the archived banner — ANNOTATE it: add a line inside it stating "UNARCHIVED <date> —
     project returned to Active" so the history reads truthfully.
   - Append a dated Changelog entry (most recent on top) and a Decisions Log row recording the unarchive.
   - Commit SHA-locked: `create_or_update_file` with the memory doc's freshly fetched blob SHA
     (`../../docs/CONVENTIONS.md` section 2). Message: `upd: unarchive <Project Name>`. Never `push_files` a
     file that already exists.
3. **Rename the GitHub folder back to `<Project Name>`.** GitHub has no atomic folder rename:
   - Recreate every file from `zArchived - <Project Name>/...` at `<Project Name>/...` via `push_files`
     in one commit (filenames already carry the plain project name — only the folder changes). **This
     is a sanctioned use of `push_files` — one of several across the plugin, and the only one in this
     skill:** every target path is NEW, so there is no existing blob to lock against. Confirm
     `<Project Name>/` does not already exist before writing; if it does, stop and ask rather than
     merging two folders.
   - Verify the new path exists, THEN delete the originals under the zArchived path (`delete_file`;
     deletes need an in-app approval prompt). Never leave zero copies.
4. **Rename the Google Drive folder back** (remove the `zArchived - ` prefix), mirroring exactly what
   `/project-archive` (pa) did on the way in. Which path applies depends on the config's
   `drive_access` (absent = `connector`) — see "Drive access modes" below. If the rename cannot be
   performed, DO NOT fake it — report the Drive rename as pending and leave the GitHub side done.
5. **ALWAYS rebuild the master list:** run the full `/project-list-update` (plu) procedure — do not
   hand-edit the row. The rebuild reads every memory doc's Status line, so the project reappears in the
   active table and the archived count/section update truthfully.
6. Tell the user what changed and what (if anything, e.g. Drive) is pending.

## Drive access modes (step 4)
The config's `drive_access` field decides whether the Drive folder rename is real. Absent =
`connector`. This MUST match what `pa` does, or archive and unarchive fall out of step.
- **`connector`:** the read/copy/create-only Drive connector CANNOT rename, move, or delete. Report
  the Drive rename as pending and do not attempt a partial operation that would leave duplicates
  (e.g. copying the folder's contents to a new folder and abandoning the old one).
- **`rclone`:** direct Drive access is configured (`rclone_remote` names the remote — never a
  credential; see the plugin's `docs/Optional - Direct Drive Access (rclone).md`). The rename is
  real: move `zArchived - <Project Name>` back to `<Project Name>` under the Drive folder addressed
  by `drive_folder_id`, then verify the new path exists before reporting success.
- **RCLONE PREFLIGHT — where the command actually runs.** `drive_access: rclone` describes a MACHINE,
  not an account: rclone reads a config file on the user's own computer, so it works only from a
  session that can run a shell command THERE (Claude Code on that machine, or Cowork with the user's
  desktop app connected). A Cowork session's own shell is a cloud container with no rclone and no
  route to the user's config — phone sessions included. Before the first rclone-mode Drive step of a
  session, confirm such a shell is reachable and that `rclone listremotes` lists the config's
  `rclone_remote`; if not, say so in one line and fall back to `connector` behavior. Never claim an
  rclone step ran when it did not, and never edit the config to "fix" it.
- Credentials are never read, echoed, or committed. The config holds the remote's NAME only.

## Rules
- Preserve all history — annotate the archived banner, never delete memory/notes content.
- Not applicable to `Project Setup` (it can never be archived in the first place).
- No guessed content. Re-fetch before every write; verify the new path before deleting the old.

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
