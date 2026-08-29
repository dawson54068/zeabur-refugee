# Getting everything out of Zeabur

Goal: a complete inventory (services, env vars, data, domains) with minimal
clicking. Zeabur confirmed the 2026-08 breach did **not** touch account
passwords or payment data, so using the dashboard and CLI is fine. If you
create a Zeabur API key for this migration, revoke it as the last step —
it's full-access (no scopes) on a platform the user is leaving.

Facts below verified 2026-08 against zeabur.com/docs and the CLI source
(github.com/zeabur/cli). If a command errors, check `npx zeabur@latest --help`
— trust the binary over this file.

## Fast path: one bulk export per service

**Env vars — dashboard:** service → **Variables** tab → **"Edit as Raw"**
button: all variables as one `.env`-style blob. Have the user copy it into a
local file (e.g. `zeabur-backup/<service>.env`) themselves — it is full of
compromised-but-still-sensitive values; keep it out of the chat and delete it
after migration. You mainly need the *names* and the non-secret values.

**Env vars — CLI** (these subcommands ship in the binary but are missing from
the docs):

```bash
npx zeabur@latest auth login                     # browser OAuth
npx zeabur@latest project ls
npx zeabur@latest service ls
npx zeabur@latest variable list --id <SERVICE_ID> --env-id <ENV_ID>
```

Add `-i=false` for non-interactive use; `-n <name>` works instead of `--id`.
⚠️ `variable env` is an *import* (writes a local `.env` file's contents TO
Zeabur, overwriting) — never run it against the old service expecting an
export. `variable list` prints values to stdout — redirect to a file
(`> file.env`) rather than letting values land in the transcript.

**Whole-project topology:** Project → Settings → General → **Export** →
"Template Resource YAML", or:

```bash
npx zeabur@latest project export --name <PROJECT_NAME> > project.yaml
```

Captures services, images/repos, env var structure, volumes — a map of what
must be recreated. Docs don't say whether secret *values* are embedded:
treat the YAML as secret-bearing until you've looked. It does NOT include
data (databases, volume contents) — those move separately, below.

## Inventory checklist (per service, while you're in there)

- **Source:** GitHub repo+branch, prebuilt image, or template — shown on the
  service page.
- **Domains:** service → Networking: note `*.zeabur.app` domains (they die
  with the service — find what links to them) and custom domains (move in
  Phase 4). Domains bought *through* Zeabur are managed under the
  account-level Domain section — those need their DNS repointed, or a
  transfer out, before the account is abandoned.
- **Cron / start command / build settings, volume mounts:** service settings.

## Databases — dump before touching anything

This section covers *taking* a dump. If the database serves a live app, a
dump alone is not a migration — db-migration.md has the full
no-data-loss procedure (rehearsal → write-freeze → final dump → verify →
7-day rollback window); the dump here is its first step.

1. Connection details: service → **Instruction** tab → **"Public
   (External)"** section (connection string + ready-made command). If public
   access is off, enable it just long enough for the dump:

```bash
npx zeabur@latest service port-forward --id <SERVICE_ID> --enable
# ... dump ...
npx zeabur@latest service port-forward --id <SERVICE_ID> --disable   # immediately after
```

2. Dump with the engine's own tool (needs client installed locally —
   `brew install libpq mysql-client` etc.):

```bash
pg_dump "$ZEABUR_PG_URL" -Fc -f backup.dump
mysqldump --single-transaction -h <host> -P <port> -u root -p <db> > backup.sql
redis-cli -u "$ZEABUR_REDIS_URL" --rdb backup.rdb    # often skippable: pure caches can be dropped
mongodump --uri "$ZEABUR_MONGO_URL" --archive=backup.archive
```

3. Verify the dump is non-trivial (`ls -lh`; count tables) before trusting it.
4. Alternative when opening a port feels wrong: service → **Backup** tab →
   Backup → download when complete. DB engines back up online; retention is
   only 7 days, so download immediately.

## Volumes / app data (n8n workflows, uploaded files …)

- **Backup tab** (service → Backup) covers mounted volume data too;
  non-database services need Settings → **Suspend Service** first for a
  consistent offline backup. Download the archive — 7-day retention.
- **File Management** (service → Overview → Files): per-file download via
  hover → Download. For directories, `tar` them first via the service's
  command execution, then download the tarball.
- Prefer the app's own export where one exists (n8n: Settings → Download
  workflows; note n8n data is useless without `N8N_ENCRYPTION_KEY` from the
  env block — capture it).

## Teardown (Phase 4 only — after the new platform is verified)

In this order, each with explicit user confirmation:

1. Delete env vars from each service (blank everything in "Edit as Raw", or
   `variable delete`) — removes secrets from the breached platform even
   before the service dies.
2. `port-forward --disable` anywhere it was enabled; suspend services
   (Settings → Suspend Service) as a reversible first step.
3. Delete services, then projects (`zeabur service delete`,
   `zeabur project delete`) — irreversible after a 2-hour grace period.
4. Revoke API keys at https://zeabur.com/account/api-keys; user changes
   their Zeabur password regardless of Zeabur's "accounts not affected"
   statement (cheap insurance).
5. Account deletion has **no self-serve path** — it goes through
   https://zeabur.com/support. "Kept account, zero services, revoked keys"
   is a fine end state, and preserves billing history for compensation
   claims about the incident.

## Common templates — what they imply for migration

High-deploy-count Zeabur templates that hold third-party LLM keys in env
vars: OpenClaw, n8n, gcli2api, SillyTavern, LobeChat, One-API, Dify,
LibreChat, ChatNio, Open WebUI, NextChat, LiteLLM (live deploy counts:
https://zeabur.com/templates). Key-aggregator apps (One-API, LiteLLM, and
chat UIs configured with several providers) mean one leaked env block = a
whole ring of provider keys — rotate every provider the app was configured
with, and for One-API/new-api also rotate the keys stored in its *database*
(channels table), not just the env block.

## Sources (official, checked 2026-08-29)

- Env vars & "Edit as Raw": https://zeabur.com/docs/en-US/deploy/config/environment-variables
- CLI docs (variable/port-forward subcommands verified in source): https://zeabur.com/docs/en-US/developer/cli and https://github.com/zeabur/cli
- GraphQL API + API keys: https://zeabur.com/docs/en-US/developer/public-api , https://zeabur.com/docs/en-US/developer/use-api-key
- DB connection details (Instruction tab): https://zeabur.com/docs/marketplace/postgresql
- Public networking / port forwarding: https://zeabur.com/docs/en-US/deploy/networking/public-networking
- Backups (downloadable, 7-day retention): https://zeabur.com/docs/en-US/operations/data/backup-restore
- File management download: https://zeabur.com/docs/en-US/operations/data/file-management
- Volumes: https://zeabur.com/docs/en-US/operations/data/volumes
- Project export YAML: https://zeabur.com/docs/en-US/deploy/manage/export-project
- Suspend service: https://zeabur.com/docs/en-US/operations/deployment/suspend-service
- Domains/DNS management: https://zeabur.com/docs/en-US/domain/management , https://zeabur.com/docs/en-US/domain/dns
