---
description: Get a journalist's laptop ready to build FA apps — Node, pnpm, git, and only the heavier tools they actually need. Use at the very start, when a command fails with "not found", when a dependency fails to compile, or when Docker runs out of memory.
---

# Getting the machine ready

The journalist is not a developer. Two rules for this whole skill:

1. **Say where to type things.** Every command belongs in one of two places, and
   they are not interchangeable. Label every single one.
2. **Install as little as possible.** Each tool is another thing that breaks on a
   Tuesday. Only add one when something actually needs it.

## Where commands go

| Where | What goes there | How you say it |
|---|---|---|
| **In Claude Code** (this chat) | Anything starting with `/` — `/plugin install`, `/fa-vibe:new-project` | *"Type this here, in our chat"* |
| **In a terminal** | `winget`, `brew`, `node`, `git`, `pnpm` | *"Open Terminal (Mac) / PowerShell (Windows) and paste this"* |

Never present the two in one block. If a journalist pastes `/fa-vibe:new-project`
into PowerShell they get an error that means nothing to them.

Most of the time **you** run terminal commands on their behalf — so say what
you're doing and why, rather than handing them homework.

## What's actually needed

| Tool | Needed for | Skip it? |
|---|---|---|
| **Node 22** | Everything | No |
| **pnpm** | Everything | No — `corepack enable` ships with Node |
| **git** | Saving work | No |
| **Docker** | *Optional.* Only to test the container build locally | **Yes, usually** — see below |
| **Build toolchain** | Only when a dependency compiles from source | Only when something fails |
| **Postgres** | Only if the app stores data | Only then, and see below |

### Install

**Windows** (PowerShell):
```powershell
winget install OpenJS.NodeJS.LTS
winget install Git.Git
```

**Mac** (Terminal):
```bash
brew install node@22 git
```

Then, both:
```bash
corepack enable
```

Close and reopen the terminal afterwards. A fresh Node install very often isn't
on the PATH in windows that were already open, and the failure looks like Node
simply not existing.

## Docker: usually skip it

**The platform builds the app for you.** Docker on the journalist's machine only
lets you catch a broken build a few minutes earlier — it is not needed to
develop, and not needed to deploy.

On a modest Windows laptop Docker Desktop wants a couple of GB through WSL2 and
will fall over. If it's already struggling: **uninstall it and move on.** Don't
spend an afternoon tuning a VM to save a round trip.

Install it only if the same deploy fails twice for reasons you can't see.

## Postgres: install it natively, don't use Docker

Only when the app stores data — check `add-database` first, since most apps
don't need one.

Running Postgres in Docker on a laptop with limited memory is the worst of both
worlds. Install it directly:

**Windows** (PowerShell, will prompt for approval):
```powershell
winget install PostgreSQL.PostgreSQL.17
```

**Mac**:
```bash
brew install postgresql@17 && brew services start postgresql@17
```

It runs as a small background service, starts with the machine, and doesn't
compete with anything for memory. Then create the local database and put its URL
in `.env` — see `add-database`.

This is **local only**. In production the platform provides Postgres and injects
`DATABASE_URL`; nothing about the journalist's machine matters there.

## When a dependency won't compile

Some Node packages build native code (`node-gyp`, `sharp`, `better-sqlite3`).
The error is a wall of C++ output. Don't paste it at the journalist — say
*"one of the building blocks needs an extra tool, installing it now."*

**Windows** — Visual Studio Build Tools, **not** GCC:
```powershell
winget install Microsoft.VisualStudio.2022.BuildTools --override "--wait --quiet --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

**Mac** — Xcode command line tools:
```bash
xcode-select --install
```

Then delete `node_modules` and install again.

Before doing any of that, check whether it's needed at all: most packages ship
prebuilt binaries for Node 22, and a compile attempt often means the machine is
on an odd Node version. `node -v` first — if it isn't 22.x, fix that instead.

## Check it works

```bash
node -v      # v22.x
pnpm -v      # 9.x or 10.x
git --version
```

Then, in the project: `pnpm install && pnpm dev` and open
`http://localhost:3000`. If that renders, the machine is ready — Docker and the
rest are problems for later, if ever.
