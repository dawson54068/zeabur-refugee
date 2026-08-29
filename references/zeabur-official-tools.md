# Zeabur official tools — when the website is slow

Use these before asking the user to click around the dashboard. All items here
come from official Zeabur surfaces checked 2026-08-30: the official CLI repo
`zeabur/cli` (latest release v0.21.0, published 2026-07-22; repo pushed
2026-08-25), Zeabur's public API docs, and schema host.

## 1. Official CLI — main tool for bulk audit

Repo: https://github.com/zeabur/cli  
Install/run without permanent install:

```bash
npx zeabur@latest version
npx zeabur@latest auth login
```

Global flags from source:

```bash
--json          # machine-readable output where supported
-i=false        # non-interactive mode where supported
--workspace     # act under a specific workspace
--debug         # debug logs
```

For a slow dashboard, this is the first path to try:

```bash
npx zeabur@latest auth login
npx zeabur@latest workspace list
npx zeabur@latest project list --json
npx zeabur@latest service list --project-id <PROJECT_ID> --json
npx zeabur@latest variable list --id <SERVICE_ID> --env-id <ENV_ID> -i=false > <service>.env
npx zeabur@latest project export --name <PROJECT_NAME> > <project>.zeabur.yaml
```

Important safety notes:

- `variable list` can print secret **values**. Redirect to a local file and
  never paste it into chat. For triage, parse names only.
- `variable env` is **import**, not export — it updates remote variables from
  a local `.env` file.
- `project export` writes Resource YAML. Treat it as secret-bearing until
  inspected; docs do not state whether env values are included.

### CLI command groups useful for migration

| Group | Commands | Use in zeabur-refugee |
|---|---|---|
| `auth` | `login`, `logout`, `status` | login once, verify auth |
| `workspace` | `list`, `current`, `switch`, `clear` | handle personal/team workspaces |
| `project` | `list`, `get`, `create`, `delete`, `clone`, `export` | enumerate projects, export topology, delete only after confirmation |
| `service` | `list`, `get`, `instruction`, `network`, `metric`, `exec`, `port-forward`, `suspend`, `restart`, `redeploy`, `delete`, `deploy`, `update tag` | enumerate services, get DB instructions, open/close DB forwarding, suspend app for live DB migration |
| `variable` / `var` | `list`, `create`, `update`, `delete`, `env` | list env names, delete compromised env vars, never use `env` as export |
| `deployment` / `deploy` | `list`, `get`, `log` | inspect deploy history/logs without dashboard |
| `domain` | `list`, `create`, `delete`, `dns`, `registrant`, `verification`, `auto-renew`, `purchase`, `renew` | list domains and DNS records; avoid purchase/renew unless user explicitly asks |
| `template` | `list`, `search`, `get`, `deploy`, `create`, `update`, `delete` | identify one-click template apps and their migration shape |
| `file` | `pull` | uploaded project files only; not reliable for volumes |
| `server` | `list`, `get`, `ssh`, `ssh-info`, `exec`, `catalog`, `providers`, `regions`, `plans`, `rent`, `reboot`, `rename` | dedicated-server products; useful only if the user used Zeabur servers |
| `ai-hub` | `status`, `usage`, `keys list/create/delete`, `auto-recharge`, `add-balance` | check/disable Zeabur AI Hub exposure if the user used it |
| `email` | domains, keys, send, records, scheduled, webhooks, batch | rotate/delete Zeabur Email API keys and inspect email usage if applicable |

For exact flags, use CLI help instead of website docs:

```bash
npx zeabur@latest help --all
npx zeabur@latest service port-forward --help
npx zeabur@latest variable list --help
npx zeabur@latest project export --help
```

## 2. Public GraphQL API — scriptable fallback

Docs: https://zeabur.com/docs/en-US/developer/public-api  
Endpoint:

```text
https://api.zeabur.com/graphql
```

Auth:

```http
Authorization: Bearer <ZEABUR_API_TOKEN>
```

Token page:

```text
https://zeabur.com/account/api-keys
```

Use this when CLI output is hard to automate or the user has many projects.
The API token appears full-access; create it only for the migration and revoke
it afterward.

Introspect with the user's token, then build queries from the live schema:

```bash
curl -s -X POST https://api.zeabur.com/graphql \
  -H "Authorization: Bearer $ZEABUR_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"query":"query{__schema{queryType{fields{name}}}}"}'
```

Unauthenticated introspection returned an error when checked, so don't assume
schema discovery works without a token.

Useful API surfaces confirmed from docs/source:

- GraphQL queries/mutations for owned resources (`ownerID` appears in many
  calls).
- `ListVariables(ctx, serviceID, environmentID)` in the CLI API package —
  returns two variable maps (likely user-defined vs injected/readonly; verify
  on a test service).
- `executeCommand` mutation and `wss://api.zeabur.com/exec/<service-id>` for
  service-container command execution.
- WebSocket subscriptions at `wss://api.zeabur.com/graphql`.

## 3. REST file surface — direct file operations

Official API exposes a file surface in addition to GraphQL:

```text
GET  /projects/{project-id}/services/{service-id}/files?path=<PATH>&environment=<ENVIRONMENT>
POST /projects/{project-id}/services/{service-id}/files?path=<PATH>&environment=<ENVIRONMENT>
```

Use only for small file inspection/export. For volume directories, prefer the
Backup tab or `service exec -- tar ...` then download. The documented upload
limit is 100 MB.

## 4. Schema host — config/template validation without dashboard

Schema index: https://schema.zeabur.app/

Available schemas:

| Schema | Use |
|---|---|
| `https://schema.zeabur.app/prebuilt.json` | prebuilt-service definitions |
| `https://schema.zeabur.app/template.json` | Zeabur template YAML validation |
| `https://schema.zeabur.app/zbpack.json` | `zbpack.json` project config |
| `https://schema.zeabur.app/upload-api/openapi.json` | upload API OpenAPI spec |

These help parse/export projects without loading the dashboard UI. They do not
include the GraphQL project/service/variable schema.

## 5. Backups and data export tools

Dashboard is still the official backup UI, but the data path can avoid most
browsing:

- DB connection details: `service instruction` in CLI, or service →
  Instruction tab → Public (External).
- Temporary public TCP forwarding:

```bash
npx zeabur@latest service port-forward --id <SERVICE_ID> --enable
# dump with pg_dump / mysqldump / mongodump / redis-cli
npx zeabur@latest service port-forward --id <SERVICE_ID> --disable
```

- Service command execution:

```bash
npx zeabur@latest service exec -- tar -czf /tmp/export.tgz /path/to/data
```

- Backup tab downloads remain the safest official path for volumes and DBs
  when available; retention is 7 days. If the site is slow, use CLI/API for
  inventory first, then only open the dashboard for the few backup downloads
  that need it.

## Recommended bulk-check flow for 30+ projects

1. Login once with the CLI.
2. List workspaces, projects, and services.
3. Export each project's Resource YAML.
4. For each service, list variables to a local file, then parse **names only**.
5. Mark services with DB/volume/custom domain/template/source repo.
6. Produce one table: delete / rotate+migrate / later / data-dump-first.
7. Only open Zeabur's website for the few actions the CLI/API can't safely do
   or where the user must visually confirm.

This is the path that avoids the slow website while staying on official
Zeabur tooling.
