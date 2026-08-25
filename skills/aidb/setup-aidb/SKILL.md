---
name: setup-aidb
description: 把 AI client 連上 MoonShine 內部的 AI-DB MCP server，涵蓋 Codex CLI 與 Claude Code CLI 的安裝、OAuth 登入與驗證。當使用者要安裝或設定 AI-DB、說連不上 AI-DB、登入沒反應、看不到 AI-DB 的 tool，或想用 AI-DB 但尚未安裝時使用。
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

# 2. 必須有 npm——兩條路徑都需要，npm 本身的安裝不在本 skill 範圍
npm --version
```

3. 使用者要有公司的 Authentik 帳號，OAuth 登入時會用到。

> **只有跑在使用者自己機器上的 CLI 連得上。** ChatGPT 與 Claude 的桌面版都是由
> 廠商雲端代連，請求從公網發出，搆不到內網的 `192.168.8.64`。那不是設定問題——
> 調參數、換 `client_id`、改 Authentik 都不會讓它們連上。理由見
> `references/limitations.md`。

## 選一條路徑

先確認使用者要設定哪一個 CLI。若不確定，就是你現在正在其中運行的那一個。

| 使用者用的是 | 走哪一節 |
|---|---|
| Claude Code | [Claude Code CLI](#claude-code-cli) |
| Codex CLI | [Codex CLI](#codex-cli) |
| 兩個都用 | 兩節都做。設定各自獨立，互不影響 |

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

## 出問題時

先做這一刀：**同一個 server，換另一個 client 測會怎樣？** 這把「server 的問題」
與「client 的 bug」分開，剩下的才值得細查。

症狀對照表、要回報什麼、不該回報什麼，見 `references/troubleshooting.md`。

**回報問題時不要貼出 token、資料庫密碼或完整的 PostgreSQL 連線字串。**
