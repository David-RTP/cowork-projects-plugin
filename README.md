# cowork-projects

A plugin that gives Claude - in Cowork or Claude Code - a memory for long-running work. Every
project gets a folder in **your own GitHub repo** with a memory doc (decisions, open questions,
changelog) and a notes file (status, tasks, session notes). `po` opens a project with a dashboard;
`upd` saves the session back to both files. Context survives across sessions, devices and model
changes, and git history is the audit trail.

The plugin ships no storage and has no server: your data stays in your repo.

## Try it

Pick the path that fits. All three need Claude with the **GitHub connector** and **code execution**
enabled (Settings).

| Path | Effort | What you get |
|---|---|---|
| **1. Demo plugin** | Install one zip | Read-only: open the example projects, see the dashboards, preview `upd`. You never name a repo |
| **2. Full plugin, pointed at the examples** | Install one zip + one line in your Claude preferences | The real commands reading the example repo. Writes fail - it is not your repo |
| **3. Full plugin on your own copy** | Install one zip + copy a repo + edit one file | Everything, including `upd`, on your own copy of the examples |

**1. Demo plugin.** Download `cowork_projects_demo_v<version>.zip` from
[Releases](https://github.com/David-RTP/cowork-projects-plugin/releases), upload it in Cowork
(Customize > Plugins), open a new session and say `pl`, then `po Dirt Bike`.

**2. Full plugin, pointed at the examples.** Install `cowork_projects_v<version>.zip`, then add this
line to Claude's personal preferences (Settings > Profile) so every session uses the example repo
without asking:

> When using the cowork-projects plugin, my projects data repo is `David-RTP/cowork-projects-examples`.

Delete the line when you set up your own repo.

**3. Your own copy.** On
[cowork-projects-examples](https://github.com/David-RTP/cowork-projects-examples), click **Use this
template** (or fork it). In your copy, edit `Project Setup/cowork-projects.config.json` and set
`owner` and `repo` to your own - **otherwise your copy still points at the original**. Install
`cowork_projects_v<version>.zip`; the commands find your copy on their own.

Starting fresh instead? Install the full plugin and run `/project-setup-init` (`psi`) - it sets up
any repo you choose.

**Install one plugin at a time.** The demo and the full plugin answer to the same commands; uninstall
the demo before installing the full plugin.

## What's in this repo

| Path | What it is |
|---|---|
| `plugin/cowork-projects/` | The full plugin: 10 commands, shared conventions, templates and guides |
| `plugin/cowork-projects-demo/` | The read-only demo plugin |
| `.claude-plugin/marketplace.json` | Lets Claude Code install either plugin straight from this repo |
| `INSTALL.md` | Click-by-click install for Cowork and Claude Code |

The plugin's own README (`plugin/cowork-projects/README.md`) explains every command and how the
memory system works.

## Examples

[cowork-projects-examples](https://github.com/David-RTP/cowork-projects-examples) holds real
projects kept with this plugin, privatized before publishing.

## License

MIT - see `LICENSE`.
