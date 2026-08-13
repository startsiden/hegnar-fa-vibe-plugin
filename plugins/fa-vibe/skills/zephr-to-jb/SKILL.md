---
description: Convert an existing Finansavisen app from Zephr auth to JournalistBoost auth so it can run on the JB app platform. Use when bringing an existing app to apps.journalistboost.ai, or when an app redirects every visitor to login, shows Unauthorized, or checks a blaize_session cookie.
---

# Zephr → JournalistBoost

An app written for `finansavisen.no` authenticates against **Zephr**. On
`*.apps.journalistboost.ai` that cookie does not exist, so the app rejects
**every journalist** — a redirect loop or a blank 401, with nothing in the logs
explaining why.

Converting is four things. If you find yourself changing more, stop and re-read
this.

## 1. Detect

Search the project. Any of these means it's a Zephr app:

```bash
grep -rn "blaize_session\|zephrAuth\|getAccountInfo\|ZEPHR_FEATURE" src/
grep -n "allowedDomains" -A5 astro.config.mjs
grep -n '"build"' package.json          # look for --base=/verktoy/...
```

| Signal | What it means |
|---|---|
| `blaize_session` cookie read | Zephr session check — **must be replaced** |
| `src/lib/zephrAuth.ts`, `getAccountInfo` | Zephr account lookup — **delete** |
| `allowedDomains` listing only `finansavisen.no` | Astro will distrust the JB host — **must be extended** |
| `--base=/verktoy/<name>` in the build script | Served at a path, not a subdomain root — **drop it** |
| `ZEPHR_FEATURE` markers / FA header components | Zephr CDN isn't in front of this domain, so these render as **bare comments** |

Tell the journalist in their words: *"This was built for the Finansavisen
website, which uses a different login. I'll switch it over to JournalistBoost so
your colleagues can use it."*

## 2. Replace the auth check

Delete the Zephr middleware block and its helper file. Replace with a JB session
check: read the `auth_token` cookie, ask
`https://www.journalistboost.ai/api/auth/check-session` server-side with that
cookie, redirect to JB login when it isn't valid. See
`working-with-journalists` for the call, and copy the template's
`src/lib/auth.ts` + `src/middleware.ts` rather than writing it fresh.

Two things to carry over, not delete:

- **Keep any CSRF / `Origin` check**, and widen its hostname pattern to include
  `journalistboost.ai`. That's separate from login and usually exists because of
  a real bug someone hit.
- **Keep `AUTH_DISABLED`** as the local-dev bypass. There is no JB cookie on
  localhost.

## 3. `allowedDomains` and the base path

In `astro.config.mjs`, add the platform domain:

```js
security: {
  allowedDomains: [
    { hostname: '**.finansavisen.no', protocol: 'https' },
    { hostname: '**.journalistboost.ai', protocol: 'https' },
  ],
},
```

**Don't skip this.** Without it Astro distrusts the forwarded `Host`, `Astro.url`
falls back to `localhost`, and the login redirect sends journalists to
`http://localhost/`. It looks like an auth bug and isn't.

In `package.json`, drop `--base=/verktoy/<name>`: on the platform the app is
served at a subdomain root.

## 4. Environment

Set `AUTH_MODE=jb` and `JB_URL=https://www.journalistboost.ai`. Remove
Zephr-only vars (`SIMULATE_ZEPHR`, `ZEPHR_COMPONENTS_URL`) from production.

**Audit every other value against where it now runs.** An app moving
environments is the single most likely place to inherit a wrong one — a `dev-`
API URL, or a placeholder key from `.env.example`. The FA stock monitor shipped
to the platform unable to fetch any data because it inherited both.

While you're in there: **check `.env.example` for real secrets.** Apps written
before this convention sometimes have live keys in them. If you find one, tell
the journalist it needs rotating — it's in git history and a placeholder doesn't
undo that.

## The FA header

`ZEPHR_FEATURE` markers are filled in by the Zephr CDN, which isn't in front of
this domain. On the platform they render as bare HTML comments — nothing
visible, no error.

Either remove the header/footer components, or leave them and tell the
journalist the Finansavisen chrome won't appear here. Don't try to reimplement
the header.

## Keep both versions working

If the app still runs on `finansavisen.no`, don't convert in place on the main
branch — you'd break the live version. Put the platform build on its own branch
and deploy from that.

`startsiden/hegnar-fa-stock-monitor` does exactly this: `master` is the Zephr/FA
build, `feat/fa-apps-platform` is the JB build. The diff between them is this
whole conversion, and it's worth reading before doing your own.

## Verify before saying it's done

```bash
docker build -t <app>:local .
docker run --rm -p 3000:3000 -e AUTH_MODE=jb <app>:local
```

- `GET /` with no cookie → **302** to `https://www.journalistboost.ai/login?from=…`
- The `from=` must name the app's real hostname. If it says `localhost`, step 3
  is wrong.
- `GET /api/...` with no cookie → **401**, not a redirect
- With `AUTH_DISABLED=1` → **200**, app renders

Then deploy with the `deploy` skill and open it as a logged-in journalist. A
conversion that looks right locally and loops in production has almost always
missed `allowedDomains`.
