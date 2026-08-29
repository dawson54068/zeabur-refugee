# zeabur-refugee

給 Zeabur 受災戶的 [Claude Code](https://claude.com/claude-code) skill。

Zeabur 在 2026-08-27 發生資安事件（[官方事件頁](https://status.zeabur.com/incident/1037896)）：
攻擊者取得內部憑證，把專案的**環境變數（包含大家放在上面的 LLM API 金鑰）**
整包撈走，其中 OpenAI、Anthropic、OpenRouter 的金鑰已確認遭到盜用。這個
skill 幫你止血、盤點、搬家：技術的部分交給 Claude，你只做非你不可的事：
開帳號、付款、產生新金鑰、改 DNS、按下最後的確認鍵。

## 它會做什麼

1. **先止血**：關掉各家的自動儲值（偷到的金鑰配上自動儲值就是無上限提款）、
   把用量截圖存證（之後向 Zeabur 申請補償要用）、撤銷外洩金鑰。附 10 家 LLM
   供應商加上 AWS / Stripe / GitHub / Telegram / Discord 的正確後台連結。
2. **大量專案快速盤點**：30 個專案，不用搬 30 次。CLI/API 一次掃完、產出一
   張分類表；金鑰照「供應商」輪替（幾家就跑幾次）；廢棄專案直接刪掉就不用
   管了，只搬還活著的。
3. **選擇合適的新家**：決策表加上各平台指南（Railway、Coolify/Dokploy
   VPS、Render、Fly.io、Cloudflare、Vercel），價格與限制逐條附官方來源連結
   （2026-08-29 查核）。
4. **資料庫不掉資料**：還在線上服務的資料庫走「演練 → 凍結寫入 → 最終備份
   → 驗證 → 保留 7 天隨時可退回」的流程搬，不是 dump 一次就了事。
5. **安全下線**：新平台驗證通過之前，Zeabur 上的東西一律不動；金鑰從頭到尾
   不會出現在對話裡。

## 安裝

**方法一：git**

```bash
git clone https://github.com/dawson54068/zeabur-refugee ~/.claude/skills/zeabur-refugee
```

**方法二：壓縮檔** — 下載 [`zeabur-refugee.skill`](./zeabur-refugee.skill)
（就是個 zip），解壓縮到 `~/.claude/skills/`。

之後重開 Claude Code，直接描述你的狀況就會觸發：

> zeabur 被駭了，我的 API key 都放在上面，怎麼辦

> 我在 Zeabur 有 30 幾個專案，一個一個查太慢了

## 備註

- 平台價格與限制以 2026-08-29 的官方頁面查核，每個參考檔結尾都附來源連結；
  skill 同時要求 Claude 以官網現況為準，數字過期就以當下查到的為準。
- `evals/` 收錄測試情境與結果（帶 skill 15/15，不帶 12/15，差在把 LobeChat
  推上 Vercel、撤銷金鑰前沒留證據、沿用已外洩的資料庫連線字串這三種會真正
  造成損失的錯誤）。
- 與 Zeabur 官方無關。事件描述皆引自 Zeabur 官方狀態頁與媒體報導。

---

## English

A Claude Code skill for users leaving Zeabur after the confirmed 2026-08-27
incident in which project environment variables — including LLM provider API
keys — were accessed, with OpenAI / Anthropic / OpenRouter keys abused in the
wild. It handles emergency key rotation (verified console URLs), bulk
auditing of many projects, platform selection with officially-sourced
pricing/constraints, and no-data-loss live database migration. Claude does
the technical work; you only handle accounts, payments, new keys, DNS, and
final confirmations.

Install: `git clone` this repo into `~/.claude/skills/zeabur-refugee` (or
unzip [`zeabur-refugee.skill`](./zeabur-refugee.skill) there), restart
Claude Code, and describe your situation. Facts verified 2026-08-29 with
source links throughout. Not affiliated with Zeabur.
