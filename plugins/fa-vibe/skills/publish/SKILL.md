---
description: Get a Finansavisen app live — self-serve on the app platform for JB apps, or a DevOps handoff for FA apps. Use when the journalist says deploy, publish, go live, put it online, release, or share it with the team.
---

# Going live

**When to load**: journalist says "deploy", "publish", "go live", "put it online", "release", "share it with the team".

**Goal**: get the app in front of the people it's for — which means two different routes depending on whether it's a JB app or an FA app. Check that first.

---

## Which route — check before doing anything

Publishing works completely differently depending on what kind of app this is.
If you don't know, look at the project: an app whose middleware checks
JournalistBoost is a JB app; one that checks Zephr / `blaize_session` and builds
with a `--base=/…` path is an FA app.

| | JB app | FA app |
|---|---|---|
| Lives at | `<name>.apps.journalistboost.ai` | `finansavisen.no/<path>` |
| Who publishes | Self-serve — the platform builds from GitHub on every save | **Profico DevOps**, via a ticket |
| How long | Minutes | Days, with review |
| Seen by | Journalists signed into JB | Finansavisen's readers |

**JB app?** You can do it yourself — load the `deploy` skill. Everything below
about pre-flight still applies; `deploy` covers the API calls.

**FA app?** None of the app-platform machinery applies. Do the pre-flight
checks, then open a DevOps ticket with: repo URL, branch, container port
`3000`, the public path it should serve at, the env vars it needs (values sent
privately, never in the ticket), and whether it sits behind Zephr. Tell the
journalist plainly: *"This one goes live on finansavisen.no, so the platform
team has to deploy it. I've prepared everything they need — it won't be
instant."*

Then stop. Don't promise a date you don't control.

## Pre-flight — do all of this before deploying

Deploys take a few minutes. Local checks take seconds. Never skip these to "just try it live".

1. **Save everything** — `skills/save.md`. The platform builds from GitHub, so anything unpushed does not exist as far as the deploy is concerned.

2. **Build locally**:
   ```bash
   pnpm install
   pnpm build
   ```
   Must pass with zero errors. Warnings are fine.

3. **Smoke-test the production build**:
   ```bash
   PORT=3000 pnpm start
   ```
   Open `http://localhost:3000` and click through every page. The Finansavisen header will be missing here — that's expected, the simulation only runs in `pnpm dev`.

4. **Build the Docker image** — this is the step that catches real deploy failures:
   ```bash
   docker build -t <app-name>:local .
   docker run --rm -p 3000:3000 -e DATABASE_URL=... <app-name>:local
   ```
   Same click-through inside the container. A broken Dockerfile found here costs seconds; found on the platform it costs a full deploy cycle plus confusion.

If any step fails, fix it before continuing. Do not deploy a red build hoping the platform is more forgiving. It isn't.

---

## First deploy — one-time setup

For an **FA app**, the platform team registers and deploys it. Give them:

| What | Value |
|---|---|
| App name | `name` from `package.json` — becomes the URL |
| GitHub repo | The journalist's repo (the platform reads it via the org GitHub App — no keys to paste) |
| Branch | Usually `main` |
| Port | `3000` |
| Health check | `/` |
| Database? | Yes/no — see `skills/add-database.md`. If yes, they provision a Postgres database and inject `DATABASE_URL` |
| Env vars | Everything from `.env.example` that isn't a placeholder, **with real values sent privately — never in a ticket, never in chat, never in the repo** |

Tell the journalist plainly: *"Someone on the platform team has to switch your app on the first time. After that, every save publishes automatically."*

---

## Environment variables in production

| Var | Value |
|---|---|
| `NODE_ENV` | `production` |
| `PORT` | `3000` |
| `HOST` | `0.0.0.0` (required by the Astro Node adapter) |
| `SIMULATE_ZEPHR` | **unset** — dev-only |
| `ZEPHR_COMPONENTS_URL` | **unset** — dev-only |
| `DATABASE_URL` | Set by the platform, if the app uses a database |
| `GITHUB_PAT`, `GIT_REMOTE` | **never in production** — those exist only for local saves |

Anything the browser needs must be prefixed `PUBLIC_`. Anything *not* prefixed stays server-side — that's the difference between a secret and a published secret.

---

## Login — you don't deploy anything for this

The platform puts every app behind the JournalistBoost login before the request reaches your container. There's nothing to configure, no env var, no library, no callback URL.

Do not add a login page "just in case". See the "Who's logged in" section of `AGENTS.md`.

---

## After deploy

1. Open `https://<app-name>.apps.journalistboost.ai` and click through everything.
2. Confirm you were asked to log in if you weren't already — that proves the gate is in front of the app.
3. If the app uses a database: add something, then **redeploy and check it's still there**. That's the test that catches a missing volume or an unprovisioned database, and it's much cheaper to find now than after a journalist has entered a week of data.
4. Tell the journalist in their words: *"Your app is live at <URL>. Anyone at Finansavisen who's logged into JournalistBoost can use it. Every time you save, your changes go live in a few minutes."*

---

## Updating after the first deploy

1. Edit the code.
2. `skills/save.md`.
3. The platform rebuilds automatically.
4. Refresh after ~2–5 minutes.

If a deploy doesn't appear, the build probably failed. Ask the platform owner to check the build log — don't guess, and don't start changing code at random.

---

## Hard don'ts

- ❌ Don't propose Vercel, Netlify, Cloudflare Pages, Heroku, Render, or a personal server. The target is the FA app platform.
- ❌ Don't hand-write Kubernetes/OKD manifests. FA's newsroom sites run on OKD; apps from this template do not.
- ❌ Don't put real secrets in `.env.example`, in the repo, in a commit message, or in a ticket. Placeholders only — a real key committed once is a key that must be rotated.
- ❌ Don't commit `.env`.
- ❌ Don't use SQLite or write data files to disk. The container's filesystem is discarded on every deploy. See `skills/add-database.md`.
- ❌ Don't change the container port away from `3000`.
- ❌ Don't deploy on the journalist's behalf if the local Docker build is failing.
