# AI-DB 連線的限制與理由

這份說明「為什麼」。要執行安裝的話回 `SKILL.md`。

## 決定能不能用的，是 OAuth 在哪裡完成

MCP server 已對外開放在 `https://aidb.moonshine-studio.net/mcp`，從哪裡都連得到
（2026-10-01 起；在那之前只有內網的 `192.168.8.64`）。現在擋人的不是網路，而是
**Authentik 只接受本機的 OAuth callback**（`localhost`／`127.0.0.1`，任何 port）：

| OAuth 在哪裡完成 | 能不能用 |
|---|---|
| **使用者自己的電腦**——Claude Code CLI、Codex CLI | 可以 |
| **使用者自己的電腦**——Claude Desktop 透過本機的 `mcp-remote` 橋接 | 可以 |
| **廠商的雲端**——ChatGPT 的 connector、Claude Desktop 的自訂連接器 | 不行，callback 在廠商的網域，Authentik 不接受 |

雲端那一列要開通，得由 AI-DB 管理者在 Authentik 登記該廠商的 callback，
使用者端調參數或換 `client_id` 都沒用。目前沒有登記，也沒有實測過。

**資料庫連線不在此列。** MCP 對外開放了，PostgreSQL 沒有：`create_database`
交付的連線字串只在公司內網連得到。

## ChatGPT / Codex 桌面版

ChatGPT 與 Codex 共用同一個桌面 app。**app 可以安裝並正常使用，只是連不上
AI-DB。** 兩個各自獨立的原因，任一個都足以擋住：

1. **OAuth callback 不在本機**——見上一節。ChatGPT 桌面版沒有等價於
   `mcpServers` 的本機 stdio 設定可用，所以 Claude Desktop 那條橋接路徑在這裡
   套不上。
2. **版本鎖不住**——AI-DB 的 OAuth 相容性要求 Codex `0.146.1`，而桌面版會自動
   更新，無法固定在這個版本（AI-DB 實測：2026-08-14）。

macOS 上桌面版與 CLI 可能讀到同一份 `~/.codex/config.toml`。**設定共用不代表
連得上**——桌面版看得到那個 server，但登入不會成功。

不要為了讓桌面版登入而修改 AI-DB 的 URL、`client_id` 或 Authentik 設定。
擋住它的是 callback 與版本，不是這些值。

## Claude Desktop 走的是本機橋接，不是自訂連接器

Claude Desktop 加遠端 MCP server 的官方路徑是「自訂連接器」（Custom connector），
那條路徑的 OAuth 由 Anthropic 的雲端完成，callback 不在本機，Authentik 會擋下。

可用的路徑是在設定檔的 `mcpServers` 區跑 `mcp-remote`：Claude Desktop 以 stdio
啟動一個本機的 Node 程序，由它去打 `https://aidb.moonshine-studio.net/mcp`。
OAuth 在本機完成，callback 收在 `http://127.0.0.1:6947/oauth/callback`。
步驟見 `SKILL.md` 的 Claude Desktop 一節。

這條路徑的代價，是它把幾件事綁在使用者的機器上：

- **Node 必須裝在跑 Claude Desktop 的那個 OS 上。** Windows 版的 Claude Desktop
  用 Windows 的 `npx`，WSL 裡裝的 Node 幫不上忙。
- **token 存在 `~/.mcp-auth`，與 CLI 各自獨立。** CLI 登入過不代表桌面版登入過。

`use-aidb` skill 與 MCP 是兩回事，可以單獨安裝。但少了 MCP tool，那個 skill 只剩
「怎麼用連線字串直連 PostgreSQL」那一半——建立、刪除、輪替憑證都做不到。

## 為什麼 Codex CLI 鎖在 0.146.1

兩個上游問題被同一次依賴升級（`rmcp` 1.8 → 3.0）分在兩邊，`0.146.1` 正好落在
夾縫中——**升上去和降下來都會壞，而且壞法不同**：

| 版本 | issuer 檢查 | token refresh |
|---|---|---|
| 0.141.0 / 0.145.0 | 寬鬆，可登入 | ✗ 過期即斷 |
| **0.146.1** | **可登入** | **✓ 正常** |
| 0.147.x 以上 | 嚴格，**擋住我們** | ✓ 正常 |

- **升上去**：0.147 把 expected issuer 的尾斜線 strip 掉，與我們公告的值不符而
  拒連（[openai/codex#37373](https://github.com/openai/codex/issues/37373)）。
  這是 Codex 單方面 strip，改 Authentik 或改我們的公告值都沒用。
- **降下去**：token refresh 時遺漏 RFC 8707 的 `resource` 參數，access token 一
  過期就斷（[openai/codex#33403](https://github.com/openai/codex/issues/33403)）。
  Authentik 的 access token **只有 5 分鐘**，所以症狀是「登入成功，五分鐘後斷線」。

因此 `codex update` 與不帶版本的 `npm install -g @openai/codex` 都會把可用的安裝
弄壞。官方的 `curl install.sh` 同樣永遠抓最新版，而且裝在 `~/.local/bin`，會蓋掉
npm 那份——症狀是「我明明裝了 0.146.1 卻還是 issuer mismatch」。

Claude Code 沒有這個版本限制。

## 已知會失去的東西

使用者拿到連線字串後直連 PostgreSQL，所以有兩件事 AI-DB 目前看不到也管不著：

- **稽核只涵蓋生命週期事件**（誰建了什麼、誰輪替了憑證），資料的增刪改不在其中。
- **沒有速率限制。** 連線數與閒置交易有上限，但連線嘗試、query 與工作負載的
  **速率**沒有任何限制。

另外，資料庫連線目前**未啟用 TLS，傳輸為明文**。連線字串裡的 `sslmode=disable`
誠實描述了這件事——把它改成 `prefer` 或 `require` 不會讓連線加密，只會讓它連不上。
不要用 AI-DB 存放需要傳輸加密保護的資料。

（2026-08-25 之前交付的是 `sslmode=prefer`，理由是留升級路徑：日後啟用 TLS 時
已交付的字串會自動協商上去、不必回收重發。那個值改掉了，因為 node-postgres
對 `prefer` 不降級而直接拋錯，使它在 JS 生態完全不能用。手上是舊字串的話改掉
這個參數即可，密碼不變。）
