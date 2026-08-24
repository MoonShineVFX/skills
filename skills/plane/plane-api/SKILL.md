---
name: plane-api
description: 使用 curl + x-api-key 操作 Plane 專案管理系統。當使用者要在 Plane 建立、查詢、更新 work items/issues，或提到「建立 Plane 任務」、「新增 work item」、「更新 Plane issue」時使用此 skill。不需要 MCP，直接透過 REST API 操作。
---

# Plane API via curl

不使用 MCP，直接以 curl 呼叫 Plane REST API。

## 連線資訊

> 連線設定由 `plane-core` 統一管理，執行前先載入：

```bash
source ~/.plane/.env
```

> 已知專案、States、欄位規範請參考 `plane-core/SKILL.md`。

## References 索引

| 操作 | 參考檔案 |
|------|---------|
| Work Items CRUD（列出、建立、更新、刪除） | `references/workitem.md` |
| 專案管理（建立、查詢、更新） | `references/project.md` |
| States 管理（建立、查詢、初始化） | `references/state.md` |
| Labels 管理 | `references/labels.md` |
| 成員查詢（取得 user_id） | `references/member.md` |
| Archive / Unarchive（非公開內部 API） | `references/archive.md` |

## 常用快速指令

> 假設已 source .env，變數 `$PLANE_BASE_URL`, `$PLANE_API_KEY`, `$PLANE_WORKSPACE` 已可用。
> ⚠️ Work Item endpoint 已更新為 `/work-items/`（舊 `/issues/` 已棄用），詳見 `references/workitem.md`。

### 列出 Work Items

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/" \
  | jq '.[] | {id, sequence_id, name, state, priority, parent}'
```

### 建立 Work Item

```bash
curl -s -X POST \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/" \
  -d '{
    "name": "Work Item 標題",
    "description_html": "<p>詳細描述</p>",
    "state": "{STATE_UUID}",
    "priority": "high",
    "parent": null
  }'
```

回傳 JSON 中的 `id` 即為新 work item 的 UUID。完整欄位說明見 `references/workitem.md`。

### 更新 Work Item

```bash
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/" \
  -d '{ "state": "{STATE_UUID}" }'
```

### 查詢專案列表

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/" \
  | jq '.[] | {id, name, identifier}'
```

### Archive / Unarchive Issue（非公開內部 API）

> ⚠️ 此為非公開內部 API，需使用瀏覽器 session cookie，不支援 x-api-key。
> 詳細指令請參考 `references/archive.md`。

## 已知限制

- **Relations API 不支援**：self-hosted CE 版本無 `/relations/` endpoint，以 description 的「前置條件」欄位代替

## 建立新專案的標準流程

1. 建立專案 → 詳見 `references/project.md`（Create Project）
2. 查詢並刪除預設 States → 詳見 `references/state.md`（List / Delete State）
3. 批次建立標準 States → 詳見 `references/state.md`（標準 States 規格 + 批次建立腳本）

> ⚠️ 建議讓使用者在 Plane 介面手動建立專案，詳見 `plane-core/SKILL.md`「建立新專案的注意事項」。

## 建立 Work Items 的標準流程

1. `source ~/.plane/.env`
2. 列出既有 work items，確認目前最大的 `sequence_id`
3. 建立 parent work item，記錄回傳的 `id`
4. 建立子 work items，填入 `"parent": "{parent_id}"`
5. 再次列出 work items 驗證父子關係

詳細指令與欄位說明見 `references/workitem.md`。  
`description_html` 模板見 `plane-core/SKILL.md`。
