<p align="center">
  <img src="assets/logo.png" alt="zeabur-refugee logo：一個人帶著包袱離開故障主機、往安全的地方走" width="160">
</p>

# zeabur-refugee

給 Zeabur 受災戶的 AI 助手手冊：可以直接貼給 Claude、Codex、ChatGPT、Gemini、Cursor 或任何能讀 GitHub 連結的 coding agent，也可以安裝成 Claude Code skill。

Zeabur 在 2026-08-27 發生資安事件（[官方事件頁](https://status.zeabur.com/incident/1037896)）：攻擊者取得內部憑證，把專案的**環境變數（包含大家放在上面的 LLM API 金鑰）**整包撈走，其中 OpenAI、Anthropic、OpenRouter 的金鑰已確認遭到盜用。

這份 repo 幫你止血、盤點、搬家：技術的部分交給 AI 助手，你只做非你不可的事：開帳號、付款、產生新金鑰、改 DNS、按下最後的確認鍵。

## 你現在可以怎麼用

### 方法 A：不用安裝，直接貼給 AI 助手

把下面這段整段貼到 Claude / Codex / ChatGPT / Gemini / Cursor / 任何能打開 GitHub 連結、讀 repo 的 AI 助手：

```text
請幫我處理 Zeabur 受災後的止血、盤點與搬家。請先讀這個 repo：
https://github.com/dawson54068/zeabur-refugee

請把 README 當入口，再依狀況讀 SKILL.md 與 references/ 裡的檔案。請用繁體中文（台灣用語）一步一步帶我做，不要一次丟一大串任務。

如果你是 Codex / Codex CLI / Cursor agent / 任何可以跑 shell 或讀檔案的 coding agent，可以先把 repo clone 到暫存目錄閱讀：

git clone --depth 1 https://github.com/dawson54068/zeabur-refugee /tmp/zeabur-refugee

然後從 /tmp/zeabur-refugee/README.md 和 /tmp/zeabur-refugee/SKILL.md 開始。不要要求我先安裝 Claude Code skill；安裝只是讓 Claude Code 使用者比較省事的做法，不是必要條件。

我的狀況：
- 我在 Zeabur 有幾個專案：<填數量；如果很多，寫「30 幾個」也可以>
- 我知道有放在 Zeabur 環境變數裡的金鑰或密碼：<OpenAI / Anthropic / OpenRouter / AWS / Stripe / GitHub / DB / 不確定>
- 最重要、不能斷太久的服務：<填服務名稱；不知道就寫不確定>
- 有沒有資料庫：<Postgres / MySQL / Redis / MongoDB / 不確定>
- 能不能接受 15–30 分鐘維護時間：<可以 / 不可以 / 不確定>
- 預算：<希望 0 元 / 可以每月幾美元 / 可以租 VPS>

請先做這件事：
1. 先判斷金鑰是否可能外洩；如果可能，先帶我關自動儲值、截圖存證、撤銷或停用舊金鑰。
2. 如果我有很多 Zeabur 專案，請不要叫我一個一個點；請照 references/bulk-audit.md 與 references/zeabur-official-tools.md 使用 Zeabur 官方 CLI/API 做總盤點（`npx zeabur@latest auth login`、`project ls`、`service ls`、`variable list`、`project export`）。
3. 如果有 live database，請照 references/db-migration.md 走「演練 → 暫停寫入 → 最後一次備份 → 驗證 → 保留 7 天」流程，不要只做一次 dump。
4. 推薦平台時，請讀對應的 references/targets/*.md，並附官方來源連結證明價格與限制。
5. 不要要求我把 API key、資料庫密碼、.env 內容貼到對話裡；需要輸入金鑰時，請用本機檔案或 CLI 的隱藏輸入方式。
6. 刪除、停用、改 DNS、撤銷金鑰之前，請先說明影響並等我明確確認。
```

如果你的 AI 助手不能讀 GitHub 網址，就請它先叫你依序貼這些檔案：`SKILL.md`、`references/key-rotation.md`、`references/bulk-audit.md`，如果有資料庫再貼 `references/db-migration.md`，最後只貼要搬去的平台那個 `references/targets/*.md`。

### 方法 B：給 Codex / CLI coding agent

如果你在 Codex CLI、Cursor agent、Claude Code，或其他可以讀 repo / 跑 shell 的環境，可以直接用這段：

```text
請 clone 並閱讀這份 Zeabur 受災搬家手冊，不要先做任何破壞性操作：

git clone --depth 1 https://github.com/dawson54068/zeabur-refugee /tmp/zeabur-refugee

先讀：
- /tmp/zeabur-refugee/README.md
- /tmp/zeabur-refugee/SKILL.md

再依我的狀況，只讀需要的 references/*。請用繁體中文（台灣用語）一步一步帶我做。不要要求我把 API key、資料庫密碼或 .env 內容貼到對話裡。需要跑 Zeabur 盤點時，優先使用 references/zeabur-official-tools.md 裡的 Zeabur 官方 CLI/API；需要搬 live database 時，照 references/db-migration.md，不要只做一次 dump。
```

### 方法 C：安裝成 Claude Code skill

```bash
git clone https://github.com/dawson54068/zeabur-refugee ~/.claude/skills/zeabur-refugee
```

或下載 [`zeabur-refugee.skill`](./zeabur-refugee.skill)（就是個 zip），解壓縮到 `~/.claude/skills/`。

之後重開 Claude Code，直接描述你的狀況就會觸發：

> zeabur 被駭了，我的 API key 都放在上面，怎麼辦

> 我在 Zeabur 有 30 幾個專案，一個一個查太慢了

## AI 助手請讀這段

如果使用者把這個 repo URL 貼給你，而不是安裝 skill，請照以下規則操作。這段適用於 Claude、Codex、ChatGPT、Gemini、Cursor、GitHub Copilot coding agent，以及任何能讀 GitHub repo 或網頁的 AI 助手。

1. **先讀 `SKILL.md`。** 它是總流程：Phase 0 止血、Phase 1 盤點、Phase 2 選平台、Phase 3 搬家、Phase 4 下線。
2. **不要一次讀完整 repo。** 依情境讀參考檔：
   - 任何金鑰可能外洩：讀 `references/key-rotation.md`。
   - 使用者有 10 個以上 Zeabur 專案：讀 `references/bulk-audit.md` 和 `references/zeabur-official-tools.md`。
   - 要匯出 Zeabur 專案、環境變數、資料庫、volume：讀 `references/zeabur-export.md`。
   - 有 live database：讀 `references/db-migration.md`。
   - 選定平台後，只讀對應的 `references/targets/*.md`。
3. **如果你能跑 shell，先 clone repo 到暫存目錄讀檔。** 例如 `git clone --depth 1 https://github.com/dawson54068/zeabur-refugee /tmp/zeabur-refugee`。不要把它當成只有 Claude Code 才能用的 skill。
4. **繁體中文優先。** 使用台灣技術社群會懂的講法；短句、步驟化，一次只給下一個步驟。
5. **金鑰比搬家急。** 如果使用者的 API key、DB URL、token、OAuth secret 曾經放在 Zeabur env var，先把它當成已外洩處理。先關自動儲值、截圖存證、撤銷或停用，再談搬去哪。
6. **不要讓使用者把機密資訊貼到對話裡。** 只看 env var 名稱；值放本機檔案，或透過 CLI 的 stdin / hidden prompt 輸入。不要輸出 `.env`、不要要求貼 API key。
7. **live database 不能只 dump 一次。** 單次 dump 會漏掉 dump 之後的新寫入。請照 `references/db-migration.md`：先演練、暫停寫入、最後一次備份、驗證、保留 Zeabur DB 至少 7 天。
8. **價格與限制要附官方來源。** 各平台參考檔結尾都有 Sources；引用價格、free tier、timeout、region、volume、DB 支援時，要附連結。若官網和本 repo 的數字不同，以官網現況為準。
9. **只有使用者能做的事才交給使用者。** 帳號註冊、OAuth 授權、付款、建立新 API key、改 DNS、刪除、停用、覆寫前的確認。其餘盤點、指令、設定檔、部署、驗證，請你主動做或給出可執行指令。
10. **刪除前一定確認。** Zeabur env vars、service、project、DB、domain、各供應商的 API key 都可能還被正式服務用到。新平台驗證前，不要刪舊平台；刪除前說清楚後果並等待明確同意。

## 它會做什麼

1. **先止血**：關掉各家的自動儲值（偷到的金鑰配上自動儲值就是無上限提款）、把用量截圖存證（之後向 Zeabur 申請補償要用）、撤銷外洩金鑰。附 10 家 LLM 供應商加上 AWS / Stripe / GitHub / Telegram / Discord 的正確後台連結。
2. **大量專案快速盤點**：30 個專案，不用搬 30 次。用 Zeabur 官方 CLI/API 一次掃完（`project ls`、`service ls`、`variable list`、`project export`）、產出一張分類表；金鑰照「供應商」輪替（幾家就跑幾次）；廢棄專案直接刪掉就不用管了，只搬還活著的。
3. **選擇合適的新家**：決策表加上各平台指南（Railway、Coolify/Dokploy VPS、Render、Fly.io、Cloudflare、Vercel），價格與限制逐條附官方來源連結（2026-08-29 查核）。
4. **資料庫不掉資料**：還在被正式服務使用的資料庫走「演練 → 暫停寫入 → 最後一次備份 → 驗證 → 保留 7 天隨時可退回」的流程搬，不是 dump 一次就了事。
5. **安全下線**：新平台驗證通過之前，Zeabur 上的東西一律不動；金鑰從頭到尾不會出現在對話裡。

## Repo 導覽

| 檔案 | 什麼時候讀 |
|---|---|
| `SKILL.md` | 總流程入口 |
| `references/key-rotation.md` | 金鑰、token、DB 密碼可能外洩時 |
| `references/bulk-audit.md` | 10 個以上 Zeabur 專案要快速盤點時 |
| `references/zeabur-official-tools.md` | Zeabur 網站很慢時：用官方 CLI/API/GraphQL/schema host 盤點 |
| `references/zeabur-export.md` | 匯出 Zeabur env vars、資料庫、volume、domain 設定時 |
| `references/db-migration.md` | 有 live database，不能掉資料時 |
| `references/targets/*.md` | 選平台後讀對應檔案：Railway / Render / Fly / Cloudflare / Vercel / self-host |

## 開源與貢獻

- 授權：MIT，見 [`LICENSE`](./LICENSE)。
- 想修正內容或新增平台指南：見 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。
- 發現安全問題：請照 [`SECURITY.md`](./SECURITY.md) 私下回報，不要把漏洞細節或任何金鑰貼到公開 issue。

## 備註

- 截至 2026-08-29，平台價格與限制已用官方頁面查核，每個參考檔結尾都附來源連結；skill 同時要求 AI 助手以官網現況為準，數字過期就以當下查到的為準。
- `evals/` 收錄測試情境與結果（有載入 skill：15/15；沒載入 skill：12/15，差在把 LobeChat 部署到 Vercel、撤銷金鑰前沒留證據、沿用已外洩的資料庫連線字串這三種會真正造成損失的錯誤）。
- 與 Zeabur 官方無關。事件描述皆引自 Zeabur 官方狀態頁與媒體報導。

---

## Logo

`assets/logo.png` 是本專案原創圖示：一個人帶著包袱離開故障主機、往安全的地方走。它不是 Zeabur 官方圖示，也沒有使用 Zeabur 商標或外部素材；跟 repo 其餘內容一起以 MIT 授權釋出。

## English

A paste-to-your-agent migration guide for users leaving Zeabur after the confirmed 2026-08-27 incident in which project environment variables — including LLM provider API keys — were accessed, with OpenAI / Anthropic / OpenRouter keys abused in the wild. It works with Claude, Codex / Codex CLI, ChatGPT, Gemini, Cursor, GitHub Copilot coding agent, or any assistant that can read a GitHub repo. It can also be installed as a Claude Code skill.

If you do not want to install anything, paste this GitHub URL into your AI assistant and ask it to read `README.md` first, then `SKILL.md`, then only the relevant files under `references/`. If the assistant can run shell commands, have it clone the repo to a temp directory and read the files locally. The core rules: rotate compromised keys before migration, never paste secret values into chat, bulk-audit many Zeabur projects instead of clicking one by one, use official source links for price/constraint claims, and migrate live databases with a rehearsal → write-freeze → final dump → verify → 7-day rollback window.

Claude Code install: `git clone` this repo into `~/.claude/skills/zeabur-refugee` (or unzip [`zeabur-refugee.skill`](./zeabur-refugee.skill) there), restart Claude Code, and describe your situation. Facts verified 2026-08-29 with source links throughout. Not affiliated with Zeabur.
