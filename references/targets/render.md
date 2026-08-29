# Render — migration target

Heroku-style PaaS: git-connected web services, background workers, cron jobs,
managed Postgres and Key Value (Redis-compatible), persistent disks, Docker
support. Simpler mental model than Fly, dashboard-first like Zeabur. Main
caveats are the free tier's spin-down and Asia coverage (Singapore region —
verify it's offered on the user's plan when latency matters).

**Fit:** classes B, D, E; C partially (no big template marketplace — deploy
the app's Docker image or repo directly; Render has some one-click "Deploy to
Render" blueprints). Class A static sites are free and fine, though
Cloudflare's static hosting is stronger.

**Cost (verified 2026-08, https://render.com/pricing + /docs/free):** free
web services exist (no card) but **spin down after 15 min idle** (cold start
on next request — wrong for bots/webhooks), are capped at 750
instance-hours/mo, and free Postgres **expires after 30 days, no backups**.
Real always-on: Starter web service + Basic-256mb Postgres ⇒ **≈$13/mo** for
app + DB — Render's own July 2026 figure; per-tier prices live only on
https://render.com/pricing (client-rendered, so check it in a browser).
Watch bandwidth: the $0 workspace includes only 5 GB/mo egress ($0.15/GB
after) — LLM-proxy apps can blow through that.

## User does

1. Sign up at https://render.com (GitHub/GitLab/Google/email).
2. Authorize Render's GitHub App for the repo (dashboard prompts).
3. Approve CLI login (`render login` opens the browser) if the CLI is used.

## Claude does

Render is dashboard-and-blueprint-first; the reliable non-clicky path is a
`render.yaml` blueprint committed to the repo:

```yaml
services:
  - type: web
    name: myapp
    runtime: docker          # or node/python with build/start commands
    plan: starter
    envVars:
      - key: OPENAI_API_KEY
        sync: false          # "sync: false" = set value in dashboard, never in git
databases:
  - name: myapp-db
    plan: basic-256mb
```

Then: user clicks **New → Blueprint**, picks the repo, and Render provisions
everything. `sync: false` env vars are entered by the user in the dashboard
env editor (values stay out of git and chat — send them the exact
service → Environment path). Sharp edge: the `sync: false` prompt appears
**only at initial Blueprint creation** — vars added to the blueprint later
must be set manually in the dashboard.

CLI alternative for existing services: `brew install render` then
`render login`, `render deploys create <service-id> --wait`. **The CLI
cannot update env vars at all** (only set them at service creation) — use
the dashboard env editor, or the REST API per-key endpoint:

```bash
curl -X PUT "https://api.render.com/v1/services/<srv-id>/env-vars/OPENAI_API_KEY" \
  -H "Authorization: Bearer $RENDER_API_KEY" -H 'Content-Type: application/json' \
  --data @- <<< "{\"value\": \"$(grep '^OPENAI_API_KEY=' .env.new | cut -d= -f2-)\"}"
```

Use the per-key `PUT …/env-vars/{key}` form — the collection-level
`PUT …/env-vars` **replaces the whole set and deletes anything omitted**.
Env changes don't auto-deploy; trigger a deploy after. Note the API key
(`rnd_…`, from Account Settings → API Keys) is account-wide with no scoping
— treat it as sensitive as the secrets it sets, and delete it after the
migration.

## Database restore

Dashboard → the Postgres instance → connection info ("External Database
URL"), then `pg_restore -d "$NEW_URL" --no-owner backup.dump` locally — for
live data follow ../db-migration.md (rehearsal → freeze → final dump →
verify), not a one-shot dump. Key Value (Redis) has no import path — treat
as cache.

## Domain & cutover

Service → Settings → Custom Domains → add domain; Render shows the
CNAME/ANAME records; user sets them at the registrar; wait for the cert, then
`curl -sI` to verify before decommissioning Zeabur.

## Gotchas

- The idle spin-down applies to *free* services only, but users who start
  free and forget will read it as "my bot is broken" — if the workload is a
  bot/webhook, put it on Starter from day one or pick Railway/Fly instead.
- Free Postgres deletion after 30 days has burned many people; never leave a
  migrated user's only copy of data there.
- Region is chosen per service at creation and can't be changed later —
  set Singapore explicitly at create time if Asia latency matters.

## Sources (official, checked 2026-08-29)

- Free tier limits (15-min spin-down, 750 instance-hours, 30-day free Postgres): https://render.com/docs/free
- Instance tiers (Free/Starter/Standard/Pro specs; prices on /pricing): https://render.com/docs/compute-plans
- "$13/month" Starter + Basic-256mb example (July 2026, Render's own article): https://render.com/articles/how-much-does-cloud-application-hosting-cost-for-small-businesses
- Regions (Singapore only in Asia): https://render.com/docs/regions
- Blueprint `sync: false` behavior: https://render.com/docs/blueprint-spec
- Bandwidth allotments and platform FAQ: https://render.com/docs/faq
