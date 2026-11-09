# CLAUDE.md

Guidance for AI agents working in this repo. Keep it accurate; don't contradict README.

## What this is

Reverse proxy + client shim so Firebase **Auth**, **Remote Config**, and
**Analytics** work from mainland China (where `*.googleapis.com` is blocked)
without a VPN. Three independent proxy implementations of one shared contract,
plus a browser fetch shim.

## Stack

- **Bun** + TypeScript (runtime; Bun runs `.ts` directly — no build/bundle step).
- Cloudflare Workers (`worker/worker.js`, plain JS) and nginx (`nginx/*.conf`)
  as alternative deploy targets.
- No production dependencies. Dev deps only: `@types/bun`, `typescript`.

## Commands (verified)

- Tests: `bun test` (or `bun run test:unit`) — 14 tests in `src/proxy.test.ts`.
- Typecheck: `bunx tsc --noEmit` (or `bun run typecheck`).
- Run server: `PORT=8080 bun run dev` (watch) or `bun run start`.
- Worker dev: `cd worker && npx wrangler dev`; deploy `npx wrangler deploy`.
- No linter/formatter is configured. Don't invent one.

## Architecture

Path-prefix routing: `/<prefix>/<rest>` → `https://<upstream>/<rest>`. Prefixes
and upstreams are defined in `src/proxy.ts` `UPSTREAMS` (and duplicated in
`worker.js` and `nginx/firebase-proxy.conf`):
`identitytoolkit`, `securetoken`, `remoteconfig`, `installations`, `gtm`, `ga`.

- `src/proxy.ts` — pure logic (routing, key allowlist, CORS, header stripping),
  unit-tested. `src/index.ts` — Bun HTTP server wrapping it, adds `/health`.
- `worker/worker.js` — Cloudflare Worker; reimplements the same contract inline
  (does NOT import `src/proxy.ts`).
- `nginx/firebase-proxy.conf` — nginx variant, same path scheme.
- `web/firebaseProxyShim.ts` — client shim; monkey-patches `fetch` and
  `navigator.sendBeacon` to rewrite Google hosts → proxy prefixes. Also shipped
  in `@sudobility/auth_lib`; this file is the standalone copy.

## Key conventions & gotchas

- **The three proxy variants must stay in sync.** Changing routing/key/CORS
  behavior means editing `src/proxy.ts`, `worker/worker.js`, AND
  `nginx/firebase-proxy.conf`. Only `src/proxy.ts` has tests.
- **Two intentional Bun↔Worker differences** (documented in `proxy.ts` header):
  Bun strips `accept-encoding` and deletes `content-encoding`/`content-length`
  (its fetch auto-decompresses); the Worker must not (it streams encoded bodies
  through). Bun drops `host`/`connection`; the Worker leaves them to its runtime.
- **Key allowlist default differs by variant** — do not "fix" this:
  - Bun + Worker: unset/empty `ALLOWED_API_KEYS` = **open mode** (forwards any
    project's traffic). `KEYED_PREFIXES` = the 4 non-analytics prefixes.
  - nginx: **fail-closed** via a literal `map` block; unedited config 403s every
    keyed request.
- **`OPTIONS` is exempt from the key check** in all three variants (CORS
  preflight can't carry `x-goog-api-key`). Preflight answered before the check.
- Cookies stripped before forwarding (Firebase web SDKs don't need them).
- Firebase Web API key is **not a secret** (ships in client bundles); the
  allowlist is anti-abuse, not confidentiality.
- Real config values never live in the repo: `.env`/`.dev.vars` are gitignored;
  deployed via Doppler (Docker) or `wrangler secret put` (Worker). Use the
  `.example` files as templates.

## Scope limits (can't be proxied — don't try)

FCM push, Google/OAuth sign-in popups, reCAPTCHA, and `@react-native-firebase`
native calls do not route through this proxy. See README "Known limits".

## CI/CD

`.github/workflows/ci-cd.yml` calls the external `johnqh/workflows`
unified-cicd. Push to `main` with a bumped `package.json` `version` builds and
pushes the Docker image; `develop` runs tests only. `skip-npm-publish: true`
(deployed as a Docker image, not an npm package).
