# Shared Conventions — cowork-projects

**This file is authoritative.** It is the single home for rules that apply to more than one
skill. Every skill that writes to the projects data repo carries a one-line pointer to this
file and is expected to obey everything in it. When a rule changes, it changes here, once —
not in nine copies that drift apart.

**A skill is bound by this file when it makes a claim this file GOVERNS — not merely when it
writes.** Writing is the commonest way to make such a claim, which is why the roster below was
once read as a list of writers, but it is not the line. Section 7 is a CLAIMS rule: a derived
figure must carry the date it was measured, and that obligation falls on whoever puts the
figure in front of the user. A read-only command that renders a cached count makes exactly
that claim and is bound by section 7 without writing a byte. Ask what a skill ASSERTS, not
whether it commits.

Skills that must obey this file: `project-new` (pn), `project-open` (po), `project-update`
(upd), `project-setup-init` (psi), `project-archive` (pa), `project-unarchive` (pua),
`project-list-update` (plu), `project-list` (pl), `project-list-archive` (pla).

Introduced in v0.11.0.

---

## 1. NO SECRETS IN THE DATA REPO — EVER

**Never write a credential into the projects data repo.** Not in memory docs, not in notes
files, not in companion files, not in the master list, not in config files, and not in commit
messages. This holds even when the user pastes a credential into the chat, and even when the
user explicitly asks for it to be saved.

Credential means: passwords, passphrases, private keys, API tokens, access tokens, session
cookies, recovery codes, connection strings containing any of the above, and anything else
that would let a holder authenticate as someone.

**What the repo MAY record:** where a credential lives, and what the authentication method
is. For example — "SSH: dedicated ed25519 key on the admin workstation, mode 600, key auth
only, no password auth" — is correct and useful. It names the location and the method and
contains nothing usable.

**What the repo MAY NOT record:** the material itself, in any form, including partially
redacted forms. A half-masked token is still a leak of its other half.

**Decisions about authentication belong in the Decisions Log.** The method, the storage
location, the rotation schedule, who holds it — all of that is project memory and should be
written down. The credential is not project memory and must not be.

**If a secret appears in the chat:** do not commit it anywhere, do not echo it back into a
document, and tell the user plainly that it will not be stored. Do not offer to store it
somewhere else in the repo instead.

**Credential FILES belong outside any folder a session can read.** Put them somewhere like
`~/.config/<tool>/`, directory mode 700 and files 600 — never in the working folder, never in a
trusted or attached folder, and never passed on a visible command line, where they land in shell
history, `ps` output, and session transcripts.

**This rule also applies to what the repo tells a human to do.** A committed setup guide that
instructs the reader to place a password into a tracked file is a leak with a delay on it —
the harm fires when someone *correctly follows* the documented procedure. Treat that as the
same violation, not a lesser one, and flag it when writing or reviewing any runbook.

---

## 2. Write safety — every commit is SHA-locked

The GitHub connector offers two write paths and they are not equivalent.

`push_files` accepts owner, repo, branch, a list of files, and a message. It accepts **no
blob SHA and no base commit**, so it cannot detect that the file changed between the read and
the write. A collision is not rejected — the losing write disappears silently, with a success
response. Nothing surfaces.

`create_or_update_file` accepts a `sha` parameter, documented as required when the file
already exists. That is a genuine optimistic lock: if the file moved under you, the call
fails loudly instead of overwriting.

**Therefore: always write with `create_or_update_file`, locked to the blob SHA that was read.
Never use `push_files` for an existing file.**

**One command is not one commit.** A skill may stay a single thing the user types while
making several locked commits underneath. That is the intended shape, not a compromise.

**Write order: notes first, memory last.** If the memory commit fails, the session merely
looks unrecorded. If the order were reversed, a memory changelog entry would be left
asserting a notes write that never landed — a document that lies about itself.

**Verification.** Re-fetch the file and confirm its blob SHA immediately before editing, so
the edit is known to be based on the live bytes. After the write, confirm the returned blob
SHA. If a skill can predict the SHA locally before sending, it should, and it should assert
equality afterward.

**The trailing newline is load-bearing.** The emission path does not preserve a final newline
on its own. It must be present in the content string deliberately, or the pushed file differs
from the intended file by one byte and every SHA prediction fails.

**Assert the expected byte count before every write** (new in v0.12.0). `create_or_update_file`
will accept a TRUNCATED body and return success: the SHA it reports is the SHA of what actually
arrived, so comparing it afterward proves the transport was faithful but never proves the
content was complete. Only a pre-write size assertion catches a partial emission, and the
damage from one lands on the branch. So: know the byte count you intend to write, compare it
against the content you are about to send, and stop if they disagree. A shrink that is not a
deliberate split or cut is a bug, not a tidy-up.

⚠️ **AND A PRE-WRITE ASSERTION CANNOT CATCH EVERYTHING — THE TRANSPORT ITSELF EDITS CONTENT
(measured 2026-08-25).** `create_or_update_file` takes its `content` as a JSON string, and **the
JSON layer DECODES unicode escapes inside it.** A source line written as a hyphen followed by two
backslash-u escapes naming **U+2013** and **U+2014** — 13 ASCII characters — arrived as `-–—`, the
three characters those escapes denote, 7 UTF-8 bytes. The file was committed 6 bytes shorter than
the file that was tested.

⚠️ **THAT SENTENCE NAMES THE CODE POINTS AS `U+2013` INSTEAD OF SHOWING THE ESCAPE FORM, AND THE
REASON IS THE RULE ITSELF: writing the escape literally here would have this paragraph rewritten in
transit by the defect it documents.** A hazard that corrupts its own description is worth stating
plainly — **when documenting an escape, name the code point rather than spelling the escape.**

⚠️ **THIS IS A DIFFERENT FAILURE FROM THE ONE THE RULE ABOVE CATCHES, AND THE RULE ABOVE CANNOT
SEE IT.** A byte-count assertion catches TRUNCATION — content that never arrived. This is content
that arrived and was **rewritten in flight**: the count is wrong *because the transport changed it*,
after the count was taken. **A hash computed before sending is a hash of the wrong object.** No
pre-write check of any kind can close this, because every pre-write check runs on the near side of
the thing doing the damage.

**What actually detects it: READ THE COMMITTED BLOB BACK AND COMPARE IT TO WHAT YOU SENT.** The
returned blob SHA is computed by the server over the bytes that landed, so comparing it against a
locally predicted SHA does catch this — which is what the verification paragraph below already
asks for. **The escape case is the reason that comparison is not ceremony.**

⚠️ **STATED HONESTLY, BECAUSE THE RULE SHOULD NOT CLAIM MORE THAN IT EARNED: this was found only
because the committed bytes were re-tested afterwards, not because any rule required it.** The
mismatch was noticed, diagnosed to one line, and the committed file then re-run through its whole
test suite — where it passed, the two forms being equivalent in that language. **Had the affected
text been load-bearing rather than equivalent, nothing in this file would have stopped it
shipping.** Recording the hazard is not the same as fixing it, and no remedy is adopted here.

**Scope, so this is not over-read: escapes are only a hazard in content that MEANS them literally**
— source code, regexes, a document quoting an escape sequence. Prose and Markdown carrying no
backslashes cannot be affected. Sweep the content you are sending, not every file you own.

**Prefer having the session that GENERATED content commit it** (new in v0.12.0). A session
re-emitting a large file it merely read is the shape that produced the partial writes this
rule exists to catch. Where a delegated agent must write, it writes content it produced.

**Every commit message carries a `[base:<sha>]` tag** (new in v0.12.0 — backlog #29). Append
the 7-character prefix of the blob SHA the edit was based on: `upd: split the backlog
[base:1a2b3c4]`. For a multi-file commit, tag the primary file's base SHA. New-path creations
have no base blob and carry no tag. **This is forward-protecting only** — nothing committed
before v0.12.0 carries one, so a clean audit of older history means "no large loss visible,"
never "nothing was lost." Its value is that an auditor can walk a path's commits and prove a
chain break. *(Recorded honestly: this rule was believed shipped in v0.11.0 by three separate
documents and had never been written. Sessions were typing the tag by hand, so practice looked
like enforcement. Verify a shipped-claim against the source, never against a status row.)*

---

## 3. Size discipline — 40,000 B warn, 60,000 B stop

Per file, in the data repo:

- **At 40,000 B — warn.** Tell the user the file has crossed the warn line, name the current
  byte count, say what the split options are, and **name the reason in one clause** — the
  lines exist to keep the file well below the fuzzier size at which a write stops being
  possible at all. Then carry on and write.
- **At 60,000 B — stop and ask.** Do not write silently past it. Name the size, name the same
  reason, and offer the split. The user may still say go ahead; the point is that they decide.

**The lines are decimal, and this section is where they are written** (pinned 2026-08-15). Until
now this section said "40 KB" and "60 KB" and never wrote either as a byte count, so 40,000 and
40,960, and 60,000 and 61,440, were all defensible readings — **and each pair disagrees about
every file between its two values.** They are **40,000 B and 60,000 B exactly**, decimal, not
1024-based. This codifies what was already computed everywhere rather than choosing something
new: every headroom figure this project has published used these two numbers, and no 1024-based
value has ever been written in either repo. Where any document still says "40 KB" or "60 KB",
read it as shorthand for the number here, and the number here governs.

**A warning that states a number without its reason trains the user to dismiss it** (new in
v0.12.0 — backlog #66). That is the same failure as chattiness, arriving by a different route,
so the fix has the same shape: the warning itself carries the one clause, and the long form
lives in the paragraph below and in the Setup Guide. A longer warning is a heavier warning —
affordable only because the once-per-sitting rule fires it far less often.

**These are not the same thing as the emission ceiling.** There is a separate, larger limit
on how big a single write can be at all, and it is a property of the model running the
session rather than of the file — writes near 72–76 KB have succeeded while writes near
83 KB could not be emitted at all. That failure is loud and safe: the call is never sent, no
partial write occurs, and the SHA lock is not consumed. The 40/60 KB rule exists to keep
files well below that region, where behaviour is predictable.

⚠️ **A READ-BACK CLIFF WAS ASSERTED HERE EARLIER ON 2026-08-24 AND IS WITHDRAWN — IT DOES NOT
EXIST.** For part of that day this section justified the two lines partly on a claim that a file
above some size *"can no longer be read back."* **That claim is false.** About sixty classified
probes from 23 B to 98,721 B, across both repos, found **zero true refusals**: every oversized
fetch named a saved path and every recovered result came back intact. No file in either repo is
unreadable, and **no file needs splitting in order to be read.** The lines above stand on the
emission ceiling alone, which is what they always stood on. This paragraph stays rather than being
deleted, because the withdrawn claim is the more instructive half: **a convention that names one
hazard licenses everything the other one forbids, and a convention that invents a hazard spends
the reader's trust on a fiction.** Both failures are cheap to make and expensive to notice.

**There IS a read-side cost. It is a PROCEDURE hazard, not an unreadability one, and it does not
justify a threshold.** Above the cut a fetch does not deliver content in the reply; it spills to a
saved file that must then be opened. That costs a step — and **it fails outright for a reader who
cannot reach the saved path.** Measured 2026-08-24: a Cowork sandbox has no `/var/folders` at all,
so a spilled result is simply out of reach there, and the host-side read tool paginates, so a large
spill takes several passes. **So the content is recoverable in principle and not always in
practice, and the difference is the reader's access, not the file.** Say this when it bites;
do not turn it into a line.

⚠️ **THE CUT IS ON CHARACTERS, NOT TOKENS AND NOT QUITE BYTES — AND NO BYTE NUMBER FOR IT IS
WRITTEN HERE, DELIBERATELY.** The decisive measurement: **46,000 B of base64 (~43,800 tokens)
returned inline while 55,000 B of prose (~17,200 tokens) spilled.** A token cap cannot produce that
ordering. The cut lands on the *serialized result*, so the byte size that trips it moves with
composition — escaping, wrapper text, and how densely the file uses multi-byte characters all shift
it. ⚠️ **And it moves between client builds.** The figure this project recorded on 2026-08-24 as
the boundary was, by later the same day, off by roughly 11 KB — **the very file it recorded as
fetching successfully now spills at a size several hundred bytes SMALLER than the figure that was
written down.** It was measured honestly and it still rotted. Within one session the cut is stable
to about 900 B; across builds, to double-digit KB. **Any number written here would become the next
stale constant, so none is** — including the character figure, which is a measurement too. What is
durable is the RELATION: on the day this was written the cut sat **above the 40,000 B warn and
below the 60,000 B stop**. That is why 40,000 B is the line that has survived every measurement,
and why it is the one to plan against. **Re-derive the cut in the session that needs it, and quote
no one else's.**

⚠️ **THE WRITE-TIME / READ-TIME ASYMMETRY IS REAL AND WORTH KNOWING — ONLY ITS CONSEQUENCE WAS
OVERSTATED.** The emission ceiling is a fact about how a file is **WRITTEN**, so it does not reach a
file no model wrote — a `cp` on a clone, a generator's output, a merge. The spill cut is a fact
about how a result is **READ**, so it reaches every file whatever put the bytes there. **What that
reach costs is a recovery step, and a class of reader who cannot take it — not a lost file.** A
session that reasoned *"this file is placed by `cp`, not emitted, so the emission ceiling does not
reach it"* was reading correctly, asked the user as this section requires, and was told to keep the
file whole. The files it produced are large and awkward to read. **They are not lost, and nothing
here asks for them to be rewritten.**

⚠️ **AND THE STOP DOES NOT REACH A FILE THAT IS *GENERATED* AND *ALWAYS-LOADED* — the maintainer, 2026-08-16.**
It follows directly from the paragraph above. The 40/60 KB lines exist to keep files well below the
**emission ceiling**, and that ceiling is a property of the model doing the writing. **A file written
to disk by a GENERATOR is never emitted by a model, so the hazard the stop guards cannot reach it.**
The worked case is a rules roll-up rendered from per-rule entry files by a script and then fetched
unconditionally at open.

⚠️ **THE PREDICATE IS *GENERATED AND ALWAYS-LOADED*, NOT *LARGE*, AND THE DIFFERENCE IS THE WHOLE
SCOPING.** A blanket exemption would strip the ceiling guard from exactly the files that do have it —
every file a model writes. Both halves must hold: a generated file that is not always-loaded, and an
always-loaded file that a model writes, are BOTH still governed.

⚠️ **WHAT IS NOT EXEMPTED, STATED SO THIS IS NOT READ AS A BLANKET PASS: CONTEXT COST.** An
always-loaded file still costs the reader's context on every open, and that hazard is untouched by
where the bytes came from. **The stop was simply the wrong instrument for it** — a threshold that
refuses a write says nothing about what a load costs. **The replacement is VISIBILITY, not a bigger
number: `project-open` reports the bytes of every file it loaded and each file's share of the open
(see its LOAD REPORT).** The two shipped together, deliberately: removing the gate without the
measurement would leave neither.

⚠️ **AND SPILL COST JOINS CONTEXT COST — SAME FAMILY, SAME REMEDY, NOT A NEW THRESHOLD (added
2026-08-24).** An always-loaded file above the spill cut is fetched, spilled and recovered **on
every open, forever** — so the cost recurs at the rate the file is READ, not once at the rate it is
written, and it lands hardest on the readers who cannot reach a saved path at all. *(An earlier
wording today justified this by saying such a file "does not load" — that was wrong; it spills and
is recovered. The scoping below survives on this weaker and better-evidenced ground instead.)*
**This is the paragraph above wearing a different hat, and it takes the same answer: VISIBILITY,
not a bigger number.** The LOAD REPORT already measures it. Measured 2026-08-24: five standard docs
in the data repo were already above the cut, so `project-open` is spilling on them at every open
and depending on path recovery to succeed. **That is a live recurring cost and an argument for
measuring it — never for a third line.**

**Never treat a hard byte number as a permanent constant.** Warn with the number and the
reason; do not refuse on arithmetic alone.

**Say the WARN once per sitting, not once per write. The STOP is never suppressed and speaks
before every write.** A file that is over the warn line stays over it for every write in the
session; repeating the warning on each one trains the user to skim past it. State the warn the
first time it is true in a sitting and at the end-of-session size re-check, and stay quiet in
between unless the file crosses a NEW line.

**The 60,000 B stop is exempt from all of that, and the exemption is not an oversight being
tidied up — it is a reversal.** The stop is not a warning; it is a decision the user has to
make, and a decision about one write is not a decision about the next. Two things make
suppressing it unsafe rather than merely quiet. **There is no line above the stop, so a file
past it can never cross a NEW line** — suppress the stop once and it stays suppressed for the
rest of the sitting, with no event able to re-open it. And **the first utterance is routinely
spent before any write happens**, by `project-open`'s health check at open. Together those mean
every `upd` in the sitting would write past the stop with nobody deciding, which is the one
outcome the stop exists to prevent. **Asking again is the cost; writing past a stop unasked is
the failure, and the two are not close.**

---

## 4. When a file must be split

**First choice: cut by date.** In log-shaped sections — Changelog, Session Notes, resolved
Open Questions, superseded decisions — everything dated before a single uniform cut date
moves verbatim to a dated archive companion, and everything a fresh session must read stays
resident. One cut date for the whole file, not a different one per section.

Leave behind a single italic pointer line naming how many items moved, their date range, the
destination file, and the date of the move. The pointer replaces the moved content; the
heading stays.

**When dates cannot cut, extract a whole section instead.** Some large sections are not
date-shaped — a live backlog, a definitions table — and no date cut will shed them. In that
case move the entire section to its own companion file, leave the heading with a pointer stub
that must never regrow a table, and add a row to the Companion Files index. This is a real
and expected case, not a failure of the date rule.

**Reconcile the split programmatically before committing.** Assert that the number of items
out equals the number in, that the moved text is byte-identical to what it replaced, and that
zero items were left behind in the source. Assert, do not eyeball.

**Order of commits: the companion first, the pointer second.** The pointer must never name a
file that does not exist yet.

### Resident budgets — a section spills on a trigger, not on a cut date (new in v0.12.0 — backlog #56)

A cut date is a judgement call made once; a **resident budget** is a standing rule that keeps
a log-shaped section bounded without anyone deciding anything. Applies to the Changelog and
the Decisions Log:

- **Budget: 6 KB resident per section.** The newest entries that fit stay; older ones spill.
- **Trigger: 16 KB.** A spill fires when the section reaches 16 KB, NOT when it exceeds the
  6 KB budget. The gap between them is deliberate hysteresis.
- **Minimum viable spill: 10 KB moved.** A spill costs roughly 2.5–3 KB of bookkeeping — the
  pointer line, the archive header, the companion-index row. Moving 400 bytes against 3 KB of
  bookkeeping makes the file bigger, so a budget with no trigger is worse than no rule at all.
  The 16 KB trigger guarantees at least 10 KB moves, which is the ≥3× ratio that makes the
  operation pay for itself.
- **The keep set is a strict prefix.** Keep newest-first until the next entry would exceed
  6 KB; everything after that point spills, in order, with nothing skipped over.

**Where a spill goes: an archive's own contract governs.** A spill file opens with a purpose
block stating whether it accepts appends. **Append until the archive reaches 40 KB**, then open
the next numbered archive and give it the same contract. An archive whose block forbids appends
is never appended to — open a new one. Never rewrite an archive's contract to make a spill
convenient.

**When a SINGLE entry exceeds the whole budget**, do not spill it and do not truncate it: warn,
keep it resident, and say which entry it is. A budget is a target for a set of entries, not a
cap on any one of them — and an entry that large is usually a §5 cap violation worth naming.

### Stubs must be self-describing, not cleverly detected

A section whose content has moved leaves a **pointer stub**, and a stub says so in its own
words: it names the companion in backticks, states that it is a pointer and not a placeholder,
and says do not regrow a table here. **A readiness check must read the stub's own words, not
infer emptiness from length.** Making the parser cleverer is the wrong fix; making the file
honest is the right one.

---

## 5. Append rules, by section type

**Decisions Log.** Newest at top. Append-only. Never delete a decision — supersede it with a
new one and mark the old row superseded.

**Open Questions.** Open items on top, then resolved ones newest-first. Never delete.

**When a row moves to an archive** — a decision superseded, a question resolved — the row
moves whole, and **only its `Status` and `Topic` cells may change.** The decision or question
text, its original date, and its rationale are never edited, reworded, or renumbered. Do not
write a rule that forbids marking the row you were just told to mark.

**Changelog.** One dated entry per session, most recent on top. Append-only; never rewrite an
existing entry's text. A correction goes in a new dated entry, or in the file's descriptive
header block, never by editing history. Split by the uniform-date rule of section 4 into
`Project Memory Changelog Archive - <Project> (through <DATE>).md`, or by the resident budget
of section 4 into a numbered archive.

**Changelog entries are capped: 200 bytes target, 400 bytes hard** (new in v0.12.0 — backlog
#59). *Previously CONVENTIONS said only "one dated entry per session" and set no length at all,
while the memory-doc template said "one line per session" — so the rule being enforced here is
newly written, not merely tightened.* A changelog entry is an INDEX: what changed, and where
the detail lives. It is not the detail.

- **Over 400 bytes, the overflow moves and the entry points at it.** For a **Project**, the
  detail's home is a dated Session Notes section in the notes file. For a **Learning
  Notebook** — which has no Session Notes, and whose Learning Log is explicitly a pointer
  index — the detail's home is the relevant `## Topic` section, edited in place. Never file a
  notebook's overflow as a dated log entry; that is the project shape, and it is exactly what
  the notebook type exists to prevent.
- **The asymmetry is the root cause, not indiscipline.** The changelog entry is mandatory
  every session and the Session Notes entry is discretionary, so everything lands in the
  mandatory field. Making the overflow's destination explicit is what fixes it.

**Structure-only changes still owe a changelog entry** (new in v0.12.0 — backlog #69). When a
session changes a project's SHAPE without judging its content — a template migration, a
section extraction, a rollout applied across projects — write one line into **that project's**
memory changelog naming what changed structurally, stating plainly that no content was judged,
and **naming which session ran it**. The memory doc is the file a fresh session reads, so a
structural change missing from it leaves the next session unable to tell an intentional shape
from drift. The name-the-session clause matters because cross-project structural writes are
routine.

**Imported entries sort by ENTRY date, never by import date** (new in v0.12.0 — backlog #64).
Imported content carries two dates: when the work happened, and when it was imported. Both are
recorded; **only the entry date decides position.** Sorting by import date piles a decade of
history on top of this week's work and makes the newest-on-top rule lie. Write it as the entry
date with the import date in parentheses — `2026-07-14 (imported 2026-08-08)` — so the pair is
visible and the sort key is unambiguous. **Imports never mark existing entries superseded**; a
conflict is flagged and asked about.

**Data Sources and Definitions.** Not log-shaped, so section 4's date cut does not apply. A
revised definition **supersedes** its predecessor rather than overwriting it: the old text
stays with a superseded marker and a date. Overwriting a definition in place destroys the
record that it ever changed, which is usually the thing someone needed.

**Backlog.** New rows take the next free number and go at the bottom. Rows are never
renumbered and never deleted. A closed item has its Status cell rewritten in place and stays
where it is.

**Backlog rows are capped: 600 bytes target, 1,200 bytes hard** (new in v0.12.0 — backlog
#68). The same discipline as the changelog cap, one register over, and for the same measured
reason: a backlog file grows out of a row-verbosity problem wearing a file-size costume, and
neither a status partition nor an ordinal cut sheds enough to matter. A row states the item and
the evidence that put it on the list; the argument for it belongs in the memory doc or a
companion. **Honest limit, recorded with the rule:** a cap slows growth, it does not bound it.

---

## 6. Markdown table lint — run before every write

Cheap checks that catch damage which is invisible in review:

- **No blank line inside a pipe table.** A blank line silently ends the table; the rows after
  it still render as text, so nothing looks wrong until a parser walks them.
- **Uniform column count within each contiguous table block.** Count cells by splitting on
  pipes that are not backslash-escaped — an escaped pipe inside a cell is content, not a
  delimiter, and counting it produces false alarms on exactly the rows most worth checking.
- **Exactly one trailing newline** at end of file.
- **No literal shell command chains in committed content.** The upstream web firewall rejects
  commits containing them. Describe commands in prose instead.

---

## 7. Every state claim carries the date it was measured

A **dated** snapshot ages honestly — a reader sees the date and discounts it. An **undated**
status field is a claim about right now, and it rots silently while continuing to sound
authoritative.

So: any row that records a file's current size, its blob SHA, a version pointer, or a count,
must either carry the date it was measured or be re-read and re-asserted at write time. Where
the writing skill already holds the number, re-asserting is nearly free and is the better
option.

⚠️ **AN EXPECTED BLOB SHA IS THE SHARPEST CASE OF THIS RULE, AND THE ONE MOST OFTEN GOT WRONG.**
It is valid only at the instant it is derived. A stale version number reads as obviously old; a
stale 40-hex SHA reads as PRECISE, and is therefore believed. **Any document, test or handoff
that names an expected SHA must either name the CONDITION instead** — *"the file's blob,
whatever it is when the run starts, is unchanged after"* — **or carry an explicit
re-derive-before-use instruction.** This is the same lesson as a release name inside a standing
order, which goes stale on the first re-target; the SHA form is worse only in that it does not
look stale. Measured case: one test's expected blob went stale three times in about thirty
hours, once at its own branch's fork point and once from the same session's earlier write.

---

## 8. Counts are computed, never hand-counted — and so are sizes

Any number that appears in a document — rows in a table, open questions, files in a folder —
is computed from the data at write time. Hand-counted numbers have been wrong every time they
have been checked, and they are wrong in the direction of understating, which is the
direction that hides problems.

If a count cannot be computed, do not write a number. Write what is actually known.

**The same rule governs byte sizes and byte deltas** (new in v0.12.0 — backlog #60). A
predicted size is not a measured size. Where a skill states what a file now weighs, or how much
an operation shed, that figure comes from the bytes actually written or actually read — never
from an estimate, and never from arithmetic on an earlier estimate.

**Sizes err in the OPPOSITE direction from counts, and the mechanism explains why.** A
hand-count understates because it is easy to skip a row. A size *prediction* overstates the
shrink, because the predictor reasons about the content it deliberately moved and forgets the
bookkeeping it added back — the pointer line, the archive header, the index row. Four
predictions in one day were wrong in that same direction. So: **state a size only after
measuring it**, and when reporting a delta, report both endpoints, both measured.

**A size delta is also a corruption check.** A write that shrinks a file it did not mean to
shrink is the signature of a partial emission or a silent compression, and the byte count is
the only thing that catches either. Compare the pre-write and post-write sizes against what the
operation was supposed to do; an unexplained shrink is reported, not smoothed over.

### The Open Questions status vocabulary — CLOSED is enumerated, OPEN is everything else (new in v0.27.0)

⚠️ **A COUNT OF OPEN ITEMS IS A DERIVED VALUE, AND ITS PREDICATE IS DEFINED HERE, ONCE.**
`project-list-update` and `project-open` both compute it and both point at this definition.

**Normalise before comparing.** Strip `*` and `_` from both ends of the Status cell and trim
whitespace. Strip nothing from the middle of the token. **Live rows write the status bolded, so a
raw comparison fails on most of them.**

**A row is CLOSED if and only if the normalised value BEGINS with one of — compared case-insensitively —**
`Resolved`, `Closed`, `RESOLVED`, `Completed`, `Dropped`, **followed by end-of-cell or a non-letter**;
**or begins** `Moved to` **and ends** `(verified <date>)`.

⚠️ **THE TRAILING BOUNDARY IS NOT DECORATION.** A bare *begins with* closes `Resolvedish`. Live rows
carry a date or a clause after the token — `Resolved 2026-07-28`, `Closed 2026-07-25 — import artifact` —
so the token is a PREFIX and the boundary is what stops it becoming a substring match.

**Everything else is OPEN**: `Open`, `Partially resolved`, `Moved to <project> (open)`, an empty
cell, and **any token nobody has written yet.**

⚠️ **THE DIRECTION IS THE RULE, NOT THE LIST.** The predicate this replaced was open-world — it
counted rows reading `Open` — so every other phrasing fell out of the count in silence, and two live
tokens already had. **A novel token must default to VISIBLE.** An over-count is a question someone
re-reads and closes; an under-count is a question no session sees again, and nothing looks wrong
while it happens.

⚠️ **THE `Moved to` PAIR IS A TOKEN DISTINCTION, NOT PREDICATE LOGIC.** `Moved to <project> (open)`
stays OPEN and `Moved to <project> (verified <date>)` is CLOSED, because the verification lives in
the token. **The set stays flat: no conditional entries, no third category.**

**Provenance, so the next reader knows where to re-measure rather than trusting this paragraph:**
the set was **MEASURED in the data repo on 2026-08-25** across 210 live Open Questions rows in 15
memory docs plus 5 Network Update spokes. **The plugin repo names the full set in four places** —
this section, both skills, and the shipped memory-doc template — **so the SET is verifiable here.
What is not is which tokens are LIVE**, and that is what the data-repo measurement above records.
`Completed` and `Dropped` had **zero live rows** and are enumerated anyway, because a historical
token that returns must not be read as novel.

### The hub layout test and the hub question-count predicate (new in v0.30.0)

⚠️ **`project-list-update` AND `project-open` BOTH DECIDE WHERE A DOC'S OPEN QUESTIONS LIVE AND BOTH
COUNT THEM ON A HUB. THE TWO PREDICATES ARE DEFINED HERE, ONCE. BOTH SKILLS POINT AT THIS
DEFINITION AND NEITHER RESTATES IT** (backlog #229). They were near-twins written separately in each
skill, and they DRIFTED on three axes measured 2026-08-14; axis 2 was closed in v0.28.0 and this
closes the other two. 🚫 **Do not answer a drift between the twins by copying one skill's wording
into the other. The wording lives here.**

**(a) THE LAYOUT NEGATIVE TEST — FIND THE SECTION BY HEADING TEXT, THEN TEST FOR THE TABLE.**
When a memory doc carries no `**Layout:**` line, treat it as **Hub-and-Spoke** if it has **no Open
Questions table** *and* **does** have a master index table whose header carries a `Kind` column
(the hub's `Decisions & Questions Index` — `Date | Item | Kind | Where`). Otherwise **Single-file**.

**A heading is the Open Questions heading when its text — after any leading section number (`6.`,
`6)`, `6.1`) is removed and surrounding `*` and `_` are stripped — equals `Open Questions`, compared
case-insensitively, at any heading level.** The doc **HAS an Open Questions table** when such a
heading exists **and** at least one markdown table row appears under it before the next heading at
the same or a higher level.

⚠️ **BIND TO THE HEADING TEXT. NEVER TO THE LITERAL STRING `## Open Questions`, AND NEVER TO THE
NUMBER.** Re-derived in the reference data repo 2026-08-27 at `[COMMIT-87]`: **21 Open Questions
headings, 17 numbered and 4 bare — and all 15 that sit in a memory doc are numbered, so a literal
`## Open Questions` match finds 0 of the docs this test actually reads.** Numbering is not stable
across projects either: one doc keeps Open Questions at 6, another at 4 and uses 6 for something else.

⚠️ **THE TABLE IS THE TEST, NOT THE HEADING.** A hub keeps its `## N. Open Questions` heading as a
POINTER STUB with no table under it. A heading match alone reports "found it" and then counts
nothing, and **a zero reached that way is byte-identical to a zero that is true.**

**(b) THE HUB QUESTION-COUNT PREDICATE.** On a hub, an index row is an **OPEN question if and only
if** its normalised `Kind` is exactly `Question` **and** its `Where` does **not** point at the
archive. Normalise as in the vocabulary above: strip `*` and `_` from both ends, trim whitespace,
strip nothing from the middle.

⚠️ **THERE IS NO STATUS CLAUSE HERE AND ITS ABSENCE IS A DECISION, NOT AN OVERSIGHT.** The index
carries no Status column, so a status token has nothing to read. **A question leaves the open set by
exactly two routes — its `Kind` becomes `Resolved log`, or its `Where` is repointed at the archive —
and the predicate already tests both.** ⚠️ **A ROW MARKED RESOLVED IN ITS ITEM TEXT WITHOUT TAKING
EITHER ROUTE THEREFORE STAYS COUNTED AS OPEN. That is the safe direction, and it is the direction
the vocabulary above already chose:** an over-count is a question someone re-reads and closes; an
under-count is a question no session sees again, and nothing looks wrong while it happens.

**Provenance, so the next reader re-measures rather than trusting this paragraph:** `plu` carried an
extra *not marked resolved* clause that `po` did not. **Both predicates computed over the live
`Network Update` index 2026-08-27 at `[COMMIT-87]` — 152 index rows, 38 with `Kind` exactly `Question`,
9 pointing at the archive: `po` returned 29, `plu` returned 29, DIVERGENCE ZERO.** ⚠️ **THAT WAS A
PROPERTY OF THE DATA AND NOT OF THE RULES** — no row was marked resolved without also being
repointed, so the extra clause selected nothing, and the first row to break that pattern would have
split the two counts with no error anywhere. **The clause is removed, not copied across.**

### The rule above reaches every DERIVABLE value, not only counts and sizes (new in v0.27.0)

This section opened as a rule about counts and grew a clause about bytes. **The principle is
neither: if a value can be derived from the artifacts, deriving it is the only way to write it.** A
carried figure is a measurement of a tree that no longer exists.

**Four instances in a single day, named rather than counted from memory:** a checker's byte size
carried from a measurement sixteen days old and wrong by 27,314 B; a tracked-file census computed
over a local mirror that was missing a file, and so short by one file and 35,235 B; a file stating
**its own size** as measured before the edit that changed it — which happened **twice**, in
consecutive commits, the second while fixing the first; and a release's acceptance claiming two
changed files where the commit list gives three.

⚠️ **THE COMMON SHAPE IS A LOCAL VIEW TRUSTED WITHOUT A CROSS-CHECK.** Every one of those was
derivable at write time and every one was read instead. **Derive it — and where the derivation runs
over a local copy, verify that copy against the source first.** The census failure was not bad
arithmetic; it was correct arithmetic over an incomplete set.

### The annotation marker — what makes *size as written* measurable (new in v0.28.0)

**Section 5's 1,200 B backlog-row cap and 400 B changelog-entry cap measure an object AS WRITTEN,
excluding later dated corrections.** Without a marker that is unmeasurable, so the operational
definition is: **an inline dated correction appended to an existing object OPENS with the sentinel
`@@ANN@@`, and *as written* means the object with every marked span removed.**

⚠️ **SCOPE, because unscoped this becomes an instruction to every session.** It governs ONE
pattern: **an inline dated correction appended to an object that cannot be rewritten.** **A new
dated changelog entry is NOT an annotation and carries no marker; a superseded status stamp is NOT
an annotation either.** `project-update` already defines a correction as a new dated entry, and that
shape is unaffected.

⚠️ **IT IS A SENTINEL RATHER THAN A DATE OR A VERB BECAUSE ORIGINAL TEXT CONTAINS BOTH** —
`**Open — filed 2026-08-22**` defeats a date matcher, and a verb list reads whole originals as
annotation. **And it is not an HTML comment: comments cannot nest**, and every file in this system
opens with a comment header, so a comment-shaped sentinel would terminate its enclosing comment at
its own first `-->`.

**The limits travel with it.** The marker makes measurement **possible, not automatic**; it applies
**forward only**; and pre-existing objects were deliberately not retro-marked, so a repo-wide *as
written* measurement is not yet valid.

---

## 9. Reconciliation is a check, not a repair

Where an index points at content held in other files, a skill can verify the pointers without
being able to rebuild them. Counts, dates, kinds and destinations all reconcile exactly, so a
missing or orphaned index row is detectable and should be reported.

It does **not** follow that the index can be regenerated. Hand-written editorial labels exist
nowhere else in the repo, there is no stable join key, and dates repeat. Report the
discrepancy and let the user resolve it. Never silently rebuild an index.

**Reconciliation is not gated on a layout.** A memory doc's companion index is worth checking
whether the doc is single-file or hub-and-spoke; conditioning the check on the layout means the
common case never gets checked. Run it on every doc that has an index.

**The check runs in BOTH directions.** Every index row resolves to a file that exists, AND
every non-standard markdown file in the folder appears in some index. A companion outside every
index is invisible to every future session, which is the same as lost — and only the second
direction finds it.

---

## 10. Never fill an unknown with a plausible guess

Write `TBD` and ask. A confident wrong value costs more than an admitted gap, because nobody
goes back to check the parts that sounded right.

---

## 11. Index markup — backticks mean live, strikethrough means dead (new in v0.12.0)

Backlog #63. Every index in the data repo — Companion Files, a hub's Decisions & Questions
Index, the notes header's link mirror — names files, and other rules key on those names. The
markup has to be unambiguous:

- **A live entry's filename is written in backticks**, path-qualified relative to the project
  folder: `` `Archives/Project Memory Archive - Project Setup 005.md` ``. Backticks are what a
  reader and a rule can both key on; a bare filename in prose is not findable and not
  distinguishable from a mention.
- **A dead entry is struck through and dated, never deleted**: ~~`old-name.md`~~ — DELETED
  2026-08-08, content moved to `new-name.md`. The row stays, so a future session can tell
  "removed on purpose" from "never existed."
- **Nothing else means anything.** Bold, italic, and plain text carry no status. Do not invent
  a third state.

**This rule is a prerequisite for #51 and #52**, both of which read backticked filenames out of
an index. Migrate an index before relying on either.

---

## 12. Entry dates are computed in the config timezone (new in v0.12.0)

Backlog #57. A session's clock is UTC; the user's day is not. An entry written at 8:10 PM ET
was being dated by UTC and came out a day late — measurably, on four entries in one evening.

- **The clock source is a shell command executed at write time**, converted to the config's
  `timezone` field (default `America/New_York`). Compute the date; do not read it off anything
  that was handed to you.
- **The `<env>` date is explicitly NOT a clock source.** It has been measured drifting by
  **18 days**. Never date an entry from it, and never sanity-check a computed date against it.
- **No-shell fallback**, for the devices where the sandbox reports no shell: use the FIRST
  commit's server timestamp from this session's writes, in UTC, converted to the config
  timezone. It is the only authoritative clock reachable without a shell. Say once, in one
  line, that the fallback is in use.
- **Daylight saving is why the offset cannot be hardcoded.** `America/New_York` is UTC−5 or
  UTC−4 depending on the date; a fixed offset is right for half the year. Convert with the
  zone name, never with a number.
- **Do NOT retro-correct existing entries.** Dates already written stay as written; the
  append-only rule outranks the fix. A correction, if one is warranted, is a new dated entry or
  a note in the file's header block — never an edit to history.

---

## 13. Reporting rules for every check in this file

Every check here is a **check, not a repair**. Report it and let the user decide. Never
silently fix, never silently rebuild an index, never delete a row because it looks orphaned,
and never quietly drop a finding because acting on it would be inconvenient.

**No silent caps.** Where a skill bounds what it reports — a top-N dashboard, a truncated
list — it says what was left out and how to see the rest. Silent truncation reads as "that was
everything" when it was not.

---

## 14. The clone preflight — a session that cannot clone says so BEFORE it starts

**Some of this plugin's work is a poor fit for the GitHub connector, and the difference is not
speed.** Renaming a folder re-creates every file in it, one SHA-locked write at a time, so a
session must reason about commit ORDER to avoid leaving the repo transiently inconsistent — a
problem one real commit does not have. Rebuilding the master list reads every project folder in
the repo. And two defect classes exist only on the connector path: the transport DECODES UNICODE
ESCAPES, and an oversized read SPILLS TO A SAVED PATH that a sandboxed session cannot reach.

**So a git clone is the DEFAULT path for work that is not provably small.** ⚠️ **A default is
not an obligation, and the difference is the rule: the connector REMAINS AVAILABLE. What changes
is that reaching for it needs a STATED REASON.** There is no requirement to always clone; there
is a requirement to say why not.

**THE TEST IS ONE TEST: escalate unless the work is PROVABLY SMALL.** There are exactly two ways
work fails to be provably small:

- **(a) You can SEE that it is not** — the work touches more than a couple of files.
- **(b) You cannot see EITHER WAY** — the work lists the repo root, iterates project folders, or
  otherwise cannot know its file count until it looks.

⚠️ **LIMB (b) EXISTS BECAUSE LIMB (a)'s INPUT IS MISSING AT THE MOMENT THE DECISION HAS TO BE
MADE. A survey cannot count its files without performing the read that IS the decision.**
Therefore **limb (b) fires BEFORE THE FIRST REPO READ of that work — not before the first WRITE,
and not once the scope is known.** The two limbs do not share a clock: (a) can fire at any point
once scope is known, (b) must fire before the first read or it cannot fire at all.

⚠️ **"OF THAT WORK" IS THE WHOLE CLOCK, AND A READ EARLIER IN THE SESSION DOES NOT DISCHARGE THE
GATE.** Read it as *the session's* first repo read and **the gate is dead in exactly the sessions
that matter most**: `project-open` is the normal first action in a project session and it reads
the repo, so by the time anyone types `plu`, `pa` or `pua` the session's own first read is long
past and the trigger moment would already have gone. **It has not. Each carrier's gate fires
before THAT COMMAND'S first repo read, however many reads came before it in the session.**

⚠️ **DO NOT CONFUSE THAT WITH THE CALLER'S RULING BELOW, WHICH IS SESSION-SCOPED AND IS A
DIFFERENT THING.** The QUESTION is asked once per unit of escalating work. The ANSWER, once the
caller gives it, stands for the rest of the session. **A session that never asked has no ruling
to stand on**, so a previous command's reads — or a previous command's silence — discharge
nothing.

### How a session decides whether it can make a clone — and this needs ZERO repo reads

The question is a property of the ENVIRONMENT, not of the repo, so it is answerable before
anything is read. All three must hold:

1. **A shell is reachable at all.** Claude Code on the user's own machine has one; Cowork with
   the user's desktop app connected has one (the device bridge runs commands there). A Cowork
   session on a phone, or any session whose only tools are connectors, does not.
2. **`git --version` succeeds in that shell.**
3. **That shell reaches GitHub WITH CREDENTIALS.** `gh auth status` reporting an authenticated
   account is the cheapest evidence; where `gh` is absent, `git ls-remote` against a repo the
   user names answers the same question. A data repo is normally PRIVATE, so reaching
   `api.github.com` is not on its own enough — the credential half is the half that decides.

⚠️ **DO NOT INFER THE ANSWER — RUN THE COMMANDS.** A sandboxed shell is reachable and still
cannot clone: `api.github.com` returns `000` and DNS fails there, while the same account on the
user's own machine fetches, pushes and clones normally. **The boundary is credentials and
egress, not a property of being a session**, and sessions have concluded the wall was absolute
after testing only from inside it. Equally, a session that merely LACKS a clone is not a session
that CANNOT MAKE one — the first is usually fixable in seconds.

### When the answer is NO

**Stop before the first repo read of that work, and say so.** Name, in a few lines:

- **the work**, and which limb fired — that it touches more than a couple of files, or that its
  file count cannot be known until it looks;
- **that this session cannot make a clone**, and which of the three checks above failed;
- **that the work wants a session that can** — Claude Code on the user's machine, or Cowork with
  the desktop app connected.

**Then hand the decision to the caller and WAIT.** ⚠️ **THERE IS NO FALLBACK IN THIS RULE.** Do
not proceed over the connector anyway, do not narrow the work to fit the path you have, and do
not offer a degraded version as though it were the same thing. **Stopping without handing the
decision over is just a slower version of deciding alone** — so the stop ends with a question,
not an announcement.

**If the caller says to go ahead over the connector, go ahead.** That is the caller deciding,
which is the entire point, and it is not a fallback the session chose. **The ruling stands for
the rest of the session** — ask once, not again at every step.

⚠️ **THIS IS NOT THE RCLONE PREFLIGHT, AND THE DIFFERENCE IS THE ENDING.** That preflight is
where this one's SHAPE and CLOCK come from — *"before the first rclone-mode Drive step of a
session, confirm such a shell is reachable… if not, say so in one line and fall back"* — and its
fallback is CORRECT there, because Drive has a second path that works. **It is not adopted here.**
A multi-file repo edit over the connector is not a degraded path; it is a different and worse one.

### Which commands carry this, and WHERE the pointer sits

**The pointer sits at the step where the not-provably-small work BEGINS** — which is not the top
of every file, and is never before the first write.

| Command | Where its pointer sits | Why there |
|---|---|---|
| `project-list-update` (plu) | above the config read | step 1 lists the repo root and step 2 iterates every project folder; the whole command IS the survey |
| `project-archive` (pa) | above the config read | step 3 re-creates every file in a folder and step 5 runs the full `plu` survey |
| `project-unarchive` (pua) | above the config read | the same two, mirrored |
| `project-list` (pl) | at its rebuild branch | `pl` is a one-file cached read and IS provably small — until the index is missing, when that branch becomes the full survey |

⚠️ **`pl`'s LATER PLACEMENT IS THE RULE WORKING, NOT AN EXCEPTION TO IT.** A top-of-file gate
there would stop the fast cached read the rule explicitly permits. `pl`'s escalating work begins
at its rebuild branch, and that is where its clock starts — the read that is the decision has
still not happened when the gate fires.

⚠️ **A CARRIER'S GATE IS UNCONDITIONAL, AND KNOWING THE REPO IS SMALL DOES NOT DISCHARGE IT.**
The *"Why there"* column above is a REASON, not a premise for a run to re-test. **A session that
believes the repo holds two projects has not LOOKED — and limb (b) is precisely *"cannot know its
file count until it looks."*** `plu` exists to rebuild an index **because that index may be
wrong**, so the cached belief a run would reason from is the very thing the command replaces.

⚠️ **THE CONTRAST WITH `pn` BELOW IS NOT REPO SIZE — IT IS WHERE THE COUNT COMES FROM.** `pn`'s
two files are fixed by its own procedure before it starts, and no state in the repo can change
that number. **A carrier's count is a DISCOVERED fact.** So: if a carrier's entry should change,
**change this table** — never discharge the gate at run time by arguing the repo looks small.

**NOT carriers — recorded so the decision can be RE-MADE rather than inherited:**

- **`project-new` (pn)** — its step 1 lists the repo root, which is limb (b)'s literal wording,
  but that is an existence probe on ONE name and `pn` knows its file count before it starts: two
  files, from two templates. Provably small. ⚠️ **This is the closest call on the page. Read
  limb (b)'s three clauses as independent triggers rather than as examples of *"you cannot see
  either way"*, and `pn` joins the roster.**
- **`project-setup-init` (psi)** — seeds a statically-known list of files as ONE atomic commit,
  so the ordering hazard the clone exists to remove does not arise; and `psi` may be creating the
  repo, in which case there is nothing yet to clone.
- **`project-update` (upd)** — a fixed, small set of SHA-locked targeted writes. That per-file
  lock catching a concurrent change is the connector's GENUINE advantage, not a concession to it.
- **`project-open` (po), `project`, `project-list-archive` (pla)** — targeted reads of named
  files. `po` lists the repo root only as a fallback when an exact path misses.

⚠️ **THE AUDIENCE PROBLEM, RECORDED BECAUSE IT BOUNDS WHAT THIS SECTION CAN DO: NOTHING HERE
REACHES WHOEVER SPAWNS THE SESSION.** Every hook in this plugin — this preflight, the rclone
preflight, `project-open`'s mandatory first action — runs inside an ALREADY-TYPED session, so for
most readers the environment was fixed before they read a word of it. **That is precisely why
this is a REQUIREMENT rather than a suggestion: escalation is the only mechanism left.**
