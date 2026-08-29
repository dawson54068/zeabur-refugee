# Bulk audit — when the user has many projects (10+)

Reframe first, because it changes the workload by an order of magnitude:
**N projects is not N migrations.** After one sweep, everything falls into
four buckets, and only one bucket involves real migration work:

1. **Dead experiments** (no custom domain, nothing depends on them) →
   delete env vars, then the service. No migration. For most heavy Zeabur
   users this is the majority of projects, and deletion is the fastest
   possible risk elimination.
2. **Alive, holds secrets** → rotate + migrate (the normal SKILL.md phases).
3. **Alive, no secrets** (static sites, toys) → migrate whenever; no urgency.
4. **Databases / volumes with real data** → dump first (zeabur-export.md),
   regardless of which bucket their app is in.

And the rotation insight that makes 30 projects tractable: **API keys belong
to provider accounts, not projects.** Thirty projects typically share a
handful of OpenAI/Anthropic/OpenRouter/… accounts, so rotation work scales
with the number of *providers* (one console visit each: money taps →
evidence → revoke everything old → mint new), not the number of projects.
When minting replacements, create **one key per app** this time — the next
leak then burns one project instead of thirty.

## Step 1 — Sweep (you do this, not the user)

Auth once — user logs in via browser, or creates an API key at
https://zeabur.com/account/api-keys (revoke it in Phase 4):

```bash
npx zeabur@latest auth login            # or: auth login --token <TOKEN>
npx zeabur@latest project ls -i=false
npx zeabur@latest service ls -i=false   # per project
npx zeabur@latest variable list --id <SERVICE_ID> --env-id <ENV_ID> -i=false > svc.env
```

The `variable`/`service` subcommands are verified in the CLI source
(github.com/zeabur/cli) but under-documented — run `--help` first and prefer
any JSON output flag you find; output formats drift between versions.
Redirect variable output straight to local files: you need the *names* for
classification; the values stay out of the transcript.

If the CLI fights the loop, use the GraphQL API with the user's token:
introspection is blocked unauthenticated (checked 2026-08-29) but works with
`Authorization: Bearer <TOKEN>` — introspect first, then build the
project → services → variables query from the live schema rather than from
memory:

```bash
curl -s -X POST https://api.zeabur.com/graphql \
  -H "Authorization: Bearer $ZEABUR_TOKEN" -H 'Content-Type: application/json' \
  -d '{"query":"query{__schema{queryType{fields{name}}}}"}'
```

## Step 2 — One inventory table

Produce a single table (CSV or markdown; keep it local) with one row per
service: project · service · source (repo / template / image) · env var
NAMES · which names are secrets · DB? · volume? · custom domain? Then mark
each row with its bucket (1–4). Show the user the table and the bucket
counts — "22 deletable, 5 migrate, 3 DB dumps" is the moment the task stops
feeling like thirty separate chores.

Aggregator apps (one-api/new-api, LiteLLM, multi-provider chat UIs) get
flagged regardless of bucket — biggest blast radius, and their *database*
holds more keys than their env block (see zeabur-export.md).

Same-named vars across projects are usually the *same* key. Confirm by
hashing locally — compare hashes, never print values:

```bash
grep '^OPENAI_API_KEY=' svc-a.env | cut -d= -f2- | shasum -a 256
```

## Step 3 — Rotate per provider, delete the dead, dump the data

- One pass per provider console, in key-rotation.md's order (money taps →
  evidence → revoke → new per-app keys). The provider list is just the union
  of secret names in the table.
- Bucket 1 gets deleted the same sitting (env vars first, then service —
  each deletion confirmed with the user; 2-hour grace period applies).
- Bucket 4 dumps run meanwhile (they're mostly waiting on `pg_dump`).

## Step 4 — Migrate the survivors in class batches

Group bucket-2/3 rows by workload class (SKILL.md Phase 1) and migrate as
batches — all template apps to Railway/Coolify in one session reuses the
same login, the same patterns, the same verification loop; all static sites
to Cloudflare in another. Per-target steps live in references/targets/.

## Realistic pacing for ~30 projects

Sweep + table: under an hour, mostly you. Provider rotation: 1–2 hours,
mostly the user in consoles with you feeding exact URLs. Dead-project
deletion: ~30 minutes. Actual migrations: usually only a handful of
services — spread over following days by priority (custom domains and
production traffic first).
