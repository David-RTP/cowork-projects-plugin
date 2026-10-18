---
name: project-update
description: >-
  Trigger/alias "upd". THE update command: commits this session's work to BOTH of a project's standard
  files in one pass - the memory doc (decisions, changelog, open questions - always) and the notes file
  (narrative and reference content - where warranted). Replaces the retired um/un commands. Use when the
  user says "upd", "update", "update memory", "update the notes", "log this", "save this", or invokes
  /project-update. Also refreshes the Master List row and re-checks file sizes, so it is the only
  end-of-session command the user needs. Run at least once per sitting.
---

# Project Update (upd) - one command, both files

Commit this session's work to the current project's memory doc AND notes file in one pass.
This command REPLACES the old separate um (memory) and un (notes) commands - if the user says
"um" or "un" out of old habit, run this combined update and mention the new name once.

## Resolve the data repo FIRST (config-driven - never hardcoded)
Every command operates on the user's projects DATA repo - the GitHub repo holding the project folders -
identified by the file `Project Setup/cowork-projects.config.json` inside that repo:
- If the data repo was already resolved earlier THIS session (or the user names one), use it.
- Else discover it: `get_me` for the user's GitHub login, then `search_code` for
  `filename:cowork-projects.config.json user:<login>`. Exactly one hit -> fetch that config and use its
  `owner`/`repo`/`branch` (tell the user which repo was resolved). Multiple hits -> ask which one.
- **ZERO HITS IS NOT PROOF OF ABSENCE — fall back to a repository search** (new in v0.12.0, backlog #18). `search_code` returns nothing for a PRIVATE repo, reproduced six times, most recently by the very session that shipped this rule. Since a data repo is normally private, the empty result is the COMMON case rather than the missing case, and treating it as "no config" sends the skill down the wrong branch: it offers to bootstrap a repo the user already has. So on zero hits, do NOT ask yet — run `search_repositories` for `user:<login>`, then probe `Project Setup/cowork-projects.config.json` with `get_file_contents` in each candidate, most-recently-updated first. Exactly one repo holds the file -> use it and say which. More than one -> ask which. Only after that probe comes back empty is the config genuinely absent.
- Still no hit -> ask ONCE for `owner/repo`. If that repo has no `Project Setup/` folder, offer to run `/project-setup-init` (psi) to bootstrap it. Never guess, and never fall back to a hardcoded repo.

Definitions used below: `<Repo>` = `owner/repo` from the config; `<Timezone>` = the config `timezone`
(default `America/New_York`); `<Today>` = the date **computed AT WRITE TIME by a shell command,
converted to `<Timezone>`**; `<Branch>` = the branch every commit in this run targets, resolved as
follows.

⚠️ **EVERY DATE THIS SKILL WRITES IS `<Today>` - the changelog entry, a decision or open-question
row, an index row's `Date` cell, the row-count restamp, a superseded or resolved stamp, and the
Drive deliverable timestamp.** **The `<env>` date is explicitly NOT a clock source** - it has been
measured drifting by 18 days - and neither is a date carried from a file's previous value or read
off a UTC clock. If no shell is reachable, use the no-shell fallback (`../../docs/CONVENTIONS.md` section
12): the FIRST commit's server timestamp from this session's writes, converted to `<Timezone>`, and
say in one line that the fallback is in use.

**`<Branch>` IS A PER-INVOCATION PARAMETER, NEVER A CONFIG FIELD.** The user supplies it by naming a
branch in the invocation - `upd branch: <name>`, or any phrasing that plainly names one ("upd on
branch test/x"). **Absent an explicit branch in THIS invocation, `<Branch>` is the config `branch`
value (default `main`).** It does not persist: it is not written to the config, not remembered across
invocations, and a later `upd` with no branch named returns to the config value. **A branch stored in
the config would leave the data repo pointing at a test branch after the test ended, and the next
real `upd` would silently write there** - a wrong-branch write nobody looks for. If the named branch
does not exist, STOP and say so; never create it, and never fall back to the config value silently.

## Determine the project
- If a project was already loaded this session (via `/project-open`) or the user names one, use that
  `<Project Name>`. `<Folder>` == `<Project Name>` for active projects, or `zArchived - <Project Name>`
  if the project is archived.
- If it is ambiguous or none is loaded, ask exactly once: "Which project should I update?" Do not
  guess. Write project names out in full.

## LOAD THE DECLARED STANDING RULES - before you compose anything (new in v0.32.0, backlog #93)

⚠️ **THE RULES GOVERN WHAT THIS COMMAND WRITES, SO THEY ARE FETCHED BEFORE ANYTHING IS COMPOSED -
NOT BEFORE THE COMMIT, AND NOT AS A CHECK AFTERWARDS.** A sitting that begins with `upd` never runs
`project-open`, and until now nothing else fetched them: the command wrote into a project whose
governing rules it had never read. **Loading them at the commit step would be too late** - by then
the content exists, and rules routinely change the required FORM of what is being written, not just
its wording.

**Fetch the memory doc's header block first, then every path its `**Standing Rules:**` line
declares.** You must fetch the memory doc anyway to hold the blob SHA the write is locked to, so
this costs one read of a header you already have.

**Parse the line exactly as `project-open` does** (`../project-open/SKILL.md`, "How the list is
written and parsed" - that file is the fuller statement and governs if the two ever disagree). The
operative half, stated here because this is the step that performs the fetch:
- **Paths are separated by SEMICOLONS.** Strip backticks and surrounding whitespace from each
  entry, ignore empty entries, and treat a single path with no separator as the one-element list.
- **A value of `None`, compared with whitespace and backticks stripped and IGNORING CASE, is a
  recorded decision and not a path.** The list is empty: fetch nothing, report nothing.
- **An entry containing a `/` resolves from the REPO ROOT; an entry with no `/` resolves as
  `<Folder>/<entry>`.** These are READ-ONLY - never write through a declared path.
- **`upd` obeys them in DECLARATION ORDER, and where two declared files conflict the EARLIER path
  wins and the conflict is REPORTED, not resolved silently.**

**Say what loaded, in one line, using the vocabulary `project-open` already uses:** `Standing
Rules: 3 declared, 2 fetched, 1 missing (<path>)`. Report EVERY failed path, not just the first. **A
declared file that is not there is a finding, not a reason to stop** - carry on with the rest and
write. **An empty list reports nothing at all.**

⚠️ **THE HONEST LIMIT, STATED HERE RATHER THAN LEFT TO BE ASSUMED: THIS LOADS GOVERNANCE. IT DOES
NOT ESTABLISH THAT A PROJECT IS OPEN.** There is no state a writing command can observe that says
`project-open` ran, so this step closes the door it names - the rules are now always read before
the write - **and it does not verify anything about how the sitting started.** Do not read a clean
`Standing Rules:` line as evidence that the rest of a proper open happened.

## The two target files (one pass, both considered)
1. `<Folder>/Project Memory Data - <Project Name>.md` - the MEMORY doc. **Always updated.**
2. `<Folder>/Project Notes - <Project Name>.md` - the NOTES file. **Updated where the session
   produced narrative, research write-ups, status changes, new tasks, or reference content worth
   keeping.** If nothing merits a notes change, say so briefly - do not pad.

⚠️ **BOTH TARGET FILES ARE FETCHED WHOLE, HERE, BEFORE ANYTHING IS COMPOSED - AND NOTHING ABOVE
HAS FETCHED THE NOTES FILE AT ALL.** The standing-rules step reads the memory doc's HEADER BLOCK
and the paths that header declares; a header is not the file, and the notes file is not named
there. **Fetch both files in full at this step and hold each one's BYTE COUNT and blob SHA.** This
is the one fetch the SIZE GATE below consumes and the one the SHA lock is taken from - **the
`Re-fetch ... immediately before editing` rule below is a re-read of THESE bytes at write time, and
it is not a substitute for holding them now**, because by then the content is already composed.
⚠️ **THE NOTES FILE IS FETCHED EVEN WHEN NOTHING MERITS A NOTES CHANGE.** *"Say so briefly"* is a
conclusion reached FROM the file, not instead of reading it - and the size gate is PER FILE, so a
notes file nobody fetched is a notes file nobody measured.

## WRITE SAFETY - every commit is SHA-locked (changed in v0.11.0)
Full rules: `../../docs/CONVENTIONS.md` section 2. The short form, because this skill is where it matters
most:
- **Write with `create_or_update_file`, locked to the blob SHA you read. Never `push_files` for a
  file that already exists.** `push_files` accepts no blob SHA and no base commit, so a concurrent
  write is not rejected - the losing write disappears silently with a success response.
  `create_or_update_file` takes a `sha` and fails loudly instead.
- **One command is not one commit.** `upd` stays one thing the user types while making two or more
  locked commits underneath. That is the intended shape, not a compromise.
- **Write order: notes first, memory last.** If the memory commit fails, the session merely looks
  unrecorded. Reversed, the memory changelog would assert a notes write that never landed - a
  document that lies about itself.
- **Re-fetch each file's content AND its blob SHA immediately before editing**, and base the edit on
  those exact bytes. Confirm the returned blob SHA after each write; if a write is rejected as stale,
  re-fetch and re-apply - never retry with the old SHA and never fall back to `push_files`.
- **PRE-COMPUTE THE EXPECTED BLOB SHA FROM THE EXACT BYTES YOU ARE SENDING, AND COMPARE IT TO THE
  SHA THE CONNECTOR RETURNS.** This is the BYTE-FIDELITY CHECK the lint rules refer to, and it is
  strictly stronger than re-reading the file and hunting for a substring: it proves that what landed
  is what was sent, byte for byte, rather than that one string survived the trip. **BOTH CHECKS STAY,
  because they fail differently** - the SHA comparison proves fidelity, the re-read proves the item
  is in the file you fetched back.
  ⚠️ **AND AN EXPECTED SHA IS VALID ONLY AT THE INSTANT IT IS DERIVED - RE-DERIVE IT AT RUN TIME,
  NEVER CARRY ONE FORWARD.** A SHA written into a doc, a test or a handoff decays exactly as fast as
  a release name inside a standing order, and it fails WORSE: a stale version number reads as
  obviously old, while a stale 40-hex SHA reads as PRECISE and is therefore believed. Anything naming
  an expected SHA must either name the CONDITION instead - *"the file's blob, whatever it is when the
  run starts, is unchanged after"* - or carry an explicit re-derive-before-use instruction. **These
  two rules are one rule: "pre-compute the SHA" without "re-derive it at run time" manufactures the
  stale-expectation failure the first half exists to catch.**
  ⚠️ **THREE LIMITS, STATED HERE RATHER THAN BURIED.** **(1) NEITHER CHECK CATCHES A WRONG
  DESTINATION.** Both operate on the path you addressed; a write routed to spoke B when it should
  have gone to spoke A passes both cleanly. **(2) ON A FULL-FILE CONNECTOR WRITE THE SHA COMPARISON
  IS A POST-HOC AUDIT, NOT A GATE.** There is no patch API and no path-based push -
  `create_or_update_file` and `push_files` both take content as a string - so the bytes round-trip
  through your own output and **the write COMPLETES before the comparison can run.** A corrupted
  emission lands, and is then detected. Never describe it as a gate. **(3) WHEN THE SHA MATCHES, THE
  RE-READ IS ENTAILED, NOT CORROBORATING** - hash(remote) = hash(local), and local holds the item,
  therefore remote does. The re-read earns its keep where the SHA comparison is unavailable.
- Insert/append/update-in-place per the rules below - preserve all existing text otherwise, including
  the memory doc's `**Status:**` line.
- If a claim in a doc looks surprising, verify it against the file it describes before acting.

## SIZE GATE - check before writing (new in v0.11.0)
Per file: **warn at 40,000 B, stop and ask at 60,000 B** — decimal, exact (`../../docs/CONVENTIONS.md`
section 3). **The byte counts come from the whole-file fetch of both target files specified above**,
so this costs nothing.
- Crossing 40,000 B: tell the user the file passed the warn line, name the byte count, name the split
  options, then carry on and write.
- At or past 60,000 B: say the size, say why it matters, offer the split, and let the USER decide. Do
  not write silently past it, and do not refuse on arithmetic alone. ⚠️ **THIS FIRES ON EVERY
  `upd` IN A SITTING - the second and later runs included, and including when `project-open`'s
  health check already named the file at open.** `../../docs/CONVENTIONS.md` section 3's
  once-per-sitting rule scopes to the WARN line only and **never** suppresses the stop. **Consent
  given for one write is not consent for the next one**, and the post-write confirmation below
  reports a breach rather than preventing one.
- Splitting is its own operation with its own rules (`../../docs/CONVENTIONS.md` section 4): cut by date
  where the section is log-shaped, extract the whole section where it is not, reconcile
  programmatically, and commit the companion BEFORE the pointer that names it.

## BEFORE WRITING THE MEMORY DOC - does it hold its own rules? (new in v0.11.0)
Some memory docs are a **hub**: navigation only, with decisions and open questions living in indexed
companion "spoke" files. Such a doc says so - usually a `**Layout:** Hub-and-Spoke` line, and always
a short "How to append" block stating where each kind of item goes, plus an index table with a `Kind`
column instead of a `## Open Questions` table.

**Where the doc carries its own append rules, read and obey THAT BLOCK rather than the layout assumed
below.** The doc is the authority on its own shape. Do not hard-code a structure the file itself
contradicts.

**Never write a decision or an open question into a hub whose own rules say it holds neither** - a
doc that documents its own structure and is then violated by the tool that read it is worse than one
that never said anything. **This refusal is unchanged and still governs every case the routing below
does not cover.**

### Routing a new decision or open question into a spoke (new in v0.15.0, backlog #20b)

Applies ONLY to FILING A NEW decision or open question. **Resolving, superseding, closing or moving
an item to an archive is NOT implemented - for those, the refusal above still applies: say what you
would have written and where, and let the user place it.**

1. **Match the item to a spoke by reading the hub's own index.** Group the index rows by their
   destination column and read the hub's own description of each destination. Choose the destination
   whose existing items are the closest topical match. **Derive the destination set from the file;
   never from a list in this skill.**
2. **STOP AND ASK WHEN THE MATCH RESTS ON A DEFINITION AND MORE THAN ONE DEFINITION FITS.** The
   threshold is a CONJUNCTION and BOTH halves must hold: **(a)** the destination you chose holds NO
   existing item on the same subject, so the match rests on the bucket's DEFINITION rather than on a
   SIBLING ITEM, **AND (b)** more than one destination's definition covers part of the item. Both
   true -> STOP: present the candidates and the exact text you propose to write, and let the user
   place it. ⚠️ **THIS REPLACES "if the match is not unambiguous", WHICH WAS GRADED BY THE PARTY
   WITH AN INTEREST IN PROCEEDING** - measured 2026-08-12, an agent named an item as its least
   confident call IN ITS OWN CONFIRMATION TABLE and then resolved it anyway. Self-assessed
   confidence is not the test, and good reasoning makes it worse rather than better; (a) and (b) are
   properties OF THE ITEM AND THE INDEX, checkable by anyone reading the same file.
   **Why neither half alone:** (a) alone fires on every genuinely new subject a project takes up,
   including the ones only one destination could possibly hold; (b) alone fires on items that route
   fine, because a correctly routed item commonly has a defensible second reading too. **The
   principle underneath: definitions are written broadly and stretch under pressure; a sibling item
   is evidence of what the project ACTUALLY DOES.**
   **THE COST, STATED PLAINLY:** roughly one false stop per new sub-project, and self-limiting - the
   second item on a subject has a precedent row. **The asymmetry decides it: a false stop costs one
   question; a false pass costs a misfiled item that nothing will ever look for.**
   ⚠️ **STOPPING DOES NOT MEAN A RIGHT DESTINATION EXISTS.** Where no bucket in the file fits the
   item, say THAT - do not pick the least-bad one. Adding a destination is the user's call, not this
   skill's.
   **This threshold governs step 1's destination match. Where another step here cites "rule 2" for a
   generic STOP AND ASK - step 5's missing-date-boundary case does - that citation still stands; the
   conjunction narrows nothing about when else to stop.**
   Never guess, and never break a near-tie by picking the larger spoke.
   **Abort-on-ambiguity is permanent and never graduates.**
3. **WRITE THE SPOKE FIRST, THEN THE INDEX ROW.** Write the spoke with `create_or_update_file`,
   SHA-locked as every write here is. That call returns the spoke's new blob SHA. **The SHA is a
   GATE and a REPORTING obligation - it is NOT written into the index row.** The index schema is
   `Date | Item | Kind | Where` and has no SHA column; putting one there would be a schema change to
   every existing row, which this release does not make.
   **THEN VERIFY THE SPOKE ACTUALLY HOLDS THE ITEM, BEFORE WRITING THE INDEX ROW.** Re-read the
   spoke with `get_file_contents` and assert THE ITEM'S OWN TEXT IS PRESENT in what comes back.
   **This assertion is the only thing standing between routing and a DANGLING INDEX ROW** - a row
   naming a spoke that does not contain the item. **It must be a positive match on text you are
   holding, never a check that the call returned something**, because a call returning something is
   what a wrong path does too. If the text is not there, **the spoke write did not land: STOP, report
   it, and do NOT write the index row.** An index row must never name an item that is not there yet.
   ⚠️ **STATED LIMIT: this catches a spoke write that silently did not land, a wrong path, and a SHA
   that was never received - because all three fail the same text assertion. It does not bind an
   agent that skips the re-read.** Nothing in a SKILL.md can. The check is cheap - one read per
   routing - and it is the check whose absence would be invisible until something followed the row.
   ⚠️ **WHERE THE SPOKE WRITE'S SHA COMPARISON MATCHED (see WRITE SAFETY), THIS RE-READ IS ENTAILED
   BY IT RATHER THAN INDEPENDENT EVIDENCE** - hash equality plus "the copy I am holding contains the
   item" yields "the remote contains the item" without a second call. Keep the re-read anyway: it
   still runs where no SHA came back, and it is the half a reader can check by eye. ⚠️ **NEITHER
   CHECK TESTS WHETHER THIS WAS THE RIGHT SPOKE.** Both operate on the path you addressed, so a
   cleanly executed write to a wrongly CHOSEN destination passes both - that is what steps 1 and 2
   are for, and it is a different failure from the wrong PATH named above, which does fail the text
   assertion.
   **AFTER THE INDEX ROW LANDS, UPDATE THE INDEX'S ROW-COUNT LINE IF IT HAS ONE** - the line stating
   the index total (e.g. `**Row count: N as of <date>**`), which typically instructs in its own words
   that it be updated and dated when rows are added. **Recount from the table; never increment a
   number you did not verify**, and restamp the date. Adding a row without touching it leaves a count
   wrong by exactly the row you just wrote, carrying a date that claims it was checked. **Part 1
   step 3's general "refresh any size or count you can compute for free" does NOT reach a reader
   following these routing steps in sequence** - which is why the obligation is restated here.
   *(Sibling of backlog #115, which fixes the same gap in the v0.13.0 spec; fixing one does not close
   the other.)*
4. **If the index write fails after the spoke write landed, say EXACTLY that.** Name the spoke, its
   new SHA, and the fact that the index row is missing. **Report it as a PARTIAL. Do not describe the
   session as saved**, and do not retry silently - a half-written pair that reads as complete is
   worse than a failure, because nothing will look for it later.
5. **The Changelog entry is ROUTED, NOT ASSUMED - AND A SECTION MAY CARRY MORE THAN ONE POINTER.**
   Read the hub's Changelog section before writing to it. If that section is a POINTER - or SEVERAL -
   to companion files, the entry goes to the companion THE POINTERS name, and **the hub is left
   byte-identical**. Only where the section actually holds entries does the entry go in the hub.
   **Where the section names MORE THAN ONE companion, choose by THE ENTRY'S OWN DATE against the
   boundary the pointer prose states** - a split changelog names a live file and a dated archive, and
   the entry belongs to whichever one covers its date. **A new entry is not automatically the live
   file's**; read the boundary and check the date against it. ⚠️ **If the pointer prose gives NO date
   rule, or gives one that does not cover the entry's date, that is the ambiguous case: STOP AND ASK
   under rule 2 above.** Do not default to the live file, do not break the tie by size, and do not
   infer a boundary from the companions' filenames or contents. *(Corrected in v0.15.0: this skill
   previously stated the changelog "still goes in the hub, because that is where the hub keeps it" -
   which is false for any hub whose changelog has been split out, and the split is common in exactly
   the large docs that become hubs.)*
   ⚠️ **THE DATE YOU COMPARE IS `<Today>`, COMPUTED AT WRITE TIME - THIS IS THE PATH WHERE AN
   UNSOURCED DATE PICKS THE WRONG FILE.** Everywhere else a bad date mis-stamps an entry; here it
   ROUTES the entry, and the archive is the file nobody re-reads.

**CONFIRMATION.** Compose every routing for the session and present **ONE table at the end, before
writing anything**: the item, the proposed destination, the runner-up, what it matched on, and how
many items already live there. Write only after approval. The evidence is part of the table because
a bare yes/no gets rubber-stamped at the end of a long session.

**GRADUATION IS PER DESTINATION, NOT PER SESSION.** A destination auto-accepts after **3 items routed
into it and left unchanged**. A session count alone can graduate a matcher that has never seen half
the destinations. **DEMOTION: if a routed item is later moved by hand, that destination demotes and
resumes confirming** - a graduation without demotion is an undated state claim in behavioural form.

**THE RUN SUMMARY ALWAYS NAMES EVERY DESTINATION IT WROTE, AND THE NEW BLOB SHA OF EACH.** Both the
spoke and the index row, by path and by SHA, on every successful run - not only when something fails.
A destination that is written and not named cannot be checked by anyone later. **This never
graduates.**

## Part 1 - MEMORY doc (always)
Follow the doc's OWN headings; structure comes from the live template. House ordering rule - PAST
lists most-recent-on-top; Open Questions open-on-top then resolved-newest-first:
1. Add a dated entry to the **TOP** of the Changelog summarizing what was decided/learned this session.
   **The Changelog is append-only** (new in v0.11.0): one dated entry per session, newest on top, and
   an existing entry's text is NEVER rewritten. A correction goes in a NEW dated entry that says what
   it corrects, or in the file's descriptive header block - never by editing history. If the changelog
   has been split, its archive is `Project Memory Changelog Archive - <Project Name> (through
   <DATE>).md` and the resident file keeps only the pointer line; do not move entries back.
2. Update whichever other sections are relevant:
   - **Decisions Log** - rows at the TOP (newest-first); mark superseded rows "Superseded", never delete.
   - **Data Sources & Definitions** - **supersede, do not overwrite** (new in v0.11.0). This section is
     not log-shaped, so it never gets cut by date and it accumulates in place - which is exactly why
     editing a definition over the top of itself is destructive. When a definition, a source, or a
     path changes, leave the old text in place, mark it superseded with the date, and add the new one
     above it. Overwriting destroys the record that it ever changed, which is usually the thing
     someone came looking for. Only a true typo is corrected in place.
   - **Open Questions** - open items on top; when resolving one, mark it Resolved with the date and move
     it into the resolved group (most-recently-resolved first). Never delete.
   - **Companion Files** index - if a companion was added/removed.
   - **Deliverables & Links** - if a Google Drive deliverable (Doc/Sheet/PDF) was added or replaced,
     record it with its link and a `<Timezone>` timestamp. (GitHub stays the source of truth for
     memory/notes; Drive holds only rich deliverables.) See DRIVE ACCESS MODES below for what
     "replaced" means in each mode.
3. **Re-verify every pointer row you touch, and every pointer row that claims a state** (new in
   v0.11.0). A row that names a companion file, a size, a blob SHA, a version, or a count is a claim
   about right now, and an undated claim rots silently while still sounding authoritative
   (`../../docs/CONVENTIONS.md` section 7). So, on each update: confirm each companion named in the index
   still exists at that path, and refresh any size or count you can compute for free from what you
   already fetched. Where the value cannot be refreshed, stamp it with the date it was measured
   instead. Report a pointer that no longer resolves - do not quietly delete the row and do not
   invent a new target for it.
4. Preserve the `**Status:**` line as-is (only `/project-archive` changes it) and the `**Type:**` line
   as-is (`Project` | `Learning Notebook`; only the user changes it, and changing it means re-shaping
   the notes file too - never flip it as a side effect of an update).

## Which notes structure? Read the memory doc's `**Type:**` line
`Project` (or the line missing, which is every older doc) -> Part 2A below. `Learning Notebook` ->
Part 2B. Read the line; never infer the type from the entry's name. If the Type line and the notes
file's actual headings disagree, say so once and ask - do not silently reshape either file.

⚠️ **MATCH THE VALUE CASE-INSENSITIVELY, AND TREAT `Reference Notebook` AND `Learning Notebook` AS
THE SAME VALUE.** `Reference Notebook` is the current spelling; `Learning Notebook` is the legacy
one, kept as a PERMANENT synonym with NO END DATE. `reference notebook`, `Reference Notebook` and
`LEARNING NOTEBOOK` are the same answer, and so are `project` and `Project`. **Wherever this file
writes `Learning Notebook` as a Type VALUE, either spelling satisfies it.** The changeover is
deliberately staged - detection first - so one repo may hold both spellings at once and
indefinitely, and **a plugin that only works when the user's capitalisation matches the template is
a trap for every repo that is not the one it was built on.**

## Part 2A - NOTES file for a PROJECT (where warranted; notes template v2 sections)
The notes file's standard shape (template v2): (1) The Project in One Paragraph, (2) Current Status,
(3) Open Tasks / Next Steps, (4) Quick Reference, (5) Phases at a Glance, (6) Gaps & Issues,
(7) Session Notes - the dated log, at the BOTTOM. Older files may predate some sections; follow the
file's own headings and never invent content to fill a section.
- **Current Status** - update IN PLACE (This week / Coming up / Ongoing); it is a snapshot, not a log.
- **Open Tasks / Next Steps** - FUTURE list, first-to-do on top; add/check off/remove per the session.
- **Quick Reference** - pin newly-surfaced durable key facts (codes, contacts, frequencies, settings).
- **Session Notes** - insert a new dated section at the **TOP of the dated area** (most recent on top)
  when the session has narrative or research worth keeping. Do NOT append at the bottom.
- **Phases/Gaps readiness check (ALWAYS run) - but read the stub's own words FIRST (new in v0.12.0,
  backlog #51):** a section whose content was EXTRACTED to a companion leaves a **pointer stub, and a
  stub is not a placeholder** (`../../docs/CONVENTIONS.md` section 4). Offering to rebuild one forks the
  content into two live copies with nothing reconciling them, and the offer arrives at the end of a
  long session when the user is agreeing to things. Decide in this order, strongest test first:
  1. **AUTHORITATIVE - the section contains a backticked filename or link that RESOLVES to a file in
     the project folder.** Structural, not prose: a genuine placeholder never names an existing file.
     Reuse the single directory listing shared with the reconciliation check below - never fetch it
     twice.
  2. Corroborating - the text matches `**Extracted <date>**` or `moved verbatim to`. Prose, so it is
     fragile to rewording.
  3. Weakest - the Companion Files index has a row naming this section. It matches prose in a
     DIFFERENT file, so it fails whenever a row describes a companion without naming the section it
     came from. **Never decisive on its own.**
  **Pointer stub -> do NOT offer.** If the session produced content belonging there, say so in one
  line and name the companion. **Genuine placeholder -> offer exactly as before**, and populate only
  on the user's yes; never automatic. **Stub whose companion no longer resolves -> report the broken
  pointer and STILL do not offer to repopulate** - a dangling pointer is a reconciliation finding
  (section 9), not an invitation to regrow the table.

## Part 2B - NOTES file for a LEARNING NOTEBOOK (topic filing)
A notebook's notes file is organized by TOPIC, not by date: (1) The Notebook in One Paragraph,
(2) Quick Reference, (3) To Learn / Open Threads, (4) theme headings (`# Theme`) holding topic
sections (`## Topic`), (5) Learning Log at the BOTTOM. **The whole point is that a reader can open a
topic and get the current best answer without reconstructing it from a log.** So:
- **File this session's learning INTO TOPICS, editing in place.** Find the topic the material belongs
  to and update it - correct it, extend it, sharpen it. Only create a new `## Topic` when the material
  genuinely has no home. Stamp what you touch: `*Added: <date>*` on a new topic, `*Updated: <date>*`
  on an edited one.
- **Corrections are written as corrections.** When new understanding overturns what the topic said,
  state that explicitly ("previously recorded as X; that was wrong because Y") rather than quietly
  replacing the text. The correction is usually the most valuable line in the topic.
- **TOPICS ARE NEVER DELETED.** A topic that no longer holds up is corrected in place. A topic that
  has stopped being useful is marked retired, or merged into another topic with a pointer left behind
  - never removed, so nothing that links to it dangles. This is the notebook's version of the
  append-only rule that governs the memory doc.
- **Themes are editorial and emerge from real material.** Add a theme when several topics start
  rhyming; split or rename one when it stops earning its place. Never invent a starter taxonomy, and
  never file into a theme that doesn't fit just to avoid making a new one. Renaming a theme is a
  content change worth a Changelog line.
- **To Learn / Open Threads** - FUTURE list, first-to-learn on top. Add threads the session opened;
  when a thread gets answered, remove it and make sure the answer landed in a topic. A thread that
  vanished without a corresponding topic edit means the learning wasn't actually captured.
- **Quick Reference** - pin only durable lookups (definitions, constants, commands, the number always
  forgotten). If an entry needs explaining, it belongs in a topic with a pointer from here.
- **Learning Log** - insert ONE line at the TOP: the date and which topics were added or changed.
  It is a pointer index, not the content. If the log line is growing into a paragraph, that paragraph
  belongs in a topic.
- **The failure mode to avoid:** writing the session's material as a dated entry at the bottom and
  calling it filed. That is the project shape, and it is exactly what the notebook type exists to
  prevent. If the material truly resists topic filing, say so and ask where it should go.

## Lint before you commit (new in v0.11.0)
Cheap checks that catch damage which is invisible in review (`../../docs/CONVENTIONS.md` section 6). Run
them against the content you are about to send, not against what you meant to write:
- ⚠️ **BEFORE EMITTING A FILE REPRODUCED FROM CHUNKED READS, RECONCILE THE BYTES: the sum of the
  chunk byte-counts you actually read MUST EQUAL the file's own byte count.** This is the only check
  that catches a region falling OUTSIDE every chunk - re-reading the chunks cannot, by construction,
  because each chunk comes back complete and correct and the missing region is in none of them.
  ⚠️ **THE TRAP IS GENERATED BY THE EDIT ITSELF:** chunk boundaries computed from the PRE-edit file
  are wrong for the POST-edit file whenever the edit changes the line count, so an inserted row can
  push the last line past the last chunk. One line of arithmetic, independent of the content, and
  unlike a re-read it cannot pass while content is missing - a missing region always changes the sum.
- **No blank line inside a pipe table.** A blank line silently ends the table; the rows after it
  still look fine to a human and stop being a table to a parser.
- **Uniform column count within each contiguous table block.** Count cells by splitting on pipes that
  are NOT backslash-escaped - an escaped pipe inside a cell is content, not a delimiter, and counting
  it raises false alarms on exactly the rows most worth checking.
- **Exactly one trailing newline** at end of file - **applied ONLY to bytes THIS WRITE INTRODUCES.**
  The trailing newline is load-bearing: it is not added for you, and without it the file differs from
  the intended file by one byte. ⚠️ **NEVER apply this check to an EOF THE FILE ALREADY HAD.** Where
  the fetched file ends with no trailing newline, or with more than one, **the WRITE SAFETY rule
  "preserve all existing text otherwise" GOVERNS and this lint is SILENT**: leave the file's final
  bytes exactly as fetched, and do not report it as a lint failure. A lint that rewrites bytes the
  session did not author is not a lint; it is an edit wearing a rule's name - and it breaks the
  byte-fidelity check that exists to prove the write was faithful.
- **No literal multi-command shell chains** in committed content - see COMMIT TRANSPORT SAFETY below.
If a check fails, fix the content and re-run it. Do not commit and clean up afterward; the cleanup
commit is a second chance to lose someone else's write.

## Commit (SHA-locked, notes first, memory last)
- **Notes file first.** `create_or_update_file` with the notes file's fetched blob SHA. Message:
  `upd: <short summary> (notes)`. Commit to `<Branch>`. Skip if there is no notes change.
- **Memory doc last.** `create_or_update_file` with the memory doc's fetched blob SHA. Message:
  `upd: <short summary>`. Commit to `<Branch>`.
- Each call returns the new blob SHA - confirm it, and say so. If a call is REJECTED as stale, that
  is the lock doing its job: re-fetch, re-apply the edit to the new bytes, and write again. Never
  retry with the stale SHA, and never work around a rejection by switching to `push_files`.
- If the notes commit lands and the memory commit fails, say exactly that. Do not describe the
  session as saved.
- Confirm to the user what landed in each file (memory always; notes or "no notes change needed").
  ⚠️ **FOR EACH FILE THIS RUN WROTE, GIVE THE POST-WRITE BYTE COUNT, THE HEADROOM, AND THE STOP
  VALUE YOU MEASURED AGAINST - all three, in one line, ON EVERY RUN AND AT EVERY SIZE** - for
  example `memory doc 12,310 B, 47,690 B of headroom to the 60,000 B stop`. Say in the same line if
  a file has crossed the 40,000 B warn line. **A BARE BYTE SIZE DOES NOT SATISFY THIS BULLET, AND
  NEITHER DOES SILENCE AT A SMALL SIZE.** The SIZE GATE above is a PRE-WRITE check whose correct
  output IS silence when no line is crossed, so nothing upstream produces this number for you - at
  12 KB the gate says nothing and this line still owes all three figures. **The headroom, not the
  size, is what decides whether the next session must split first**, and the stop value is named
  so the subtraction can be checked: `../../docs/CONVENTIONS.md` section 3 pins the warn at
  40,000 B and the stop at 60,000 B, decimal. **Name the stop on every run even so** - an installed
  tree predating that pin reads section 3 unpinned, so naming the value is what makes two sessions
  reporting different headroom for one file visible instead of invisible. Past the stop, report the
  headroom as a NEGATIVE with
  its magnitude - `61,102 B, 1,102 B PAST the 60,000 B stop` - never as an unquantified "over the
  limit", which reads identically at one byte over and at four thousand.
  **SCOPE: the two target files only - the memory doc and the notes file - and the Master List is
  deliberately EXCLUDED because this run does not hold its post-write size**: `upd` never fetches
  it, the `plu` call in step 1 below is what writes it, `create_or_update_file` returns a blob SHA
  and not a size, and section 8 forbids predicting one - so say nothing about the Master List's
  size rather than estimating one, and do not add a fetch to obtain it.
  **NAME `<Branch>` IN THIS CONFIRMATION ON EVERY
  RUN, and say whether it came from the invocation or from the config** - not only when it differs
  from the config value. An unnamed branch is how a wrong one goes unnoticed. **This never
  graduates.**

## What `upd` runs for you (new in v0.11.0 - one command, the whole update)
`upd` is the one command a user should have to remember at the end of a sitting. It therefore calls
the other skills that would otherwise have to be run by hand. Run these in order, after the two
commits above have landed:
1. **`project-list-update` (plu)** - refresh this project's row in the Master List: its one-line
   description, its open-question count, and its last-updated date. Always run; the Master List going
   stale between sittings is the single most common drift in the repo.
2. **Companion-file reconciliation - run it whenever the Companion Files index HAS ROWS, not when
⚠️ **A SECTION-8 POINTER IS FOLLOWED HERE TOO (new in v0.25.0, backlog #182).** The reconciliation
below runs when the Companion Files index has rows **OR when that section is a POINTER naming a
companion file** - zero rows plus a backticked `.md` filename in its prose. **Fetch the named file
and reconcile against it.** ⚠️ **THIS COMMAND DOES NOT INHERIT `project-open`'s RULE AND NEVER DID:
its reconciliation is built here, independently, so a fix there leaves this blind.** Before this
release, "run it whenever the index HAS ROWS" meant a stub silently disabled the check.
   the layout is hub-and-spoke (new in v0.12.0, backlog #52).** Layout was the wrong trigger: it is
   set solely by the memory doc's `**Layout:**` line, so splitting a NOTES file into companions never
   set it, and conditioning the check on the layout means the common case never gets checked
   (`../../docs/CONVENTIONS.md` section 9). Layout stops being the trigger and becomes an input to HOW:
   every doc with an index gets the three checks below, and hub-and-spoke ADDITIONALLY reconciles
   index and spokes link-by-link as before. **Zero companions -> skip silently.** **Share ONE
   directory listing** with the readiness check in Part 2A and with the orphan check - two
   independent listings per `upd` is pure waste. All of it is a CHECK, not a repair: report and let
   the user resolve; never silently rebuild an index.
   - **Direction 1 - every index row resolves to a live path.** Report a row that does not. Never
     quietly delete it and never invent a new target for it.
   - **Direction 2 - every non-standard markdown file in the folder appears in some index.** A
     companion outside every index is invisible to every future session, which is the same as lost,
     and only this direction finds it.
   - **The two-way assertion - the check that makes this item worth shipping.** Where an index row
     declares that its companion holds a named source section - machine-readably, as an HTML comment
     ending the description cell, `<!-- holds: memory doc section 3 -->` - assert that **that source
     section is a STUB, not content.** Walk #51's failure through to see why existence checking
     cannot catch this: the bad offer is accepted, the table regrows in the source file, the
     companion still holds its copy - the index row names a file that exists, the companion sits in
     the folder and is indexed, and both copies are well-formed, so **every existence check passes
     while two live copies of one table diverge.** Existence was never the problem. Rows carrying no
     marker get no assertion and produce no false finding; until a project's rows are backfilled the
     assertion is simply silent there, which is missing coverage rather than a bug.
   - **Report, never merge.** Told there are two copies, say so and STOP. The plugin cannot know
     which is authoritative - the notes copy may carry rows added since the fork - and merging is a
     judgement with no safe default, where a wrong answer silently loses work.
   - ⚠️ **The hub-and-spoke reconciliation branch shipped UNVERIFIED in v0.12.0 and is STILL
     UNVERIFIED.** Its acceptance test requires running `upd` against the pilot project that the
     do-not-`upd` standing order covers, so the test was unrunnable for that whole cycle.
     *(Corrected in v0.15.0: this note said the order lifts when "#20 ships in v0.13.0". #20 was
     SPLIT - the read-only half shipped in v0.14.0 and the write half is #20b, this release - and
     v0.13.0 carried neither. The order lifts on that item's acceptance tests passing, not on a
     version number. A release name inside a standing order goes stale on the first re-target.)*
3. **Size gate re-check - SPECIFIED IN THE `## Commit` CONFIRMATION ABOVE, NOT HERE.** That
   bullet's post-write byte count + headroom + named-stop line IS this re-check; report it there,
   once, and do not restate it in a second form here.
Each of these is reported in one line. If one of them fails, say which one and keep the rest - a
failed Master List refresh does not undo a landed memory commit, and must not be described as if it
did.

## Incident banners (data-integrity recovery convention)
- If the memory doc opens with a "⚠️ READ FIRST" incident banner, it is confirmed ground truth:
  honor it over conflicting entries, and NEVER remove, soften, or contradict it in an update.
  Only the user closes an incident (then move the banner's content into a dated changelog entry).
- If THIS session uncovers corrupted memory (e.g. a double/mis-dated import): reconstruct the truth
  from git history, get the user to confirm ground truth, ANNOTATE wrong entries in place (never
  delete), and ADD a "⚠️ READ FIRST" banner at the very top of the memory doc stating what happened
  and what the confirmed truth is. Fetch-first guarantees every future session and import sees it.
- Migration imports NEVER mark existing entries superseded - if imported content conflicts with the
  doc's current state, flag it and ask (see the Migration Runbook's Safety section).

## DRIVE ACCESS MODES (deliverables only - never memory or notes)
The config's `drive_access` field decides how Drive deliverables are handled. Absent = `connector`.
- **`connector`** (default): the Drive connector can create Docs/Sheets and inline SMALL files, but
  CANNOT rename or delete, and cannot carry a large binary (base64 inflates it past a single
  response). So: upload what fits, record the new deliverable with its `<Timezone>` timestamp, and
  TELL THE USER which older file to delete by hand (identify it by its timestamp). Anything too
  large to inline is reported as pending with a one-line reason - never silently degraded, never
  faked.
- **`rclone`**: the user has direct Drive access configured (see the plugin's
  `docs/Optional - Direct Drive Access (rclone).md`); the config's `rclone_remote` names the remote.
  Uploads run at full fidelity at any size, and replace means REPLACE: upload the new file, VERIFY
  it (compare size or checksum against the local original), and only then delete the superseded
  copy. Address folders by `drive_folder_id`, never by name.
- **Either mode, same honesty rule:** report what actually happened. If an rclone call fails, fall
  back to reporting the step as pending - do not claim a Drive step succeeded when it did not, and
  never delete an old copy before the new one verifies.
- Credentials are never read, echoed, or committed. The config holds the remote's NAME only.

### RCLONE PREFLIGHT - where the command actually runs
`drive_access: rclone` describes a MACHINE, not an account. rclone reads a config file on the user's
own computer, so it is usable only from a session that can run a shell command THERE: Claude Code on
that machine, or Cowork with the user's desktop app connected (the device bridge runs commands on
their machine). A Cowork session's own shell is a cloud container with no rclone and no route to the
user's config - phone sessions included. So before the FIRST rclone-mode Drive operation of a
session: confirm such a shell is reachable and that `rclone listremotes` lists the config's
`rclone_remote`. If it is not reachable, say so in one line and use `connector` behavior for the rest
of this session. Never claim an rclone step ran when it did not, and never edit the config to "fix"
it - the config is correct; this session just isn't on the right machine, and the next one from that
machine picks rclone back up automatically.

## Standing rules
- Run at least once per sitting - sessions compact; these files are the persistence layer.
- Never fill unknowns with plausible guesses - write "TBD" and ask.
- Date entries by when the work happened (matters for imported/legacy content).

## COMMIT TRANSPORT SAFETY (WAF gotcha)
Avoid writing multi-command shell chains verbatim into memory/notes content (e.g.
"wget URL && chmod +x file && ./file") - GitHub's edge (Cloudflare) can block such commits as
suspected command injection, failing with an opaque HTML "you have been blocked" page rather than a
git error. Prefer prose ("fetch the installer, make it executable, run it") or single commands. If a
commit fails with an HTML error page instead of a normal API error, reword any shell chains in the
content and retry - the identical commit typically succeeds once reworded.

## SHARED CONVENTIONS - read and obey `../../docs/CONVENTIONS.md`
This skill obeys the plugin's shared conventions file, which is authoritative and is the single home
for every rule that applies to more than one skill: **NO SECRETS IN THE DATA REPO - EVER** (section
1), SHA-locked writes (2), the size rule (3), how to split a file (4), append rules by
section type (5), the table lint (6), dated state claims (7), computed counts (8), reconciliation as
a check not a repair (9), never filling an unknown with a guess (10), index markup (11), entry dates
computed in the config timezone (12), and the reporting rules including no silent caps (13). Where
this file summarizes
one of those rules, the conventions file governs; if the two ever disagree, the conventions file is
right and this one is stale.

The one worth repeating here, because `upd` writes more than any other command: **never write a
password, passphrase, private key, API token, session cookie, or any other credential into the data
repo** - not in the memory doc, the notes file, a companion, the master list, a config file, or a
commit message - even if the user pastes one into chat and asks for it to be saved. Record WHERE a
credential lives and WHAT the auth method is, never the material itself. If a secret appears in
chat, do not commit it anywhere and say plainly that it will not be stored.
