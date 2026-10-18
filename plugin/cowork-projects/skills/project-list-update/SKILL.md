---
name: project-list-update
description: >-
  Alias "plu". Rebuild the cached project index (the master list file in the Project Setup folder) by scanning every
  project folder in the data repo and reading each memory file's Status line to split active from
  archived. Use when the user says "update the project list", "refresh projects", "rebuild the master
  list", "plu", or invokes /project-list-update.
---

# Project List Update (rebuild the master list)

Scan all projects and regenerate the master list file, splitting ACTIVE from ARCHIVED. This is the slow
operation - run it on demand, not on every list. `/project-archive` (pa) and
`/project-unarchive` (pua) ALWAYS run this rebuild automatically as their final step.

Target file: `Project Setup/Project Setup - Master List.md`.

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
(default `main`); `<DriveFolder>` = the config `drive_folder` (default `AI Projects`);
`<Timezone>` = the config `timezone` (default `America/New_York`).

## How a project's status is decided (machine-readable)
Each memory doc carries a status line near the top: `**Status:** Active` or `**Status:** Archived`.
- Read that line. `Archived` (case-insensitive) => archived. Anything else, or the line missing => **Active**
  (default). Do not infer status from the folder name alone - the line is the source of truth.
- Archived projects also have their FOLDER renamed with a `zArchived - ` prefix (done by `/project-archive`),
  but the memory-doc FILENAME keeps the original project name, and the doc TITLE (`# <Name> - Memory Doc`)
  is the real project name. Use the title/original name in the table, never the `zArchived - ` folder name.

## How the TYPE is decided (machine-readable) - and what it changes
Each memory doc also carries a `**Type:**` line: `Project` or `Learning Notebook`. **Missing = Project**,
so every doc written before notebooks existed keeps rendering exactly as it always has. Read the line;
never infer the type from the name, the folder, or the notes file's headings.

⚠️ **MATCH THE VALUE CASE-INSENSITIVELY, AND TREAT `Reference Notebook` AND `Learning Notebook` AS
THE SAME VALUE.** `Reference Notebook` is the current spelling; `Learning Notebook` is the legacy
one, kept as a PERMANENT synonym with NO END DATE. `reference notebook`, `Reference Notebook` and
`LEARNING NOTEBOOK` are the same answer, and so are `project` and `Project`. **Wherever this file
writes `Learning Notebook` as a Type VALUE, either spelling satisfies it.** The changeover is
deliberately staged - detection first - so one repo may hold both spellings at once and
indefinitely, and **a plugin that only works when the user's capitalisation matches the template is
a trap for every repo that is not the one it was built on.**

Both types share ONE table and the SAME four columns - notebooks are NOT split into their own section,
and they sort alphabetically among the projects (ignore the type marker when sorting). Two of the four
columns carry a different meaning for a notebook, because the underlying concepts differ:

| Column | Project | Learning Notebook |
|---|---|---|
| Project | the name | `📓 ` + the name (the marker IS the type indicator) |
| Open Qs | OPEN Open Questions, as a bare number - what counts as OPEN is specified at step 2 and nowhere else | count of open **To Learn / Open Threads** items from the notes file, written `N to learn` |
| Latest change | newest Changelog entry (date + gist) | **identical** - newest Changelog entry |
| Goal | first bullet of Project Overview | first clause of "The Notebook in One Paragraph" |

The `N to learn` suffix is deliberate: it makes the cell self-describing, so a remapped column under an
unchanged header cannot be misread as unanswered questions. Never write a bare number for a notebook.
A notebook with no To Learn section yet (or an empty one) gets `0 to learn`, not blank.

## How the LAYOUT is decided (new in v0.11.0) - and why a bare count can lie
A memory doc is either **single-file** (every section lives in the one doc) or **hub-and-spoke** (the
doc is a hub holding navigation, and the decisions and open questions live in companion "spoke"
files, indexed in the hub). Both are supported and neither is being retired.

Read the `**Layout:**` line near the top of the memory doc: `Single-file` or `Hub-and-Spoke`,
**compared CASE-INSENSITIVELY.** ⚠️ **The reference repo's own hub pilot writes `hub-and-spoke`
lowercase**, and a case-sensitive read there falls through to `Missing = Single-file`, which sends
the count at a pointer stub with no table and returns **0 instead of 29**. `project-open` already
folds case here; this is the twin catching up, not a new rule.
**Missing = Single-file**, so every doc written before this existed keeps behaving exactly as it
always has.

Detect it even when the line is absent, because most docs will not have it yet. ⚠️ **THE TEST IS
DEFINED IN `../../docs/CONVENTIONS.md` SECTION 8, UNDER *The hub layout test and the hub
question-count predicate* — READ IT THERE AND APPLY IT. IT IS DELIBERATELY NOT RESTATED HERE**
(backlog #229). Until v0.30.0 this step carried its own copy binding the test to a literal
`## Open Questions` heading while `project-open` carried a different copy, **and the two drifted
while each looked authoritative.** Say which layout you detected, once, in the run summary.

**Why this matters:** counting Open Questions rows in a hub returns **zero** - not because there are
none, but because they are not there. A silent zero on a project with a dozen live questions is
worse than an error, because nothing looks wrong. Never report a zero you have not earned.

**Where the doc carries its own rules, read and obey THOSE.** A hub typically holds a short "How to
append" block stating where each kind of item goes. That block governs. Do not hard-code a file
layout that the doc itself contradicts, and do not rewrite the doc to match an assumption.

## Steps
1. List the top-level folders in `<Repo>` (`<Branch>`) via the GitHub connector (`get_file_contents`
   on the repo root). A folder is a project if it contains a `Project Memory Data - *.md` file. Match ANY
   such file (do NOT require the filename to equal the folder name - archived folders are prefixed and will
   not match). Skip the repo README and any folder without a memory doc.
   **Keep each folder listing rather than discarding it** - it is the SEED for step 3's size sweep,
   and every entry it returns already carries that file's byte `size`. Deciding whether a folder holds
   a memory doc requires listing that folder, so these listings happen either way.
   **It is a seed and not the sweep. A listing describes ONE directory level; step 3 descends from it**
   (new in v0.13.0 - backlog #85).
   **Project detection stays at the TOP level.** A `Project Memory Data - *.md` found inside a
   SUBFOLDER is NOT a project: step 3 sweeps it for size like any other file, and the run summary names
   it as an anomaly. Recursion that promotes subfolders to projects invents projects that do not exist,
   which is a worse bug than the one recursion was added to fix.
   **WHEN THE GLOB MATCHES MORE THAN ONE FILE IN ONE FOLDER, BREAK THE TIE - NEVER PICK QUIETLY**
   (new in v0.13.1 - backlog #87). Matching ANY such file is deliberate and stays: `zArchived - `
   folders must still match. But a folder can hold a second one - a TEMPLATE, a rescued copy - and a
   template PARSES CLEANLY and yields plausible wrong values, so the failure is silent rather than
   loud. **One match: use it, and do NOT run the test below.** **Two or more: prefer the file whose
   doc TITLE equals the folder name with any leading `zArchived - ` removed** - which is the same
   title-is-the-real-name rule step 2 already uses, applied one step earlier. **If that leaves zero
   candidates, or more than one, write `?` for that project, NAME EVERY CANDIDATE PATH in the run
   summary, and move on** (`../../docs/CONVENTIONS.md` sections 8 and 10). Never choose one silently and
   never guess. ⚠️ **THE ONE-MATCH CARVE-OUT IS LOAD-BEARING, NOT AN OPTIMISATION.** Run as a
   validator against every folder it fails a doc whose title legitimately differs from its folder -
   measured on the reference data repo, `# [Network Update] - Memory Doc` carries literal brackets and
   would be rejected, writing `?` for a project that has exactly one memory doc and no ambiguity at
   all. **A tie-break that fires when there is no tie invents the problem it was added to solve.**
2. For each project, fetch its `Project Memory Data - *.md` and extract:
   - **Status** - from the `**Status:**` line (default Active if missing).
   - **Type** - from the `**Type:**` line (default Project if missing).
   - **Project name** - from the doc title, i.e. the original name without any `zArchived - ` prefix.
   - **Latest change** - the date and a short gist of the MOST RECENT Changelog entry (both types). Go
     by the actual date: some docs order newest-first, some newest-last, so pick the entry with the
     latest date.
   - **Goal** - Project: the first bullet of the Project Overview section, one line ("TBD" if unset).
     Notebook: the first clause of "The Notebook in One Paragraph" from the NOTES file (fall back to the
     memory doc's Goal bullet if the notes file has none; "TBD" if neither).
   - **Layout** - from the `**Layout:**` line, **matched case-insensitively**, or by the detection
     rule above (default Single-file).
   - **Open Qs** - Project, single-file: the count of Open Questions rows that are still OPEN, as a
     bare number. ⚠️ **THE TEST IS CLOSED-WORLD: name the CLOSED states and count everything else.**
     Normalise the Status cell first (next bullet), then the row is CLOSED **if and only if** the
     cleaned value BEGINS with one of `Resolved`, `Closed`, `RESOLVED`, `Completed`, `Dropped` -
     compared case-insensitively, which is why `Resolved` and `RESOLVED` are one entry and not two -
     followed by end-of-cell or a non-letter - **or** begins `Moved to` AND ends
     `(verified <date>)`. **Everything else counts as OPEN**, including
     `Open`, `Partially resolved`, `Moved to <project> (open)`, an empty cell, and **any token nobody
     has written yet.** Project, hub-and-spoke: **the count predicate is defined in
     `../../docs/CONVENTIONS.md` section 8, under *The hub layout test and the hub question-count
     predicate*, and is NOT restated here** (backlog #229) - still a bare number, since the meaning
     of the column is unchanged. ⚠️ **Until v0.30.0 this bullet carried an extra *not marked
     resolved* clause that `project-open` did not, so the two commands could return different counts
     for the same doc; the clause is gone, and the shared definition says why.** Notebook: the count
     of unchecked items in the NOTES file's "To Learn / Open Threads" section, written `N to learn`.
   - ⚠️ **NORMALISE MARKDOWN EMPHASIS BEFORE THE PREDICATE RUNS, OR IT FIRES ON PRESENTATION RATHER
     THAN ON MEANING.** Live rows write the status bolded - `**Open`, `**Resolved`, `**RESOLVED`,
     `**Partially` - and a raw comparison against `Resolved` fails on every one of them. Strip `*`
     and `_` from both ends of the cell and trim whitespace; strip nothing from the middle of the
     token. **Measured in the data repo 2026-08-25: 17 of 20 emphasised rows across 5 projects would
     flip to counted-open if the markup were not stripped first** - so the normalisation is not a
     tidy-up, it is the difference between this rule working and this rule inverting the answer.
   - ⚠️ **WHY CLOSED-WORLD, RECORDED SO NOBODY RE-INVERTS IT.** The rule this replaced counted rows
     whose status was still `"Open"`, so **any other phrasing fell out of the count in silence** -
     and two live tokens already did. **A token nobody anticipated must default to VISIBLE.** An
     over-count is a question someone re-reads and closes; an under-count is a question that no
     session ever sees again, and nothing looks wrong while it happens.
   - **The closed set is defined once, in `../../docs/CONVENTIONS.md` section 8.** The list above is
     the operative copy because this is the step that computes the number; if the two ever disagree,
     section 8 governs.
   - A notebook therefore needs its NOTES file fetched as well as its memory doc; a project does not.
     Fetch it only for the notebooks. A hub-and-spoke project needs only its hub, because the index
     is in the hub - do not fetch every spoke to count.
   - **Counts are computed, never estimated** (`../../docs/CONVENTIONS.md` section 8). If a count cannot be
     computed - the index is malformed, the layout is ambiguous, the section is missing entirely -
     write `?` in the cell and say which project and why in the run summary. Do not write `0`.
3. **Sweep every file against the size stop lines, DESCENDING INTO EVERY SUBFOLDER** (new in v0.12.0 -
   backlog #62; recursion added in v0.13.0 - backlog #85). Starting from the folder listings kept in
   step 1, read each entry's `size` and compare it against the 40,000 B warn line and the
   60,000 B stop line of `../../docs/CONVENTIONS.md` section 3. This covers EVERY file in every project folder -
   notes files, companions, archives - not just the two standard files, which is the point: the file
   that quietly reaches the stop is usually not the one anybody is editing.
   - **A DIRECTORY COMES BACK AS ONE ENTRY WITH `size: 0`, SO IT TRIPS NO LINE** (backlog #85). That is
     not a small omission. A sweep that reads only the top level of a project folder reports
     confidently on the top level and says NOTHING about the rest - silently, because a `0` reads as a
     small file rather than as an unopened door, and the sentence above claiming it covers archives
     stayed in this file for a release while being false. Measured on the reference data repo,
     2026-08-09 late evening: **Project Setup 63 files / 1,009,829 B invisible - 82.9% of that folder's
     markdown; Network Update 6 files / 42,065 B.** Four files already at or over the 40 KB warn line
     were invisible, including the LIVE backlog at 53,544 B, the tightest file in the repo. Those
     figures are DATED to that evening (section 8, backlog #83); the walk re-derives them.
   - **Recurse. For every entry of `type: dir`, issue another `get_file_contents` listing on its path
     and sweep that listing too, repeating until no unvisited directory remains.** Pass
     `fields: ["type","name","path","size"]` - it keeps each response small, and `path` comes back
     fully qualified, which is what the report needs.
   - **NO DEPTH CAP AND NO SKIP LIST.** Both were considered and both were rejected. A depth cap
     reproduces this exact bug one level lower and fires the first time an archive folder acquires a
     subfolder of its own. A skip list is a hardcoded assumption about a data repo whose layout this
     plugin does not own. **A boundary nobody can see is worse than a cost everybody can** - which is
     the entire content of #85.
   - **Bound the walk LOUDLY, never silently** (section 13). If the walk reaches **200 listings**, stop
     walking, write the master list anyway, and name in the run summary every directory left unvisited
     and how many. A cap that announces itself is a limit; a cap that does not is the bug being fixed
     here.
   - **Report the listing count in every run summary** - "swept N directories, M files." The cost of
     this sweep is then a measured number in front of the user on every run, rather than a claim
     recorded once in this file and believed thereafter.
   - **Sizes are READ, never predicted** (`../../docs/CONVENTIONS.md` section 8). The listing reports the
     bytes actually stored; do not compute a size from an estimate or from arithmetic on one.
   - **Report as a table in the run summary:** project, **full repo-relative path**, measured bytes,
     and which line it is over. A bare filename does not say where the file lives, and archive
     filenames repeat across projects. Name the reason the lines exist once, per section 3 - a warning
     that states a number and nothing else gets dismissed.
   - **An ARCHIVE at 40 KB is over its own CONTRACT, which is not the section 3 warn line, and it is
     reported as a breach rather than as a warning.** Section 4 governs: append until an archive
     reaches 40 KB, then open the NEXT numbered archive and give it the same contract. So an archive at
     41 KB is not "a large file worth an eye" - it is a file that must stop receiving appends now, and
     the report names the successor archive as the remedy. **This is the case that made recursion
     urgent:** a backlog archive sat 1,534 B over its cap for three sessions, with a successor NAMED in
     advance and never created, and `plu` was structurally unable to say so because `Archives/` is a
     subfolder. **Naming a successor is not creating one, and the check that should have shouted could
     not see the file.**
   - **Report EVERY file over a line. No top-N and no truncation** (`../../docs/CONVENTIONS.md` section 13).
     A silent cap here reads as "nothing else is close," which is the one thing a sweep must never say
     falsely.
   - **This is a check, not a repair** (section 13). Do not split anything, do not open an archive, do
     not edit any project's files from here. `plu` writes exactly one file: the master list.
   - A file over a line never blocks the rebuild. The master list is written either way.
   - **Honest cost, RE-MEASURED 2026-08-09 late evening: this sweep now DOES add connector calls -
     one listing per subfolder.** On the reference data repo a full recursive walk is **37 listings
     against step 1's 14**: 23 added, of which 14 are a vendored copy of this plugin sitting inside the
     data repo (`Project Setup/plugin/` and everything beneath it). 153 files swept. DATED to that
     evening; the run summary's own listing count is the LIVE figure.
     *(This SUPERSEDES the note carried here since 2026-08-08, which said the sweep added no new calls.
     That note was TRUE ON ITS DATE and true of a one-level sweep: step 1 must list every top-level
     folder anyway, and a listing already carries each entry's `size`, so the sweep read a field
     fetched for another reason. The rev 2.4 audit's finding A5 said the opposite - that #62 "costs one
     directory listing per project, 13 today" - and was overruled on exactly that ground, correctly.
     **Recursion is what makes A5's concern real, one level down:** the subfolder listings are new work
     that nothing else needs. The old note is not stale and is not being called stale - it is
     DISCHARGED, correct until this change, and replaced because this change falsified it. Section 8,
     backlog #83.)*
4. Write `Project Setup/Project Setup - Master List.md` containing, in order:
   - A short auto-maintained HTML comment (regenerated by /project-list-update; a row is added by
     /project-new; do not hand-edit). Include a brief COLUMN MEANING BY TYPE note in that comment
     recording that `📓` rows are Learning Notebooks and how their Open Qs / Goal columns are derived
     (the comment does not render, and it keeps the remapping discoverable by whoever reads the raw file).
   - A title, then `**Last Refresh:** <Today> (by /project-list-update, tree <short-sha>)`, where
     **`<Today>` is computed at write time by a shell command converted to `<Timezone>`** - never
     read off `<env>`, never carried from the file's previous value, never a UTC date
     (`../../docs/CONVENTIONS.md` section 12, backlog #57). This line dates the entire snapshot, so a
     UTC clock backdates or postdates every row beneath it at once. If no shell is reachable,
     use section 12's no-shell fallback and say in one line that the fallback is in use.

     ⚠️ **`<short-sha>` IS THE TREE THIS REBUILD READ, AND WHICH TREE IT IS MATTERS (new in
     v0.14.0, backlog #94).** This table is the largest whole-tree claim the plugin produces, and
     **a date does not identify a tree** - two rebuilds on one day against different heads write
     the same line, which is exactly the confusion this is here to prevent.

     **NAME THE REMOTE BRANCH HEAD THE REBUILD READ, CAPTURED BEFORE THE FIRST FILE IS READ.**
     Three properties, each ruling out an alternative that looks equivalent and is not:
     - **REMOTE, not local.** A clone routinely sits ahead of `origin`. A local-only SHA resolves
       in one working copy and FAILS AS ABSENT everywhere else - and "cannot resolve" is
       indistinguishable from "not an ancestor", which is the null-means-two-things defect landing
       inside the check meant to catch staleness.
     - **BEFORE the first read, not at write time.** The rebuild reads every project's memory doc
       and then writes. A concurrent push in between makes a write-time SHA name a tree the numbers
       were NOT computed from.
     - **Whichever channel is in use.** Connector session: the head SHA from the commit list for
       `<Branch>`, taken before the first file fetch. Mounted clone: `git rev-parse origin/<Branch>`
       - **the remote-tracking ref, NOT `HEAD`.**

     ⚠️ **IF LOCAL WORK IS NOT ON THE REMOTE, SAY SO IN THE LINE RATHER THAN HIDING IT.** When the
     clone has commits `origin/<Branch>` does not, write the remote SHA and append
     `+ local commits not on origin - NOT REPRODUCIBLE FROM THE REMOTE`. **A SHA NOBODY ELSE CAN
     RESOLVE MUST ANNOUNCE THAT FACT**; silently writing the local one produces a line that looks
     checkable and is not.

     **WHAT THE SHA IS FOR, so it is not "simplified" away later:** any reader can assert
     `git merge-base --is-ancestor <named-sha> HEAD`. If that fails the snapshot is STALE and no
     number in it should be trusted until a rebuild. The check goes quiet by itself at the next
     rebuild. Before this line carried a SHA the rule existed and had nothing to read.
   - The ACTIVE table `| Project | Open Qs | Latest change | Goal |`, one row per ACTIVE entry of EITHER
     type, sorted alphabetically by name **ignoring the `📓 ` marker** (so notebooks interleave with
     projects rather than clustering).
   - A single archived-count line: `**Archived:** <M> project(s) not listed above - run /project-list-archive
     (pla) to see them.` (Use "🗄️ " prefix; if M is 0 say "0 archived projects.")
   - An `## Archived Projects` heading, then the SAME-format table `| Project | Open Qs | Latest change |
     Goal |` with one row per ARCHIVED entry, sorted alphabetically, same per-type column rules. (Omit the
     table if M is 0.)
   - A footer: `*N active + M archived projects. "Project Setup" is the meta/system folder (hosts the
     config and templates; exempt from archiving).*` **If at least one active entry is a notebook**, break
     the active count out: `*N active (P projects + K learning notebooks) + M archived projects. ...*`
     With zero notebooks the footer stays exactly as it is today.
5. **Lint the table before committing** (`../../docs/CONVENTIONS.md` section 6): uniform column count in
   each table block, counting cells by splitting on pipes that are NOT backslash-escaped; no blank
   line inside a table; exactly one trailing newline. A project name containing a pipe must have it
   escaped, or it silently adds a column to that row.
6. **Commit SHA-locked.** Re-fetch the file's blob SHA immediately before writing and commit with
   `create_or_update_file` using that SHA - never `push_files` for a file that already exists
   (`../../docs/CONVENTIONS.md` section 2). Message: `project-list: rebuild master list (N active, M
   archived)`. If the write is rejected as stale, re-fetch and rebuild rather than retrying.

## Rules
- This is a FULL rebuild from the live folders - do not carry over old rows by hand.
- Report only what the memory docs contain - never invent details. Unknown -> "TBD".
- Write project names out in full, without the `zArchived - ` prefix.
- **The whole table is a dated snapshot, and the `**Last Refresh:**` line is what makes it honest**
  (`../../docs/CONVENTIONS.md` section 7). Never write a row asserting a current state without that line
  being refreshed in the same commit.
- **This rebuild is also a check.** Report, in the run summary, any folder holding a memory doc that
  could not be parsed, any project whose layout was ambiguous, and any count written as `?`. Reporting
  is the job; do not repair a project's memory doc from here.

## SHARED CONVENTIONS - read and obey `../../docs/CONVENTIONS.md`
This skill obeys the plugin's shared conventions file (new in v0.11.0). It is authoritative and is
the single home for every rule that applies to more than one skill: **NO SECRETS IN THE DATA REPO -
EVER** (section 1), SHA-locked writes (2), the size rule (3), how to split a
file (4), append rules by section type (5), the markdown table lint (6), dated state claims (7),
computed counts (8), reconciliation as a check and not a repair (9), never filling an unknown with a
plausible guess (10), index markup (11), entry dates computed in the config timezone (12), and the
reporting rules including no silent caps (13). Read it before writing anything to the data repo. If this file and the
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
