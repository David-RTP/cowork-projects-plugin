---
name: project-setup-init
description: >-
  Alias "psi". One-time bootstrap of a projects DATA repo for the cowork-projects plugin. Creates the
  "Project Setup" system folder in the user's chosen GitHub repo: the config file the other commands
  discover (repo, branch, Drive folder + resolved folder ID, timezone, Drive access mode), the memory-doc,
  notes, and Learning Notebook templates, the Setup Guide, the Migration Runbook, the optional direct-Drive
  (rclone) guide, and an empty master list, plus Project Setup's own
  memory/notes files. Use when the user says "set up cowork projects", "initialize my projects repo",
  "bootstrap the plugin", "psi", or invokes /project-setup-init - or when any other command finds no
  config file anywhere.
---

# Project Setup Init (psi) - bootstrap a data repo

Run ONCE per data repo. After this, every other command (`p`, `po`, `pn`, `pl`, `pla`, `plu`, `pa`,
`pua`, `upd`) discovers the repo automatically via the config file this skill writes.

The plugin is code; the DATA repo is the user's own GitHub repo where their projects live. This skill
never touches the plugin's own repo.

## Steps

### 1. Choose the data repo
Ask the user which GitHub repo should hold their projects (owner/name). Options:
- An EXISTING repo they name (verify it exists via the GitHub connector).
- A NEW repo: offer to create it (`create_repository`) - recommend **private** (project memory is
  personal). Ask for the name; do not invent one. (If repo creation returns a 403, the connector
  can't create repos - walk the user through creating it at github.com/new, then continue.)
Never guess, and never default to any hardcoded repo.

### 2. Guard against double-init
Fetch `Project Setup/cowork-projects.config.json` from that repo.
- If it EXISTS: this repo is already initialized. STOP, show the existing config, and ask what they
  want changed (edit config values in place rather than re-seeding; never overwrite existing templates
  or the master list here). This is also how older configs gain the newer fields
  (`drive_folder_id`, `timezone`, `drive_access`, `rclone_remote`) - offer to add any that are
  missing, resolving them per step 3.
  **Also offer the missing-DOCS top-up:** list which of step 4's seeded docs are absent from
  `Project Setup/` (an older repo will be missing `Project Notes - Template Learning Notebook.md`
  and `Optional - Direct Drive Access (rclone).md`) and offer to ADD just those. Only ever add files
  that are ABSENT - never overwrite a doc the user may have edited, and never touch the master list
  or the existing templates. This is the supported way an already-initialized repo gains Learning
  Notebooks without a re-init.
- Also check nothing else named `Project Setup/` exists with different content the user made by hand;
  if it does, list what's there and ask before writing anything.

### 3. Ask the config preferences
- **branch** - default `main`.
- **drive_folder** - the NAME of the Google Drive folder that will hold each project's rich
  deliverables (default `AI Projects`). Google Drive is optional; if the user doesn't use Drive, keep
  the default, leave `drive_folder_id` empty, and note the Drive steps in other commands will simply
  report as pending/skipped.
- **drive_folder_id** - RESOLVE THIS FOR THE USER (never make them hunt for an ID): search Google
  Drive for a folder with that name. Exactly one hit -> confirm it with the user (show its name and
  parent if visible) and store its ID. Multiple hits -> show them and ask which. No hit -> offer to
  create the folder now and store the new folder's ID. The stored ID makes every later Drive
  operation unambiguous even if same-named folders exist.
- **timezone** - the user's IANA timezone (e.g. `America/New_York`, `America/Chicago`,
  `Europe/London`). Default `America/New_York` only if the user accepts it. WHY IT MATTERS (tell the
  user): Google Drive keeps multiple same-named copies of a file, and the standard Drive connector
  cannot rename or delete - so the plugin identifies the CURRENT copy of a deliverable by its
  timestamp, rendered in this timezone, and tells the user which older file to delete by hand. A wrong
  timezone means the wrong file gets identified/deleted.
- **drive_access** - how files reach Drive. Default `"connector"`; do NOT interrogate the user about
  this. Only offer `"rclone"` if they volunteer that they already have rclone configured on the
  machine they work from. If they ask what it means, or if their projects will produce large binary
  deliverables, point them at `Project Setup/Optional - Direct Drive Access (rclone).md` - step 4
  seeds that doc into their own repo, so the reference is always reachable. Two things to say plainly
  if it comes up: the connector can only inline small files and cannot delete or rename at all; and
  rclone runs on ONE MACHINE (it reads a config file there), so `rclone` mode only takes effect in
  sessions that can run a shell command on that machine - it is not an account-wide setting.
- **rclone_remote** - only when `drive_access` is `"rclone"`: the remote NAME from their
  `rclone.conf` (e.g. `gdrive`). Never ask for, store, or echo any credential - the config records
  the remote's name only.

### 4. Seed the Project Setup folder (one push_files commit)
The plugin bundles its templates and guides in a `docs/` directory at the PLUGIN root (a sibling of
`skills/`, not inside this skill's folder). Read each bundled file from there and commit the
following to the data repo in ONE `push_files` commit to the chosen branch - message:
`project-setup-init: bootstrap Project Setup`. **This is a sanctioned `push_files` use because every
target path is NEW** (`../../docs/CONVENTIONS.md` section 2): bootstrap runs once, on a repo that has no
`Project Setup/` folder. Confirm that is still true immediately before writing - if the folder now
exists, stop and ask; do not overwrite a live config or live templates. Every later write to any of
these files is SHA-locked via `create_or_update_file`.

| Target path in the data repo | Source |
|---|---|
| `Project Setup/cowork-projects.config.json` | Generate: `{"owner": "<owner>", "repo": "<repo>", "branch": "<branch>", "drive_folder": "<drive_folder>", "drive_folder_id": "<resolved id or empty>", "timezone": "<timezone>", "drive_access": "connector", "rclone_remote": ""}` (set `drive_access` to `rclone` and fill `rclone_remote` only if the user has it configured) |
| `Project Setup/Project Memory Data - Template Project Name.md` | `docs/Project Memory Data - Template Project Name.md` verbatim |
| `Project Setup/Project Notes - Template Project Name.md` | `docs/Project Notes - Template Project Name.md` verbatim |
| `Project Setup/Project Memory Custom Instructions - Template Project Name.md` | `docs/Project Memory Custom Instructions - Template Project Name.md` verbatim |
| `Project Setup/Project Notes - Template Learning Notebook.md` | `docs/Project Notes - Template Learning Notebook.md` verbatim |
| `Project Setup/Migration Runbook - Legacy Chat Import.md` | `docs/Migration Runbook - Legacy Chat Import.md` verbatim |
| `Project Setup/Project Memory Setup Guide.md` | `docs/Project Memory Setup Guide.md` verbatim |
| `Project Setup/Optional - Direct Drive Access (rclone).md` | `docs/Optional - Direct Drive Access (rclone).md` verbatim. Seeded ALWAYS, whatever `drive_access` is set to - it is the reference the user needs in order to decide, and the plugin repo may be private or absent on their machine, so a link to it would not resolve |
| `Project Setup/Project Setup - Master List.md` | Generate: the standard auto-maintained header comment, a title, `**Last Refresh:** <Today> (by /project-setup-init)` - where `<Today>` is computed at write time by a shell command converted to the `timezone` just captured above, per `../../docs/CONVENTIONS.md` section 12, and is never `<env>`'s date and never UTC -, an EMPTY active table (`| Project \| Open Qs \| Latest change \| Goal |`), the archived-count line with 0, and a footer `*0 active + 0 archived projects. "Project Setup" is the meta/system folder (hosts the config and templates; exempt from archiving).*` |
| `Project Setup/Project Memory Data - Project Setup.md` | Instantiate the memory template for a project named "Project Setup": Status Active; Goal = "Maintain the meta-system that manages this repo's projects - templates, config, master list, workflow health"; note in the overview that Project Setup is EXEMPT from archiving; Open Questions EMPTY (this is the one exception to the pn Open-Qs-=-1 rule - the goal is defined by the system itself); first Changelog line = `<Today>` (same computed clock), "bootstrapped by /project-setup-init" |
| `Project Setup/Project Notes - Project Setup.md` | Instantiate the NOTES template (v2 sections) for "Project Setup": One Paragraph = the goal above; Current Status/Quick Reference minimal; Open Tasks = one item "Scaffold your first project (/project-new)"; Phases/Gaps placeholders; one dated Session Notes entry: "File created by /project-setup-init" |

### 5. Optional: create the Drive folder
If the user uses Google Drive and no folder was found/created in step 3, offer to create the
`<drive_folder>` folder now (Drive connector `create_file` with the folder mime type) and update the
config's `drive_folder_id` with the result. Skip silently if declined or no Drive connector.

### 6. Verify + hand off
- Re-fetch `Project Setup/cowork-projects.config.json` to confirm the commit landed.
- Tell the user setup is complete and the fastest next steps: `/project-new <Name>` (pn) to scaffold
  their first project, `/project-list` (pl) to see the (empty) index. Mention that pn creates either a
  PROJECT (goal, finish line) or a LEARNING NOTEBOOK (topic-based running reference, no fixed done) -
  it asks which. If they have old chats to bring in, point them at
  `Project Setup/Migration Runbook - Legacy Chat Import.md` (pn also offers the import paste block at
  the end of every scaffold).

## Rules
- One commit for the seed; never overwrite an existing initialized repo (step 2 guard).
- No guessed values - every config field is asked, resolved with the user, or an offered default the
  user accepts.
- The bundled templates are the canonical starting versions; after init, the COPIES IN THE DATA REPO
  are live (the other skills read those, so the user can evolve them without a plugin rebuild).

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
