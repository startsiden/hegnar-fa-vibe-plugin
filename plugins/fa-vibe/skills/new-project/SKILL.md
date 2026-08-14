---
description: MUST be used before writing any code for a new Finansavisen app or tool. Triggers on any request to build, make, create, or start an app, tool, page, dashboard, tracker, or widget — however small it sounds. Also when the user describes an idea and there is no project on disk yet. Without it the wrong stack gets chosen and the result cannot be deployed to Finansavisen at all.
---

# Start a new Finansavisen app

The person you're working with is a **journalist, not a developer**. They speak product, not code. Never show them a stack trace, never ask them to pick a framework, and never explain git.

## Which kind of app — decide this first

Three paths. They differ in **who sees the app** and **where it runs**, and that
decision is hard to reverse later, so get it right before cloning anything.

| Path | Who sees it | Where it runs | Template |
|---|---|---|---|
| **JB app** | Journalists only, signed into JournalistBoost | `<name>.apps.journalistboost.ai` — self-serve Coolify, live in minutes | `startsiden/vibecode-template` |
| **FA app** | Finansavisen readers, public or paywalled | `finansavisen.no/<path>` on OKD — **needs a DevOps deployment**, not self-serve | `startsiden/vibecode-template` |
| **Backend app** | Nothing visual — an API or service | Not yet defined | ⚠️ **Not available yet** |

### Just ask

> "Where should this live?
> 1. **Inside JournalistBoost** — for the newsroom, and I can put it live myself
> 2. **On finansavisen.no** — for readers, and the platform team deploys it
> 3. **Neither — it's a backend service** with no pages"

Take the answer at face value; they know which system they're building for.

- **1 → JB app.** The good case: self-serve, live in minutes, already behind the
  JB login. If they hesitate or say "not sure", pick this one — it's cheap and
  reversible, and an FA app can be made from it later.
- **2 → FA app.** Say up front: *"That one lives on finansavisen.no, so the
  platform team has to put it live — it's not instant, and there's a review."*
- **3 → Backend app. Stop.** The NestJS template doesn't exist yet. Tell them:
  *"That's not something I can start yet — the team is still setting up the
  template for it. Worth asking them directly."* Don't improvise a backend from
  scratch.

### What differs beyond the URL

| | JB app | FA app |
|---|---|---|
| Login | JournalistBoost, automatic — `AUTH_MODE=jb` | Zephr / paywall at the CDN edge — `AUTH_MODE=zephr` |
| The FA header | Don't add it — Zephr isn't in front of this domain and the markers stay bare | `add-zephr-header` skill |
| Publishing | Self-serve, minutes | DevOps ticket, wait |
| Served at | A subdomain root | A path like `/verktoy/<name>`, so the build needs a base path |

**Never mix them up.** Copying Zephr cookie checks into a JB app locks out every
journalist — that cookie doesn't exist on `journalistboost.ai`. Building a JB-style
app for readers leaves it behind the wrong login.

### Worked example of both

`startsiden/hegnar-fa-stock-monitor` is the same app built both ways, so the
difference is a diff you can read:

- `master` → **FA app**: Zephr/`blaize_session` auth, `allowedDomains` for
  `**.finansavisen.no`, built with `--base=/verktoy/stock-monitor`, deployed to
  OKD by DevOps.
- `feat/fa-apps-platform` → **JB app**: Zephr auth deleted, JB `check-session`
  instead, `allowedDomains` for `**.journalistboost.ai`, no base path, deployed
  to Coolify.

Converting between them touches exactly four things: the auth middleware, the
`allowedDomains` list, the build's base path, and the deploy target. If you find
yourself changing more than that, stop and re-read the diff.

If what they describe fits none of the three, say so and stop. Ask them to check
with the platform team rather than improvising a stack — an app off-template
can't be deployed by the normal route.

## Step 1 — Understand what they want

Ask, in one message, in plain language:

> "Three quick things:
> 1. **What should it do?** One or two sentences is plenty.
> 2. **What should we call it?** Lowercase with dashes, e.g. `market-tracker`.
> 3. **Do you have a GitHub repo and access token from the team?** The token starts with `ghp_` or `github_pat_`."

If they don't have the repo or token, tell them to ask whoever set them up. Don't proceed without both — the first save will fail and they won't understand why.

From their answer, choose the template. Say which you picked and why, in one sentence.

## Step 2 — Clone and detach

```bash
git clone <template repo> <project-name>
cd <project-name>
git remote remove origin
git remote add origin "<their repo URL>"
```

**Detaching matters.** A fresh clone still points at the shared template. A push would either be rejected or damage the template for every other journalist.

## Step 3 — Credentials, never in the code

```bash
cp .env.example .env
```

Fill `GIT_REMOTE` and `GITHUB_PAT` in `.env`, which is already gitignored.

Never echo the token back, never `cat .env`, never put it in a commit message. If you need to check a field, read `.env.example`, not the live file.

## Step 3b — Set the login mode

In `.env`, set `AUTH_MODE` to match the answer from earlier:

- JB app → `AUTH_MODE=jb` (the default; the template's middleware validates the
  JournalistBoost session)
- FA app → `AUTH_MODE=zephr` (the middleware stands down; Zephr gates at the
  edge, and readers have no JB session to check)

This must also be set in the deployed environment, not just locally. Getting it
wrong doesn't fail loudly — it just refuses everyone.

## Step 4 — Make it theirs

- `package.json` → set `name` to the project name
- `README.md` → one sentence saying what this app is for
- `src/pages/index.astro` → title and copy that read as their app

Leave everything else alone. Only touch other files when a skill tells you to.

## Step 5 — Git identity, locally only

```bash
git config user.name "<their name>"
git config user.email "<their @finansavisen.no or @hegnar.no address>"
```

Never `--global`. Don't touch their machine-wide git config.

## Step 6 — First save

Run the template's own `save` skill (in the cloned project's `skills/`). If the push succeeds, the token and remote are both correct — and you've proved that before any real work exists to lose.

## Step 7 — Check their machine

Run `setup-machine` if anything is missing — Node 22, pnpm, git. **Say where
every command goes**: `/` commands here in the chat, everything else in Terminal
(Mac) or PowerShell (Windows). Don't mix the two in one block.


Run the template's own `tools-init` skill (in the cloned project's `skills/`): Node 22, pnpm, git, and `pnpm dev` serving `http://localhost:3000`. Don't skip it. Windows machines frequently have stale PATH state after a fresh Node install, and the failure looks like nothing at all.

## Then

Tell them what they have, in their words:

> "Your project is set up and saved. You've got a Finansavisen-branded starter running on your machine. Tell me what you'd like on the first page."

## Don'ts

- ❌ Don't start from scratch or from a different framework, however simple the request sounds.
- ❌ Don't build a login. Apps on the platform sit behind the JournalistBoost login already — see the `publish` skill.
- ❌ Don't use SQLite or write data to files. See `add-database`.
- ❌ Don't explain git, npm, or the difference between a branch and a commit unless they ask.

## After bootstrap

Stack-specific work — pages, styling, interactivity, the Zephr header — is covered by the **template's own** `AGENTS.md` and `skills/` in the cloned project. Read them before building. This plugin only covers what's true for every template.
