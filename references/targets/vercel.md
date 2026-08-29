# Vercel — migration target

The best developer experience for Next.js and other frontend frameworks.
Serverless: request-scoped functions, not processes. If the Zeabur service is
a framework app that renders pages and calls APIs, Vercel is a near-drop-in.
If anything in it must stay resident — bot long-polling, queue consumer,
cron loop in-process — Vercel is the wrong shape; don't bend the app to fit
(that's a rewrite, and Railway/Fly take it as-is). WebSockets are in public
beta as of mid-2026 — usable for new work, not something to bet a same-week
rescue migration on.

**Fit:** class A (frontend frameworks, static). Class E partially (Vercel
Cron hits an endpoint on schedule — but Hobby cron runs at most once per day
with ±59 min slack; per-minute cron needs Pro). Not B, not C, not D (no
database hosting — the Marketplace resells Neon/Upstash etc., which is fine
and often the right DB answer). Note Hobby deploys compute to a single
region — pick the closest (hnd1 Tokyo / sin1 Singapore) explicitly.

**Cost (verify at https://vercel.com/pricing):** Hobby is free, generous —
and **non-commercial use only**. Anything revenue-touching (the course site,
the client project, ads) needs Pro at **$20/user/mo**. Ask the commercial
question out loud before recommending Hobby; it is the most common gotcha in
this migration. No card on Hobby.

## User does

1. Sign up at https://vercel.com (GitHub preferred — enables git deploys).
2. Approve `vercel login` (browser).
3. If the project deploys from GitHub: authorize the Vercel GitHub App when
   the dashboard asks. (Git-connected deploys give preview URLs per PR —
   worth it.)
4. Pro upgrade + card if commercial.

## Claude does

```bash
npm i -g vercel
vercel login                  # OAuth device flow: CLI prints a code, user approves in browser
vercel link                   # run in the repo, create/link the project
vercel --prod                 # or: vercel (preview first), then promote
```

(`vercel login --github`-style flags were removed in 2026 — the CLI's device
flow is the only path now; some Vercel doc pages still show the dead flags.)

Framework config is auto-detected for Next.js/Astro/Nuxt/SvelteKit — the
usual porting work from Zeabur is only:

- Anything reading `PORT`/listening manually (Zeabur ran it as a server) —
  on Vercel the framework's own start path is used; custom `server.js`
  wrappers must go.
- Long tasks: function duration caps at 300 s on Hobby / 800 s on Pro
  (verified 2026-08) — move genuinely long work elsewhere. Request bodies cap
  at 4.5 MB — file-upload flows need direct-to-storage uploads.
- File writes: only `/tmp`, ephemeral — anything persisted to disk on Zeabur
  needs object storage (Vercel Blob / R2 / S3).

Env vars — piped stdin or the interactive prompt both keep values out of the
transcript; production/preview vars are **sensitive by default** now (value
unreadable later, which is what we want post-breach):

```bash
grep '^OPENAI_API_KEY=' .env.new | cut -d= -f2- | vercel env add OPENAI_API_KEY production
vercel env add OPENAI_API_KEY production,preview   # or interactive: prompts for the value
vercel env ls                                      # names only — safe
```

Redeploy after env changes (`vercel --prod`) — env vars bake in at
build/deploy time.

## Database

Marketplace → Neon (Postgres, has Singapore region) or Upstash (Redis) from
the dashboard's Storage tab, or create directly at neon.tech/upstash.com and
paste the URL as an env var. Restore:
`pg_restore -d "$NEW_URL" --no-owner backup.dump` — but if the database has
live traffic, follow ../db-migration.md (rehearsal → freeze → final dump →
verify) instead of a one-shot dump. Serverless functions open many
short-lived connections: use Neon's **pooled** connection string.

## Domain & cutover

`vercel domains add <domain>` on the project (or dashboard → project →
Settings → Domains); Vercel shows the A/CNAME records; user updates the
registrar; verify with `curl -sI https://<domain>` (look for `server:
Vercel`) before decommissioning Zeabur.

## Gotchas

- Hobby's non-commercial rule is enforced by ToS, not by a paywall — the
  deploy will *work*; the account is what's at risk. Flag it even if the user
  shrugs.
- Serverless + LLM streaming works (functions stream), but long streams eat
  function duration — check the plan's limit against the app's longest
  response.
- Env vars from Zeabur that pointed at sibling services
  (`http://service.zeabur.internal`) have no meaning here — every internal
  URL needs a public replacement.

## Sources (official, checked 2026-08-29)

- Plans, Pro $20/seat: https://vercel.com/pricing
- Hobby non-commercial rule: https://vercel.com/docs/limits/fair-use-guidelines
- Function duration (300s/800s), memory, 4.5 MB body: https://vercel.com/docs/functions/limitations
- Hobby cron once/day: https://vercel.com/docs/cron-jobs/usage-and-pricing
- Regions, Hobby single compute region: https://vercel.com/docs/regions
- WebSockets public beta: https://vercel.com/changelog/websocket-support-is-now-in-public-beta
- Docker = stateless functions only: https://vercel.com/kb/guide/does-vercel-support-docker-deployments
- CLI device-flow login (old flags removed): https://vercel.com/changelog/new-vercel-cli-login-flow
- `vercel env` commands: https://vercel.com/docs/cli/env
