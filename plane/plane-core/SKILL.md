---
name: plane-core
description: 核心知識庫，供 plane-api 和 okr-to-plane 引用。不直接觸發，僅作為共享參照。包含連線設定、欄位規範、命名慣例、Member lookup、專案建立注意事項。
---

# Plane 核心知識庫

> 此文件為共享基礎，由 `plane-api` 和 `okr-to-plane` 引用。

## 連線資訊

`.env` 位於此 skill 目錄（`plane-core/`），使用前先載入：

```bash
# 載入環境變數（curl 前先執行）
source ~/.cursor/skills/plane/plane-core/.env
# 取得：PLANE_BASE_URL, PLANE_API_KEY, PLANE_WORKSPACE

# 或直接 export 供後續使用
export $(cat ~/.cursor/skills/plane/plane-core/.env | xargs)
```

## 已知專案快取

讀取 `~/.cursor/skills/plane/plane-core/cache.md` 取得已快取的專案 ID 與 States。

**寫入時機：**
- 查詢到新專案且確認會重複使用 → 寫入「已知專案」
- 查詢到新專案的 States → 寫入「已知 States」對應小節

**移除時機：**
- 使用者明確表示專案不再使用 → 移除對應條目

**快取格式（新增專案時依此格式寫入）：**

```markdown
## 已知專案
| 專案 | Project ID |
|------|-----------|
| {專案名稱} | `{project-uuid}` |

## 已知 States
### {專案名稱}
| 狀態 | ID |
|------|---|
| {State 名稱} | `{state-uuid}` |
```

## 欄位規範

| 欄位 | 可用值 |
|------|--------|
| `priority` | `urgent` / `high` / `medium` / `low` / `none` |
| `description_html` | HTML 字串（`<p>`, `<ul>`, `<li>`, `<strong>` 等） |
| `state` | State UUID |
| `assignees` | `["user-uuid"]`（陣列） |
| `parent` | Issue UUID 或 `null` |

## 命名慣例（OKR 類型）

| 類型 | 前綴 | 範例 |
|------|------|------|
| Objective | `[O]` | `[O] 建立 4DGS 能力 (Q1/Q2)` |
| Key Result | `[KR]` | `[KR] 完成 Blender 基礎建模培訓` |

跨季度拆分時，在標題加上季度標記：`(Q1/Q4)` 表示全年度計畫的 Q1 部分。

## 優先級建議

| 類型 | priority |
|------|----------|
| Objective | `high` |
| Key Result | `medium` |

## Member Lookup

```bash
# curl 方式
curl -s \
  -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/members/" \
  | jq '.[] | {id, display_name, email}'

# MCP 方式
plane:get_workspace_members
plane:get_project_members  # 限定專案範圍
```

找到 user_id（UUID 格式）後，帶入 `assignees: ["user-uuid"]`。

## 建立新專案的注意事項

⚠️ **不要用 MCP `plane:create_project` 建立新專案。**
MCP 建立的專案缺少預設 State 設定，導致後續建立 Work Item 時失敗。

正確流程：
1. 讓使用者在 **Plane 介面手動建立**專案
2. 取得 Project ID 後再繼續

若使用者堅持用 API 建立，請參考 `plane-api.md` 的「建立新專案的標準流程」，並在建立後**手動初始化 States**。

## 標準 States 規格（新專案用）

| 名稱 | group | color | default |
|------|-------|-------|---------|
| Backlog | `backlog` | `#D9D9D9` | false |
| Todo | `unstarted` | `#D9D9D9` | false |
| In Progress | `started` | `#F59E0B` | **true** |
| Review | `started` | `#9900EF` | false |
| Done | `completed` | `#16A34A` | false |
| Cancelled | `cancelled` | `#ABB8C3` | false |

## description_html 模板

```html
<h3>背景</h3>
<p>{背景說明}</p>

<h3>目標</h3>
<p>{本 issue 要達成的目標}</p>

<h3>實作步驟</h3>
<ol>
  <li>{步驟 1}</li>
  <li>{步驟 2}</li>
</ol>

<h3>前置條件</h3>
<ul>
  <li>依賴 {序號}：{說明}</li>
</ul>

<h3>驗收條件</h3>
<ul>
  <li>{驗收項目 1}</li>
</ul>
```

## 已知限制

- **Relations API 不支援**：self-hosted CE 版無 `/relations/` endpoint，改用 description 的「前置條件」欄位代替
- **Archive API 為非公開內部 API**：需要瀏覽器 session cookie，詳見 `plane-api.md`
