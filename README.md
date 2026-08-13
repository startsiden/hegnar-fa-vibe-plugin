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

Then start your app with:

```
/fa-vibe:new-project
```

**Type that command.** Don't just describe what you want — see below.

## What's in it

| Skill | For |
|---|---|
| `working-with-journalists` | Tone, vocabulary, and the platform rules that hold for every template |
| `new-project` | Start a new app from a template |
| `add-database` | Store data that survives a redeploy |
| `publish` | Decides the route: JB app self-serve, or FA app via DevOps |
| `deploy` | Put a JB app live, and push updates to it |
| `coolify` | What the platform actually is — Coolify's model, API and quirks |
| `zephr-to-jb` | Convert an app built for finansavisen.no so it runs on JB |

Stack-specific skills — pages, styling, interactivity, the Zephr header,
local tooling, saving — stay in each **template's** own `skills/` directory, because
they only make sense for that stack. The plugin holds what's true regardless
of which template an app came from.

## Templates

Registered in `skills/new-project/SKILL.md`. Today there's one — `fa-astro`
([vibecode-template](https://github.com/startsiden/vibecode-template)). Adding
another is a row in that table, not a fork of the plugin.

## Start with the command, not a description

Skills are *model-invoked*: Claude decides whether a description matches what
you asked. Measured behaviour (print mode, `--plugin-dir`, `Skill` allowed):

| Prompt | Result |
|---|---|
| `/fa-vibe:new-project I want a stock movers tool` | ✅ asks JB vs FA vs backend, then the bootstrap questions |
| `I want to build a little tool that shows which stocks moved the most today` | ❌ wrote a Python CLI using yfinance — wrong stack, wrong platform, undeployable |

The second was retried with a `MUST be used before writing any code…`
description and behaved the same. So **the slash command is the contract**, and
the failure is quiet: you get a confident, working, completely unusable app.

Once a project exists this matters less — the template's `AGENTS.md` is read
automatically and pins the stack. It's the *first* step, before any repo exists,
that depends on the journalist typing the command. Put it in the onboarding
message, not just here.

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
