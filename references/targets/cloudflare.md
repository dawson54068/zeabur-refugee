# Cloudflare — migration target

Edge platform, not a container host: static assets + Workers (V8 isolates),
with storage primitives (D1 SQLite, KV, R2 objects, Hyperdrive for external
Postgres/MySQL, Durable Objects, Queues, Cron Triggers). The free tier is the
best in this comparison and allows commercial use — the strongest "budget is
zero" answer for class A.

**Fit:** class A (static and framework sites — Next.js via the OpenNext
Cloudflare adapter, Astro/Nuxt/SvelteKit via their CF adapters). Class E for
HTTP-shaped cron (Cron Triggers). Class B only as a *rewrite* to
Workers/Durable Objects — a Node server does not run as-is; be honest that
this is porting work, not migration, and route to Railway/Fly if the user
just wants their app back up. Cloudflare Containers (GA since April 2026,
needs the $5 Workers Paid plan) runs real Docker images, but with blockers
for typical Zeabur workloads: **all disk is ephemeral** (no volumes), no
inbound raw TCP (HTTP only), no autoscaling yet, 1–3 s cold starts — so
n8n-style stateful templates and databases still don't belong here; verify
current limits at https://developers.cloudflare.com/containers/ before
proposing it. Class D: D1/KV/R2 are excellent but are *different databases* —
migrating Zeabur Postgres to D1 is a schema-and-code project, not a restore;
default to keeping Postgres (Neon + optional Hyperdrive) instead.

**Cost (verify at https://developers.cloudflare.com/workers/platform/pricing/):**
free: 100k Worker requests/day, static assets effectively unlimited, D1/KV/R2
free allotments, commercial use OK, no card. Paid: $5/mo (Workers Paid)
lifts CPU limits, adds Containers/bigger quotas. A framework site + small D1
app typically runs $0.

## User does

1. Sign up at https://dash.cloudflare.com/sign-up (email; no card for free).
2. Approve `wrangler login` in the browser.
3. If the domain's DNS doesn't already live on Cloudflare: either move the
   zone to Cloudflare (they change nameservers at the registrar — best
   integration) or CNAME from the current DNS host.

## Claude does

Static / framework app:

```bash
npm i -D wrangler            # project-local install is the recommended pattern
npx wrangler login           # browser OAuth; use `--device` on headless/SSH machines
# Framework apps: add the adapter first —
#   Next.js: npx create-cloudflare@latest --existing  (or add @opennextjs/cloudflare per current docs)
#   Astro/Nuxt/SvelteKit: the framework's cloudflare adapter
wrangler deploy            # Workers project (wrangler.jsonc with assets binding)
```

Check the current adapter recipe at https://developers.cloudflare.com/workers/frameworks/
rather than trusting memory — this area moves fast. Verify the deploy on the
`*.workers.dev` URL before cutover.

Env vars & secrets:

```bash
npx wrangler secret put OPENAI_API_KEY   # interactive prompt (no --value flag exists)
npx wrangler secret bulk .env.new        # bulk from a local .env or JSON file (≤100 keys)
npx wrangler secret list                 # names only — safe
# plain (non-secret) config goes in wrangler.jsonc "vars" — those are visible
# in the dashboard; anything sensitive must be a secret, not a var
```

Secrets survive deploys; they're write-only after creation (not readable in
dashboard or CLI) — exactly right for post-breach hygiene.

## Database

- Keep Postgres: create at https://neon.tech (Singapore region available),
  restore the dump (`pg_restore -d "$NEW_URL" --no-owner backup.dump`), and
  connect from Workers via Hyperdrive (connection pooling for the edge) or a
  serverless driver. Live data → ../db-migration.md (rehearsal → freeze →
  final dump → verify), never a one-shot dump.
- D1 only for new-ish/simple schemas where moving to SQLite is worth it —
  that's an offer, not a default.
- Redis: Upstash (has a Workers-native client).

## Domain & cutover

If the zone is on Cloudflare: add the custom domain on the Worker
(`wrangler.jsonc` routes / dashboard → Workers → domain) — cutover is
instant and TLS is automatic. If not, CNAME per the dashboard's instructions.
Verify with `curl -sI` before decommissioning Zeabur.

## Gotchas

- Workers are isolates, not Node: no filesystem, no arbitrary TCP (DB via
  drivers/Hyperdrive), 128 MB memory per isolate. `nodejs_compat` (on by
  default with 2026 compat dates) covers much of the Node API, but
  `child_process`, `worker_threads`, `cluster`, and `http2` are
  non-functional stubs — audit the app's deps before promising a port.
- CPU-time limits (not wall-clock) bound each request — LLM proxying/
  streaming is fine (waiting on upstream is free), heavy compute is not.
- WebSockets need Durable Objects — that's part of the class-B rewrite tax.
- If the user's whole reason for Cloudflare is "free", confirm their traffic
  fits 100k requests/day before promising $0.

## Sources (official, checked 2026-08-29)

- Workers pricing (free 100k req/day, $5/mo paid): https://developers.cloudflare.com/workers/platform/pricing/
- Workers limits (CPU time, 128 MB isolate, script size): https://developers.cloudflare.com/workers/platform/limits/
- Containers GA announcement (2026-04-13): https://developers.cloudflare.com/changelog/post/2026-04-13-containers-sandbox-ga/
- Containers pricing: https://developers.cloudflare.com/containers/pricing/
- Containers ephemeral disk / HTTP-only ingress: https://developers.cloudflare.com/containers/platform-details/ and https://developers.cloudflare.com/containers/faq/
- Node.js compat coverage and stubs: https://developers.cloudflare.com/workers/runtime-apis/nodejs/
- D1 pricing/limits: https://developers.cloudflare.com/d1/platform/pricing/
- R2 pricing (zero egress): https://developers.cloudflare.com/r2/pricing/
- Hyperdrive (included, external DBs only): https://developers.cloudflare.com/hyperdrive/platform/pricing/
