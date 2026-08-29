---
name: zeabur-refugee
description: >-
  Migrate deployments off Zeabur to Vercel, Cloudflare, Railway, Render, Fly.io,
  or self-hosted Coolify/Dokploy — including emergency key rotation when secrets
  stored on Zeabur (LLM provider API keys, database passwords) may be compromised.
  For anyone who calls themselves a Zeabur refugee (Zeabur 難民), or asks in
  zh-TW about 搬離 Zeabur / Zeabur 金鑰外洩 / 環境變數外洩.
  Use this skill whenever the user mentions leaving Zeabur, a Zeabur security
  incident or breach, moving a Zeabur-hosted app or template (n8n, one-api,
  new-api, LobeChat, NextChat, a Next.js app, a Discord/Telegram bot) to another
  platform, rotating API keys that were stored on Zeabur, comparing Zeabur
  alternatives, auditing or triaging many Zeabur projects at once ("I have 30
  projects there, checking one by one"), or asks "where should I host this
  instead of Zeabur" — even if they don't use the word "migrate".
---

# Zeabur Migration

Help a Zeabur user move their deployments to a safer platform and contain the
damage if their secrets were exposed. Many Zeabur users deployed one-click
templates (LobeChat, one-api, n8n) and are not professional developers — assume
nothing about their skill level, use plain language, and take the work off their
plate wherever a CLI or file edit can do it. The user should only ever be asked
to do things that genuinely require their identity, their browser session, or
their payment details.

Work one phase at a time. Tell the user which phase you are in and what remains.

## Phase 0 — Triage (always do this first)

Establish two facts before anything else:

1. **Are secrets at risk?** Ask whether API keys, database passwords, or other
   secrets were stored in Zeabur environment variables. Context: Zeabur
   officially confirmed unauthorized access to project environment variable
   data on 2026-08-27/28 (status.zeabur.com incident "Unauthorized Access to
   Project Environment Variable Data"); Anthropic, OpenAI, and OpenRouter keys
   were abused in the wild. Zeabur states account passwords and payment data
   were NOT accessed — so logging into the dashboard to export is fine. If the
   user had secrets in env vars, treat **every secret that ever lived on
   Zeabur as compromised**. Read
   [references/key-rotation.md](references/key-rotation.md) and start the
   rotation track immediately. Key rotation beats migration in urgency: a
   stolen OpenAI key burns money every minute; a slow migration costs nothing.
2. **What is running on Zeabur?** A git-deployed app they own? A one-click
   template? A database? Several services? This determines the migration path.

If secrets are at risk, run Phase 0.5 before (or in parallel with) everything
else. Otherwise skip to Phase 1.

## Phase 0.5 — Contain (compromised keys)

Follow [references/key-rotation.md](references/key-rotation.md). Summary of the
order, and why:

1. **Check for abuse now** — have the user open each provider's usage page
   (exact URLs in the reference). Evidence of abuse → revoke that key
   immediately, even though it takes the Zeabur service down. Downtime is
   recoverable; a drained account or a poisoned account reputation is worse.
2. **No abuse yet →** create NEW keys, keep them local (never in chat), deploy
   the new platform with the new keys, verify, then revoke the old keys and
   scrub Zeabur. This avoids downtime while still closing the window.
3. Rotate **everything**, not just LLM keys: database passwords, OAuth client
   secrets, webhook signing secrets, `ACCESS_CODE`/admin passwords of template
   apps. Attackers read whole env blocks, not single variables.

## Phase 1 — Inventory

Read [references/zeabur-export.md](references/zeabur-export.md) for how to get
data out of Zeabur (dashboard paths, CLI, API, database dumps).

**More than a handful of projects?** Read
[references/bulk-audit.md](references/bulk-audit.md) first and run the sweep
yourself instead of walking the dashboard together: one CLI/API pass over
every project, one inventory table, then rotate per *provider* (not per
project), delete the dead projects outright, and migrate only what's alive.
Thirty projects is usually one sweep + a handful of console visits + a few
real migrations — not thirty migrations.

Build a small table with the user and keep it updated through the migration:

| Service | Type (git app / template / DB) | Data to move | Secrets in env | Domain |

Classify each service into one of these workload classes — the class picks the
target platform:

- **A. Static or frontend-framework site** (Next.js/Astro/Nuxt, no persistent
  server process)
- **B. Long-running server** (Express/Fastify/Hono server, WebSocket, Discord
  or Telegram bot, any Dockerfile service)
- **C. One-click template app** (LobeChat, one-api/new-api, n8n, Umami …) —
  these are Docker images with env vars, sometimes plus a database
- **D. Database or stateful store** (Postgres/MySQL/Redis/Mongo, volumes)
- **E. Cron / background job**

## Phase 2 — Choose the target

Recommend, don't interrogate: propose one primary target per service with a
one-line reason and cost, and mention at most one alternative. Ask only about
budget (free vs ~$5/mo ok) and whether they want "click a dashboard" or "own a
server" if it actually changes the recommendation.

| Workload | First choice | Why | Also fine |
|---|---|---|---|
| A. Static / frontend framework | Cloudflare (free, no spin-down) | Free tier is real and commercial use is allowed | Vercel (nicest Next.js DX; Hobby is non-commercial) |
| B. Long-running server / Dockerfile | Railway (~$5–15/mo) | Closest to Zeabur: git deploy, Dockerfile, usage pricing, Singapore region | Fly.io, Render |
| C. Template app (one-api, LobeChat, n8n …) | Railway template, or self-host Coolify/Dokploy on a VPS | Railway has most of the same one-click templates; Coolify replicates the whole Zeabur experience on a ~€6 Hetzner VPS the user controls | Render |
| D. Database | Managed DB near the app: Railway (only one with managed MySQL/Mongo too), Neon Postgres, Upstash Redis | Vercel/Cloudflare don't host classic always-on databases | Keep on a VPS with Coolify |
| E. Cron / worker | Same platform as the app it belongs to | Fewer moving parts | Cloudflare Workers cron for tiny jobs |

Hard constraints that veto a target (details + pricing in each target file):

- **Vercel / Cloudflare Workers are not container hosts.** No long-lived
  processes, no persistent disk, no first-party databases. Cloudflare
  Containers (GA 2026) runs Docker images but with ephemeral-only disk and
  HTTP-only ingress — see the target file. Class B/C/D workloads should not
  go to either unless rewritten.
- **Vercel Hobby (free) forbids commercial use** — broadly (ads, donations,
  client work all count). Anything revenue-touching needs Pro ($20/user/mo)
  — often the deciding factor.
- **Render free tier spins down** after 15 min idle, and free Postgres
  self-deletes after 30 days; fine for demos, wrong for bots and webhooks.
- **Fly.io has no free tier** — card required, pay-as-you-go (~$2–6/mo
  small machine).
- **Asia latency matters to most Zeabur users.** Prefer targets with
  Tokyo/Singapore regions or an edge network; note region choice explicitly.

Every price and constraint above is evidence-backed: each target file ends
with a **Sources** section of official links (checked 2026-08-29). When
quoting a number to the user, cite its source link; if a claim matters to
their decision and the link's content has changed, the current page wins —
update the recommendation, not the user's expectations.

Then read the matching target file — only the one(s) actually chosen:

- [references/targets/vercel.md](references/targets/vercel.md)
- [references/targets/cloudflare.md](references/targets/cloudflare.md)
- [references/targets/railway.md](references/targets/railway.md)
- [references/targets/render.md](references/targets/render.md)
- [references/targets/fly.md](references/targets/fly.md)
- [references/targets/self-host.md](references/targets/self-host.md) (Coolify/Dokploy on a VPS)

## Phase 3 — Migrate

Universal order, regardless of target — the new deployment must be alive and
verified before anything on Zeabur is touched:

1. Export from Zeabur: env var names (values only if still needed and not
   secret-compromised), database dumps, config. Per
   [references/zeabur-export.md](references/zeabur-export.md). **If a
   database has live traffic, a single dump is not a migration** — follow
   [references/db-migration.md](references/db-migration.md)
   (rehearsal → write-freeze → final dump → verify → 7-day rollback window);
   writes made after an unfrozen dump are silently lost.
2. Prepare the project for the target (config files, Dockerfile tweaks) — do
   this yourself; explain what changed in one or two sentences.
3. Walk the user through the target's signup + CLI login (their part), then
   deploy (your part). The target file has the exact commands and current UI
   paths.
4. Set env vars on the target using NEW secrets — via CLI, never by pasting
   values into the conversation (see Secret hygiene below).
5. Verify: hit the deployed URL / run the app's own health check / send a test
   message to the bot. Show the user it works before going further.

## Phase 4 — Cut over and decommission

1. Move the domain: add it on the target, update DNS at the registrar (user
   does this in their registrar UI unless there's API access; give exact
   records to change). Remind them DNS TTL means both platforms serve traffic
   for a while.
2. Confirm the domain serves from the new platform.
3. Only now revoke old keys (if not already revoked in Phase 0.5).
4. Scrub Zeabur: delete env vars first, then the services, then (user's call)
   the account. These steps are destructive — list exactly what will be
   deleted and get an explicit yes before each.

## Division of labor — remove the hassle

Do everything that can be done with files, CLIs, and code. The user's job list
should end up this short:

**Only the user can (guide them precisely, one step at a time):**

- Create accounts / sign in (OAuth happens in their browser). Give the exact
  URL and what to click; tell them which signup method to prefer and whether a
  credit card will be asked.
- Approve CLI logins (`vercel login`, `wrangler login`, `railway login`,
  `fly auth login` open a browser — tell them what the approval screen looks
  like and to come back to the terminal when it says success).
- Enter payment details, choose paid plans.
- Create/revoke API keys in provider consoles (their identity). Give the exact
  URL, the exact button, and where to put the new key (a local file or CLI
  prompt — never the chat).
- Change DNS records at their registrar.
- Say yes to destructive steps.

**Never do, even if asked:** create accounts on the user's behalf, click
through provider consoles for them with their session unless they explicitly
hand over a browser tab and understand what will be done, accept payment
details in chat, or store their secrets anywhere outside their machine.

If a dashboard's UI doesn't match the instructions in a target file (UIs
change), don't insist — search the dashboard's settings for the landmark terms
given in the reference, or check the platform's current docs with web search,
then update the user with the corrected path.

## Secret hygiene (applies to every phase)

- Never ask the user to paste secret values into the conversation, and never
  print them in command output (`set -x`, `env`, `cat .env` are all leaks —
  transcripts may be stored). When showing env state, show names only.
- New secrets go into a local `.env` file the user edits themselves, or
  directly into an interactive CLI prompt (`wrangler secret put KEY`,
  `vercel env add KEY` prompt for the value without echoing it to the
  transcript — prefer these).
- Ensure `.env*` is gitignored before any commit; check, don't assume.
- Old Zeabur-era secrets are compromised by assumption: never copy them to the
  new platform. A migration that moves stolen keys to a new host has migrated
  the problem, not solved it.

## Reference index

| File | Read when |
|---|---|
| references/key-rotation.md | Any chance secrets were exposed (Phase 0/0.5) |
| references/zeabur-export.md | Inventorying or exporting anything from Zeabur (Phase 1/3) |
| references/bulk-audit.md | The user has many projects (10+) — sweep and triage before anything else |
| references/db-migration.md | Any database with live traffic is moving (Phase 3) — the no-data-loss procedure |
| references/targets/*.md | A target platform is chosen (Phase 2/3) — only the chosen one |
