# AI-DB 安裝排錯

## 先做這一刀

**同一個 server，換另一個 client 測會怎樣？**

- 兩個 client 都失敗 → server 或網路的問題，收集下方資訊回報給 AI-DB 管理者
- 只有一個失敗 → 那個 client 的設定或版本問題，往下查對應的表

在那之前先確認 server 活著。在**你自己的機器**上執行：

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.8.64:8000/mcp
```

- `401` — 正常。server 活著，認證中介層在線上。接著查 client 端。
- 連線失敗／逾時 — 你不在公司內網，或 server 沒起來。先確認網路。
- `200` — 異常，請回報 AI-DB 管理者。

## Claude Code

| 症狀 | 處理方式 |
|---|---|
| `does not support dynamic client registration` | 加入 server 時漏了 `--client-id`。`claude mcp remove ai-db` 後重新加入 |
| `MCP server ai-db already exists` | 該 scope 已有同名設定。先 `claude mcp remove ai-db` 再加入 |
| `! Needs authentication` | 尚未登入或 token 已失效。執行 `claude mcp login ai-db`，或在 `/mcp` 選 Re-authenticate |
| `✘ Failed to connect` | 先確認在公司內網，且上面那條 curl 回 401 |
| 登入時 Authentik 回報 redirect URI 不符 | Claude Code 預設用隨機 port 的 `http://localhost:<port>/callback`。加 `--callback-port 8080` 固定 port 重新加入 server，並請 AI-DB 管理者把 `http://localhost:8080/callback` 加進 Authentik client 的 redirect URI |
| WSL 或 SSH 環境瀏覽器開不了 | 改用 `claude mcp login ai-db --no-browser`，把印出的網址在有瀏覽器的機器打開，完成後貼回**完整的 callback 網址** |
| `/skills` 看不到 `use-aidb` | 確認路徑是 `~/.claude/skills/use-aidb/SKILL.md`。必要時重開 `claude` |
| 設定好了但另一個環境看不到 | WSL 與 Windows 各有一份 `~/.claude.json`，不同步。兩邊都要設 |

## Codex CLI

| 症狀 | 處理方式 |
|---|---|
| `Authorization server issuer mismatch` | 版本不對。`codex --version` 必須是 `0.146.1` |
| 版本確認是 0.146.1 卻仍 issuer mismatch | `which -a codex`。`install.sh` 裝在 `~/.local/bin` 的那份會蓋掉 npm 這份，刪掉它或調整 `PATH` |
| `Dynamic client registration not supported` | 加入 server 時漏了 `--oauth-client-id`。`codex mcp remove ai-db` 後重新加入 |
| `No MCP server named 'ai-db' found` | 尚未加入 server。`login` 只查已註冊的 server，不會建立。先 `codex mcp add` |
| 登入成功但約五分鐘後斷線 | 版本低於 `0.146.1`，token refresh 沒帶必要參數。重新檢查版本與 `PATH` |
| 瀏覽器沒開或登入沒反應 | 先確認在公司內網。仍不行就回報，附上下方三項輸出 |
| WSL 設定好了但 Windows 桌面版看不到 | 兩邊使用不同的 `~/.codex/config.toml`，不同步。目前只有 CLI 這條路可用 |
| 設定改了卻沒生效 | Codex 對**未知的 config key 是靜默忽略**的，打錯字不會有任何錯誤訊息。逐字核對 key 名稱 |

## 兩邊共通

| 症狀 | 處理方式 |
|---|---|
| 登入流程走完，但 agent 看不到 AI-DB 的 tool | 重開 CLI。仍看不到就回報 |
| tool 呼叫回「找不到」 | AI-DB 對「不存在」與「不屬於你」一律回報找不到。確認 `db_id` 沒打錯，且那個資料庫是你自己建的 |
| 拿到連線字串但連不上資料庫 | 那不是安裝問題，見 `use-aidb` skill。JDBC 使用者特別注意：`dsn` 是 PostgreSQL URI，**不是** JDBC URL |
| 密碼弄丟了 | 平台不儲存密碼，取不回來。唯一途徑是 `rotate_database_credentials`，而它會中斷既有連線 |

## 回報時附上什麼

Claude Code：

```bash
claude --version
claude mcp list
claude mcp get ai-db
```

Codex CLI：

```bash
codex --version
which -a codex
codex mcp list
```

**不要貼出 token、資料庫密碼或完整的 PostgreSQL 連線字串。** 上面這些指令的輸出
不含密碼，可以直接貼；若你另外複製了連線字串，把 `://` 與 `@` 之間那段遮掉。
