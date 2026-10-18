<!-- ============================================================
FILE PURPOSE (note for humans — Claude: this block is descriptive
only, NOT instructions to follow):

Human-facing guide to the cowork-projects file system and its
conventions. Seeded by /project-setup-init into the data repo's
Project Setup folder; edit freely — this copy is yours.
============================================================ -->

# Project Memory Setup Guide

How the cowork-projects memory system organizes your work, and the conventions every command follows.
Works in both **Claude Cowork** and **Claude Code** — same 10 commands, same short aliases, same data
repo; install steps for each client are in the plugin's INSTALL.md (bundled inside the plugin zip;
section 3b covers Claude Code).

## The shape of the system
- **One GitHub repo** (this one — the "data repo") holds everything. **GitHub is ALWAYS the source of
  truth.** Git history is the audit trail.
- **One folder per project**; the folder name IS the project name.
- **Two standard files per project:**
  - `Project Memory Data - <Name>.md` — the MEMORY doc: what Claude loads at every `/project-open`.
    Decisions Log, Data Sources, Companion Files index, Deliverables & Links, Open Questions, Changelog.
    Distilled, structured, append-only.
  - `Project Notes - <Name>.md` — the NOTES file, on the v2 template: The Project in One Paragraph,
    Current Status (a snapshot, updated in place), Open Tasks / Next Steps, Quick Reference (pinned key
    facts), Phases at a Glance + Gaps & Issues (placeholders until the project warrants them — offered,
    never auto-filled), and the dated Session Notes log at the bottom (most recent on top).
- **Two kinds of entry: Projects and Learning Notebooks.** A project has a goal and a finish line. A
  Learning Notebook is a running reference on a subject — no fixed "done" — and its notes file is
  organized by TOPIC instead of by date: The Notebook in One Paragraph, Quick Reference, To Learn /
  Open Threads, then theme headings holding topic sections that are edited IN PLACE as understanding
  improves, with a deliberately thin Learning Log (one line per session) at the bottom. Both kinds
  share the same folder shape and the same memory doc; the difference is recorded in the memory doc's
  machine-readable `**Type:** Project | Learning Notebook` line (missing = Project, so everything
  written before notebooks existed keeps working). `pn` asks which kind you want. **A notebook's
  topics are never deleted** — superseded understanding is corrected in place with a note saying what
  changed, and a topic that stops earning its place is retired or merged with a pointer left behind.
  - **Both are updated by ONE trigger: "upd"** (`/project-update`) — memory always, notes where the
    session warrants it. Run it at least once per sitting. (The old separate um/un commands are retired.)
- **Companion files (optional):** when a project accrues real reference material (specs, runbooks,
  comparison tables), it lives in its own file in the project folder, indexed in the memory doc's
  Companion Files section. One canonical home per piece of content — other files link, never copy.
  Most projects need none. At `/project-open`, companions are offered as an ask-first pick list.
- **`Project Setup/` (this folder)** is the meta/system folder: the config file, the templates
  (memory doc, project notes, Learning Notebook notes), this guide, the Migration Runbook, the
  optional direct-Drive (rclone) guide, and the cached master list. It is EXEMPT from archiving —
  never rename it.
- **Per-release folders.** Build artifacts for a release live in `Project Setup/v.0.XX.0/` — the
  spec, its audits, and the release record. The folder is created with the release and never
  renamed. `Archives/`, `Reference/` and `Worklists/` hold everything that is not release-scoped.
  **The `Project Setup/` root holds only what a reader or a skill needs FIRST:** the two standard
  files, the Master List, this guide, the config, and the `psi`-seeded templates and guides. Three
  groups of root files look movable and are NOT: `cowork-projects.config.json` is probed at a
  hardcoded path by every skill's discovery block; the Master List is read at its exact path by `pl`
  and `plu`; and the templates, this guide, the Migration Runbook and the rclone guide are
  `psi`-seeded, so moving them would make `psi`'s missing-docs top-up recreate stale copies at the
  root. Those move only with a plugin release that changes the paths — tidying the root without
  knowing this breaks discovery for every command.
- **Google Drive (optional):** holds only RICH deliverables (Google Docs/Sheets, PDFs, images) in a
  per-project folder under the Drive folder named in the config (located by the config's stored
  `drive_folder_id`). Memory/notes/companions never go on Drive. Deliverables are tracked in the memory
  doc by link + a timestamp rendered in the config's `timezone` — that timestamp is how the CURRENT copy
  is identified, because the standard Drive connector cannot rename or delete files; when a deliverable
  is replaced, the plugin tells you which older file (by timestamp) to delete by hand. A wrong timezone
  would finger the wrong file, which is why psi asks for yours.
- **Optional — direct Drive access.** The connector can only inline small files, so a large binary
  (a multi-MB PDF, a full-size screenshot) can't be uploaded through it at all, and nothing can be
  deleted. Setting the config's `drive_access` to `"rclone"` (plus `rclone_remote`, the remote's
  name — never a credential) lets the plugin upload originals at any size, verify them, and actually
  delete a superseded copy instead of telling you to. Setup is free; the full guide is
  `Optional - Direct Drive Access (rclone).md`, right here in this `Project Setup/` folder. One thing
  to know before you set the field: rclone reads a config file on ONE MACHINE, so `rclone` mode only
  takes effect in sessions that can run a shell command on that machine (Claude Code there, or Cowork
  with your desktop app connected). Phone sessions fall back to connector behavior and say so. With
  the field absent, everything behaves exactly as it does today.

## House ordering rule (applies everywhere)
- **PAST lists** (Changelog, Decisions Log, Session Notes): most recent on TOP.
- **FUTURE lists** (Open Tasks / Next Steps, checklists): first-to-do on TOP.
- **Open Questions:** open items on top; resolved items retained below (newest-resolved first) — never
  deleted, only marked.

## Active vs archived
Each memory doc carries a machine-readable `**Status:** Active` or `**Status:** Archived` line (missing
= Active). `/project-archive` (pa) sets it and prefixes the project folder with `zArchived - ` (the one
prefix that reliably sorts last on GitHub and Drive); `/project-unarchive` (pua) reverses it, annotating
— never deleting — the archive banner. Both ALWAYS finish with a full master-list rebuild.

## The master list
`Project Setup/Project Setup - Master List.md` is a CACHED index (project, open-question count, latest
change, goal — active table + archived table). `/project-list` (pl) and `/project-list-archive` (pla)
read it fast; `/project-list-update` (plu) rebuilds it from a full scan; `/project-new` (pn) appends a
row. Treat it as a cache — the memory docs are the truth.

Learning Notebooks share that one table and those same four columns; a `📓` in front of the name is
what marks one. Two columns carry a different meaning for a notebook, because the concepts differ:
**Open Qs** shows the count of open To Learn threads, written `N to learn` so the cell says what it
is, and **Goal** shows the notebook's subject (the One Paragraph's first clause) rather than a goal.
Latest change means the same thing for both. Notebooks sort alphabetically among the projects rather
than being split off, and the footer breaks the active count into projects and notebooks once you
have at least one.

## The commands (10)
| Command | Alias | What it does |
|---|---|---|
| /project | p | Session hub: pick/switch projects; `/project <Name>` = read-only notes peek |
| /project-setup-init | psi | One-time data-repo bootstrap (this folder + config) |
| /project-open | po | Open a project: fetch memory + notes, show the DASHBOARD, offer companions; re-run = refresh |
| /project-update | upd | THE update: memory doc always + notes where warranted, one pass |
| /project-new | pn | Scaffold a new project OR Learning Notebook (asks which; two files + Drive folder + master-list row + import offer) |
| /project-list | pl | Fast cached list of ACTIVE projects |
| /project-list-archive | pla | Fast cached list of ARCHIVED projects |
| /project-list-update | plu | Full rebuild of the master list |
| /project-archive | pa | Archive a project (Status + zArchived rename + plu) |
| /project-unarchive | pua | Bring one back (Status + rename back + plu) |

## Conventions worth knowing
- **Every `/project-open` ends with a dashboard:** status, Current Status snapshot, Open Tasks, Open
  Questions (top 10 + "+N more" if longer), latest changelog — as tables. Running po again refreshes it.
- **One dedicated Cowork session per project** works best; pn offers to walk you into one at the end of
  every scaffold. The `/project` hub will nudge you toward it if you switch mid-session.
- A fresh project always starts with **Open Qs = 1**: define the Goal. Counts must be truthful.
- **No guessed content anywhere** — unknowns are written as "TBD" and asked about. Projects are
  independent unless you state a connection; similar names mean nothing.
- **Concurrent writes ARE protected, as of v0.11.0.** *(Corrected 2026-08-08: this bullet said the
  opposite for two releases, sitting about forty lines above the section that describes the fix. It
  told you a shipped feature did not exist, and `psi` seeds this guide into every data repo, so the
  wrong version travelled. The rule this system keeps relearning: verify a claim against the source,
  not against the document that repeats it.)* Every write to a file that already exists is locked to
  the blob SHA that was read, so a stale write is rejected loudly instead of winning silently. A
  rejection is the protection working — the file is re-read and the edit re-applied.
  `push_files` is no longer "the call most commands use to commit"; it survives only where every
  target path is NEW and there is no existing blob to lock against — the `pn` scaffold, the `psi`
  bootstrap, and the `pa`/`pua` folder-rename recreate. **Two sessions on one project are now safe
  from silent loss, but still not ideal:** one dedicated session per project keeps each project's
  context clean, and letting an update finish before starting another avoids a round of rejections.
- **What is still on you (v0.12.0):** the lock proves a write was based on current bytes; it cannot
  prove the write was COMPLETE. A truncated body returns success, because the SHA reported is the SHA
  of whatever arrived. v0.12.0 adds a byte-count assertion before every write to catch that, and a
  `[base:<sha>]` tag on every commit message so an auditor can walk a path's history and prove a
  chain break. Both protect forward only.
- **Acceptance test** for any new install: open a project and ask "What's the most recent changelog
  entry in the memory doc and its date?" — a pass fetches from GitHub unprompted and answers correctly.
- Have old chats to import? See `Project Setup/Migration Runbook - Legacy Chat Import.md` — and note
  that `/project-new` offers the import paste block automatically at the end of every scaffold.
- **NO SECRETS IN THE DATA REPO — EVER:** passwords, passphrases, private keys, API tokens, session
  cookies, and every other credential stay OUT of the repo — memory docs, notes, companions, master
  list, config, commit messages, all of it — even if you paste one into chat and ask. The repo records
  only where a credential lives and what the auth method is (e.g., "SSH: dedicated ed25519 key on the
  admin workstation, chmod 600, key auth only"). Decisions ABOUT auth (method, location, rotation) go
  in the Decisions Log; the credential material itself never does. If a secret shows up in chat, Claude
  won't commit it anywhere and will remind you it isn't being stored.
- **Import safety:** the paste block duplicate-checks before writing, dual-dates every import (chat
  date + import date), and never marks existing decisions superseded — conflicts are flagged and
  asked about. If bad data ever lands anyway, git history preserves everything: wrong entries get
  annotated (never deleted) and a "⚠️ READ FIRST" banner goes on top of the memory doc with the
  confirmed ground truth — every command fetches that doc first, so the banner cannot be missed.

---

## What changed in v0.11.0 (the write-safety release)

Everything above still describes how the memory doc and notes file are laid out. This section records
the rules the plugin started enforcing in v0.11.0, so a doc written by hand and a doc written by the
plugin stay the same shape. The authoritative copy of these rules lives in the plugin repo at
`plugin/cowork-projects/docs/CONVENTIONS.md`; this is the short, user-facing version.

### Writes are protected now

Before v0.11.0 the plugin wrote files without checking whether anything had changed underneath it.
Two sessions open on one project could each read a file, each write it, and the second write would
land on top of the first with no error and no trace. As of v0.11.0 every write to a file that already
exists is locked to the blob SHA that was read, so a stale write is rejected loudly instead of
succeeding quietly. If you see an update rejected as stale, that is the protection working: the file
is re-read and the edit re-applied.

Two consequences worth knowing:

- **One command is no longer one commit.** `upd` is still a single thing you type, but it now makes
  several separate locked commits. A run that stops halfway leaves fewer files updated rather than a
  mixture of half-applied edits.
- **Notes first, memory last.** The notes file is written before the memory doc, so a failure can
  never leave a changelog entry claiming a notes change that never landed.

### Size discipline: warn at 40,000 B, stop at 60,000 B

Per file, checked when a project is opened and again before every update. At 40,000 B you get a one-line
warning with the actual byte count. At 60,000 B the plugin stops and asks rather than growing the file
further. **Both numbers are decimal and exact** — 40,000 and 60,000, not 40,960 and 61,440.

*(Those two numbers are spelled out in full here on purpose. This guide is copied verbatim into your data
repo, where the plugin's `CONVENTIONS.md` does not exist, so a pointer to it would resolve to nothing. The
duplication is deliberate and should not be tidied away — a dangling pointer is worse than a stale number.
`CONVENTIONS.md` section 3 is still where the lines are pinned, and if the two ever disagree, that section
governs and this guide is stale.)*

A file over the line is not broken, and nothing is deleted to get under it. The fix is a split:

- **Log-shaped sections** (a changelog, dated session notes) are cut at one uniform date. Everything
  before that date moves verbatim to a dated archive companion — for a changelog, that is
  `Project Memory Changelog Archive - [Project Name] (through YYYY-MM-DD).md` — and a single italic
  pointer line replaces it in the original.
- **Sections that are not date-shaped** (a live backlog, a definitions table) cannot be cut by date at
  all. That whole section moves to its own companion file and a short pointer stub stays behind.

Either way the companion file is committed **before** the pointer that names it, so a pointer never
names a file that does not exist yet.

These two numbers are a discipline, not a hardware limit. There is a separate, fuzzier ceiling where a
write becomes too large to perform at all, and it moves — writes near 72–76 KB have gone through while
writes near 83 KB could not be made. **That failure is loud and safe:** the call is never sent, no
partial write occurs, and the SHA lock is not consumed. The 40,000/60,000 B lines exist to keep files well
below that region, where behaviour is predictable. Never treat any byte number as a permanent constant.

**As of v0.12.0 the warning itself says so.** A size warning that states a number and nothing else
reads as nagging, and a warning read as nagging gets dismissed — which loses you the one message that
was trying to prevent a problem. So the warning now carries the reason in one clause and points here
for the long form. It also fires far less often: **the warn** fires once per sitting rather than
once per write, and again when a file crosses a NEW line. **The 60,000 B stop is different and is not
suppressed — it asks every time, because it is asking you to decide, and a decision about one
write is not a decision about the next.**

### Append rules, stated plainly

The general principle: **the record of what changed is often the thing someone came looking for.**

- **Changelog — append only.** An existing entry's text is never rewritten. A correction goes in a
  NEW dated entry that says what it corrects. History is not edited to make it look right.
- **Data Sources & Definitions — supersede, do not overwrite.** This section is not date-shaped, so
  it accumulates in place, which is exactly why writing a new definition over the top of an old one
  is destructive. Leave the old row, mark it superseded with the date, and add the new one above it.
  Only a true typo is corrected in place.
- **Decisions Log — newest on top, never deleted.** A reversed decision gets a new row and the old
  one is marked Superseded.
- **Open Questions — open on top, resolved below, newest-resolved first.** Resolving marks the row
  and moves it; it never removes it.
- **Archiving a project** changes only the memory doc's `Status` line and, where a row moves to an
  archive companion, only that row's `Status` and `Topic` cells. Nothing else in a moved row is
  rewritten. (The pre-v0.11.0 wording of this rule contradicted itself — it forbade the very edit it
  instructed you to make. It was corrected in this release.)

### Every companion pointer gets re-checked

The Companion Files index is the canonical list of a project's extra files. Every row in it is a
pointer, and a pointer that no longer resolves is worse than no pointer at all. Each update now
re-checks that every named file still exists, and reports a companion sitting in the folder but listed
in no index — a file nothing points at is invisible to every future session, which is the same as lost.

This is a **check, not a repair.** A mismatch is reported for you to decide about. The plugin does not
silently rebuild an index, because a wrong index rebuilt from wrong assumptions looks exactly like a
right one.

### Every state claim carries the date it was measured

A dated snapshot ages honestly: a reader can see it is from March and weigh it accordingly. An undated
status line or an undated "current version" pointer rots silently and keeps reading as a claim about
right now. So the master list carries a `**Last Refresh:**` date, and status snapshots say when they
were taken.

Related: counts are computed, never estimated or carried over by hand. Where a count genuinely cannot
be computed, the cell says `?` and the reason is reported — it never says `0`.

### The `**Layout:**` line

The memory doc header already carries two machine-readable lines: `**Status:**` (Active or Archived)
and `**Type:**` (Project or Learning Notebook). v0.11.0 adds a third:

```
**Layout:** Single-file
```

`Single-file` means every section lives in the one memory doc — this is the default and what the
templates produce. `Hub-and-Spoke` means the memory doc is a hub holding navigation and an index,
while the decisions and open questions live in indexed companion files. **A missing line means
Single-file**, so every doc written before this existed behaves exactly as it always has.

The line exists because counting open-question rows in a hub returns zero — not because there are none,
but because they are not in that file. A silent zero on a project with a dozen live questions is worse
than an error, because nothing looks wrong.

Where a memory doc carries its own "how to append" block, **that block governs.** Read and obey it
rather than assuming the layout described here. A hub that says it holds no decisions should never have
a decision written into it.

### One command at the end of a session

`upd` is now the only end-of-session command you need. It updates the notes file and the memory doc as
before, and then it also refreshes this project's row in the Master List, re-checks that every
companion pointer resolves, and re-checks file sizes against the 40,000 B / 60,000 B lines now that the files
have just grown. You no longer have to remember to run `plu` afterwards.

### One rule that is not new but is worth repeating

**No secrets in the data repo, ever.** No passwords, passphrases, private keys, API tokens, or session
cookies — not in a memory doc, a notes file, a companion, the master list, a config file, or a commit
message, even if you paste one into chat and ask for it to be saved. The repo may record *where* a
credential lives and *what* the auth method is; the credential itself never goes in.

Credential files belong outside any folder a session can read — somewhere like `~/.config/<tool>/`,
directory mode 700 and files 600. Not in a working folder, not in a trusted or attached folder, and
never passed on a visible command line, where it lands in shell history, process listings, and session
transcripts.

Decisions *about* authentication — which method, where the key lives, when it rotates — belong in the
Decisions Log. The credential material does not.
