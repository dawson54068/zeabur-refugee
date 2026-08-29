# Self-host with Coolify (or Dokploy) on a VPS

The closest thing to "Zeabur, but nobody else holds your secrets". Coolify and
Dokploy are open-source PaaS dashboards: git deploys, Docker/Compose services,
one-click templates (n8n, Umami, LobeChat …), databases, automatic HTTPS. The
user owns the server, so this is the right answer when the breach broke their
trust in third-party PaaS — and the wrong answer if they never want to think
about a server again (updates and backups become their job; say this plainly).

**Fit:** every workload class (A–E), including databases with volumes — the
only target in this skill with no runtime constraints. Especially good for
template apps (class C) and "several small services + a DB" setups where PaaS
per-service pricing adds up.

**Cost (verified 2026-08):** the VPS only. Hetzner CX23 (2 vCPU / 4 GB /
40 GB NVMe) is **€5.49/mo + ~€0.50 IPv4** (prices as of the June 2026
adjustment; https://www.hetzner.com/cloud). Hetzner has Singapore, but its
included traffic there is far smaller than in the EU — officially 0.5–5 TB/mo
depending on plan, then $8.49/TB (https://www.hetzner.com/cloud-singapore/);
budget for that before hosting bandwidth-heavy apps there. Alternatives with
Asia regions: Vultr/Linode Tokyo, DigitalOcean Singapore (DO's 2 vCPU / 4 GB
Droplet is a pricier $24/mo). Coolify self-hosted is free with unlimited
servers; Dokploy is free OSS (its paid cloud tier is unnecessary here).

## User does

1. Create the VPS account, add payment, create the smallest 4 GB Ubuntu LTS
   server in a nearby region, and add an SSH key (offer to generate one:
   `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_vps`; they paste the `.pub`
   into the provider's form).
2. Give you the server IP.
3. Point DNS: an `A` record for the app domain (and a wildcard or one record
   per app) to that IP, at their registrar.

## Claude does

Everything else, over SSH from the user's machine:

```bash
ssh root@<ip>   # confirm access works first
# Coolify (recommended default — larger template catalog, bigger community):
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
# Dokploy (lighter alternative):
curl -sSL https://dokploy.com/install.sh | sh
```

The installer prints a dashboard URL (`http://<ip>:8000` for Coolify,
`:3000` for Dokploy). The user opens it and sets the admin password
themselves — that password never goes through the chat.

Then, in the dashboard (guide by landmark, the UI evolves):

1. **Git app:** add the GitHub repo (user authorizes the GitHub App
   integration in their browser), pick the branch, set build (Nixpacks or
   Dockerfile — same choice Zeabur made), add env vars with NEW secrets.
2. **Template app:** find it in the template/service catalog and deploy; set
   env vars fresh.
3. **Database:** create it as a service, then restore the dump taken in
   Phase 1 (`pg_restore`/`mysql`/`redis-cli --pipe` against the container —
   run these over SSH so credentials stay off the chat). Live data →
   ../db-migration.md (rehearsal → freeze → final dump → verify); self-host
   is the one target that can also mount a Redis RDB file directly.
4. **Domain + HTTPS:** set the domain on each service; Coolify/Dokploy
   provision Let's Encrypt automatically once DNS points at the server.

## Hardening (do this — the user just migrated *because of* a breach)

- Enable the provider's firewall: allow 22, 80, 443 only; the dashboard port
  gets closed or bound behind the proxy/domain after setup.
- Disable SSH password auth (`PasswordAuthentication no`) — key-only.
- Enable unattended security upgrades (`unattended-upgrades` on Ubuntu).
- Turn on Coolify's built-in backup schedule for every database, targeting
  off-server storage (S3-compatible; R2 free tier works) — Zeabur was their
  backup story until yesterday.

## Gotchas

- 4 GB RAM comfortably runs Coolify + a handful of small apps; n8n or
  LobeChat + Postgres + Redis together want 4 GB minimum. Don't sell a 1–2 GB
  server.
- The server is now the single point of failure — that's the trade for
  ownership. Mention the backup schedule again at handoff.
- Updates: tell the user the one-liner (Coolify updates from its own UI) and
  suggest a monthly reminder rather than "remember to update".

## Sources (official, checked 2026-08-29)

- Hetzner June 2026 price adjustment (CX23 €5.49 etc.): https://docs.hetzner.com/general/infrastructure-and-availability/price-adjustment/
- Hetzner Cloud plans/locations: https://www.hetzner.com/cloud/
- Hetzner Singapore traffic (0.5–5 TB included, $8.49/TB after): https://www.hetzner.com/cloud-singapore/
- DigitalOcean Droplet pricing ($24/mo 2 vCPU/4 GB): https://www.digitalocean.com/pricing/droplets
- Coolify free self-hosting: https://coolify.io/pricing
- Dokploy plans: https://dokploy.com/pricing
