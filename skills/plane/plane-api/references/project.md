# Project API Reference

> Base URL: `$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE`
> Auth header: `X-API-Key: $PLANE_API_KEY`

---

## Endpoints

| Method | Path | 說明 |
|--------|------|------|
| `POST` | `/projects/` | 建立專案 |
| `GET` | `/projects/` | 列出所有專案 |
| `GET` | `/projects/{project_id}/` | 取得單一專案 |
| `PATCH` | `/projects/{project_id}` | 更新專案 |
| `DELETE` | `/projects/{project_id}` | 刪除專案（不可復原） |

> ⚠️ 注意尾斜線：List/Get 有 `/`，Update/Delete **沒有**。

---

## Create Project

**POST** `/projects/`

```bash
curl -s -X POST \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/" \
  -d '{
    "name": "專案名稱",
    "identifier": "PROJ",
    "description": "說明（選填）",
    "network": 2
  }'
```

| 參數 | 必填 | 說明 |
|------|------|------|
| `name` | ✅ | 專案名稱 |
| `identifier` | ✅ | 短代號，用於 work item ID |
| `description` | — | 說明 |
| `network` | — | `0`=私有, `2`=公開（預設 2） |

回傳：`201` + 專案物件（含 `id`）

---

## List Projects

**GET** `/projects/`

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/" \
  | jq '.[] | {id, name, identifier}'
```

回傳：陣列，依建立時間**倒序**排列（最新在前）

---

## Get Project

**GET** `/projects/{project_id}/`

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/"
```

---

## Update Project

**PATCH** `/projects/{project_id}` （無尾斜線）

```bash
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}" \
  -d '{
    "name": "新名稱",
    "description": "新說明"
  }'
```

只需傳要修改的欄位，未傳的欄位保持不變。

---

## Delete Project

**DELETE** `/projects/{project_id}` （無尾斜線）

```bash
curl -s -X DELETE \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}"
```

回傳：`204 No Content`

> ⚠️ 永久刪除，包含所有 work items、cycles、modules，**不可復原**。

---

## Project 物件欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | uuid | 專案 ID |
| `name` | string | 名稱 |
| `identifier` | string | 短代號（如 `PROJ`） |
| `description` | string | 說明 |
| `network` | int | `0`=私有, `2`=公開 |
| `total_members` | int | 成員數 |
| `total_cycles` | int | Cycle 數 |
| `total_modules` | int | Module 數 |
| `project_lead` | uuid | 負責人 user_id |
| `default_assignee` | uuid | 預設指派人 |
| `cycle_view` | bool | 是否啟用 Cycle |
| `module_view` | bool | 是否啟用 Module |
| `archive_in` | int | 幾個月後自動封存（0=不自動） |
| `close_in` | int | 幾個月後自動關閉（0=不自動） |