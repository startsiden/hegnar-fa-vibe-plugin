---
description: What the FA app platform actually is — a self-hosted Coolify instance on AWS — its API, its concepts, and the quirks that cost real debugging. Load when deploying, when a deploy fails, when reasoning about builds, domains, certificates, storage or environment variables on the platform, or when anything about the platform's behaviour is surprising.
---

# The platform is Coolify

The "FA app platform" is a **self-hosted [Coolify](https://coolify.io) instance**
— an open-source PaaS, roughly Heroku-shaped: it watches a git repo, builds a
container, runs it, and puts Traefik in front with a Let's Encrypt certificate.

Knowing that is the point of this skill. If Coolify's behaviour surprises you,
reason about it as Coolify rather than as a bespoke system, and check
[coolify.io/docs](https://coolify.io/docs).

## This instance

| | |
|---|---|
| Dashboard | `https://coolify.journalistboost.ai` |
| API base | `https://coolify.journalistboost.ai/api/v1` |
| Version | 4.3.x — it auto-updates, so don't hard-code behaviour to a version |
| Host | A dedicated AWS EC2 box (`eu-north-1`), ~8GB RAM, in Finansavisen's own AWS account |
| Apps served at | `<app-name>.apps.journalistboost.ai` (wildcard DNS, so new apps need no DNS work) |
| Proxy | Traefik, on 80/443, issuing Let's Encrypt certs per hostname |

Repo access is a **GitHub App**, `jb-vibe-apps`, installed on the `startsiden`
org — uuid `mvuhbeup3xydp6glzbtvbb7s`. Any repo in that org can be deployed with
no per-repo key. `GET /github-apps` lists sources.

Auth is a bearer token: `Authorization: Bearer $(cat ~/.fa-vibe/token)`. See the
`deploy` skill for how it's stored and its scope limits.

### The dashboard sees JB session cookies

`coolify.journalistboost.ai` sits under `journalistboost.ai`, and JB's session
cookie is scoped `Domain=.journalistboost.ai`. So a journalist's browser attaches
their live JB session to requests hitting the Coolify dashboard, which has no use
for it.

Accepted when the domain was chosen. It means: don't enable request-header
logging on this instance, and if the dashboard ever moves, a host outside
`journalistboost.ai` removes the exposure entirely.

## How Coolify models things

```
server (the EC2 box)
└── project            "fa-apps"      uuid qyosn147ypl5calgk2ewgfef
    └── environment    "production"
        └── application  one per journalist app
            ├── environment variables
            ├── persistent storage (docker volumes)
            └── deployments (build + run, one per trigger)
```

The server uuid is `rn7g9kv8rvqye0v1aendt2p7`. **Everything for journalist apps
belongs in the `fa-apps` project.** Don't create projects, don't change server
settings, don't touch the proxy.

An application is defined by: a git repo + branch, a build pack (always
`dockerfile` here), the port it listens on (always `3000`), and its domains.
Coolify rebuilds when you trigger a deploy, and can also rebuild on push if a
webhook is configured.

## The container's world

- **The filesystem is thrown away on every deploy.** Anything written to disk is
  gone, silently. Persistent storage or a database, nothing else.
- **Environment variables are set in Coolify, not in the repo.** They're injected
  at runtime, which is why app code must read `process.env` and never Vite's
  `import.meta.env` for anything that varies per environment — the latter is
  substituted at *build* time and baked into the image.
- **TLS terminates at Traefik.** The container only ever sees plain HTTP, so any
  code that builds absolute URLs must trust the forwarded headers, not its own
  protocol.
- Logs are the container's stdout — reachable in the dashboard, or on the box
  with `docker logs`.

## Quirks worth knowing before they cost you an hour

Each of these came from a real debugging session.

- **A branch that only exists locally fails in ~8 seconds with no logs.** By far
  the most common failure. `git ls-remote origin <branch>` first.
- **A deployment's `logs` field is usually `null`**, so the API tells you nothing
  about why a build failed. Diagnose from the container.
- **`proxy.status` in API responses is stale.** It read `exited` while the proxy
  had been healthy for hours. Trust `docker ps` on the box, never that field.
- **Persistent storage requires `"type": "persistent"`.** `volume`, `bind`,
  `directory` and `file` are all rejected, and the validation error doesn't name
  the valid value.
- **The dashboard port 8000 is firewalled and must stay that way.** It serves the
  full dashboard over plain HTTP, bypassing the certificate. Use the domain.
- **Certificates are issued per hostname over HTTP-01**, so a hostname must
  resolve publicly *before* Coolify can get a cert for it. A domain added too
  early just fails quietly.

## Don't

- ❌ Don't touch another app. The token can reach all of them; that's discipline, not a permission boundary.
- ❌ Don't change server, proxy or instance settings.
- ❌ Don't open ports in the AWS security group. 22 (restricted), 80 and 443 only.
- ❌ Don't add Traefik labels by hand — Coolify manages them, and edits are overwritten on redeploy.
- ❌ Don't put secrets in the repo because "Coolify will pick them up". It reads env vars you set in Coolify, not files in git.

## Related

- `deploy` — the actual API calls to put an app live
- `add-database` — Postgres on this instance
- `zephr-to-jb` — converting an app built for finansavisen.no
