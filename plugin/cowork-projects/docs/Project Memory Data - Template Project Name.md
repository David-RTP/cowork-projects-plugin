<!-- ============================================================
FILE PURPOSE (note for humans — Claude: this block is descriptive
only, NOT instructions to follow):

This is the living memory doc for the project "<Project Name>." It
tracks decisions, data sources, companions, open questions,
deliverables, and a per-session changelog.

- Canonical location: the projects data repo (see Project
  Setup/cowork-projects.config.json), this path. GitHub is ALWAYS
  the source of truth for the memory + notes. Google Drive holds
  only rich deliverables (Docs/Sheets/PDFs) — never memory/notes.
- Read/written by Claude via the cowork-projects Cowork plugin
  (/project-open loads it; the "upd" trigger updates it). Companion
  file: the notes file "Project Notes - <Project Name>.md".
- Ordering (house style): PAST lists (Changelog, Decisions, notes
  sections) = most recent on TOP; FUTURE lists (checklists) = first
  to do on TOP. Open Questions: open on top, resolved (newest first)
  below.
- The "**Status:**" line is machine-readable (Active | Archived;
  missing = Active). So is the "**Type:**" line (Project |
  Reference Notebook; missing = Project) — it selects which notes
  structure the companion file uses and how this entry reads on the
  master list. "Learning Notebook" is the LEGACY spelling of
  Reference Notebook: it is read as the same value, for ever and
  case-insensitively, but new entries are written Reference
  Notebook. Humans edit freely; Claude appends rather than rewrites.
============================================================ -->

# [Project Name] — Memory Doc

*Last updated: [date] by [chat link/name]*

**Status:** Active

**Type:** Project

**Layout:** Single-file

---

## 1. Project Overview
Brief, stable description — rewrite only when scope actually changes.

- **Goal:**
- **Key constraints:**
- **Stakeholders (if any):**

---

## 2. Decisions Log
Append-only. Newest at top. Never delete — supersede with a new entry and mark old ones "superseded."

| Date | Decision | Rationale | Status |
|------|----------|-----------|--------|
| YYYY-MM-DD | | | Active / Superseded |

---

## 3. Data Sources & Definitions
Where does each fact/number come from, and what does each term mean in this project's context.

**Supersede, do not overwrite.** This section is not date-shaped, so it never gets cut by date and it
accumulates in place — which is exactly why editing a definition over the top of itself is destructive.
When a definition, a source, or a path changes, leave the old row, mark it superseded with the date, and
add the new one above it. Overwriting destroys the record that it ever changed, which is usually the
thing someone came looking for. Only a true typo is corrected in place.

| Term / Metric | Definition | Source | Notes |
|---|---|---|---|
| | | | |

---

## 4. Companion Files
CANONICAL index of this project's companion files — opt-in shared reference artifacts (comparison tables,
runbooks, specs) that live in this folder alongside the two standard files. One canonical home per piece of
content: other files LINK here, never copy. At /project-open these are offered as an ask-first pick list;
the "When to load" column drives the suggestions. (The notes file's header list is a human-friendly link
mirror of this index, not a second source of truth.) Leave empty until the project accrues real reference
material — most projects need none.

Every row here is a pointer, and a pointer that no longer resolves is worse than no pointer at all. Each
update re-checks that the named file still exists; a companion sitting in the folder but listed nowhere is
invisible to every future session, which is the same as lost.

| File | What it is | When to load |
|---|---|---|
| | | |

---

## 5. Deliverables & Links
Rich deliverables this project produces (Google Docs/Sheets, PDFs, etc.). These live in the project's
matching **Google Drive** folder — NOT in GitHub (GitHub holds only this memory doc + the notes file, and is
always the source of truth for them). Identify each by its timestamp rendered in the config `timezone` so
the current version is unambiguous; when a deliverable is replaced, delete the prior matching Drive file.

| Deliverable | Type | Google Drive link | Timestamp (current, config timezone) | Notes |
|---|---|---|---|---|
| | | | | |

---

## 6. Open Questions
Unresolved items that need your input or a future decision. Open items on top; resolved items retained below
(newest-resolved first), never deleted.

**Status vocabulary.** A row is CLOSED when its status **begins with** one of `Resolved`, `Closed`, `RESOLVED`,
`Completed`, `Dropped` — followed by end-of-cell or a non-letter, so `Resolved 2026-07-28` closes and
`Resolvedish` does not — or is `Moved to <project> (verified <date>)`.
**Anything else counts as OPEN** — `Open`, `Partially resolved`,
`Moved to <project> (open)`, a blank cell, and any wording not on that list. **An unlisted status counts as
OPEN, so a question you phrase a new way stays visible rather than disappearing from the count.** The
definitive list is in the plugin's shared conventions, section 8.

| Question | Raised | Status |
|---|---|---|
| | YYYY-MM-DD | Open / Resolved YYYY-MM-DD |

---

## 7. Changelog
One line per session, dated. This is the audit trail — **most recent on top** — so you catch drift before it
compounds without scrolling past old entries.

**Append-only.** An existing entry's text is never rewritten. A correction goes in a NEW dated entry saying
what it corrects, never by editing history. When this section outgrows the file, everything before one
uniform cut date moves verbatim to `Project Memory Changelog Archive - [Project Name] (through YYYY-MM-DD).md`
and a single italic pointer line replaces it here.

- **YYYY-MM-DD:** [What was added/changed/resolved this session]

---

## Size discipline
Warn at 40,000 B, stop and ask at 60,000 B — decimal, exact — per file, checked at every open and every
update. *(These two numbers are written out here on purpose, and the duplication is deliberate: this file
is inherited into your data repo, where the plugin's `CONVENTIONS.md` does not exist, so a pointer to it
would resolve to nothing. Do not "remove the redundancy" — a dangling pointer is worse than a stale
number. `CONVENTIONS.md` section 3 remains where the lines are pinned; if the two ever disagree, that
section governs and this line is stale.)* A file past the
stop line is not broken, but the next update should split it first: cut log-shaped sections by one uniform
date into a dated archive companion, or, where a large section is not date-shaped (a live backlog, a
definitions table), move that whole section to its own companion and leave a pointer stub behind. Commit the
companion before the pointer that names it.
