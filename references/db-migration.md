# Live database migration — no-data-loss procedure

The trap: `pg_dump` copies a snapshot, but a live app keeps writing after the
snapshot. Restore that dump, cut over, and every write since the dump is
silently gone — the app looks fine, the loss surfaces days later as a missing
order or a vanished workflow. So for any database with real traffic, the
migration is **rehearsal → freeze → final dump → verify → cutover**, never a
single dump. Nothing on Zeabur gets deleted until the new platform has run
clean for days.

Ask one question up front: **can this app be down for ~15–30 minutes?**

- Yes (almost every personal/template app) → the freeze-window procedure
  below. Simple and genuinely lossless.
- Truly no (busy production) → Postgres logical replication (publication on
  old, subscription on new, cut over when caught up). That's an advanced
  path — offer it, but be honest that for most Zeabur-scale apps the freeze
  window is safer than a first-time replication setup under incident
  pressure.
- Redis that's only a cache/session store → don't migrate data; users
  re-login. Say that explicitly and get agreement — it's often the whole
  Redis answer.

## The procedure

**1. Rehearsal (zero risk — old DB untouched, app still live).**
Take an initial dump (zeabur-export.md has the connection/port-forward
steps), restore it into the NEW database, point the new deployment at it,
and test the app against it. This finds version, extension, and driver
surprises while nothing is at stake. Treat this restored copy as a dress
rehearsal — its data will be thrown away.

**2. Belt and suspenders before the real window.**
Trigger a Zeabur Backup-tab backup AND keep your own dump, stored in two
places (local disk + a cloud drive/R2 bucket). Verify sizes and table
counts. You are about to change a live system; the cost of an extra copy is
zero, the cost of needing one and not having it is the whole project.

**3. Freeze writes (the maintenance window starts here).**
Suspend the Zeabur **app** service (Settings → Suspend Service — it
preserves env vars, volumes, and the DB; the DB service stays up). If users
exist, tell them the window in advance. Confirm writes actually stopped:

```sql
-- Postgres: run twice, 30s apart — numbers must not move
SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
```

**4. Final dump → restore into a CLEAN target.**
Drop/recreate the rehearsal database (or restore into a fresh one) before
the final restore — never restore over the rehearsal data and hope it
merges; a mixed state is worse than a failed restore. Then restore the
final dump.

**5. Verify BEFORE unfreezing anything.**
- Row counts per table, old vs new, side by side — they must match.
- Latest-timestamp spot check on the busiest tables (newest order/message
  present on the new side).
- App-level smoke test on the new stack: read a recent record, create a
  test record, delete it.

**6. Cut over** (app env → new DB URL, DNS per Phase 4). The old app stays
suspended — do not resume it, or you'll have two writers again.

**7. Keep the rollback door open.**
Old DB service stays up (or suspended) and intact for **at least 7 days**
of clean running on the new platform. Rollback = repoint to Zeabur and
resume — but any writes made on the new platform during the divergence are
then lost, so rolling back is also a decision, not a reflex. Only after the
quiet week does Phase 4 teardown delete the old DB (its deletion has a
2-hour grace period, then it's gone forever).

## Engine notes

- **Postgres:** `pg_dump -Fc` → `pg_restore --no-owner --no-acl -d "$NEW"`.
  Target major version ≥ source (`SELECT version()` on both). Check
  extensions (`\dx`) — common ones exist on Neon/Railway; anything exotic,
  verify against the target's docs before the real window. Serverless
  targets (Vercel/Workers) need the pooled connection string.
- **MySQL:** `mysqldump --single-transaction --routines --triggers`
  (consistent snapshot without locking); mind utf8mb4 on the target.
- **MongoDB:** `mongodump --archive` → `mongorestore --archive`, same
  freeze-verify bracket.
- **Redis with durable data** (n8n queues, app state — not just cache):
  freeze writers first, then `redis-cli --rdb`. Managed Redis targets
  usually can't ingest an RDB file directly — check the target's own import
  tooling at migration time; self-hosted (Coolify) can mount the RDB
  straight into the container. If import is painful and the data is
  re-derivable, prefer regenerating over migrating.

## What NOT to do, ever

- Don't run old and new apps against their own databases "temporarily" —
  divergence has no merge story.
- Don't delete the Zeabur DB in the same sitting as the cutover, no matter
  how well it went.
- Don't skip the rehearsal to save 20 minutes; the rehearsal is where every
  real-world restore problem shows up safely.
- Don't leave the DB's public port/port-forward open between the dump and
  teardown (breach hygiene — zeabur-export.md).
