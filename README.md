# fa-vibe

Claude Code plugin for **Finansavisen journalists** building small web apps.

The journalist describes what they want; the agent starts from an approved
template, keeps the app on-brand, and gets it onto the FA app platform at
`*.apps.journalistboost.ai`.

## Install

```
/plugin marketplace add startsiden/fa-vibe-plugin
/plugin install fa-vibe@fa-vibe
```

Then just say what you want to build.

## What's in it

| Skill | For |
|---|---|
| `new-project` | Start a new app from a template |
| `tools-init` | Check Node / pnpm / git and get it running locally |
| `new-page` | Add a page |
| `theme` | Colours and styling, within the FA identity |
| `add-react` | Add interactivity, only where it's genuinely needed |
| `add-database` | Store data that survives a redeploy |
| `add-zephr-header` | The real Finansavisen header |
| `rewrite-existing` | Bring an existing project onto the FA stack |
| `save` | Save and publish changes |
| `publish` | Get it live |

## Templates

Registered in `skills/new-project/SKILL.md`. Today there's one — `fa-astro`
([vibecode-template](https://github.com/startsiden/vibecode-template)). Adding
another is a row in that table, not a fork of the plugin.

## Notes for maintainers

Skills are journalist-facing: plain English, no jargon, no stack traces.
Operator tooling — the Coolify API, deploy keys, the box itself — deliberately
lives elsewhere. A token that can deploy and delete every app does not belong
in a plugin journalists install.

Local testing:

```bash
claude --plugin-dir ./plugins/fa-vibe
claude plugin validate ./plugins/fa-vibe
```

Changes reach journalists when they run `/plugin marketplace update`. Bump
`version` in `plugins/fa-vibe/.claude-plugin/plugin.json` so they're offered it.
