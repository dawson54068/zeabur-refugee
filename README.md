# zeabur-refugee

A [Claude Code](https://claude.com/claude-code) skill for anyone leaving
[Zeabur](https://zeabur.com) after the confirmed **2026-08-27 security
incident** ([official incident page](https://status.zeabur.com/incident/1037896))
in which project **environment variables — including LLM provider API keys —
were accessed**, with OpenAI / Anthropic / OpenRouter keys abused in the wild.

Claude does the technical work; you only do what genuinely requires you
(accounts, payments, creating new keys, DNS, final confirmations).

## What it does

1. **Contain first** — kill auto-recharge money taps, capture abuse evidence
   for Zeabur's compensation process, revoke compromised keys (verified
   console URLs for 10 LLM providers + AWS/Stripe/GitHub/Telegram/Discord).
2. **Audit at scale** — 30 projects ≠ 30 migrations: one CLI/API sweep, one
   triage table, rotate per *provider*, delete dead projects, migrate only
   what's alive.
3. **Choose a real replacement** — decision table + per-platform guides
   (Railway, Coolify/Dokploy on a VPS, Render, Fly.io, Cloudflare, Vercel)
   with prices and constraints, every claim linked to an official source
   (checked 2026-08-29).
4. **Migrate without losing data** — live databases get
   rehearsal → write-freeze → final dump → verify → 7-day rollback window,
   never a one-shot dump.
5. **Decommission safely** — new platform verified before anything on Zeabur
   is deleted; secrets never pass through the chat.

## Install

**Option A — git:**

```bash
git clone https://github.com/dawson54068/zeabur-refugee ~/.claude/skills/zeabur-refugee
```

**Option B — packaged:** download `zeabur-refugee.skill` from
[Releases](../../releases), unzip into `~/.claude/skills/`.

Then restart Claude Code and just describe your situation:

> zeabur got breached and my API keys were in there, help

> 我在 Zeabur 有 30 幾個專案，一個一個查太慢了

## 中文快速說明

這是給 Zeabur 事件受災戶的 Claude Code skill：先止血（關自動儲值、截圖用量
證據、撤銷外洩金鑰），再一次掃完所有專案分類（該刪的刪、該搬的搬），
最後帶你把服務搬到 Railway / Coolify VPS / Render / Fly / Cloudflare /
Vercel — 資料庫用「彩排 → 凍結寫入 → 最終備份 → 驗證」流程搬，不會掉資料。
金鑰絕不經過對話內容。裝好後直接用中文描述你的狀況即可。

## Notes

- Platform prices/constraints were verified against official pages on
  **2026-08-29** and each reference file ends with its source links; the
  skill instructs Claude to trust live pages over stale numbers.
- Eval results (3 scenarios, with-skill 15/15 vs. baseline 12/15) and the
  test harness live in `evals/`.
- Not affiliated with Zeabur. Incident facts come from Zeabur's official
  status page and cited press coverage.
