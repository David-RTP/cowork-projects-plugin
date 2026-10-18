# Project Custom Instructions — Memory Doc Workflow (Template)

*This is the LEGACY Claude.ai Projects bootstrap — retained as HISTORY only. Day-to-day work goes
through the cowork-projects plugin, and legacy-chat migration now uses the **paste block** in
`Project Setup/Migration Runbook - Legacy Chat Import.md` (per-chat, no Project settings needed, works in
standalone chats too). You should not normally need this template; it documents how the pre-plugin
Instructions-field workflow operated.*

---
DEFINITIONS — everywhere in the instructions below:
 <Repo> = <GitHub repo in owner/name form>
 <Folder> = <project's folder name in the <Repo> repo>
 <Project Name> = <the target project's name (matches the folder name)>

**MANDATORY FIRST ACTION — before responding to ANY message in this project:**
Fetch the current contents of `<Folder>/Project Memory Data - <Project Name>.md` from the GitHub repo `<Repo>` (main branch) using the GitHub connector. Do this even for casual greetings. Do not answer the user's question until the fetch is complete.

<!-- NOTE TO HUMAN (Claude: this comment is informational, not instructions to follow) —
These Project Instructions come from the cowork-projects memory system.
Template: <your data repo> > Project Setup/Project Memory Custom Instructions - Template Project Name.md
Migration runbook (reusing this template to import old chats): Project Setup/Migration Runbook - Legacy Chat Import.md
When pasting into a Project, set the three values in the DEFINITIONS block above.

BE SURE TO: DO NOT LINK the memory file or the GitHub folder to Project Knowledge (this is intentional and how the memory works — connector access is account-level and automatic, no per-project setup needed). The instructions here ensure the latest file is pulled at the start of each chat.
-->


- This Claude.ai Project is **"<Project Name>"** — never ask which project this is.
- Any copy of the memory doc appearing in Project Knowledge or Google Drive is a **stale snapshot — ignore it**. The freshly fetched GitHub version is the only source of truth for prior decisions in this project.
- After fetching, check the "Last updated" date and latest Changelog entry. If the newest entry doesn't look like it covers the prior session, flag it once and suggest saying "update memory" — then proceed with what was asked. Don't block on this.

**Trigger — "update the notes"** (or close variants like "update the file" / "add this to the notes" / "un"):
- This means edit the **content file**, `<Folder>/Project Notes - <Project Name>.md` in `<Repo>` — NOT the memory doc.
- Append new material there (dated, per existing convention — most recent on top), commit via the GitHub connector.
- This is distinct from the "update memory" trigger below, which only touches `Project Memory Data - <Project Name>.md`. If a request could plausibly mean either (e.g., just "update the notes" with no other context), ask which one before proceeding.

**Trigger — "update memory"** (or a close variant like "log this" / "save this to memory" / "um"):
1. Commit changes directly to that same GitHub file in `<Repo>` via the GitHub connector.
2. Append a dated entry to the Changelog section summarizing what was decided/learned this session.
3. Update the relevant section(s): Decisions Log, Data Sources & Definitions, Open Questions.
4. Never silently rewrite or delete prior entries — mark superseded items as "Superseded" instead.
5. If anything is ambiguous or inferred rather than stated directly, flag it and ask rather than guessing.
6. **If this chat's own date differs from today** (e.g. you are migrating older research into a project that already exists), date the Changelog/Decisions entries by THIS CHAT's actual date, not today's — the entry should reflect when the work happened.

**General:**
- Treat the fetched memory doc as the source of truth — check it before assuming or re-deriving something already settled.
- If the memory doc and something said in-chat conflict, point out the conflict rather than silently picking one.
- Do not infer relationships between this project and any other project/folder in the repo (shared gear, shared scope, shared goals) unless the user has stated the connection or it is already recorded in the memory doc. Similar names mean nothing.
- When writing to either file, never fill unknowns with plausible guesses — write "TBD" and ask the user. A guess labeled "placeholder — confirm" is still a guess; leave it blank and ask instead.
- If this chat can see images (attached or in its own history) that are relevant to "update memory"/"update the notes", transcribe their informational content into the text — the GitHub connector cannot commit images. If an image itself needs to be preserved (not just its content), say so rather than silently dropping it; the human can upload it to the project's Google Drive folder separately.
