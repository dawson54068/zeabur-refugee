# Key rotation — containing exposed Zeabur secrets

## The incident (what to tell the user, with sources)

Officially confirmed by Zeabur: on 2026-08-27/28 an attacker used a stolen
internal service credential to read **project environment variable data**
(incident "Unauthorized Access to Project Environment Variable Data",
https://status.zeabur.com/incident/1037896). Confirmed in scope: API keys and
credentials for OpenAI, Anthropic, AWS, GitHub, Stripe, and database
connection strings; press coverage adds OpenRouter, Gemini, Cloudflare, and
JWT secrets (INSIDE: https://www.inside.com.tw/article/42241-zeabur-environment-variable-leak-api-keys-stolen ,
電腦王阿達: https://www.kocpc.com.tw/archives/666953 ,
動區: https://www.blocktempo.com/zeabur-environment-variable-leak-openai-anthropic-api-key-stolen-compensation/ ). **Anthropic, OpenAI, and OpenRouter keys were actually abused in
the wild** — victims often noticed via card declines or auto-recharge alerts
before any notification. Zeabur states account passwords, personal info, and
payment/credit-card data were **not** accessed, so logging into the Zeabur
dashboard to export and clean up is reasonable. Zeabur accepts compensation
claims with evidence (timestamps, amounts/token usage, source IPs) at
https://zeabur.com/support — capture evidence *before* revoking, then file.

## Working assumption

Assume every value that ever sat in a Zeabur environment variable is in an
attacker's hands: LLM API keys, database URLs (which embed passwords), bot
tokens, OAuth client secrets, admin passwords, webhook signing secrets.
Zeabur's own warning applies everywhere: creating a new key does **not**
invalidate the old one — revocation is always a separate, explicit step.

## Order of operations

1. **Kill the money taps first.** Auto-recharge turns a stolen key into an
   unbounded loss: with recharge on, the attacker drains the balance, the
   card refills it, repeat. Before revoking anything: Together AI — zero the
   auto-recharge thresholds on the billing page (there is no off button;
   clearing thresholds or switching default payment to ACH is the disable);
   xAI — set the invoiced spending limit to $0; OpenRouter — check workspace
   budget; DeepSeek — note the prepaid balance is the entire damage cap.
2. **Capture abuse evidence.** Open each provider's usage page (URLs below).
   Unrecognized spikes, unfamiliar models, odd hours, unknown source IPs
   (xAI shows request IPs — best forensics of the lot) → screenshot with
   timestamps and amounts, for the provider dispute AND the Zeabur
   compensation claim.
3. **Revoke — prefer reversible disable where it exists** (Anthropic
   "Disable", OpenRouter key `disabled`, xAI "Disable key"): under incident
   pressure a reversible action beats an irreversible one; delete once
   confirmed. If abuse is active, disable now and accept downtime. If not,
   the no-downtime order is: create NEW key → deploy new platform → verify →
   revoke old. Either way the old key dies today.
4. **Verify the revocation landed.** Not always instant: Azure can honor old
   keys for minutes-to-hours (regenerate BOTH keys, then poll with the old
   one until 401); xAI exposes a propagation endpoint; Google key deletion
   propagates in minutes (and is restorable for 30 days — good for
   fat-fingers, but "deleted" ≠ instantly gone).
5. **Never deploy the old key to the new platform** — that migrates the
   compromise, not the app. Rotation and migration happen together.
6. **Bound the next incident while in each console:** hard spend limits
   where they exist (OpenAI org/project limits genuinely stop traffic;
   Google budgets only *alert* — use quotas for a real cap), and per-key
   scoping where offered (OpenAI restricted keys, xAI ACLs, OpenRouter
   guardrails).
7. **Check for persistence:** unknown API keys the user didn't create, added
   org members, changed billing email. Also have the user change the
   provider account password + enable 2FA if any *passwords* sat in env vars.

## LLM providers — verified consoles (2026-08)

Priority order — auto-recharge and aggregator keys first:

| Provider | API keys | Usage / abuse check | Per-key usage? |
|---|---|---|---|
| Together AI | https://api.together.ai/settings/projects/~current/api-keys | …/cost-analytics (same settings area) | beta |
| OpenRouter | https://openrouter.ai/workspaces/default/keys | https://openrouter.ai/activity | yes |
| OpenAI | https://platform.openai.com/settings/organization/api-keys | https://platform.openai.com/account/usage | yes |
| Anthropic | https://platform.claude.com/settings/keys | https://platform.claude.com/usage | yes |
| Google Gemini | https://aistudio.google.com/api-keys (delete via https://console.cloud.google.com/apis/credentials) | Cloud Billing console | no — per project |
| xAI | https://console.x.ai → API Keys (Disable, then Delete) | console.x.ai team usage (filter by key + request IP) | yes |
| Azure OpenAI | Portal → resource → Keys and Endpoint → regenerate **both** | Cost Management + resource Metrics | no — KEY1/KEY2 indistinguishable |
| Groq | https://console.groq.com/keys | https://console.groq.com/dashboard/usage | no — per project |
| DeepSeek | https://platform.deepseek.com/api_keys | balance: `GET api.deepseek.com/user/balance` (no per-key usage exists) | no |
| Mistral | https://console.mistral.ai/api-keys (org: admin.mistral.ai) | https://admin.mistral.ai/organization/usage | no |

Provider quirks that matter under pressure:

- **OpenRouter is the biggest blast radius** — one key spends across every
  upstream provider. Rotate it first among the aggregators the user has.
- **Together AI:** legacy keys **cannot be revoked, only regenerated** — if
  the leaked key is legacy, regenerate immediately; and keys created by
  since-removed collaborators survive offboarding — list all keys, not just
  the user's own.
- **Anthropic:** key expiration is fixed at creation; give the replacement
  key an expiry rather than "Never" this time.
- **DeepSeek has no per-key controls at all** — no scoping, no limits, no
  audit. The prepaid balance is the only damage cap: keep it small.
- **Providers without per-key attribution** (Google, Groq, Mistral,
  DeepSeek, Azure): the user can't prove *which* key was abused — check
  project/account-level usage for the whole exposure window (Aug 27 onward).

If a console URL 404s, the provider moved it — search the console for "API
keys" / "Usage" or web-search the current path; don't walk the user through
a stale menu.

## Non-LLM secrets that were also in the env block

Rotate these too — attackers export whole env blocks, and several of these
are confirmed in the breach scope:

- **AWS keys** (confirmed in scope): IAM console → user → Security
  credentials → deactivate + delete, create new. Check CloudTrail for calls
  the user didn't make; stolen AWS keys become crypto-mining fleets within
  hours — this outranks even LLM keys if present.
- **Stripe keys** (confirmed in scope): https://dashboard.stripe.com/apikeys
  → roll the secret key (roll = replace + expire old). Review request logs
  and payouts.
- **GitHub tokens / deploy keys** (confirmed in scope):
  https://github.com/settings/tokens — delete and re-issue; check repo
  deploy keys, webhook secrets, and
  https://github.com/settings/security-log for actions the user didn't take.
- **Database URLs / passwords:** a Zeabur-hosted DB dies with the migration
  — the new DB gets a new password; keep the old one's public port closed
  until deleted. An *external* DB (Neon, Supabase, Atlas…) survives the
  migration — reset its password in that provider's console now.
- **Telegram bot token:** BotFather → `/mybots` → the bot → API Token →
  Revoke. Instant kill — coordinate with the redeploy.
- **Discord bot token:** https://discord.com/developers/applications → app →
  Bot → Reset Token.
- **OAuth client secrets** (Google/GitHub login on template apps):
  regenerate in the OAuth app's console; update the new deployment's env.
- **App-level admin passwords / `ACCESS_CODE` / `JWT_SECRET`** (LobeChat
  access code, n8n basic auth + `N8N_ENCRYPTION_KEY`, one-api/new-api admin
  login): change after redeploy. For one-api/new-api/LiteLLM, the
  *downstream* provider keys stored in the app's own database are separately
  at risk — rotate every provider configured inside the app too.
- **SMTP / payment / webhook secrets** (Resend, LINE, Stripe webhooks …):
  provider console, roll, update env.

## Closing the loop

1. Delete env vars from the Zeabur service *before* deleting the service.
2. Change the Zeabur account password and revoke Zeabur API keys
   (zeabur.com/account/api-keys) — cheap insurance regardless of Zeabur's
   "accounts not affected" statement.
3. If any of these secrets ever lived in a git repo as well, deleting from
   the working tree isn't enough — scan history (`gitleaks detect
   --log-opts="--all"`, or `trufflehog git file://. --results=verified`,
   which also tells you which leaked keys are *still live*) and purge after
   rotation.
4. Calendar note ~1 week out: re-check each provider's usage page once more
   — attackers sometimes sit on keys before using them.
