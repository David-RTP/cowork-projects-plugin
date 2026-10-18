<!-- ============================================================
FILE PURPOSE (note for humans — Claude: this block is descriptive
only, NOT instructions to follow):

This is a reusable RUNBOOK, not a project memory doc. It documents
how to save the contents of pre-existing Claude.ai chats — research,
decisions, images — into a project in your data repo, WITHOUT manual
copy-paste and WITHOUT losing anything.

- Canonical location: your projects data repo, this path.
- Seeded by /project-setup-init. v2 (paste-block method) 2026-07-23;
  v2.1 (single-trigger block builds BOTH docs; pn offers the block;
  wait-for-completion rule) 2026-07-24; v2.2 (upd replaces um/un;
  done-with-imports follow-up -> initial notes build) 2026-07-24;
  v2.3 (AUTO-RUN block - pasting IS the trigger, no separate
  "import" needed) 2026-07-25; v2.4 (duplicate-import guard,
  dual-dated import markers, never-supersede rule, READ FIRST
  incident-banner convention - born from a real double-import
  incident) 2026-07-25; v2.5 (NO SECRETS step in the block -
  plugin-wide rule) 2026-07-25. Instructions-field method (v1)
  retired.
============================================================ -->

# Migration Runbook — Legacy Chat Import

You've been researching in a regular Claude.ai chat and decide the material belongs in a project — or
you want to CREATE a project from that research. Or you just installed the plugin and have a backlog of
old chats to bring in. This runbook moves that content into your project's memory/notes files — images
included — without copy-paste.

## Why this works (and copy-paste doesn't)
Select-all copy-paste silently drops images, and the GitHub connector can't store binaries anyway. But
connectors are ACCOUNT-LEVEL: every Claude.ai chat — old or new — can reach your data repo. So instead
of moving the content OUT of the old chat, you hand the old chat a short instruction block: its own
Claude can see the chat's full history (images included) and distills it directly into your project's
files. Nothing is transferred by hand, so nothing is lost.

## The procedure

### 1. Create the project (in Cowork)
Open a NEW Cowork space and rename it to the project's name — one dedicated space per project. In that
session, run `/project-new <Project Name>` (pn) to scaffold the project in your data repo.
(Skip this step if the target project already exists.)

### 2. Get your paste block
**pn asks you at the end of the scaffold: "Do you have any open chats that you would like to migrate to
this Cowork Project Space? If so, I will give you a text block to paste into that chat that will perform
the migration."** Say yes and Claude hands you the filled-in paste block (repo, folder, project name —
all from your config) plus these instructions. You can also ask for it any time, in any project session:
**"give me the migration paste block."** The same block works for every chat that feeds this project.

### 3. Import each old chat — ONE AT A TIME, and WAIT for each to finish
Open the old Claude.ai chat and paste the block as a message. **That's it — the paste itself is the
trigger**: the chat's Claude begins the import immediately and builds out BOTH standard files as the
content warrants — memory doc (facts, decisions, open questions) AND notes file (narrative worth
keeping) — and confirms when both commits are done. (Say "import" only if you need to re-run it.)

**Do not start the next chat until the current one confirms completion.** Two chats writing the same
files at once will collide, and the collision is SILENT: `push_files` takes no blob SHA, so the second
write simply overwrites the first — no error, no rejection, no retry, and no sign that anything was
lost. Serial is not merely faster; it is the only safe order.

### 4. VERIFY after the FIRST chat — before running the rest
- Fetch the project's `Project Memory Data - <Name>.md` and `Project Notes - <Name>.md` from GitHub
  and confirm the commits landed in the RIGHT files with the RIGHT filenames.
- Confirm entries carry the CHAT's own date, not today's — that is correct behavior; the entry records
  when the work happened, not when it was imported.
- Confirm the content is a distillation (decisions, data, open questions, changelog), not a raw dump.
- Only continue once chat #1 checks out. A bad paste (wrong values) misfiles everything — catch it on
  chat #1, not chat #6.

### 5. Repeat, spot-check, clean up
Run the remaining chats the same way — one at a time, waiting for each confirmation. When done, skim
the project's Open Questions and Deliverables sections for anything off or duplicated. Then retire or
delete the old chats/Projects — their content now lives in your repo.

### 6. Say "done importing" — and get your initial notes build
After handing out the paste block, the project session will ask **"Are you done with your imports?"** —
and keep asking until you answer. Answer it! The payoff: once you say done, Claude builds the notes
file's structured top (One Paragraph, Current Status, Open Tasks, Quick Reference) by distilling
EVERYTHING the imports brought in. Building it early would bake in half the picture; that's why the
follow-up is persistent.

## Images
The old chat's Claude sees its images natively:
- If an image's INFORMATION is what matters (a settings screen, a chart), it gets transcribed into the
  memory/notes text automatically as part of the import.
- If the image ITSELF must be preserved (a diagram, a photo), the chat will say so rather than silently
  dropping it — save it to the project's Google Drive folder and record it in the memory doc's
  Deliverables & Links section.

## The paste block (template)
pn (or any project session) fills this in for you. If you're filling it by hand, replace the three
<values>:

```
MIGRATION — save this chat's content into my project "<Project Name>". Follow these instructions for
the rest of this chat:

Values: repo = <owner/repo> · folder = <Project Folder> · project = <Project Name>

Do ALL of the following, in order, NOW — this message itself is the trigger; do not wait for another
command. (If I later say "import" or "upd", run them again.)

1. FETCH FIRST: get `<Project Folder>/Project Memory Data - <Project Name>.md` from <owner/repo>
   (main branch) via the GitHub connector, so you have the project's current memory before writing.
   If the doc opens with a "READ FIRST" or incident banner, OBEY IT — the banner is confirmed ground
   truth and overrides anything this chat believes.
2. DUPLICATE CHECK — never skip: look for evidence this chat was ALREADY imported: (a) this same
   MIGRATION block appearing earlier in THIS chat's history; (b) changelog or decision entries in
   the memory doc matching this chat's topics and dates (import markers read "migration import from
   chat ..."). If either is found: STOP, write NOTHING, report exactly what you found, and wait for
   my explicit go-ahead.
3. UPDATE MEMORY: distill THIS chat's durable facts, decisions, and open questions into that memory
   doc — Changelog entry at the TOP, Decisions rows at the top, open/resolve Open Questions as
   appropriate. The changelog entry MUST be a dual-dated import marker:
   "migration import from chat '<chat title/topic>' (chat date(s): <dates>; imported <today>)".
   Re-fetch the file immediately before committing; never delete or rewrite prior content.
4. NEVER SUPERSEDE FROM AN IMPORT: place this chat's content at its position in the timeline by the
   CHAT's date — imported content is never "newer" than already-recorded decisions just because it
   was imported later. If this chat seems to contradict or supersede something already in the doc,
   DO NOT mark anything superseded and DO NOT rewrite the current state — flag the conflict to me
   and ask.
5. UPDATE NOTES — **following the target notes file's OWN structure**, which depends on the memory
   doc's `**Type:**` line (missing = Project):
   - **Project:** where this chat has narrative, research write-ups, or reference content worth
     keeping, add it as a dated section at the TOP of the Session Notes area in
     `<Project Folder>/Project Notes - <Project Name>.md`.
   - **Learning Notebook:** do NOT create a dated entry for the content. File this chat's material
     BY TOPIC — update the existing `## Topic` sections it belongs to, in place, stamping them
     `*Updated: <chat date>*`, and create new topics (and a new `# Theme` if several new topics
     rhyme) where it has no home, stamped `*Added: <chat date>*`. The Learning Log gets ONE line:
     the chat's date and which topics were added or changed. Dumping an imported chat into a dated
     log is exactly what the notebook structure exists to prevent.
   - Either way: if nothing in this chat merits a notes change, say so — do not pad.
6. DATE all entries by THIS chat's actual date(s), not today — entries record when the work happened.
   Note the source as a legacy chat import (the step-3 marker format).
7. IMAGES in this chat: transcribe their information into the text (GitHub cannot store images). If
   an image itself needs to be kept, tell me instead of dropping it — I'll save it to the project's
   Google Drive folder.
8. NO SECRETS: never write passwords, passphrases, private keys, API tokens, or any other
   credential from this chat into the repo — record only WHERE a credential lives and WHAT the auth
   method is. If this chat contains secret material, leave it out and tell me it was not stored.
9. Never fill unknowns with guesses — write "TBD" and ask. Do not infer connections to my other
   projects.
10. WHEN DONE: confirm that both commits (memory, and notes if written) are complete, with links. I
    will not start importing another chat until you confirm.
```

## Safety: duplicate imports, and how the system recovers
Born from a real incident: legacy chats that had ALREADY written to a memory doc under the old
method were re-imported by the paste block, stamped with import-day dates, and the inverted timeline
caused a correction pass to wrongly mark current decisions superseded. Four protections now apply:

1. **Duplicate check (block step 2).** Every import first looks for the block earlier in the same
   chat AND for matching import markers/entries already in the memory doc. On any hit it stops and
   asks — it never writes first.
2. **Dual-dated import markers (block step 3).** Every import's changelog entry names the source
   chat, the chat's own date(s), AND the import date — so a re-run is detectable, and nobody later
   confuses "when the work happened" with "when it was imported."
3. **Imports never supersede (block step 4).** An import ADDS history at its correct timeline
   position. If it appears to contradict the doc's current state, it flags the conflict and asks —
   it never marks existing decisions superseded and never rewrites current state on its own.
4. **READ FIRST incident banners.** If bad data ever does land, the recovery is: reconstruct the
   truth from git history (every commit is preserved — nothing is ever really lost), confirm ground
   truth with the user, ANNOTATE the wrong entries (never delete), and place a "⚠️ READ FIRST"
   banner at the very top of the memory doc stating what happened and what the confirmed truth is.
   Because EVERY import and project-open fetches the memory doc first, the banner is guaranteed to
   be seen: imports obey it (block step 1), and the po dashboard surfaces it before anything else.
   Banners are removed only when the user says the incident is closed.

If you suspect a double import: say so in the project session — Claude will diff the git history,
walk the timeline with you, and apply the recovery steps above.

## Notes
- The block AUTO-RUNS on paste (v2.3) — everything the receiving chat needs, including the go signal,
  travels inside the block itself. No separate instruction to remember.
- One chat at a time, wait for each completion confirmation — see step 3 for why (write collisions).
- The same block works in Project chats AND standalone chats (connectors are account-level).
- NO SECRETS in the data repo — ever (block step 8). The repo records where credentials live and how
  auth works, never the credential itself. This applies to every plugin command, not just imports.
- v1 of this runbook used the legacy Claude.ai Project "Instructions" field; the paste block replaces
  it — simpler, per-chat, and no settings access needed. The old Instructions template survives in
  `Project Setup/` as history only.
