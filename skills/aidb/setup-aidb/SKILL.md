---
name: setup-aidb
description: 把 AI client 連上 MoonShine 內部的 AI-DB MCP server，涵蓋 Claude Code CLI、Codex CLI 與 Claude Desktop（透過 mcp-remote 本機橋接）的安裝、OAuth 登入與驗證。當使用者要安裝或設定 AI-DB、說連不上 AI-DB、登入沒反應、看不到 AI-DB 的 tool，或想用 AI-DB 但尚未安裝時使用。
---

# 安裝 AI-DB MCP server

AI-DB 是 MoonShine 內部的資料庫平台：使用者以公司帳號建立自己的 PostgreSQL
資料庫，取得專屬連線字串後，由自己的應用程式直連操作資料。

本 skill 只負責**連上**。連上之後怎麼用，見 `use-aidb` skill。

連線資訊（兩個值都是公開識別值，不是密碼）：

```
MCP server   http://192.168.8.64:8000/mcp
client_id    8MGJHGH157nKGeP2o5pinEghwzS6FGUU9bLASnTo
```

## 開始前的三項檢查

任一項不成立就停下來告訴使用者，不要繼續往下裝。

```bash
# 1. 必須在公司內網——AI-DB 沒有對外路徑
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.8.64:8000/mcp
#    預期 401（代表 server 活著且認證中介層在線）。連線失敗代表不在內網。

# 2. 必須有 npm／npx——三條路徑都需要，npm 本身的安裝不在本 skill 範圍
npm --version
```

3. 使用者要有公司的 Authentik 帳號，OAuth 登入時會用到。

> **請求必須從使用者自己的機器發出。** 走廠商雲端代連的路徑——ChatGPT 的
> connector、Claude Desktop 的「自訂連接器」——請求從公網出去，搆不到內網的
> `192.168.8.64`，那不是調參數或換 `client_id` 能解決的。Claude Desktop 要用
> AI-DB，走本機的 `mcp-remote` 橋接（見下）。ChatGPT／Codex 桌面版目前沒有可用
> 路徑，理由見 `references/limitations.md`。

## 選一條路徑

先確認使用者要設定哪一個 client。若不確定，就是你現在正在其中運行的那一個。

| 使用者用的是 | 走哪一節 |
|---|---|
| Claude Code | [Claude Code CLI](#claude-code-cli) |
| Codex CLI | [Codex CLI](#codex-cli) |
| Claude Desktop | [Claude Desktop](#claude-desktop) |
| 不只一個 | 各節都做。設定各自獨立，互不影響 |

**設定寫在你執行指令的那個環境裡。** WSL 與 Windows 各有一份設定檔，兩邊不同步；
在 WSL 裡設定完成後，Windows 上的同名 CLI 仍然看不到 server。

## Claude Code CLI

適用 macOS、Linux 與 WSL。

`claude mcp login` 需要 Claude Code v2.1.186 以上（`claude --version` 可確認）。
更舊的版本改在 `claude` 裡用 `/mcp` 完成登入，其餘步驟相同。

### 1. 加入 server

```bash
claude mcp add --transport http --scope user \
  --client-id 8MGJHGH157nKGeP2o5pinEghwzS6FGUU9bLASnTo \
  ai-db http://192.168.8.64:8000/mcp
```

- `--client-id` **不能省略**。Authentik 不支援 Dynamic Client Registration，
  省略時會收到 `does not support dynamic client registration`。
- **不要**加 `--client-secret`。這是 public client，沒有 secret。
- `--scope user` 讓所有專案都載得到。省略時預設是 local scope，只在當下目錄生效。

同名 server 已存在時先移除再重加：`claude mcp remove ai-db`

### 2. 登入

```bash
claude mcp login ai-db
```

瀏覽器開啟後以公司帳號完成 Authentik 登入與授權，再回到終端機。

WSL、SSH 等開不了瀏覽器的環境改用 `claude mcp login ai-db --no-browser`：
它會印出授權網址，在有瀏覽器的機器打開，完成後把瀏覽器網址列上的**完整
callback 網址**貼回終端機。

也可以在 `claude` 裡輸入 `/mcp`，選 `ai-db` 後執行 Authenticate，結果相同。

> **這一步必須由使用者在自己的終端機執行，不能請 Claude Code 這個 agent 代跑。**
> Claude Code 執行 Bash 工具時沒有 TTY，`claude mcp login` 會直接失敗並回報
> `stdin isn't a terminal, so authentication can't be completed here`——加
> `--no-browser` 也一樣，因為問題是缺 TTY，不是缺瀏覽器。遇到使用者要求「幫我
> 登入 ai-db」時，回覆請他在自己的終端機跑這條指令，不要嘗試用 Bash 工具執行。

### 3. 驗證

```bash
claude mcp list
claude mcp get ai-db
```

`ai-db` 必須顯示 `✔ Connected`，URL 是 `http://192.168.8.64:8000/mcp`。
顯示 `! Needs authentication` 代表第 2 步還沒完成。

在 `claude` 裡輸入 `/mcp` 應能看到 `ai-db` 與它的 tool。

### 4. 安裝 use-aidb skill

```bash
npx skills add MoonShineVFX/skills --skill use-aidb -g -a claude-code -y
```

`-g` 裝到 `~/.claude/skills/`，所有專案共用；只想給單一專案用就拿掉 `-g`。

驗證：在 `claude` 裡執行 `/skills`，清單中應有 `use-aidb`。
日後更新用 `npx skills update use-aidb`。

### 5. 端到端確認

請 Claude Code「列出我在 AI-DB 的資料庫」。**沒有資料庫時回傳空列表是正確結果**，
不是失敗。收到錯誤的話走 `references/troubleshooting.md`。

## Codex CLI

適用 macOS、Linux 與 WSL。Windows 使用者建議在 WSL 內完成。

### 1. 安裝指定版本

**AI-DB 鎖定 Codex CLI `0.146.1`。不要裝最新版，也不要用官方的 `install.sh`。**
其他版本會在 OAuth 登入或 token 更新時失敗，理由見 `references/limitations.md`。

```bash
npm install -g @openai/codex@0.146.1
codex --version      # 必須是 0.146.1
which -a codex       # 必須只有一份
```

`install.sh` 裝的那份在 `~/.local/bin/codex`，會蓋過 npm 這份。`which -a` 列出
兩份時，先移除舊版或調整 `PATH`，再重新確認。

> 不要執行 `codex update`，也不要跑不帶版本的 `npm install -g @openai/codex`。
> 升級前須先確認 AI-DB 已解除版本鎖定。

### 2. 登入 Codex 本身

```bash
codex login
```

依提示完成 ChatGPT 帳號登入。**這與下一步的 AI-DB 登入是兩組獨立的登入，
兩者都必須完成。**

### 3. 加入 server

```bash
codex mcp add ai-db \
  --url http://192.168.8.64:8000/mcp \
  --oauth-client-id 8MGJHGH157nKGeP2o5pinEghwzS6FGUU9bLASnTo
```

`--oauth-client-id` **不能省略**，省略時會收到
`Dynamic client registration not supported`。

同名 server 已存在時先移除再重加：`codex mcp remove ai-db`

### 4. 登入 AI-DB

```bash
codex mcp login ai-db
```

瀏覽器開啟後以公司帳號完成 Authentik 登入與授權，再回到終端機。

### 5. 驗證

```bash
codex mcp list
```

`ai-db` 必須已啟用、URL 是 `http://192.168.8.64:8000/mcp`，且 Auth 欄位是
**OAuth 而不是 `Not logged in`**。

啟動 `codex` 後輸入 `/mcp`，應能看到 `ai-db` 與它的 tool。

> macOS 上 AI-DB 的 MCP token 存在 Keychain 的 `Codex MCP Credentials`，
> **不在 `~/.codex/auth.json`**（那只是 ChatGPT 帳號自己的登入）。確認登入狀態
> 一律以 `codex mcp list` 為準。

### 6. 安裝 use-aidb skill

```bash
npx skills add MoonShineVFX/skills --skill use-aidb -g -a codex -y
```

### 7. 端到端確認

請 Codex「列出我在 AI-DB 的資料庫」。**沒有資料庫時回傳空列表是正確結果。**

## Claude Desktop

Claude Desktop 的「自訂連接器」連不上 AI-DB——那條路是 Anthropic 的雲端代連，
搆不到內網。改走 `mcp-remote`：它在**使用者自己的電腦**上跑一個 stdio ↔ HTTP 的
橋接程式，請求就從本機發出。

**Claude Desktop 執行的是它所在作業系統的 `npx`。** Windows 版跑 Windows 的 Node，
不是 WSL 裡那份；Node 要裝在 Windows 這一側，而且那台機器要在公司內網。

### 1. 編輯設定檔

Settings → Developer → Edit Config 會打開 `claude_desktop_config.json`：

- macOS `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows `%APPDATA%\Claude\claude_desktop_config.json`

在 `mcpServers` 區加入 `ai-db`。**已經有其他 server 就併進去，不要整段覆蓋。**

```json
{
  "mcpServers": {
    "ai-db": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://192.168.8.64:8000/mcp",
        "6947",
        "--allow-http",
        "--static-oauth-client-info",
        "{\"client_id\":\"8MGJHGH157nKGeP2o5pinEghwzS6FGUU9bLASnTo\"}"
      ]
    }
  }
}
```

`args` 裡的每一項都不能省：

| 參數 | 為什麼需要 |
|---|---|
| `6947` | 固定 OAuth callback 的本機 port，redirect URI 因此是 `http://127.0.0.1:6947/oauth/callback`。不給的話 `mcp-remote` 每次挑隨機 port，Authentik 會擋掉沒登記過的 URI。要換 port 得先請 AI-DB 管理者把新的 URI 加進 Authentik client |
| `--allow-http` | AI-DB 是 `http://` 而非 https，`mcp-remote` 預設拒絕非 https 的 server |
| `--static-oauth-client-info` | Authentik 不支援 Dynamic Client Registration，`client_id` 必須自己帶 |

最後那一項的值是**字串形式的 JSON**，內層引號的 `\"` 是必要的跳脫，不是筆誤。

### 2. 重開 Claude Desktop 並登入

**完全結束 Claude Desktop 再開啟**——關掉視窗不算，macOS 用 Cmd+Q，Windows 從
系統匣結束。啟動後 `mcp-remote` 會開瀏覽器帶你走 Authentik 登入，完成後回到 app。

token 存在 `~/.mcp-auth`（Windows 是 `%USERPROFILE%\.mcp-auth`）。要重新登入就
刪掉這個目錄再重開 app。

### 3. 驗證

Settings → Developer 裡 `ai-db` 應顯示為 running，聊天框的工具選單看得到 AI-DB
的 tool。

### 4. 安裝 use-aidb skill

`npx skills add ...` 只裝給 CLI，Claude Desktop 讀不到。桌面版請在
Settings → Capabilities → Skills 加入 `use-aidb`（把 `skills/aidb/use-aidb/`
打包成 zip 上傳）。

### 5. 端到端確認

請 Claude「列出我在 AI-DB 的資料庫」。**沒有資料庫時回傳空列表是正確結果。**

## 出問題時

先做這一刀：**同一個 server，換另一個 client 測會怎樣？** 這把「server 的問題」
與「client 的 bug」分開，剩下的才值得細查。

症狀對照表、要回報什麼、不該回報什麼，見 `references/troubleshooting.md`。

**回報問題時不要貼出 token、資料庫密碼或完整的 PostgreSQL 連線字串。**
