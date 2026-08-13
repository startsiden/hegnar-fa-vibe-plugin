---
description: Put a JB app live on the FA app platform, or push an update to one that's already live. Use when the journalist says deploy, publish, go live, put it online, or update the live version — for apps on apps.journalistboost.ai. FA apps on finansavisen.no do not use this; see the publish skill.
---

# Deploy a JB app

Deploys to the FA app platform at `<app-name>.apps.journalistboost.ai`.

**Only for JB apps.** If this app is served on `finansavisen.no` (`AUTH_MODE=zephr`),
stop and use the `publish` skill — those are deployed by the platform team and
nothing here applies.

## The deployment key

You need a Coolify API token. Ask the platform team for it once.

Store it in `~/.fa-vibe/token`, readable only by the journalist:

```bash
mkdir -p ~/.fa-vibe && chmod 700 ~/.fa-vibe
# have them paste it, then:
chmod 600 ~/.fa-vibe/token
```

Read it with `TOKEN=$(cat ~/.fa-vibe/token)` and use it as
`Authorization: Bearer $TOKEN`.

**Never** print it, echo it, put it in a commit, paste it into chat, or write it
into the project. It is not their password — it's a key to the whole platform,
so treat it like one. If they don't have it yet, stop and tell them to ask the
platform team; don't improvise another way to deploy.

## Before every deploy

1. **Save first** — the platform builds from GitHub, so anything unpushed does
   not exist as far as the deploy is concerned. Run the template's `save` skill.
2. **Confirm the branch is on GitHub**: `git ls-remote origin <branch>`. Empty
   output means it's local only, and the deploy will fail in about 8 seconds
   with no logs at all. This is the single most common failure.
3. **Build the container locally**: `docker build -t <app>:local .`. Catching a
   broken Dockerfile here takes seconds; catching it on the platform takes a
   round trip and a confusing error.

## Updating an app that's already live

This is the common case and it's one call. Find the app by name, then deploy:

```bash
BASE=https://coolify.journalistboost.ai/api/v1
TOKEN=$(cat ~/.fa-vibe/token)

# find the uuid for this app
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/applications" \
  | python3 -c "import json,sys;print([a['uuid'] for a in json.load(sys.stdin) if a['name']=='<app-name>'][0])"

# deploy
curl -s -X POST -H "Authorization: Bearer $TOKEN" "$BASE/deploy?uuid=<uuid>&force=false"
```

Then poll until it settles — takes 90–130 seconds:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$BASE/deployments/<deployment_uuid>" \
  | python3 -c "import json,sys;print(json.load(sys.stdin).get('status'))"
```

`finished` is success. `failed` means look at the container, not the API — the
deployment record's `logs` field is usually null and tells you nothing.

## First deploy of a new app

The app has to be registered once. Repo access is the part that needs the
platform team, unless Coolify's GitHub App is already installed on the org — if
it is, skip step 1.

**1. Repo access (private repos, no GitHub App)**

Needs `gh` and admin on the repo. If the journalist doesn't have both, stop and
ask the platform team to register the app — don't hand them a key generation
they can't verify.

```bash
ssh-keygen -t ed25519 -N '' -C 'coolify' -f /tmp/dk -q
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d "$(python3 -c 'import json;print(json.dumps({"name":"<app>-deploy-key","private_key":open("/tmp/dk").read()}))')" \
  "$BASE/security/keys"                      # returns the private_key_uuid
gh repo deploy-key add /tmp/dk.pub --repo <org>/<repo> --title "coolify (read-only)"
rm -f /tmp/dk /tmp/dk.pub                    # Coolify holds it now
```

**2. Create the application**

```json
POST /applications/private-deploy-key
{
  "project_uuid": "qyosn147ypl5calgk2ewgfef",
  "server_uuid": "rn7g9kv8rvqye0v1aendt2p7",
  "environment_name": "production",
  "private_key_uuid": "<from step 1>",
  "git_repository": "git@github.com:<org>/<repo>.git",
  "git_branch": "<branch>",
  "build_pack": "dockerfile",
  "dockerfile_location": "/Dockerfile",
  "ports_exposes": "3000",
  "name": "<app-name>",
  "domains": "https://<app-name>.apps.journalistboost.ai",
  "instant_deploy": false
}
```

Keep `instant_deploy: false` — env vars must exist before the first build.

**3. Environment variables** — one `POST /applications/{uuid}/envs` per variable,
body `{"key":…,"value":…,"is_preview":false}`. Always set:

```
AUTH_MODE=jb          PORT=3000          HOST=0.0.0.0
JB_URL=https://www.journalistboost.ai
```

Plus anything from `.env.example` the app genuinely needs. **Check every value
against where it's actually running** — an example file is a shape reference,
not a source of real values. A placeholder key and a `dev-` URL copied straight
from `.env.example` is exactly how the stock monitor shipped unable to fetch any
data.

**4. Persistent storage**, only if the app stores files:

```json
POST /applications/{uuid}/storages
{"name":"<app>-data","mount_path":"/app/data","type":"persistent"}
```

`type` **must** be `"persistent"`. `volume`, `bind`, `directory` and `file` are
all rejected, and the error doesn't tell you the valid value.

**5. Deploy**, as above.

## Stay inside your lane

The token can reach every app on the platform, so the discipline is yours:

- Only create apps in the `fa-apps` project, on `*.apps.journalistboost.ai`.
- Only deploy the app you are working on. Never delete one. Never read or change
  another app's environment variables.
- If an API call looks like it would affect anything other than this journalist's
  app, don't make it.

## After it's live

1. Open `https://<app-name>.apps.journalistboost.ai` — you should be sent to the
   JournalistBoost login, and back to the app once signed in.
2. If it stores data: add something, deploy again, confirm it survived. That's
   the check that catches a missing volume, and it's much cheaper now than after
   a week of someone's work is in there.
3. Tell the journalist plainly: *"It's live at <URL>. Anyone at Finansavisen
   signed into JournalistBoost can use it. Every time you save and deploy, your
   changes go live in a couple of minutes."*

## When it fails

- **Fails in ~8 seconds, no logs** → the branch isn't on GitHub. Push it.
- **Build fails** → reproduce with `docker build .` locally; the platform is not
  more forgiving than your laptop.
- **App is up but broken** → its own logs, not the deploy logs. Ask the platform
  team to check the container if you can't reach it.
- **401 from the API** → the token is wrong or expired. Ask the platform team;
  don't go hunting for another route in.
