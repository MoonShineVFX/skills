# AI-DB 連線的限制與理由

這份說明「為什麼」。要執行安裝的話回 `SKILL.md`。

## 為什麼只有 CLI 連得上

**AI-DB 沒有對外路徑**，只在公司內網的 `192.168.8.64`。這一條決定了哪些 client
用得了：

| client 怎麼連 MCP server | 能不能用 |
|---|---|
| 在**你自己的電腦**上直接連 | 可以——Codex CLI、Claude Code CLI 都是這一類 |
| 交給**廠商的雲端**代連（ChatGPT 的 connector、Claude 的自訂連接器） | 不行，那些請求從公網發出，搆不到內網 |

兩個桌面版都屬於後者。**這不是設定問題**——調參數、換 `client_id` 或改 Authentik
都不會讓它們連上。要支援它們得先決定把 AI-DB 對外暴露，那是另一個題目。

## ChatGPT / Codex 桌面版

ChatGPT 與 Codex 共用同一個桌面 app。**app 可以安裝並正常使用，只是連不上
AI-DB。** 兩個各自獨立的原因，任一個都足以擋住：

1. **沒有對外路徑**——見上一節。
2. **版本鎖不住**——AI-DB 的 OAuth 相容性要求 Codex `0.146.1`，而桌面版會自動
   更新，無法固定在這個版本（AI-DB 實測：2026-08-14）。

macOS 上桌面版與 CLI 可能讀到同一份 `~/.codex/config.toml`。**設定共用不代表
連得上**——桌面版看得到那個 server，但登入不會成功。

不要為了讓桌面版登入而修改 AI-DB 的 URL、`client_id` 或 Authentik 設定。
擋住它的是網路路徑與版本，不是這些值。

## Claude Desktop

Claude Desktop 加遠端 MCP server 的官方路徑是「自訂連接器」（Custom connector），
而那條路徑要求 MCP server **從公網、由 Anthropic 的 IP 連得到**。AI-DB 只在內網，
加了不會連上。

需要在 Claude 裡使用 AI-DB，請用 Claude Code CLI。

> **關於本機橋接。** 技術上可以在自己的電腦跑一個 stdio 轉 HTTP 的橋接程式
>（例如 `mcp-remote`），讓請求從本機發出而不經過雲端。AI-DB 尚未驗證這條路徑，
> 因此**不列為支援的安裝方式**。若之後驗證通過會補上。

`use-aidb` skill 與 MCP 是兩回事，Claude Desktop 這一側可以單獨安裝。但少了 MCP
tool，那個 skill 只剩下「怎麼用連線字串直連 PostgreSQL」那一半——建立、刪除、
輪替憑證都做不到。

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

另外，資料庫連線目前**未啟用 TLS，傳輸為明文**。連線字串裡的 `sslmode=prefer`
是為了日後啟用 TLS 時已交付的字串會自動協商上去，不必回收重發——**但它現在不保證
加密**。不要用 AI-DB 存放需要傳輸加密保護的資料。
