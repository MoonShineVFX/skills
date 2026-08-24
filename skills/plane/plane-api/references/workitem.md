# Work Item API Reference

> Base URL: `$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}`
> Auth header: `X-API-Key: $PLANE_API_KEY`

---

## ⚠️ 重要：Endpoint 路徑

官方已改為 `/work-items/`，舊的 `/issues/` 路徑**已棄用**。

| Method | Path | 說明 |
|--------|------|------|
| `POST` | `/work-items/` | 建立 Work Item |
| `GET` | `/work-items/` | 列出所有 Work Items |
| `GET` | `/work-items/{work_item_id}/` | 取得單一 Work Item |
| `PATCH` | `/work-items/{work_item_id}/` | 更新 Work Item |
| `DELETE` | `/work-items/{work_item_id}/` | 刪除 Work Item（不可復原） |

> **尾斜線規律**：Work Item **全部有尾斜線**（包含 Get / Update / Delete），與 Label 不同。

---

## Create Work Item

**POST** `/work-items/`

```bash
curl -s -X POST \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/" \
  -d '{
    "name": "實作登入頁面",
    "description_html": "<p>支援 SSO 登入</p>",
    "state": "{STATE_UUID}",
    "assignees": ["{USER_UUID}"],
    "labels": ["{LABEL_UUID}"],
    "priority": "high",
    "start_date": "2026-03-10",
    "target_date": "2026-03-20",
    "parent": null
  }'
```

### Body Parameters

| 參數 | 必填 | 類型 | 說明 |
|------|------|------|------|
| `name` | ✅ | string | Work Item 標題 |
| `description_html` | — | string | HTML 描述，建議用 `<p>...</p>` |
| `state` | — | uuid | 狀態 ID（省略則使用專案預設 backlog state） |
| `assignees` | — | uuid[] | 指派人員（陣列） |
| `labels` | — | uuid[] | 標籤（陣列） |
| `priority` | — | string | `none` / `urgent` / `high` / `medium` / `low`（預設 `none`） |
| `parent` | — | uuid | 父 Work Item（sub-issue） |
| `estimate_point` | — | int | 估算點數 `0–7`，null 表示未估算 |
| `start_date` | — | string | `YYYY-MM-DD` |
| `target_date` | — | string | `YYYY-MM-DD` |
| `type` | — | uuid | Work Item Type ID |
| `module` | — | uuid | Module ID |

回傳：`201` + Work Item 物件

---

## List Work Items

**GET** `/work-items/`

```bash
# 列出全部（回傳是分頁 envelope，要用 .results[]，不是 .[]）
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/" \
  | jq '.results[] | {id, sequence_id, name, state, priority}'

# 篩選特定 state + 分頁
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/?state={STATE_UUID}&limit=50&offset=0"

# 展開 state / assignees / labels 為完整物件
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/?expand=state,assignees,labels"
```

### Query Parameters

| 參數 | 說明 |
|------|------|
| `state` | 依 state UUID 篩選 |
| `assignee` | 依 assignee user UUID 篩選 |
| `project` | 依 project ID 篩選（跨專案查詢時使用） |
| `limit` | 每頁筆數 |
| `offset` | 分頁偏移量 |
| `expand` | 展開欄位（見下方） |

### `expand` 可用值

`type`, `module`, `labels`, `assignees`, `state`, `project`

逗號分隔多個：`?expand=state,assignees,labels`  
展開後對應欄位從 UUID 變為完整物件（省去二次查詢）。

---

## Get Work Item

**GET** `/work-items/{work_item_id}/` （有尾斜線）

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/"

# 同時展開所有關聯欄位
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/?expand=state,assignees,labels,type,module"
```

---

## Update Work Item

**PATCH** `/work-items/{work_item_id}/` （有尾斜線）

```bash
# 更新優先度與狀態
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/" \
  -d '{
    "state": "{STATE_UUID}",
    "priority": "urgent"
  }'

# 設定 parent（轉為 sub-issue）
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/" \
  -d '{
    "parent": "{PARENT_WORK_ITEM_UUID}"
  }'

# 批次替換 assignees（傳入新陣列，非 append）
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/" \
  -d '{
    "assignees": ["{USER_UUID_1}", "{USER_UUID_2}"]
  }'
```

> PATCH 所有欄位皆為選填，只需傳入要修改的欄位。
> `assignees` / `labels` 是**完整替換**，不是 append。要保留原有值需先 GET 再合併。

---

## Delete Work Item

**DELETE** `/work-items/{work_item_id}/` （有尾斜線）

```bash
curl -s -X DELETE \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/work-items/{WORK_ITEM_ID}/"
```

回傳：`204 No Content`

> ⚠️ 永久刪除，**不可復原**。Sub-issues 行為待確認（是否連帶刪除）。

---

## Work Item 物件欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | uuid | Work Item ID |
| `sequence_id` | int | 人類可讀流水號（專案內唯一）。與專案 `identifier` 合成使用者口中的識別碼，如 `AIDB-3` = 專案 `AIDB` 的 `sequence_id` 3。換算方式見 `SKILL.md`「Work Item 識別碼」 |
| `name` | string | 標題 |
| `description_html` | string | HTML 描述 |
| `description_stripped` | string | 純文字版（自動產生） |
| `priority` | string | none / urgent / high / medium / low |
| `state` | uuid | 狀態 ID（可 expand 為物件） |
| `assignees` | uuid[] | 指派人員（可 expand） |
| `labels` | uuid[] | 標籤（可 expand） |
| `parent` | uuid / null | 父 Work Item |
| `estimate_point` | int / null | 估算點 0–7 |
| `start_date` | date | 開始日 |
| `target_date` | date | 截止日 |
| `completed_at` | timestamp / null | 進入 completed group state 的時間 |
| `archived_at` | timestamp / null | 封存時間 |
| `is_draft` | bool | 是否為草稿 |
| `type` | uuid | Work Item Type（可 expand） |
| `module` | uuid | Module（可 expand） |
| `sort_order` | float | 排序權重（系統自動） |
| `project` | uuid | 所屬專案 |
| `workspace` | uuid | 所屬 workspace |

---

## 各 Resource 尾斜線對照

| Resource | Create | List | Get | Update | Delete |
|----------|--------|------|-----|--------|--------|
| Project | ✅ | ✅ | ✅ | ❌ | ❌ |
| State | ✅ | ✅ | ✅ | ✅ | ✅ |
| Label | ✅ | ✅ | ❌ | ❌ | ❌ |
| Work Item | ✅ | ✅ | ✅ | ✅ | ✅ |
