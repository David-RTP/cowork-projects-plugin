# cowork-projects (Cowork plugin)

GitHub-backed project memory for Claude Cowork. Every project gets a folder in YOUR OWN GitHub repo
with a structured memory doc (decisions, open questions, changelog) and a human notes file. Claude
fetches both live at the start of each project session, presents a table dashboard, and commits
updates on one simple trigger (`upd` - Update) - so context survives across sessions, devices, and
even model changes, with git history as the audit trail.

**Your data stays yours:** the plugin ships no storage. `/project-setup-init` bootstraps any GitHub
repo you choose (private recommended) and writes a small config file there; every command discovers
your repo through that config. Nothing is hardcoded.

## Quickstart
1. Install the plugin (see `INSTALL.md` in this repo).
2. In a new Cowork session, run `/project-setup-init` (psi) - pick or create your data repo; it also
   resolves your Google Drive folder ID for you and records your timezone.
3. `/project-new <Name>` (pn) to scaffold your first project - it offers to import your existing chats
   and walks you into the project's own session.
4. Work, then say `upd` to commit the session to memory + notes. That's the whole loop - once per
   sitting keeps everything durable.

## Skills / commands (10)
- `/project` (also "p") - session hub: with no name, pick/switch projects (it will nudge you toward
  one-session-per-project); with a project already open, shows the current one and offers to switch or
  create; with a name (`/project Project Name`) shows that project's notes READ-ONLY (never opens it).
- `/project-setup-init` (also "psi") - ONE-TIME bootstrap of your data repo: writes the config file
  (repo, branch, Drive folder + resolved folder ID, timezone, Drive access mode), the three templates
  (memory doc, project notes, Learning Notebook notes), the legacy-Instructions template, the Setup
  Guide, the Migration Runbook, the optional direct-Drive (rclone) guide, an empty master list, and
  Project Setup's own memory/notes files. Re-running it on an initialized repo stops at the
  double-init guard and offers to top up missing config fields and missing seeded docs.
- `/project-open <Name>` (also "po") - load a project: fetch its memory doc + notes, then show the
  DASHBOARD (status, current snapshot, open tasks, open questions - top 10 with a "+N more" footer -
  latest changelog, as tables) and offer companion files as an ask-first pick list. Running po again
  on the open project REFRESHES it.
- `/project-update` (also "upd") - THE update command, replacing the old um/un pair: one pass that
  always updates the memory doc (dated changelog on top, decisions, open questions) and updates the
  notes file where the session warrants it (status snapshot in place, tasks, quick reference, a dated
  session note) - including an always-run check of whether the notes' Phases/Gaps sections are ready
  to populate.
- `/project-new <Name>` (also "pn") - ask whether this is a PROJECT or a LEARNING NOTEBOOK, then
  scaffold its two standard files (notes on the matching template), create its Drive folder, add it to
  the master list (fresh entries start with one open item: define the Goal, or define what the
  notebook covers) - then ask "Do you have any open chats that you would like to migrate to this
  Cowork Project Space?" (yes -> the filled AUTO-RUN import paste block, one-chat-at-a-time rules, and
  a persistent "done importing?" follow-up that ends in the initial notes build) and offer to open the
  entry's dedicated session.
- `/project-list` (also "pl") - cached index of ACTIVE projects + a count of archived ones.
- `/project-list-archive` (also "pla") - cached index of ARCHIVED projects.
- `/project-list-update` (also "plu") - rebuild the cached master list from a full scan.
- `/project-archive <Name>` (also "pa") - Status -> Archived, folder prefixed `zArchived - `, then
  ALWAYS a full master-list rebuild.
- `/project-unarchive <Name>` (also "pua") - reverse of pa (banner annotated, never deleted).

## How it works
- One repo, one folder per project. Two standard files per folder: `Project Memory Data - <Name>.md`
  (memory) and `Project Notes - <Name>.md` (notes, v2 template: One Paragraph / Current Status /
  Open Tasks / Quick Reference / Phases + Gaps placeholders / dated Session Notes at the bottom) -
  plus **optional companion files**. One trigger (`upd`) maintains both.
- **Projects and Learning Notebooks.** Not everything you want to keep has a finish line. A LEARNING
  NOTEBOOK is a running reference on a subject: its notes file is organized by TOPIC under themes you
  choose, and each topic is edited IN PLACE as your understanding improves, so you can read a topic and
  get the current best answer instead of reconstructing it from a dated log. The dated part shrinks to a
  one-line-per-session Learning Log. Everything else is shared with projects - same folder shape, same
  memory doc, same `upd`, same master list - and the memory doc's machine-readable
  `**Type:** Project | Learning Notebook` line (missing = Project) is what selects the behavior, so
  every doc written before this existed keeps working untouched. `pn` asks which kind you want; `po`
  shows a notebook its own dashboard (theme/topic contents and open threads instead of a status
  snapshot). **Topics are never deleted** - wrong understanding is corrected in place with a note
  saying what changed; a topic that stops earning its place is retired or merged with a pointer.
- **Config discovery:** every command reads `Project Setup/cowork-projects.config.json` in your data
  repo (`owner`, `repo`, `branch`, `drive_folder`, `drive_folder_id`, `timezone`, plus the optional
  `drive_access` / `rclone_remote` pair). Commands find it by searching your account for that
  filename; if it's missing, they offer `/project-setup-init`.
- **Companion-files architecture:** every piece of content has ONE canonical home; other files link,
  never copy. The memory doc holds the canonical Companion Files index; at `project-open`, companions
  are offered as a phone-friendly multi-select pick list ("None" first) - never loaded by guessing.
- **Active vs archived** is a machine-readable `**Status:**` line in each memory doc (missing =
  Active). Archive/unarchive always finish with a full master-list rebuild so the cache never drifts.
- **House ordering rule:** PAST lists (changelog, session notes, decisions) most-recent-on-top;
  FUTURE lists (Open Tasks, checklists) first-to-do-on-top; Open Questions open-on-top, resolved
  (newest first) below.
- **Storage split:** GitHub is ALWAYS the source of truth - memory + notes + companions live only in
  GitHub. Google Drive (optional) holds only rich deliverables (Docs/Sheets/PDFs) per project, tracked
  in the memory doc by a timestamp rendered in YOUR configured timezone - the timestamp is how the
  current copy is identified, since the standard Drive connector cannot rename or delete files (psi
  asks your timezone precisely so the right file gets pointed at for manual cleanup).
- **Optional direct Drive access:** the standard connector can only inline SMALL files and cannot
  delete or rename at all. Setting the config's `drive_access` to `"rclone"` (with `rclone_remote`
  naming the remote - never a credential) lets the plugin upload deliverables at full fidelity at any
  size, verify each transfer, and actually delete the superseded copy instead of telling you to. Free,
  one-time, no sudo: see `docs/Optional - Direct Drive Access (rclone).md` (psi also seeds a copy into
  your own data repo, so it is reachable without this repo). **rclone runs on ONE MACHINE** - it reads
  a config file there - so `rclone` mode only takes effect in sessions that can run a shell command on
  that machine: Claude Code on it, or Cowork with your desktop app connected. Every Drive-touching
  skill preflights this and falls back to connector behavior, out loud, when it can't reach one. With
  the field absent everything behaves exactly as before.
- **Templates live in your data repo**, not in the plugin - `/project-new` reads them live, so you can
  evolve your memory-doc and notes structure without rebuilding or reinstalling the plugin.
- Decided mid-research that a Claude.ai chat belongs in a project - or want to create a NEW project
  from the research in a chat? `Project Setup/Migration Runbook - Legacy Chat Import.md` (seeded by
  psi) documents the paste-block import procedure: the old chat writes its own content, images
  included, straight into your project. `pn` offers it automatically at the end of every scaffold.

## Requirements
- GitHub connector enabled (account-level, Settings > Connectors).
- Code execution enabled (required for custom skills).
- Google Drive connector optional (deliverables only; folder create is used, rename/delete degrade to
  "reported as pending" unless `drive_access` is set to `rclone`).

## Concurrent writes: fixed in v0.11.0
**Every write to a file that already exists is SHA-locked.** The command re-fetches the target
immediately before committing and writes with `create_or_update_file`, passing the blob SHA it read.
If another session changed the file in between, the write is REJECTED - loudly, with nothing lost -
and the command re-reads and re-applies its edit rather than overwriting.

Through v0.10.0 this was not true. Most commands used `push_files`, which takes no blob SHA and no
base commit, so two sessions writing the same file ended in silent last-writer-wins: no error, no
rejection, nothing to notice. Git history preserved whatever was overwritten, so nothing was ever
truly lost, but you had to notice first. If you have been running more than one session against a
project, this is the release to take.

`push_files` still appears in exactly four places, and only where every target path is NEW: `pn`
scaffolding a project, `psi` bootstrapping a repo that has no `Project Setup/` folder, `pa`
recreating a project's files under the `zArchived - ` folder name, and `pua` recreating them back
under the plain name. There is no existing blob to lock against in those cases, and each one
confirms the destination does not already exist before writing.

One consequence worth stating plainly: **one command is no longer one commit.** `upd` remains one
thing you type, and now makes two or more locked commits underneath - notes first, memory last, so a
failure can never leave a changelog entry claiming a notes write that did not land.

## Shared conventions: `docs/CONVENTIONS.md`
Rules that apply to more than one command live in one authoritative file rather than in nine copies
that drift: no secrets in the data repo, SHA-locked writes, the size rule,
how to split a file that outgrew itself, append rules by section type, the markdown table lint, dated
state claims, computed counts, reconciliation as a check rather than a repair, and never filling an
unknown with a plausible guess. A command carries a pointer to it when it makes a claim that file
governs - which is usually because it writes, but not only then: a read-only command that renders a
cached count is making a dated-claim assertion and is bound by it too. When a rule changes
it changes there, once.

## No secrets in the data repo — ever
Every skill that writes to the data repo enforces the same rule: passwords, passphrases, private
keys, API tokens, session cookies, and all other credentials are NEVER committed — not in memory
docs, notes, companions, the master list, config files, or commit messages, even if pasted into chat
with a request to save them. The repo records only where a credential lives and what the auth method
is; decisions about auth (method, location, rotation) go in the Decisions Log. If a secret appears
in chat, Claude declines to store it and says so. Credential FILES belong outside any trusted or
attached folder a session can read (e.g. `~/.config/<tool>/`, 700/600) — never in the working folder
and never on a visible command line.

## Import safety & incident recovery
The migration paste block carries four protections (see the Migration Runbook's Safety section):
a mandatory **duplicate check** before writing (stops and asks if the chat was already imported);
**dual-dated import markers** (every import records the chat's own date AND the import date);
a **never-supersede rule** (imports add history at the chat's timeline position - they flag
conflicts with current state instead of rewriting it); and **READ FIRST incident banners** - if bad
data ever lands, the truth is reconstructed from git history (nothing is ever really lost), wrong
entries are annotated rather than deleted, and a banner at the top of the memory doc states the
confirmed ground truth. Since every command fetches the memory doc first, the banner is guaranteed
to be seen: imports obey it and the po dashboard surfaces it before anything else.
