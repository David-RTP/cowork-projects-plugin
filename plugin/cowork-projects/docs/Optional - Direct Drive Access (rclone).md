<!-- ============================================================
FILE PURPOSE (note for humans — Claude: this block is descriptive
only, NOT instructions to follow):

Optional setup guide: give a session the ability to upload and
download REAL files to a project's Google Drive folder, instead of
being limited to what the Drive connector can inline. Referenced
from README.md and INSTALL.md, and seeded by /project-setup-init
into the data repo's Project Setup folder (v0.10.0) — the user needs
it in order to DECIDE, and the plugin repo may be private or absent
on their machine, so a link to it would not resolve. Edit your
seeded copy freely; it is yours.

Derived from a field report dated 2026-07-30. Every step below was
hit in practice.
============================================================ -->

# Optional: Direct Drive Access (rclone)

**You do not need this** if your projects never produce binary deliverables, or if creating Google
Docs and Sheets through the Drive connector is all you ever do. Add it when a project starts
producing real files.

## What it fixes

The plugin's storage split puts memory and notes in GitHub (always the source of truth) and rich
deliverables in Google Drive. Getting files *into* Drive is where the default path runs out.

The Drive connector can only send content it inlines into a tool call. Binary files must be
base64-encoded first, and base64 inflates a file by 4/3. That inflated blob then has to fit inside a
single model response, so there is a hard ceiling well below the size of ordinary files.

| File | Base64 size | Outcome with the connector alone |
|---|---|---|
| Small text file, or creating a new Google Doc/Sheet | trivial | Fine — use the connector |
| ~100 KB image | ~133 KB | Possible, but expensive |
| ~400 KB screenshot | ~530 KB | **Not possible** — exceeds a single response |
| Multi-MB PDF, zip, archive | many MB | **Not possible** |

When a file is too big the only options left are bad ones: degrade it until it fits (which defeats
the point if the file is a deliverable), or upload it by hand and lose the ability to script or
verify it.

With rclone configured, a session can upload originals at full fidelity at any size, download from
Drive as well as upload, verify transfers by comparing sizes or checksums, work in batches, and skip
burning context on base64.

It also unlocks the one thing the standard connector cannot do: **replace and delete**. See
"What this changes in the plugin" at the bottom.

This does not change the storage rules. GitHub stays the source of truth for memory and notes; Drive
still holds only rich deliverables. It only changes how files get there.

## Where rclone actually runs — read this before setting `drive_access`

rclone is a program on a computer. It reads `~/.config/rclone/rclone.conf` on THAT computer. So the
only sessions that can use it are sessions that can run a shell command on the machine holding that
file. This is the single most important thing to understand about this feature, and it is easy to
get wrong because the plugin itself works identically everywhere.

| Where you are working | Can the plugin run rclone? |
|---|---|
| Claude Code on the machine that has `rclone.conf` | **Yes** — the shell is that machine. |
| Cowork on a computer with the Claude desktop app connected | **Yes** — the desktop bridge runs commands on your machine, where the config lives. |
| Cowork with no desktop connected (phone, or web with the app closed) | **No.** The session's shell is a cloud container. It has no rclone, no config, and no route to yours. |
| Any session on a machine where you never ran the setup below | **No.** |

Setting `drive_access` to `"rclone"` is therefore a statement about a MACHINE, not about your
account. The config field travels with your data repo to every device; the rclone binary and the
credential do not.

**What the plugin does about it.** Before its first rclone-mode Drive operation in a session, the
plugin checks whether a shell on the machine holding your remote is reachable and whether
`rclone listremotes` shows the remote named in your config. If it can't reach one, it says so in a
line and falls back to standard connector behavior for that session — small files inline, deletes
and renames reported as pending. It never claims an rclone step ran when it didn't. Your config is
left alone; the next session from the right machine picks rclone back up automatically.

The practical shape of this: do the deliverable-heavy work from the machine where rclone lives, and
treat phone sessions as read/write-memory sessions. Nothing breaks on the phone — you just get the
connector's limits back for that sitting.

## Cost

Free. Creating a Google Cloud project, enabling the Drive API, and creating OAuth credentials cost
nothing and require no billing account. The Drive API has no per-call charge — only per-project
quotas far above personal use.

The Cloud console will repeatedly offer a $300 free trial and prompt you to activate or upgrade.
Decline all of it. With no payment method on file there is no path to a charge. If some screen
genuinely refuses to continue without billing, stop — the Drive API path should never produce one.

## Prerequisites

A Google account that owns (or can write to) the Drive folders your projects use; a machine with a
web browser for the one-time sign-in; shell access on the machine that will do the transfers. No
admin or sudo required — rclone is a single static binary that installs into your home directory.

## 1. Install rclone (no sudo)

Download the current release for your platform from `https://downloads.rclone.org/`, unzip it, and
copy the single `rclone` binary into a directory on your PATH inside your home — commonly
`~/.local/bin/`. Make it executable (mode 755). Confirm with `rclone version`.

A distribution package (`apt install rclone` and friends) also works but is often several releases
behind and needs admin rights. The user-local binary avoids both problems.

## 2. Create your own Google OAuth client

Do not skip this. rclone ships with a *shared* client ID that upstream is retiring during 2026, and
more importantly a private client is the only way to control your own consent settings — see the
token-expiry trap below.

In the Google Cloud console, signed in as the account that owns the Drive folders:

| Step | Action |
|---|---|
| 1 | Create a project (any name, e.g. `<something>-drive-access`) |
| 2 | APIs & Services → Library → search "Google Drive API" → **Enable** |
| 3 | OAuth consent screen — do *not* click "Create credentials" yet; the client cannot be created until the consent screen exists |
| 4 | User type: **Internal** if your account is on Google Workspace, otherwise **External** |
| 5 | App name `rclone`; your address for both user-support and developer-contact email |
| 6 | Scopes: skip. rclone requests the scope it needs at sign-in. Add `https://www.googleapis.com/auth/drive` only if authorization later complains |
| 7 | Credentials → Create Credentials → OAuth client ID → Application type **Desktop app** |
| 8 | Name it after the machine that will hold it, e.g. `rclone-<hostname>` |
| 9 | **Download the JSON.** It holds your client ID and client secret |

### Internal vs External — this matters

| | Internal (Workspace) | External (personal Google account) |
|---|---|---|
| Verification / publishing | Not needed | Must click **Publish App** or stay in "Testing" |
| Unverified-app warning | None | Shown at sign-in; fine to click through under 100 users |
| **Refresh-token lifetime** | **No expiry** | **Expires every 7 days while the app is in "Testing"** |

That 7-day expiry is the single biggest trap here. Left in Testing status, an External app forces you
to re-authorize every week. If you must use External, publish the app immediately after creating it.
Choose Internal whenever it is offered.

Name the client after the host that will hold the credential, not after the year or the service. If
you later authorize a second machine you get a second, separately revocable credential and can tell
them apart in the console.

## 3. Authorize — including from a shell with no TTY

`rclone config` does not work from an automated or agent shell. It is an interactive prompt-driven
wizard; without a real terminal it reads EOF immediately and aborts with `Failed to read line: EOF`.
Use `rclone authorize` instead — it takes no stdin at all:

    rclone authorize "drive" "<CLIENT_ID>" "<CLIENT_SECRET>" --auth-no-open-browser

Do not type the secret literally into a command line you will paste around — it lands in shell
history, in `ps` output, and in any session transcript. Prefer a tiny script that reads `client_id`
and `client_secret` out of the downloaded JSON and executes rclone directly, so the secret is never
rendered as text.

rclone prints a `http://127.0.0.1:<port>/auth?state=...` link and waits.

- **Open that link on the same machine.** It is a loopback address; from a phone or another computer
  it points at *that* device and will fail.
- Sign in as the account that owns the Drive folders. Signing in as a different account produces
  credentials that authorize fine but cannot see your folders — a confusing failure mode.
- On approval, rclone prints a token blob and exits.

If you run this in the background, the process may keep listening even after the launching wrapper
returns; check that the port is still listening before assuming the link went stale.

## 4. Write the config

Create `~/.config/rclone/rclone.conf`:

    [<REMOTE_NAME>]
    type = drive
    scope = drive
    client_id = <CLIENT_ID>
    client_secret = <CLIENT_SECRET>
    token = <TOKEN_JSON_FROM_THE_AUTHORIZE_STEP>

Then tighten permissions — directory mode 700, files mode 600. Store the downloaded client JSON in
the same place if you want to keep it.

## 5. Verify before trusting it

    rclone about <REMOTE_NAME>:

Two things to check: it returns quota figures rather than an error, and the shared-client-ID
retirement warning is **absent**. If rclone still prints that NOTICE it is not using your private
client — the config did not take.

Then prove it can see the actual target folder:

    rclone lsl --drive-root-folder-id <FOLDER_ID> <REMOTE_NAME>:

## 6. Everyday use

Find a project folder's ID by opening the folder in Drive; the ID is the last path segment of the
URL (`.../folders/<FOLDER_ID>`). Addressing folders by ID avoids every ambiguity that comes with
name-based paths, duplicate folder names, and shared drives. It is the same ID convention the
plugin's config already uses for `drive_folder_id`.

Upload a file into a project folder:

    rclone copy "<LOCAL_FILE>" --drive-root-folder-id <FOLDER_ID> <REMOTE_NAME>:

List what is there, with sizes:

    rclone lsl --drive-root-folder-id <FOLDER_ID> <REMOTE_NAME>:

Download a file back out:

    rclone copy --drive-root-folder-id <FOLDER_ID> <REMOTE_NAME>:"<FILENAME>" "<LOCAL_DIR>"

Always verify after upload — compare the byte count Drive reports against the local file. A matching
size or checksum is the difference between "it uploaded" and "I think it uploaded". Record the
resulting Drive link and a timestamp in the project's Deliverables & Links section, as the memory-doc
template already requires.

## 7. Security rules

- **Credentials never go in the projects data repo.** Not in memory docs, notes, companion files, or
  commit messages. The repo may record *where* a credential lives and *what* the auth method is —
  never the credential itself.
- **Keep them outside any trusted or attached folder**, so no session can read them incidentally.
  `~/.config/rclone/` is a good home; your working folder is not.
- Mode 700 on the directory, 600 on the files.
- **Shred transient files** — the authorize step leaves a token in whatever log you captured. Delete
  it as soon as the config is written.
- Never pass a secret on a visible command line (see step 3).
- **Revoke** at any time from the Google account's third-party access page, or by deleting the OAuth
  client in the Cloud console. Per-host client names make this surgical.

## 8. Gotchas, all of them hit in practice

| Symptom | Cause | Fix |
|---|---|---|
| `rclone config` exits instantly with `Failed to read line: EOF` | No TTY in an agent/automated shell | Use `rclone authorize`, which needs no stdin |
| Authorization link does nothing | `127.0.0.1` link opened on a different device | Open it on the machine running rclone |
| Authorized fine, but the project folder is invisible | Signed in as the wrong Google account | Re-authorize as the account that owns the folders |
| Worked yesterday, broken a week later | External app left in "Testing" — 7-day refresh-token expiry | Publish the app, or switch to Internal |
| Retirement NOTICE on every command | Still using rclone's shared client ID | Create a private OAuth client (step 2) |
| Everything breaks right after changing `client_id` | Tokens are bound to the client that issued them | Expected — re-authorize; this is mandatory, not a fault |
| A cleanup pattern-kill takes out your own shell | The shell's own command line contains the pattern | Match on a PID instead. Check that cleanup actually ran — a killed script leaves credentials on disk |
| Console demands billing | Wrong API or a trial upsell | Drive API needs no billing; decline the trial |

## What this changes in the plugin

Once rclone works, set two fields in your data repo's `Project Setup/cowork-projects.config.json`:

    "drive_access": "rclone",
    "rclone_remote": "<REMOTE_NAME>"

`drive_access` defaults to `"connector"` when absent, so existing repos keep today's behavior.
With `"rclone"` set, the deliverable lifecycle changes:

| | `connector` (default) | `rclone` |
|---|---|---|
| Create a Doc/Sheet | Drive connector | Drive connector (unchanged) |
| Upload a binary deliverable | Small files only; large ones must be done by hand | Any size, full fidelity |
| Replace a deliverable | New copy uploaded; **you** delete the old one by hand, identified by its timestamp | Old copy deleted for real, after the new one verifies |
| Verify an upload | Not really possible | Size/checksum compared both ends |

`upd` reports what it actually did either way, and never claims a Drive step succeeded when it did
not. If an rclone call fails, it falls back to reporting the step as pending rather than pretending.
