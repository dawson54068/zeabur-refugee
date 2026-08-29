# Railway — migration target

The most Zeabur-shaped target: git deploys with auto-detected builds
(Nixpacks/Railpack), Dockerfile support, one-click templates, managed
Postgres/MySQL/Redis/Mongo, volumes, cron, usage-based pricing. Most Zeabur
setups map 1:1 — same env vars, same Dockerfile, a database sibling service.

**Fit:** classes B (long-running server), C (template apps — its template
marketplace has most of the same apps: n8n, one-api/new-api, LobeChat, Umami),
D (databases), E (cron/workers). Overkill for a pure static site (class A —
use Cloudflare).

**Cost (verified 2026-08 at https://railway.com/pricing):** a Free plan
exists — $0/mo with a $1/mo usage credit, capped at 1 vCPU / 0.5 GB and 5
services, no card — enough to *evaluate*, not to run a real app. Hobby is
$5/mo including $5 of usage; a small always-on app + small Postgres
realistically lands **≈$10–15/mo** (usage rates ≈$20 per vCPU-month, ≈$10 per
GB-RAM-month, volumes $0.15/GB-mo). Unverified accounts get restricted
outbound network access and limited ports — which can silently break an LLM
app's API calls — so have the user run the automated verification at
https://railway.com/verify (checks GitHub account age/activity) early.

**Regions:** us-west2, us-east4, europe-west4, **asia-southeast1
(Singapore)** — pick Singapore explicitly for Zeabur-refugee latency; the
default is US. Railway is also the only managed-DB PaaS in this comparison
offering MySQL and MongoDB, not just Postgres/Redis — decisive if the Zeabur
DB was one of those.

## User does

1. Sign up at https://railway.com with GitHub (preferred — enables repo
   deploys), add a card on the Hobby plan.
2. Approve `railway login` when it opens the browser.
3. If deploying from GitHub: authorize the Railway GitHub App for the repo
   when the dashboard asks.

## Claude does

```bash
brew install railway            # or: npm i -g @railway/cli
railway login                   # user approves in browser
railway init                    # create project (run in the repo)
railway add --database postgres # managed DB if needed (also: mysql/redis/mongo)
railway up                      # build & deploy current directory
railway domain                  # mint the public *.up.railway.app URL
railway logs                    # verify boot
```

Template apps: deploy from the dashboard's template page instead of `railway
up` — find the app at https://railway.com/deploy (e.g. `/deploy/n8n`), user
clicks Deploy, then set env vars fresh. Faster and less error-prone than
reconstructing the compose setup by hand.

Env vars — the CLI reads values from stdin, which keeps them out of the
transcript and shell history (verified against CLI v5.45; `railway variable`
singular is canonical, `variables` still aliases):

```bash
grep '^OPENAI_API_KEY=' .env.new | cut -d= -f2- | railway variable set OPENAI_API_KEY --stdin
railway variable set NODE_ENV=production --skip-deploys   # non-secrets inline is fine
railway variable list --json | jq 'keys'                  # verify names without printing values
```

Each `set` triggers a redeploy by default — batch with `--skip-deploys` and
redeploy once at the end.

References between services use Railway's template variables, e.g.
`DATABASE_URL=${{Postgres.DATABASE_URL}}` — set that instead of a literal so
credentials rotate with the DB.

## Database restore

Live data? Follow ../db-migration.md (rehearsal → freeze → final dump →
verify) — a single dump loses every write made after it. Restore commands:

```bash
railway connect postgres        # psql shell straight to the new DB, or:
pg_restore -d "$NEW_DATABASE_URL" --no-owner backup.dump
```

Get `$NEW_DATABASE_URL` from `railway variables --json` on the DB service —
into a shell variable, not the chat.

## Domain & cutover

`railway domain <custom-domain>` (or service → Settings → Networking →
Custom Domain), then the user adds the CNAME Railway shows at their
registrar. Verify with `curl -sI https://<domain>` before decommissioning
Zeabur.

## Gotchas

- Usage pricing means an app that spins CPU (crashloop, busy poll) costs real
  money — set the usage limit in Account → Billing (user does; suggest $10–20
  cap) so surprises are bounded.
- Railway sleeps nothing by default (good for bots) — but that also means
  every service bills 24/7; delete experiments.
- Nixpacks/Railpack may pick a different Node/Python version than Zeabur's
  buildpack — pin via `.nvmrc`/`runtime.txt`/Dockerfile if the first deploy
  misbehaves.

## Sources (official, checked 2026-08-29)

- Plan structure and usage rates: https://railway.com/pricing and https://docs.railway.com/reference/pricing/plans
- Free/Trial tiers, limited-trial network/port restrictions: https://docs.railway.com/reference/pricing/free-trial (verification: https://railway.com/verify)
- Regions incl. Singapore: https://docs.railway.com/reference/regions
- Managed databases incl. MySQL and MongoDB: https://docs.railway.com/reference/databases
- `railway variable` CLI (incl. `--stdin`, `--skip-deploys`): https://docs.railway.com/cli/variable
