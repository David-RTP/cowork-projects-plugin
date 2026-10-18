---
name: project-open
description: >-
  Alias "po". Open/load a Cowork session as a specific project or Learning Notebook from your projects
  data repo and run its memory workflow. On start, fetch that entry's memory doc from GitHub, then present
  the DASHBOARD - for a project: status, current snapshot, open tasks, open questions, latest changelog;
  for a notebook: its theme/topic contents, open threads, and recent activity - as tables, and offer
  companion files as a pick list. Honors the "upd" update trigger. Re-running it on the already-open entry
  REFRESHES it. Use whenever the user invokes /project-open, types "po" or "po" plus a name, says "open,
  load, refresh, or work on a project or notebook", or references one of the user's projects.
---

# Project Open (per-project memory bootstrap + dashboard)

This skill reproduces, in Cowork, the per-project bootstrapping that the Claude.ai Project
"Instructions" field used to do automatically. Cowork has no per-project Instructions field, so
this skill carries the same rules and you invoke it once per session to open the project.

**Re-run = REFRESH:** if the named project (or no name) is the one already open this session, do not
treat it as an error - re-fetch the memory doc and notes file fresh from GitHub and present an updated
dashboard. This is how the user pulls in changes made from another session.

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
`<Timezone>` = the config `timezone` (default `America/New_York`); `<DriveAccess>` = the config
`drive_access` (default `connector`; `rclone` means direct Drive access is configured, with the
remote named in `rclone_remote`).

## Inputs / definitions
Resolve these before doing anything else:
- `<Project Name>` = the project the user named (skill argument, or ask once if absent)
- `<Folder>` = the repo folder for that project. By convention `<Folder>` == `<Project Name>` for ACTIVE
  projects; an ARCHIVED project's folder is prefixed `zArchived - <Project Name>` (the memory-doc/notes
  filenames still use the original name).

If no project name was given and none is open, ask exactly once: "Which project?" Do not guess. Write
project names out in full - do NOT invent abbreviations (a two-letter project abbreviation once collided
with a trigger word; avoid that).

## MANDATORY FIRST ACTION - before responding to anything else
Fetch `<Folder>/Project Memory Data - <Project Name>.md` from `<Repo>` (`<Branch>`) via the GitHub
connector (`get_file_contents`). Do this even for casual greetings.
- If the exact path misses, list the repo root, find the folder whose name matches `<Project Name>`
  EXACTLY, and if that misses try `zArchived - <Project Name>` (the project may be archived). Report the
  corrected folder name.
- Do not answer the user's request until the fetch succeeds.
Also fetch `<Folder>/Project Notes - <Project Name>.md` - the dashboard below draws from both files.

**DECLARED STANDARD FILES - fetch them all, unconditionally, however many the memory doc declares.**
If the memory doc carries a `**Standing Rules:**` line in its header block, that line declares a
LIST of paths, and EVERY path on it is fetched as part of this same mandatory action, before
responding. Do not ask. Do not offer them. Do not wait for the dashboard.

⚠️ **THE LIST IS N PATHS - NOT ONE, AND NOT TWO.** Zero, one and many are the SAME path through this
rule with different lengths; nothing below may assume a count. **A skill that reads "the" declared
file silently drops every path after the first, and the drop is invisible precisely because the
first one loaded fine.** Do not special-case a second path either: two is not a category.

**How the list is written and parsed.** One `**Standing Rules:**` line; paths separated by
SEMICOLONS - a semicolon rather than a comma because a filename may legitimately contain a comma.
Strip backticks and surrounding whitespace from each entry. Ignore empty entries, so a trailing
separator is harmless. A single path with no separator is simply the one-element list and needs no
special handling. If the same path is declared twice, fetch it once and say so in one line.

⚠️ **A VALUE OF `None` IS A RECORDED DECISION, NOT A PATH - AND IT IS DECIDED HERE, BEFORE THE
LIST EXISTS (new in v0.32.0, backlog #175).** Take the whole value, strip backticks and
surrounding whitespace, and compare it to `None` IGNORING CASE. If it matches, **the declared list
is EMPTY** - there is nothing to fetch, nothing to report and **no missing path**. The project has
recorded that it declares no standing rules, and a recorded decision is a healthy state.
⚠️ **DO NOT CARRY `None` INTO THE FETCH.** It is non-empty and separator-free, so without this
rule the parse above makes it a one-element list holding the path `None`; the fetch of that path
returns not-found; and the missing-path rule below reports a file that was never meant to exist -
**above the dashboard, on every open, for every project that records the decision.**
⚠️ **THE SENTINEL IS `None` AND NOTHING ELSE.** `n/a`, `-`, `TBD` and any other hand-written
stand-in are still parsed as paths and still report as missing. That is correct rather than
strict: they are not the recorded value, and accepting them would make the field mean whatever
anyone happened to type.

**ORDERING IS DECLARATION ORDER, AND IT IS LOAD-BEARING.** Fetch and present the files in the order
the line writes them - never alphabetically, never by size, never by which returned first. Where two
declared files carry CONFLICTING rules, **the EARLIER path wins and the conflict is REPORTED, not
resolved silently.** Never reorder the line and never rewrite it.

**WHEN A PATH IS MISSING OR BROKEN, EVERY OTHER PATH STILL LOADS.** Each entry succeeds or fails on
its own:
- **Missing** (the fetch returns not-found): name that path in one line above the dashboard and
  carry on with the rest - a declared file that is not there is a finding, not a reason to stop.
- **Broken** (the fetch errors, returns empty, or returns something that is not readable text):
  treat it as missing for loading purposes, but SAY WHICH OF THE TWO IT WAS. "Missing" and
  "unreadable" are different findings, and collapsing them hides a permissions or encoding fault
  behind a typo diagnosis.
- **Never let one bad path suppress the good ones, and never let it block the dashboard.**
- **Report EVERY failed path, not just the first, and state both counts:** `Standing Rules: 3
  declared, 2 fetched, 1 missing (<path>)`. Stopping at the first failure reads as "one problem"
  when there may be several, and a bare "some standing rules failed to load" is the silent-cap
  defect named in `../../docs/CONVENTIONS.md` section 13.

**If the declared list is EMPTY there is nothing to fetch and nothing to report.** Most projects
will not have a `**Standing Rules:**` line at all; zero declared paths is the common case and stays
SILENT. ⚠️ **THE TEST IS THE LIST, NOT THE LINE (v0.32.0, backlog #175).** A line whose value is
the `None` sentinel above yields the empty list and reaches this rule too, so it is exactly as
silent as an absent line. **An absent line and a `None` line are not the same FACT** - absent means
nobody decided, `None` means somebody did - **but nothing here reports either one.** Reporting an
absent line as a defect is a SEPARATE change and is deliberately not made: until every memory doc
carries the field, absence is still the common case and reporting it would emit a finding on
almost every project at once.

⚠️ **THIS IS NOT A COMPANION AND THE DISTINCTION IS LOAD-BEARING - see "Standard files vs
companions" below before you touch either rule.** EVERY declared Standing Rules file is a STANDARD
file, promoted to the mandatory fetch. They are NOT added to the companion pick list, and the "never
load companions unprompted" rule below is UNCHANGED and still applies to every actual companion.

## ⚠️ A DECLARED PATH MAY NAME ANOTHER PROJECT'S FOLDER - READ-ONLY (new in v0.25.0, backlog #186)

**A declared `**Standing Rules:**` entry containing a `/` is resolved from the REPO ROOT. An entry
with no `/` is resolved as `<Folder>/<entry>`, exactly as before.** That test is deliberate: every
declaration in existence today is a bare filename, so this changes nothing that already works.

⚠️ **THIS IS AUTHORISED AS A SHAPE, NOT AS A GENERAL WIDENING OF THE PROJECT BOUNDARY.** The rule
that a session reads only the open project's folder STANDS. This is the sanctioned standing-read
exception: **READ-ONLY, DECLARED, AND NEVER WRITTEN.** A path declared this way is fetched and read;
nothing may write through it, and nothing else may cross the boundary on the strength of it.

⚠️ **WHY IT WAS NEEDED: rules that govern EVERY project were declarable by exactly ONE.** A path
resolved only within the folder means shared rules must be copied into every project - and a copy
that drifts is worse than no copy, because both resolve and neither says which governs.

⚠️ **A `../` ESCAPE IS NOT THE MECHANISM AND DOES NOT WORK.** Measured: `<Folder>/../Other/x.md`
returns *"The path does not point to a file or directory"*. Use a repo-root-relative path.

## After fetching
- **CHECK FOR AN INCIDENT BANNER FIRST:** if the memory doc opens with a "READ FIRST" / incident
  banner (placed after a data-integrity incident, e.g. a double import), it is confirmed ground
  truth - honor it over any conflicting entry in the doc, surface it at the VERY TOP of the
  dashboard, and never remove or soften it (only the user closes an incident).
- Read the `**Status:**` line, the "Last updated" line, and the newest Changelog entry (top of the
  Changelog - this system orders past-lists most-recent-on-top).
- **If Status is Archived:** tell the user this project is ARCHIVED (and surface any redirect the banner
  states). You can still read/answer, but flag that changes normally go elsewhere; confirm before writing.
  (`/project-unarchive` / pua brings it back to Active.)
- If the newest Changelog entry does not appear to cover the prior session, flag it once and suggest
  saying "upd" - then proceed. Do not block on it.
- The freshly fetched GitHub file is the ONLY source of truth. Any copy in Google Drive or Project
  Knowledge is a stale snapshot - ignore it.
- Memory docs are the ONLY cross-session channel: you cannot read other chats/sessions.

## HEALTH CHECK at open (new in v0.11.0) - one line, or silence
You already hold both files, so these cost nothing. Report each in ONE line above the dashboard, and
say nothing at all when everything is fine. **Every one of these is a CHECK, not a repair**
(`../../docs/CONVENTIONS.md` section 9): report it and let the user decide. Never silently fix, never
silently rebuild an index, and never delete a row because it looks orphaned.

1. **Size.** Name the byte count of any file at or past **40,000 B** (warn) and say what the split
   options are; at or past **60,000 B** (stop) say plainly that the next update should split the file
   first, and name which section is carrying the weight if you can see it. Opening is the right
   moment for this - the alternative is discovering it mid-write at the end of a long session.
2. **Standing Rules declarations resolve - ALL of them.** If the memory doc declares any Standing
   Rules paths, confirm EACH one exists and was fetched, and report the count fetched against the
   count declared. **A declared standard file that is missing is worse than a missing companion** -
   the companion was never guaranteed to load, this one was. **Checking only the first declared path
   is the defect this check exists to catch.**
3. **Companion pointers resolve.** For each row in the memory doc's **Companion Files** index,
   confirm a file actually exists at that path (one directory listing of `<Folder>` covers all of
   them). Report any row pointing at a file that is not there, and any markdown file in the folder
   that is not a STANDARD file (memory doc, notes file, or a declared Standing Rules file) and is
   not listed in the index. A companion outside
   every index is invisible to every future session, which is the same as lost.
4. **Reconciliation, wherever the doc HAS an index - not only where the layout is hub-and-spoke**
   (backlog #52). Counts, dates, kinds and destinations should agree between the index and what it
   points at. A mismatch is reportable and worth reporting. **Additionally, where an index row
   declares its companion holds a named source section** - as an HTML comment ending the description
   cell, `<!-- holds: memory doc section 3 -->` - **assert that the source section is a STUB, not
   content.** A section that has regrown a table while its index row still claims a companion holds
   it is a divergence that every existence check passes straight over, because both copies exist and
   both are well-formed. Rows without the marker get no assertion and no false finding. It does NOT
   follow that the index can be regenerated: the editorial labels in it exist nowhere else, there is
   no stable join key, and dates repeat. Report the discrepancy and stop there - never merge two
   copies, because the plugin cannot know which one is authoritative.
5. **Undated state claims.** Flag any row asserting a current size, blob SHA, version or count that
   carries no measurement date (`../../docs/CONVENTIONS.md` section 7). A dated snapshot ages honestly; an
   undated one rots while still sounding authoritative.

## LOAD REPORT at open (new in v0.23.0; refusals counted separately in v0.24.0) - ALWAYS printed, never silent
⚠️ **THIS IS A MEASUREMENT, NOT A CHECK, AND IT IS THE ONE THING ABOVE THAT IS NEVER SILENT.** The
health checks above say nothing when everything is fine. This one always reports, because there is no
"fine" to be quiet about — the whole point is that the number is visible on every open rather than
discovered when something breaks.

**Report, in one line above the dashboard: the BYTES of every file the mandatory first action
READ, and each file's SHARE of the total.** That is the memory doc, the notes file, and EVERY
declared Standing Rules file — the same set the mandatory fetch names, however many that is.

⚠️ **COUNT WHAT ARRIVED, NOT WHAT WAS ASKED FOR (backlog #185).** A fetch can come back REFUSED —
the read path declines an oversized result outright — and **a refused file costs ZERO context,
because not one character of it entered the session.** So the report is a THREE-STATE count, in the
same shape health check 2 already uses one screen above: **`N declared, M read, K refused`**. That
vocabulary exists a line away; do not invent a second one for the same idea.

    Loaded 3 of 3 declared files, 120000 B - Standing Rules 40000 B (33%), memory 50000 B (42%), notes 30000 B (25%)

    ⚠️ 1 of 3 declared files was REFUSED and is NOT in context.
    Loaded 2 of 3 declared files, 80000 B - memory 50000 B (63%), notes 30000 B (37%)
    REFUSED: Standing Rules - <path> (the read path declined it; its bytes are NOT in the total above)

**Both examples use ILLUSTRATIVE round numbers and are not a measurement of any real repo.**

⚠️ **THE BYTE TOTAL COUNTS ONLY FILES THAT WERE READ.** Folding a refused file's size into the total
does two wrong things at once: **it INFLATES the reported context cost, and it HIDES the absence
behind a bigger number.** The case this ships for: a run reported three files loaded and a total that
included one which had been refused — and that file was **39% of the total claiming it was present.**
**A number confidently right about a file nobody received is worse than no number**, because it turns
an invisible failure into a visible reassurance.

⚠️ **REPORT WHAT THE TOOL RETURNED. DO NOT INFER READABILITY.** A refusal is OBSERVABLE: the fetch
returns an error instead of content. **Read that, and report it.** Do NOT inspect the content and
judge whether it "looks complete", do not compare against an expected size, and do not treat a short
file as suspicious. **A heuristic that is sometimes right is worse than a plain report of what
happened** — it produces confident findings on files that are fine and silent passes on files that
are not, and neither is checkable by the person reading the line.

⚠️ **A SPILLED RESULT IS A READ, NOT A REFUSAL — AND THE SAVED PATH DECIDES, NOT THE WORDING.** A
large fetch often does not come back inline: the client writes the result to a file and returns a
message naming that SAVED PATH. **That fetch SUCCEEDED — only the inlining did not.** ⚠️ **IF A
SAVED PATH IS NAMED, GO AND READ IT, then count the file as READ and put its bytes in the total.**
Do not classify on the message's vocabulary: a size-shaped or token-shaped message can arrive
WITH a saved path, so the wording is not the test and the path is. **Only a result that names NO
saved path and carries no content is a file this report may count as not read.** ⚠️ **AND FAILING
TO RECOVER A NAMED PATH IS A SESSION ERROR, NOT A LOAD FAILURE** — say which one happened, and
never report a file as absent when it is sitting on disk. **A false refusal discredits this report
in the direction that looks responsible**, which is the harder one to notice. Fuller statement -
including the reader who cannot reach a saved path, and why no byte number for the cut is written
anywhere - `../../docs/CONVENTIONS.md` section 3.

⚠️ **LOUD WHEN THEY DIFFER, AND NEVER FOLDED INTO PROSE.** If `M` is less than `N`, the refusal is on
its own line, names EVERY refused path, and says plainly that those files are not in context.
**Silence about a refused STANDARD file is the exact defect this exists to fix** — it is the one
class of file the session was promised would always load.

⚠️ **BOTH HALVES ARE REQUIRED AND NEITHER SUBSTITUTES FOR THE OTHER.** A total alone cannot tell
anyone WHICH file grew. A per-file byte count without the share cannot tell anyone whether it
MATTERS. The share is what turns a number into a judgement a person can act on.

⚠️ **REPORT, NEVER REPAIR — `project-open` MUST NOT DECLINE TO LOAD ON SIZE.** Do not truncate, do
not summarise, do not skip a declared standard file because it is large, and do not offer to. The
session is not usable without these files; that is why they are standard. **Reporting the cost and
refusing to pay it are different acts, and only the first is authorised here** (`../../docs/CONVENTIONS.md`
section 9). ⚠️ **A REFUSAL BY THE READ PATH IS NOT `po` DECLINING** — it is the harness refusing, and
the only correct response is to REPORT it, never to retry with less or to summarise what did arrive.

⚠️ **WHY THIS EXISTS, so it is not later "simplified" away as chatter.** `CONVENTIONS.md` section 3
scopes the 60,000 B stop so it does not reach a GENERATED always-loaded file — the emission ceiling
cannot reach one. **The hazard that remains is CONTEXT COST, and until this report existed nothing
measured it.** The gate and its replacement shipped in the same release on purpose: removing the
first without the second would leave neither. **v0.24.0 is the correction to the replacement: the
measurement existed, and was wrong in the one case that mattered.**

## Read the `**Type:**` line - it selects the dashboard
The memory doc carries a machine-readable `**Type:**` line: `Project` or `Learning Notebook`
(missing = `Project`, so every pre-existing doc keeps working untouched). The type does NOT change
what you fetch or how the memory doc is read - only how the notes file is shaped and therefore which
dashboard makes sense. Never infer the type from the name or from the notes file's headings; read the
line. If the line says `Learning Notebook` but the notes file is clearly on the project template (or
vice versa), say so once and ask - do not silently reshape either file.

⚠️ **MATCH THE VALUE CASE-INSENSITIVELY, AND TREAT `Reference Notebook` AND `Learning Notebook` AS
THE SAME VALUE.** `Reference Notebook` is the current spelling; `Learning Notebook` is the legacy
one, kept as a PERMANENT synonym with NO END DATE. `reference notebook`, `Reference Notebook` and
`LEARNING NOTEBOOK` are the same answer, and so are `project` and `Project`. **Wherever this file
writes `Learning Notebook` as a Type VALUE, either spelling satisfies it.** The changeover is
deliberately staged - detection first - so one repo may hold both spellings at once and
indefinitely, and **a plugin that only works when the user's capitalisation matches the template is
a trap for every repo that is not the one it was built on.**

## Read the `**Layout:**` line - it selects WHERE the questions are (new in v0.14.0, backlog #20a)
**READ-ONLY. This changes what `po` READS, never what it writes.** Routing, index reconciliation and
repairing a stub are `upd`'s job and are NOT in this command (backlog #20b).

Some memory docs are a **hub**: navigation only, with decisions and open questions in indexed
companion "spoke" files. `**Layout:** Hub-and-Spoke` says so; **missing means the classic
single-file doc**, so every doc written before this line existed keeps behaving exactly as it always
has.

⚠️ **MATCH THE VALUE CASE-INSENSITIVELY, ignoring hyphens and spacing.** `hub-and-spoke`,
`Hub-and-Spoke` and `HUB AND SPOKE` are the same answer. The reference repo's own pilot project
wrote it lowercase against a template that writes it capitalised, and **a plugin that only works
when the user's capitalisation matches the template is a trap for every repo that is not the one it
was built on.**

**Detect it even when the line is absent**, because most docs will not carry it. ⚠️ **THE TEST IS
DEFINED IN `../../docs/CONVENTIONS.md` SECTION 8, UNDER *The hub layout test and the hub
question-count predicate* - READ IT THERE AND APPLY IT. IT IS DELIBERATELY NOT RESTATED HERE**
(backlog #229).

⚠️ **`project-list-update` USED TO SHIP A NEAR-TWIN OF THIS RULE AND THE TWO HAD DRIFTED.** This
file once claimed the rule was "stated identically on purpose, so the two commands cannot drift into
disagreeing about the same file." **That assurance was false in shipped text, and an assurance
sitting beside a rule that HAS drifted is worse than silence, because it actively discourages the
check.** Measured 2026-08-14 against v0.17.0, the two texts differed on three axes, each able to
change the answer. **All three are now closed, and closed the same way: by DELETING both copies and
pointing at one definition, never by copying one skill's wording into the other.**
- **The negative test** - `plu` bound it to a literal `## Open Questions` heading while real docs
  NUMBER their headings, so the literal form could miss. **Closed in v0.30.0 (`#229` axis 1);
  the definition now lives in section 8.**
- **Case folding of the `**Layout:**` value** - stated here, absent there, so `hub-and-spoke`
  lowercase read as a hub here and as single-file there. **Closed in v0.28.0 (`#227` axis 2).**
- **The hub question-count predicate** - `plu` carried an extra *not marked resolved* clause, so
  **the two commands could return different counts for the same doc.** **Closed in v0.30.0 (`#229`
  axis 3); the definition now lives in section 8.**

⚠️ **THE STANDING RULE IF THEY EVER DISAGREE AGAIN IS UNCHANGED, AND IT IS WHY THE REMEDY WAS A
SHARED DEFINITION RATHER THAN A PATCH: neither twin is automatically right, so report the
disagreement rather than reconciling it silently, and do not edit the other skill from this one.**

**Two traps, both met live on the reference repo. The FIRST is now handled inside the shared
definition and is named here only so you recognise it; the SECOND is broader than that definition
and governs every section this command reads:**
- ⚠️ **TEST FOR THE TABLE, NOT THE HEADING** - carried by the section 8 definition above, which is
  where it is specified. A hub keeps its `## N. Open Questions` heading as a POINTER STUB with no
  table under it, and a heading match reports "found it" and then counts nothing.
- ⚠️ **FIND SECTIONS BY HEADING TEXT, NEVER BY NUMBER.** Section numbering is not stable across
  projects - one project keeps Open Questions at 6, another at 4 and uses 6 for something else
  entirely. A number-based read silently reads the wrong section and reports its contents.
  **This applies to EVERY section this command reads, not only Open Questions.**

**Counting open questions on a hub: the predicate is defined in `../../docs/CONVENTIONS.md`
section 8, under *The hub layout test and the hub question-count predicate*, and is NOT restated
here** (backlog #229).

⚠️ **IF THE DOC DECLARES HUB-AND-SPOKE AND NO INDEX IS FOUND, SAY SO AND RENDER NO NUMBER. NEVER
FALL BACK TO THE OPEN QUESTIONS HEADING.** That fallback returns 0, and **a fallback zero is
byte-identical to the defect this exists to fix** - nothing looks wrong, which is what made the
original bug survive. Report: *"declares hub-and-spoke, no `Kind` index found - questions not
counted."* **Never report a zero you have not earned.**

## THE DASHBOARD - every open (and every refresh) ENDS with this
Present a compact overview built from the two fetched files, AS TABLES, in this order. Keep it tight;
it must fit a phone screen without heavy scrolling.

**For a Project (`Type` = Project, or absent):**
0. **⚠️ Incident banner** - ONLY if the memory doc carries a READ FIRST banner: summarize it first,
   before anything else.
1. **Header line** - project name, Status (Active/Archived), memory doc's "Last updated".
2. **Current Status** - the notes file's Current Status section (This week / Coming up / Ongoing),
   as a table if it has structure.
3. **Open Tasks / Next Steps** - from the notes file, first-to-do on top.
4. **Open Questions** - from the memory doc, OPEN items only, as a table.
   ⚠️ **WHAT COUNTS AS OPEN IS DEFINED HERE, CLOSED-WORLD, BECAUSE THIS IS THE STEP THAT BUILDS THE
   PANEL.** Until v0.27.0 this line said "OPEN items only" and defined nothing, so the test was
   whatever the running session assumed - which is not a test. Normalise the Status cell first
   (strip `*` and `_` from both ends, trim whitespace; strip nothing from the middle), then the row
   is CLOSED **if and only if** the cleaned value BEGINS with one of `Resolved`, `Closed`,
   `RESOLVED`, `Completed`, `Dropped` - compared case-insensitively - followed by end-of-cell or
   a non-letter - **or** begins `Moved to` AND ends
   `(verified <date>)`. **Everything else is OPEN**: `Open`, `Partially resolved`,
   `Moved to <project> (open)`, an empty cell, and any token not on that list.
   ⚠️ **A ROW WHOSE STATUS NOBODY ANTICIPATED IS SHOWN, NEVER HIDDEN** - the panel errs toward
   visible, because a question that vanishes from the dashboard is a question nobody re-reads.
   The closed set is defined once in `../../docs/CONVENTIONS.md` section 8, and section 8 governs if
   the two ever disagree. **Render at most the top 10;
   if more are open, add a footer line: "+N more - say 'show all questions'".** (Render truncation
   only - never drop items from the file.) ⚠️ **ON A HUB, READ THE INDEX, NOT THE STUB** - see the
   `**Layout:**` section above. **NAME THE SOURCE in the panel header, always, on both layouts:**
   `Open Questions (26, from the hub index)` or `Open Questions (16, from the memory doc)`. A
   hub-counted number sitting in a panel that historically read the Open Questions section will
   otherwise be quoted later with no way to tell which channel produced it - and the two channels
   disagreed by 26 on the reference repo, which is the entire reason this line exists.
5. **Latest changelog** - the newest Changelog entry (date + summary).
6. **Companion files pick list** - see below (only if the index has entries).
End with: "What would you like to work on?" - or, if the user already stated their task, skip the
question and get to work.

**For a Learning Notebook (`Type` = Learning Notebook):** a notebook has no "this week" and no
finish line, so a status snapshot would be empty theatre. Show what's IN it and what's still open:
0. **⚠️ Incident banner** - same rule, first, if present.
1. **Header line** - notebook name, Status, memory doc's "Last updated", and the notebook's subject
   (the One Paragraph's first clause).
2. **Contents** - the theme headings with a count of topics under each, and (if it fits) the topic
   names under the theme most recently touched. This is the map; it is the most useful thing on a
   notebook's dashboard and it replaces Current Status.
3. **To Learn / Open Threads** - from the notes file, first-to-learn on top, same top-10-plus-footer
   truncation rule as Open Questions.
4. **Open Questions** - from the memory doc, OPEN items only, same truncation rule.
   ⚠️ **THE OPEN TEST IS NAMED HERE RATHER THAN INHERITED, AND THE REASON IS THAT "same rule" USED
   TO MEAN THE TRUNCATION RULE ALONE.** A predicate carried by an ambiguous pointer is a predicate
   that quietly does not run on this path, and the notebook dashboard is a path nobody watches.
   So: normalise the Status cell, then CLOSED **if and only if** it BEGINS with `Resolved`,
   `Closed`, `RESOLVED`, `Completed`, `Dropped` - case-insensitively - followed by end-of-cell or
   a non-letter - **or** begins `Moved to` AND ends
   `(verified <date>)`. **Everything else is OPEN**, including any token not on that list. Identical
   to the project dashboard above and to `../../docs/CONVENTIONS.md` section 8, which governs.
   (Notebooks keep these too; they are questions about the notebook, not about the subject.)
5. **Recent activity** - the newest 2-3 Learning Log lines (they are one line each), then the newest
   Changelog entry.
6. **Companion files pick list** - same rule.
End with: "What would you like to add or look up?" - or get straight to work if the user already said.
- **Never dump topic bodies into the dashboard.** A notebook's topics are the payload and can be
  long; the dashboard lists them and the user asks for the one they want.

## Standard files vs companions - READ THIS BEFORE CHANGING EITHER RULE

Two categories, two opposite loading rules, and collapsing them would break one of them:

| | STANDARD files | COMPANIONS |
|---|---|---|
| Which | memory doc, notes file, and EVERY declared `**Standing Rules:**` file | everything in the Companion Files index |
| Loaded | **ALWAYS, by the mandatory first action. Never offered, never asked.** | **ONLY when the user picks them.** Never unprompted |
| Why | the session is not usable without them | the user is the dispatcher and knows the topic; guessing wastes context |

⚠️ **A STANDING RULES FILE IS PROMOTED TO STANDARD, NOT SMUGGLED INTO THE PICK LIST.** It is fetched
unconditionally *because it was declared a standard file in the memory doc*, not because something
decided a companion looked important. **The rule below - "NEVER load companions unprompted" - is
untouched by that promotion and governs every actual companion exactly as before.** If a future
change ever makes a companion load automatically, that is a DIFFERENT decision and it contradicts
the rule below; do not treat this section as precedent for it.

**The test, if you are unsure which a file is:** does the memory doc *declare* it in a header line
(standard), or *index* it in Companion Files (companion)? A file cannot be both. If one appears in
both, report it - that is a defect, not a preference.

## ⚠️ A SECTION-8 POINTER IS FOLLOWED, NOT SKIPPED (new in v0.25.0, backlog #182)

**If the memory doc's Companion Files section has NO ROWS but NAMES a companion file, FETCH that
file and build the pick list from it.** A section that says *"the index now lives in `X.md`"* is a
POINTER, and a pointer is an instruction to go there - not an empty index.

**How to tell:** the section has zero table rows AND its prose contains a backticked `.md`
filename. That is the whole test. **Take the FIRST such filename**; if there are several, name them
all in one line and use the first, rather than guessing.

⚠️ **WHY THIS EXISTS, AND IT IS A SILENT FAILURE, WHICH IS THE WORST KIND.** Before this release the
pick list, the pointer-resolution health check and `project-update`'s companion reconciliation all
built from that section, and where it was a stub they **skipped without saying anything**. On the
reference repo that hid an 89-row index from every mechanism designed to consume it: the index was
canonical, complete, and unreachable. **Nothing reported a problem, because nothing looked.**

⚠️ **IF THE NAMED FILE DOES NOT FETCH, SAY SO ON ITS OWN LINE AND CARRY ON.** A pointer naming a
file that is not there is a FINDING - report it above the dashboard exactly as a missing declared
standard file is reported. **Never fall back to "the index is empty": that is the defect.**

⚠️ **THE POINTED-AT FILE IS A COMPANION, NOT A STANDARD FILE.** It is fetched to BUILD the pick
list, and its own contents are still offered rather than loaded. This does not promote anything to
the mandatory fetch.

## Companion files - ASK-FIRST pick list (never guess what to load)
If the memory doc has a **Companion Files** section (the canonical index: file -> what it is -> when to
load), offer the companions as part of the dashboard:
- Present them as a **phone-friendly multiple-choice PICK LIST** (AskUserQuestion, multi-select): the
  FIRST option is always "None - just the memory doc"; list each companion as its own option; mark as
  "(Recommended)" any whose "when to load" condition plausibly matches what the user has said so far.
- Load ONLY what the user selects. Default (no selection / dismissed) = load nothing extra.
- If the user selects "None" TOGETHER WITH one or more files (the multi-select allows it), the files
  win: load the selected files and ignore "None" - do not ask again.
- NEVER load companions unprompted, and never guess the session topic. The user is the dispatcher.
- If the doc has no Companion Files section, skip this step silently.

## The standard files (structure you are reading/writing)
Each project folder has **two standard files, plus every file the memory doc declares on its
`**Standing Rules:**` line - zero, one or many** (and optional companion files beyond that):
1. **Memory doc** - `Project Memory Data - <Project Name>.md`. Starts with a `<!-- FILE PURPOSE -->` HTML
   comment, a title, a `*Last updated*` line, and a `**Status:**` line, followed by sections.
   **SINGLE SOURCE OF TRUTH FOR STRUCTURE:** the section set is defined by the live template
   `Project Setup/Project Memory Data - Template Project Name.md` and by whatever sections the fetched doc
   already has. Do NOT assume a fixed list. Follow the fetched doc's own headings.
2. **Notes file** - `Project Notes - <Project Name>.md`. Its shape depends on the memory doc's
   `**Type:**` line:
   - **Project** (notes template v2): (1) The Project in One Paragraph, (2) Current Status (a snapshot,
     updated in place), (3) Open Tasks / Next Steps (first-to-do on top), (4) Quick Reference (pinned key
     facts), (5) Phases at a Glance (placeholder until warranted), (6) Gaps & Issues (placeholder until
     warranted), (7) Session Notes - the dated newest-on-top log at the BOTTOM.
   - **Learning Notebook** (notebook template): (1) The Notebook in One Paragraph, (2) Quick Reference,
     (3) To Learn / Open Threads (first-to-learn on top), (4) theme headings (`# Theme`) each holding
     topic sections (`## Topic`) that are updated IN PLACE and stamped Added/Updated, (5) Learning Log -
     one line per session, newest on top, at the BOTTOM. The theme set is editorial and differs per
     notebook; there is no required list.
   Older files may predate some sections; follow the file's own headings.
3. **Companion files** (opt-in, only when a project accrues real reference material - comparison tables,
   runbooks, specs). One canonical home per piece of content; the memory doc's Companion Files index is
   canonical; the notes header list is a human-friendly link mirror, not a second source of truth.

## Dated-list ordering rule (the house style - apply everywhere)
- **PAST lists** (Changelog entries, Session Notes sections, Decisions Log rows, revision history) sort
  **MOST RECENT ON TOP** - so you never scroll past old material to find the latest.
- **FUTURE lists** (Open Tasks / Next Steps, checklists, setup steps) sort **FIRST-TO-DO ON TOP**
  (runbook order).
- **Open Questions**: open items on top; resolved items retained below (never deleted), and when a
  resolved item carries a resolved date, order the resolved ones most-recently-resolved first.
- Decisions Log: newest on top; mark superseded rows "Superseded" rather than deleting.

## Trigger - "upd"  (aka "update", "update memory", "update the notes", "log this", "save this")
Run the `/project-update` (upd) procedure: ONE pass that always updates the memory doc (dated Changelog
entry on TOP; Decisions/Data Sources/Open Questions/Companion index/Deliverables as relevant; preserve
the `**Status:**` line) and updates the notes file where the session warrants it (Current Status in
place; Open Tasks; Quick Reference; a dated Session Notes section on TOP of the dated area; ALWAYS check
whether Phases/Gaps placeholders are ready to populate and offer - but a **pointer stub naming a companion
in backticks is NOT a placeholder** and is never offered for rebuild, per backlog #51). Never delete or silently rewrite
prior entries. Commit message: `upd: <short summary>`.
(The old um / un commands are retired; treat them as requests for this combined update.)

**Once per sitting:** if a working session is clearly wrapping up and no `upd` has been committed this
sitting, remind the user ONCE - sessions compact, and these files are the only persistence layer.

## Storage split (GitHub vs Google Drive)
- **GitHub is ALWAYS the source of truth.** The memory doc, notes file, and companion files (markdown)
  live ONLY in GitHub. Never store memory or notes on Google Drive.
- **Google Drive holds rich deliverables only** (Google Docs/Sheets, PDFs, etc.) in the project's
  matching Drive folder (under `<DriveFolder>`; use the config `drive_folder_id` when present). Track
  them in the memory doc's **Deliverables & Links** section, identified by a `<Timezone>` timestamp;
  when a deliverable is replaced, only one current version should remain.
- **How that replacement happens depends on `<DriveAccess>`:** in `connector` mode the connector
  cannot rename or delete, so the new file is uploaded (if small enough to inline) and the user is
  TOLD which older file to delete by hand, identified by its timestamp; anything too large to inline
  is reported as pending. In `rclone` mode the upload runs at full fidelity at any size and the
  superseded copy is deleted for real - but only AFTER the new file verifies (size or checksum).
  Either way, report what actually happened; never fake a Drive step. Credentials are never read or
  committed - the config holds the remote's NAME only.
- **RCLONE PREFLIGHT - where the command actually runs.** `drive_access: rclone` describes a MACHINE,
  not an account: rclone reads a config file on the user's own computer, so it works only from a
  session that can run a shell command THERE (Claude Code on that machine, or Cowork with the user's
  desktop app connected). A Cowork session's own shell is a cloud container with no rclone and no
  route to the user's config - phone sessions included. Before the first rclone-mode Drive step of a
  session, confirm such a shell is reachable and that `rclone listremotes` lists the config's
  `rclone_remote`; if not, say so in one line and use `connector` behavior for the rest of this
  session. Never claim an rclone step ran when it did not, and never edit the config to "fix" it.

## Archiving
To archive a project, use `/project-archive` (`pa`); to bring one back, `/project-unarchive` (`pua`).
Both end with a full master-list rebuild. Never archive-rename the `Project Setup` folder (the plugin
depends on its paths).

## WRITE SAFETY - every commit is SHA-locked (changed in v0.11.0)
- **Write with `create_or_update_file`, locked to the blob SHA you read. Never `push_files` for a
  file that already exists** (`../../docs/CONVENTIONS.md` section 2). `push_files` takes no blob SHA and no
  base commit, so a concurrent write is silently last-writer-wins with a success response;
  `create_or_update_file` rejects a stale write loudly. A rejection is the lock working: re-fetch,
  re-apply, write again - never retry with the stale SHA.
- **Re-fetch immediately before writing.** Base the edit on those exact bytes, not on the
  session-start copy, and confirm the returned blob SHA afterwards.
- **One command is not one commit.** Several SHA-locked commits under one typed command is the
  intended shape. Notes first, memory last.
- **Append/insert, never overwrite.** Add rows/entries; preserve all existing text verbatim.
- **Lint tables before committing** (`../../docs/CONVENTIONS.md` section 6) and keep exactly one trailing
  newline - it is not added for you.
- **Verify inventory against the repo, not memory.** List the repo before claiming what exists.
- **Distrust cross-session inferences.** Verify surprising claims against the file they describe.

## GitHub mechanics (Cowork specifics)
- Commit to `<Branch>` unless told otherwise. Connectors are account-level; no per-project linking.
- Deletes require an in-app approval prompt. There is NO atomic folder rename - recreate files at the
  new path, then delete the old ones.
- Code search is unreliable on private repos - it can return zero hits for a file that plainly
  exists. Never treat an empty search result as proof of absence; list the directory and read.

## Standing rules
- The fetched memory doc is the source of truth - check it before re-deriving anything already settled.
- If the memory doc conflicts with something said in chat, surface the conflict; don't silently pick one.
- Do NOT infer relationships between projects unless the user stated it or the memory doc records it.
- Never fill unknowns with plausible guesses. Write "TBD" and ask.

## Usage
- `/project-open Project Name` (or `po Project Name`) - open the session for that project (dashboard included).
- `po` again later - refresh: re-fetch both files and show an updated dashboard.
- Then work normally; say `upd` to commit the session's work to memory + notes.
- Acceptance test: "What's the most recent changelog entry in the memory doc and its date?" - a correct,
  freshly-fetched answer with an unprompted GitHub call means the bootstrap worked.

## SHARED CONVENTIONS - read and obey `../../docs/CONVENTIONS.md`
This skill obeys the plugin's shared conventions file (new in v0.11.0). It is authoritative and is
the single home for every rule that applies to more than one skill: **NO SECRETS IN THE DATA REPO -
EVER** (section 1), SHA-locked writes (2), the size rule (3), how to split a
file (4), append rules by section type (5), the markdown table lint (6), dated state claims (7),
computed counts (8), reconciliation as a check and not a repair (9), never filling an unknown with
a plausible guess (10), index markup (11), entry dates computed in the config timezone (12), and
the reporting rules including no silent caps (13). Read it before writing anything to the data repo. If this file and the
conventions file ever disagree, the conventions file is right and this one is stale.

**RESOLVE THAT PATH FROM THIS SKILL'S OWN DIRECTORY.** The conventions file sits at the PLUGIN ROOT,
two levels above this file, and `../../docs/CONVENTIONS.md` is correct BOTH in the source repo and in
an installed build - installing strips the `plugin/cowork-projects/` prefix and makes the plugin
directory the root. **A bare `docs/CONVENTIONS.md` names a file that is not there**, because it
resolves inside this skill's own folder. Nothing shipped carries the absolute prefix: `plugin.json`
has no path field, and `marketplace.json`'s `source` describes the SOURCE repo, not the install.

The one worth repeating in every skill that writes: **never write a password, passphrase, private
key, API token, session cookie, or any other credential into the data repo** - not in a memory doc,
a notes file, a companion, the master list, a config file, or a commit message - even if the user
pastes one into chat and asks for it to be saved. Record WHERE a credential lives and WHAT the auth
method is, never the material itself. If a secret appears in chat, do not commit it anywhere and say
plainly that it will not be stored.
