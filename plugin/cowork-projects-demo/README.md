# cowork-projects-demo (read-only)

A try-before-you-install build of the `cowork-projects` plugin. It reads the public example repo
`David-RTP/cowork-projects-examples` - you never have to name a repo - and it never writes anything.

## What you can do
| Say | What happens |
|---|---|
| `pl` | List the example projects and notebooks |
| `po Dirt Bike` | Open one: the full dashboard - status, open tasks, open questions, latest changelog, companion files |
| `p` | The session hub: see what is open, switch to another example |
| `pla` | List archived examples (there may be none) |
| `upd` | PREVIEW of the update command: shows what would be committed to the memory doc and notes file, and commits nothing |

Anything that writes - new projects, archive, unarchive, setup - is in the full plugin only.

## Requirements
- The GitHub connector enabled (Settings > Connectors). The example repo is public, so no access
  to it needs to be granted.
- Code execution enabled.

## Moving to the full plugin
Uninstall this demo first - both plugins answer to `po`, `pl` and `upd`. Then install
`cowork-projects` and run `/project-setup-init` (psi) to set up your own data repo. To start from the
examples, use the example repo's **Use this template** button and set `owner` and `repo` in your
copy's `Project Setup/cowork-projects.config.json` to your own.
