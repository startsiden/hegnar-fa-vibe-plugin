---
description: Start a new Finansavisen app from an approved template. Use when the journalist says they want to build something new, start a project, make an app or a tool, or when they describe an idea and have no project on disk yet.
---

# Start a new Finansavisen app

The person you're working with is a **journalist, not a developer**. They speak product, not code. Never show them a stack trace, never ask them to pick a framework, and never explain git.

## Templates

Pick from this list. Don't invent a stack, and don't start from an empty directory.

| Template | Use when | Repo |
|---|---|---|
| `fa-astro` *(default)* | Almost everything: pages, dashboards, lists, forms, small tools, anything shown to readers or colleagues | `https://github.com/startsiden/vibecode-template` |

If nothing fits what they're describing, say so and stop. Ask them to check with the platform team rather than improvising a stack — an app off-template can't be deployed by the normal route.

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

Run the `save` skill. If the push succeeds, the token and remote are both correct — and you've proved that before any real work exists to lose.

## Step 7 — Check their machine

Run the `tools-init` skill: Node 22, pnpm, git, and `pnpm dev` serving `http://localhost:3000`. Don't skip it. Windows machines frequently have stale PATH state after a fresh Node install, and the failure looks like nothing at all.

## Then

Tell them what they have, in their words:

> "Your project is set up and saved. You've got a Finansavisen-branded starter running on your machine. Tell me what you'd like on the first page."

## Don'ts

- ❌ Don't start from scratch or from a different framework, however simple the request sounds.
- ❌ Don't build a login. Apps on the platform sit behind the JournalistBoost login already — see the `publish` skill.
- ❌ Don't use SQLite or write data to files. See `add-database`.
- ❌ Don't explain git, npm, or the difference between a branch and a commit unless they ask.
