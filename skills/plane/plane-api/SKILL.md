---
name: plane-api
description: 使用 curl + x-api-key 操作 Plane 專案管理系統。當使用者要在 Plane 建立、查詢、更新 work items/issues，或提到「建立 Plane 任務」、「新增 work item」、「更新 Plane issue」時使用此 skill。使用者以 `AIDB-3`、`SHO-12` 這種「專案代號-數字」形式指稱某個 work item 時（例如「來做 AIDB-3」、「把 SHO-12 改成 Done」）也用此 skill。不需要 MCP，直接透過 REST API 操作。
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

## Work Item 識別碼：`AIDB-3` 這種形式

使用者通常直接以 `AIDB-3`、`SHO-12` 指稱某個 work item。格式是
`{專案 identifier}-{sequence_id}`：

- **前綴就是專案**。`AIDB` 是專案的 `identifier` 欄位（專案短代號），不是 UUID。
  看到前綴就等於已經指定了專案，**不要再問使用者「這是哪個專案」**。
- 後面的數字是 `sequence_id`（專案內流水號），**不是** work item 的 UUID。
  所有 API 路徑吃的都是 UUID，一定要先換算。

> ⚠️ `sequence_id` 與標題開頭的編號常常不一致。例如 AIDB 專案的 `AIDB-3`
> 標題是「2. 建立 Authentik OAuth provider」。**以 `sequence_id` 為準**，
> 使用者說的號碼指的是識別碼，不是標題裡的序號。

### 換算：`AIDB-3` → project UUID + work item UUID

REST API 沒有「用識別碼直接查」的 endpoint，要兩段式換算：

```bash
source ~/.plane/.env

IDENT=AIDB   # 前綴
SEQ=3        # 數字

# 1) 前綴 → project UUID
PROJECT_ID=$(curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/" \
  | jq -r --arg i "$IDENT" '.results[] | select(.identifier == $i) | .id')

# 2) sequence_id → work item UUID
WORK_ITEM_ID=$(curl -s -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/$PROJECT_ID/work-items/" \
  | jq -r --argjson s "$SEQ" '.results[] | select(.sequence_id == $s) | .id')

echo "$PROJECT_ID / $WORK_ITEM_ID"
```

拿到兩個 UUID 後，就能套用下方所有 `{PROJECT_ID}` / `{WORK_ITEM_ID}` 的指令。

> 用 MCP 的話 `plane:retrieve_work_item_by_identifier("AIDB-3")` 一步到位，
> 但只回傳 work item；要 project UUID 仍需讀回傳的 `project` 欄位。

### ⚠️ 兩個實測踩過的坑

- **`?sequence_id=3` 這個 query 參數無效**——會被靜默忽略、回傳整份清單。
  只能用 jq 過濾。
- **列表回傳是分頁 envelope，不是陣列**。要用 `.results[]`，`jq '.[]'` 取不到東西。
  項目多時要看 `next_page_results` / `next_cursor` 翻頁，否則號碼大的會查不到。

## 常用快速指令

> 假設已 source .env，變數 `$PLANE_BASE_URL`, `$PLANE_API_KEY`, `$PLANE_WORKSPACE` 已可用。
> ⚠️ Work Item endpoint 已更新為 `/work-items/`（舊 `/issues/` 已棄用），詳見 `references/workitem.md`。

### 列出 Work Items

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/" \
  | jq '.results[] | {id, sequence_id, name, state, priority, parent}'
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
  | jq '.results[] | {id, name, identifier}'
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
