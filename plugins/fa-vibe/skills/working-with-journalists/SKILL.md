---
description: How to work on a Finansavisen app with a journalist — vocabulary, tone, and the platform rules that hold for every template. Load at the start of any work on an FA app, and whenever unsure whether something is allowed on the platform.
---

# Working on a Finansavisen app

Rules that hold regardless of which template the app came from. Stack-specific
guidance — pages, styling, the Zephr header, interactivity — lives in the
template's own `AGENTS.md` and `skills/`. Read those too.

## Who you're talking to

A **journalist, not a developer**. They speak product, not code.

- No stack traces, no error dumps, no framework debates.
- When something breaks, say what it means for them and what you're doing about it.
- Don't ask them to choose between technical options. Choose, and say why in one sentence.
- Don't explain git, npm, or branches unless they ask.

## Vocabulary

They don't speak git. Translate silently:

| They say | You do |
|---|---|
| "save" | `git add -A && git commit && git push` |
| "publish" / "go live" | save first, then the `publish` skill |
| "undo" | `git reset --soft`, or `--hard` after confirming |
| "it's broken" | reproduce first, then fix. Don't guess out loud |

## The platform, and which login applies

Two kinds of app, two different logins. **Neither one is written by you or the
journalist** — but they are not interchangeable, and picking the wrong one locks
out exactly the audience the app is for.

| | JB app | FA app |
|---|---|---|
| Lives at | `<name>.apps.journalistboost.ai` | `finansavisen.no/<path>` |
| Audience | Journalists signed into JournalistBoost | Finansavisen readers, per the paywall |
| Login enforced by | The app, via its middleware (`AUTH_MODE=jb`) | Zephr, at the CDN edge (`AUTH_MODE=zephr`) |
| App-level login code | Ships in the template — don't touch it | **None at all** |

In both cases:

- ❌ Never build a login page, password field, session, or user table.
- ❌ Never add an auth library — no Auth.js, Clerk, Supabase Auth, Passport, Lucia.

And the mistake that actually happens:

- ❌ **Never put a JB `check-session` call in an FA app.** Readers have no
  JournalistBoost session, so it rejects all of them.
- ❌ **Never copy Zephr / `blaize_session` checks into a JB app.** That cookie
  only exists on `finansavisen.no`; on `journalistboost.ai` it locks out every
  journalist.

### Who is the visitor — JB apps only

FA apps don't get an identity from the platform. Readers are anonymous as far
as the app is concerned; the paywall decided whether they get in.

For a JB app, ask JB server-side:

```ts
const res = await fetch("https://www.journalistboost.ai/api/auth/check-session", {
  headers: { cookie: <the incoming request's cookie header> },
});
const { valid, user } = await res.json(); // user: { id, email, role, isActive }
```

Server-side only. Never expose that cookie to browser JavaScript, never log it.
There is no name field — `email` is the most human thing available.

"Only certain people should see this" is an allowlist of email addresses in
config, not a login system.

## Storing data

The container's disk is thrown away on every deploy. Anything written to a file
is gone the next time the app publishes — silently, with no error.

- ❌ No SQLite, no `better-sqlite3`, no JSON files as storage.
- ✅ A Postgres database provisioned by the platform. See the `add-database` skill.

First ask whether it needs a database at all. A fixed list belongs in a file in
the repo; a per-visitor preference belongs in `localStorage`.

## Hosting

Deployment target is the FA app platform — a self-hosted Coolify instance on
Finansavisen's own AWS. See the `coolify` skill for how it works. Don't propose Vercel, Netlify,
Cloudflare Pages, Heroku, Render, or a personal server. Don't hand-write
Kubernetes or OKD manifests — FA's newsroom sites run on OKD, apps from these
templates do not.

## Secrets

- Never write a real key, token or password into the repo, into `.env.example`,
  or into a commit message. Placeholders only.
- Never paste a secret back into the chat, even one the journalist just gave you.
- `.env` is gitignored. Keep it that way.

A key committed once is a key that must be rotated, and that's someone else's
afternoon.

## Before you say it's done

1. It runs locally without errors.
2. You clicked through what you changed — not just "the build passed".
3. `docker build .` succeeds, if the change touched the Dockerfile or dependencies.
4. No secrets in the diff.
5. You can describe what changed in one sentence a journalist would understand.
