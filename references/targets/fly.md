# Fly.io — migration target

Docker-native micro-VMs ("Machines") with the best Asia coverage of the PaaS
options: Tokyo (nrt), Singapore (sin), Sydney (syd) — no Hong Kong or Seoul.
More knobs than Railway: great when the user is comfortable with a `fly.toml`
and wants cheap always-on containers; skip it for users who wanted Zeabur's
zero-config feel (no git-push deploys either — flyctl or GitHub Actions).

**Fit:** classes B and E; C works but there's no template marketplace — you
write the config. D: volumes + Postgres exist, but Fly's Managed Postgres
starts at ~$38/mo — for small apps either self-run Postgres on a machine +
volume (~$7–8/mo, user operates it) or prefer a managed DB elsewhere
(Neon/Upstash/Railway).

**Cost (verified 2026-08 at https://fly.io/docs/about/pricing/):** no free
tier — the trial is 2 hours of machine runtime or 7 days, whichever first;
card required after that. Pay-as-you-go: `shared-cpu-1x` ~US$2/mo at 256 MB,
~$6/mo at 1 GB (varies by region); volumes $0.15/GB-mo; APAC egress
$0.04/GB; dedicated IPv4 $2/mo.

## User does

1. Sign up at https://fly.io (GitHub or email), add a card.
2. Approve `fly auth login` in the browser.

## Claude does

```bash
brew install flyctl
fly auth login
fly launch --no-deploy      # detects Dockerfile, writes fly.toml — review it, set region (nrt/sin/hkg), memory
fly deploy
fly status && fly logs      # verify
```

Key `fly.toml` decisions to make deliberately, not by default:

- `auto_stop_machines` / `min_machines_running = 0` saves money but sleeps
  the app — **wrong for bots/webhooks/websockets**; set
  `min_machines_running = 1` (or `auto_stop_machines = "off"`) for class B.
- Region list: put the primary near the users, not near the DB dump.

Secrets (encrypted, not in fly.toml):

```bash
fly secrets import < .env.new    # reads KEY=VALUE lines from stdin — nothing echoes
fly secrets list                 # names + digests only, safe to show
```

`fly secrets import` is the cleanest bulk path of any platform here — use the
local `.env.new` file the user filled with rotated keys. Two behaviors to
plan around: every `secrets set`/`import` **restarts the machines** (batch
with `--stage`, then one `fly secrets deploy`), and there is no interactive
prompt mode — values come from args or stdin only. For GitHub Actions
deploys (Fly has no built-in git integration), mint a scoped token with
`fly tokens create deploy -a <app> -x 720h` — always pass `-x`; the default
expiry is 20 years.

## Database

Prefer managed: Neon (Postgres, Singapore region available), Upstash (Redis,
built into `fly redis` too). Restore the Phase 1 dump with
`pg_restore -d "$NEW_URL" --no-owner backup.dump` — for live data follow
../db-migration.md (rehearsal → freeze → final dump → verify), not a
one-shot dump. Fly's own options are a
poor fit for small apps: Managed Postgres starts ~$38/mo, and self-run
Postgres on a machine means backups, upgrades, and failover are on the user —
after a breach, most users want *less* ops surface, not more.

## Domain & cutover

```bash
fly certs add <domain>     # prints the A/AAAA or CNAME records to set
fly certs show <domain>    # wait until issued
```

User updates the records at the registrar; verify with `curl -sI` before
touching Zeabur.

## Gotchas

- `fly launch` sometimes writes a generated Dockerfile for framework apps —
  if the repo already has one (it did on Zeabur), make sure it's used as-is.
- Machines bill while running even with zero traffic; for cron-only work use
  `fly machine run --schedule` instead of an always-on machine.
- No usage cap setting — bound spend by keeping machine count/size explicit;
  tell the user roughly what the monthly bill should read.

## Sources (official, checked 2026-08-29)

- Machine/volume/egress pricing: https://fly.io/docs/about/pricing/
- Trial terms (2 VM-hours or 7 days; card ends trial): https://fly.io/docs/about/free-trial/
- Managed Postgres tiers (from ~$38/mo): https://fly.io/docs/mpg/
- Regions (Tokyo/Singapore/Sydney; no HK/Seoul): https://fly.io/docs/reference/regions/
- Deploy tokens and expiry defaults: https://fly.io/docs/security/tokens/
- Signup methods: https://fly.io/docs/getting-started/sign-up-sign-in/
