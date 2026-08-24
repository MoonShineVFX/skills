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

| Skill | 說明 | 相依 |
|-------|------|------|
| [`plane-core`](skills/plane/plane-core/) | Plane 共用知識庫：連線設定、欄位規範、命名慣例、快取格式 | — |
| [`plane-api`](skills/plane/plane-api/) | 用 curl + x-api-key 操作 Plane REST API（不需 MCP） | `plane-core` |
| [`okr-to-plane`](skills/plane/okr-to-plane/) | 把表格格式的 OKR Markdown 匯入 Plane，用 Cycles 做季度追蹤 | `plane-core` |

> `plane-core` 是被引用的共用基礎，安裝 `plane-api` 或 `okr-to-plane` 時請一併安裝。

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
└── plane/              # 分類目錄（安裝時會被攤平）
    ├── plane-core/
    ├── plane-api/
    └── okr-to-plane/
```

新增 skill 時放在 `skills/<分類>/<skill-name>/SKILL.md`，`SKILL.md` 的 frontmatter 需有 `name` 與 `description`；`name` 就是安裝後的目錄名，全 repo 不可重複。
