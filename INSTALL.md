# Installing the cowork-projects Cowork plugin

Click-by-click steps. Install once from a computer (claude.ai web or the Claude desktop app);
afterward the commands work in any Cowork session, including on your phone.

## 1. Prerequisites (one-time, in Settings)
- **GitHub connector** enabled at the account level (Settings > Connectors) - the plugin reads/writes
  YOUR projects data repo through it.
- **Code execution** enabled (required for custom skills/plugins).
- Google Drive connector optional (only used for rich deliverables).

## 2. Get the plugin file
Download the zip from this repo's **Releases** page (`David-RTP/cowork-projects-plugin` > Releases,
newest release). Each release carries two:

| File | What it is |
|---|---|
| `cowork_projects_v<version>.zip` | The full plugin. Works on YOUR OWN data repo; every command, including writes |
| `cowork_projects_demo_v<version>.zip` | A read-only demo, built in to the public example repo `David-RTP/cowork-projects-examples`. No setup, never writes. Try this first if you only want to look |

Each zip bundles this INSTALL.md and the Claude Code `marketplace.json`, so it is self-contained for
either client.
- IMPORTANT: upload the release zip itself. Do NOT upload GitHub's whole-repo "Download ZIP" - its
  `.claude-plugin/plugin.json` is not at the archive root, so the upload fails.
- Install one or the other, not both: both answer to `po`, `pl` and `upd`. Uninstall the demo before
  installing the full plugin.

## 3. Install
1. Open Claude on a computer and go to the **Cowork** tab.
2. In the left sidebar, open **Customize** > **Plugins** tab.
3. Upload/install the `cowork_projects_v<version>.zip` file (or the demo zip) and confirm when
   prompted. (Re-uploading a newer version replaces the installed one.) Open a NEW chat to see
   command changes.
4. **Demo installed?** Skip sections 4 and 5 below. In a new session say `pl` to list the example
   projects, then `po Dirt Bike` (or any name from the list) to open one, and `upd` to preview an
   update.

## 3b. Installing in Claude Code (CLI) instead of Cowork
Claude Code has no plugin-upload UI - it installs from a *marketplace* (a directory whose
`.claude-plugin/` contains `marketplace.json`; the plugin bundles one since v0.9.1).

1. Unzip `cowork_projects_v<version>.zip` to a folder, e.g. `~/plugins/cowork_projects_v<version>/`.
2. In Claude Code:

       /plugin marketplace add ~/plugins/cowork_projects_v<version>
       /plugin install cowork-projects@cowork-projects-mkt
       /reload-plugins

   For the demo zip, unzip `cowork_projects_demo_v<version>.zip` to its own folder and use:

       /plugin marketplace add ~/plugins/cowork_projects_demo_v<version>
       /plugin install cowork-projects-demo@cowork-projects-demo-mkt
       /reload-plugins

   ⚠️ Pass the directory that CONTAINS `.claude-plugin` - never `.claude-plugin` itself and never
   the `.zip` file. Both mistakes produce confusing "Marketplace not found" errors.
3. Skills are addressable as `/cowork-projects:project-list` etc., and the short aliases (pl, po,
   upd, pn, ...) trigger from plain text - no slash needed.
4. Upgrading: uninstall the old version FIRST (two marketplaces offering the same plugin name
   collide):

       /plugin uninstall cowork-projects@cowork-projects-mkt
       /plugin marketplace remove cowork-projects-mkt

   then add/install the new folder.

More gotchas seen in the field:
- **`/plugin` is not available over Remote Control** - run the commands directly on the machine.
- **If the "Add Marketplace" dialog opens, enter ONLY the source** (`David-RTP/cowork-projects-plugin`,
  or a local folder path) - typing the full `/plugin marketplace add ...` command into the dialog
  makes Claude Code treat the whole string as a path ("Path does not exist").

**Direct-from-GitHub alternative** (repo-root marketplace, added v0.9.1):

       /plugin marketplace add David-RTP/cowork-projects-plugin
       /plugin install cowork-projects@cowork-projects

or, for the read-only demo, `/plugin install cowork-projects-demo@cowork-projects`. This route CLONES
the repo, so it requires `git` installed. The repo is public, so no GitHub sign-in is needed for it.
Later, `/plugin marketplace update cowork-projects` picks up new versions. No git -> use the zip
route above.

## 4. First run - bootstrap YOUR data repo
1. Start a new Cowork session and run `/project-setup-init` (or type "psi").
2. Pick an existing GitHub repo or let it create a new **private** one for you. It also resolves your
   Google Drive folder ID for you (you just give the folder name) and asks your timezone.
3. It seeds a `Project Setup/` folder there: the config file every other command discovers, the
   memory-doc template, the project-notes template, the Learning Notebook template, the
   legacy-Instructions template, the Setup Guide, the Migration Runbook, the optional direct-Drive
   (rclone) guide, and an empty master list - plus Project Setup's own memory/notes files.
   (Already initialized from an older version? Re-run psi: it stops at the double-init guard and
   offers to add any config fields and seeded docs you're missing, without overwriting anything.)

## 5. Verify it worked
1. In a NEW session, type `/` and confirm you see the 10 `/project*` commands.
2. Run `/project-new Test Project` (pn), then in another new session `/project-open Test Project` (po)
   and ask: "What's the most recent changelog entry in the memory doc and its date?" A pass fetches
   from GitHub unprompted and reports the scaffold entry. (Archive the test with `pa Test Project`
   when done.)

## 6. Day-to-day use
- One Cowork session per project works well: open it once with `/project-open <Name>` (po) and return
  to that session for that project's work. Running po again refreshes the project dashboard.
- `upd` to commit the session's work to memory + notes (one command does both; run it at least once
  per sitting).
- `pn` new project (it offers chat import + opens the project's session); `pl` list active; `pla` list
  archived; `plu` rebuild the index; `pa`/`pua` archive/unarchive (both rebuild the index automatically).
- Old chats to import? `pn` offers the paste block automatically; the full procedure is in
  `Project Setup/Migration Runbook - Legacy Chat Import.md` in your data repo.
- **Projects vs Learning Notebooks.** `pn` asks which kind you want. A PROJECT has a goal and a finish
  line (dated Session Notes, Current Status). A LEARNING NOTEBOOK is a running reference on a subject
  with no fixed "done" - its notes are organized by TOPIC and edited in place, with only a thin dated
  Learning Log. Both share the same folder shape and memory doc; the memory doc's `**Type:**` line
  records which (missing = Project). Notebooks appear in the same master list marked with a `📓`.
- **Optional - direct Drive access.** The Drive connector can only inline small files and cannot
  delete or rename. If your projects produce large binary deliverables, the plugin can use rclone
  instead (config fields `drive_access` + `rclone_remote`): full-fidelity uploads at any size,
  verified transfers, and real replace/delete. Free, no sudo, one-time setup - the full guide is
  `Optional - Direct Drive Access (rclone).md`, which psi seeds into your data repo's
  `Project Setup/` folder (it also ships in the zip's `docs/` folder). Note that rclone reads a
  config file on ONE MACHINE, so `rclone` mode only takes effect in sessions that can run a shell
  command there - Claude Code on that machine, or Cowork with your desktop app connected. Phone
  sessions fall back to connector behavior and tell you so.

- **Shared conventions (new in v0.11.0; three more rules added in v0.12.0).** Rules that apply to
  more than one command now live in one file, `docs/CONVENTIONS.md`, bundled in the zip: no
  secrets in the data repo, SHA-locked writes, the size rule, how to split
  a file, append rules by section type, the table lint, dated state claims, computed counts,
  reconciliation as a check not a repair, never guessing at an unknown, index markup, entry dates
  computed in your configured timezone, and the reporting rules for every check. Every writing
  skill points at it rather than carrying its own copy. This is a plugin file, not a data-repo
  file - psi does not seed it and you do not need to do anything with it; it is listed here so you
  know where the rules actually live if a command's behavior surprises you.
- **Writes are protected as of v0.11.0.** Earlier versions committed with a call that carried no blob
  SHA, so two sessions updating the same project could silently overwrite each other with no error.
  Updates now lock to the blob SHA that was read and are REJECTED if the file moved underneath - the
  command re-reads and re-applies instead of clobbering. If you have been running two sessions on one
  project, this is the upgrade worth taking.

## 7. Updating the plugin later
A reinstall is ONLY needed when COMMANDS change. When a new release is published, download its
zip and upload it the same way - it replaces the installed version. (Claude Code users on the
direct-from-GitHub route run `/plugin marketplace update cowork-projects` instead.)
Your memory-doc template lives in YOUR data repo and is read live - section/format changes there never
require a plugin update.

## Known install gotchas
- **Zip structure:** `plugin.json` not at the archive root fails with "Zip must contain a
  .claude-plugin/plugin.json file...". Use the release zip, not GitHub's "Download ZIP".
- **XML in descriptions:** a SKILL.md `description` cannot contain angle brackets (the validator reads
  them as XML tags). Skill bodies may use `<placeholders>`; descriptions may not.
- **Description length:** the plugin.json `description` must be at most 500 characters or the upload
  fails validation ("Plugin description must be at most 500 characters").
- Install from a **computer**; phone upload of plugin files is unverified.
- Connectors are account-level: no per-project linking is ever needed.
- **Google Drive limits:** the standard Drive connector can search/read/copy/create but NOT
  rename/move/delete - the plugin degrades those steps to "reported as pending" rather than faking them.
