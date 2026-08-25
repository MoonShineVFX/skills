# MoonShine Skills

MoonShine 的 Agent Skills 集合，可用 [`npx skills`](https://github.com/vercel-labs/skills) 安裝到 Claude Code、Cursor、Codex 等 agent。

## 安裝

```bash
# 列出這個 repo 有哪些 skill
npx skills add MoonShineVFX/skills --list

# 安裝指定 skill 到目前專案（./.claude/skills/）
npx skills add MoonShineVFX/skills --skill plane-api --skill plane-core

# 安裝到全域（~/.claude/skills/），所有專案共用
npx skills add MoonShineVFX/skills --skill '*' -g

# 指定 agent
npx skills add MoonShineVFX/skills --skill plane-api -a claude-code -a cursor
```

安裝後 `skills/plane/` 這層分類目錄會被攤平，實際落點是 `<agent>/skills/<skill-name>/`。

## Skills

### AI-DB

| Skill | 說明 | 相依 |
|-------|------|------|
| [`setup-aidb`](skills/aidb/setup-aidb/) | 把 Codex CLI 或 Claude Code CLI 連上 AI-DB MCP server：安裝、OAuth 登入、驗證、排錯 | — |
| [`use-aidb`](skills/aidb/use-aidb/) | 連上之後怎麼用：哪些事走 MCP、哪些事用連線字串直連 PostgreSQL | — |

### Plane

| Skill | 說明 | 相依 |
|-------|------|------|
| [`plane-core`](skills/plane/plane-core/) | Plane 共用知識庫：連線設定、欄位規範、命名慣例、快取格式 | — |
| [`plane-api`](skills/plane/plane-api/) | 用 curl + x-api-key 操作 Plane REST API（不需 MCP） | `plane-core` |
| [`okr-to-plane`](skills/plane/okr-to-plane/) | 把表格格式的 OKR Markdown 匯入 Plane，用 Cycles 做季度追蹤 | `plane-core` |

> `plane-core` 是被引用的共用基礎，安裝 `plane-api` 或 `okr-to-plane` 時請一併安裝。

## 首次設定（AI-DB）

AI-DB 是 MoonShine 內部的資料庫平台：用公司帳號建立自己的 PostgreSQL 資料庫，
拿到專屬連線字串後由你的應用程式直連。**需要在公司內網。**

先裝這兩個 skill，然後讓 agent 幫你完成安裝：

```bash
npx skills add MoonShineVFX/skills --skill setup-aidb --skill use-aidb -g -a claude-code -y
```

裝好之後跟 Claude Code 說「幫我把 AI-DB 裝起來」，它會照 `setup-aidb` 逐步執行並驗證。
用 Codex 的話把 `-a claude-code` 換成 `-a codex`。

<details>
<summary>完全沒有 agent 可用？最短手動路徑（Claude Code）</summary>

```bash
# 1. 加入 server（--client-id 不能省略，且沒有 secret）
claude mcp add --transport http --scope user \
  --client-id 8MGJHGH157nKGeP2o5pinEghwzS6FGUU9bLASnTo \
  ai-db http://192.168.8.64:8000/mcp

# 2. 用公司帳號登入（開不了瀏覽器的環境加 --no-browser）
claude mcp login ai-db

# 3. 確認：ai-db 必須顯示 ✔ Connected
claude mcp list
```

Codex CLI 的版本鎖、排錯對照表與桌面版為何連不上，都在
[`setup-aidb`](skills/aidb/setup-aidb/) 裡。

</details>

## 首次設定（Plane 系列）

連線設定放在 `~/.plane/.env`，與 skill 安裝位置無關：

```bash
mkdir -p ~/.plane
cat > ~/.plane/.env <<'ENVEOF'
PLANE_BASE_URL=https://your-plane.instance.com
PLANE_API_KEY="你自己的 api key"
PLANE_WORKSPACE=your-workspace
ENVEOF
chmod 600 ~/.plane/.env
```

API Key 取得：Plane → 右上角頭像 → **Profile** → **API Tokens**。

`~/.plane/cache.md` 由 agent 自動維護，記錄已查詢過的 Project ID 與 States，不需手動建立。

## 目錄結構

```
skills/
├── aidb/               # 分類目錄（安裝時會被攤平）
│   ├── setup-aidb/
│   │   ├── SKILL.md
│   │   └── references/     # 排錯表與「為什麼」，agent 需要時才讀
│   └── use-aidb/
└── plane/
    ├── plane-core/
    ├── plane-api/
    └── okr-to-plane/
```

新增 skill 時放在 `skills/<分類>/<skill-name>/SKILL.md`，`SKILL.md` 的 frontmatter 需有 `name` 與 `description`；`name` 就是安裝後的目錄名，全 repo 不可重複。
