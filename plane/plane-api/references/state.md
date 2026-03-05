# State API Reference

> Base URL: `$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}`
> Auth header: `X-API-Key: $PLANE_API_KEY`

---

## Endpoints

| Method | Path | 說明 |
|--------|------|------|
| `POST` | `/states/` | 建立 State |
| `GET` | `/states/` | 列出所有 States |
| `GET` | `/states/{state_id}/` | 取得單一 State |
| `PATCH` | `/states/{state_id}/` | 更新 State |
| `DELETE` | `/states/{state_id}/` | 刪除 State（不可復原） |

> State 所有端點 URL 都**有尾斜線**（與 Project Update/Delete 不同）。

---

## Create State

**POST** `/states/`

```bash
curl -s -X POST \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/" \
  -d '{
    "name": "In Progress",
    "color": "#F59E0B",
    "group": "started",
    "default": false
  }'
```

| 參數 | 必填 | 說明 |
|------|------|------|
| `name` | ✅ | State 名稱 |
| `color` | ✅ | Hex 色碼，例如 `"#F59E0B"` |
| `group` | ✅ | 所屬群組（見下方群組列表） |
| `default` | — | 是否為預設 State（`true`/`false`） |

回傳：`201` + State 物件（含 `id`）

**group 合法值：**

| group | 說明 |
|-------|------|
| `backlog` | 待辦積壓 |
| `unstarted` | 尚未開始 |
| `started` | 進行中 |
| `completed` | 已完成 |
| `cancelled` | 已取消 |

> ⚠️ 官方 Create 文件只列 `name` 和 `color` 為必填，但 Overview 明確指出 `group` 也是 required。實際使用**務必帶入 `group`**，否則行為未定義。

---

## List States

**GET** `/states/`

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/" \
  | jq '.[] | {id, name, group, color, default}'
```

回傳：陣列，依各 group 內的 `sequence` 順序排列。

---

## Get State

**GET** `/states/{state_id}/`

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/{STATE_ID}/"
```

---

## Update State

**PATCH** `/states/{state_id}/`

```bash
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/{STATE_ID}/" \
  -d '{
    "name": "新名稱",
    "color": "#16A34A"
  }'
```

只需傳要修改的欄位。

---

## Delete State

**DELETE** `/states/{state_id}/`

```bash
curl -s -X DELETE \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/{STATE_ID}/"
```

回傳：`204 No Content`

> ⚠️ 永久刪除，**不可復原**。若有 work item 仍在此 State，需先移轉。

---

## State 物件欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | uuid | State ID |
| `name` | string | 名稱 |
| `color` | string | Hex 色碼 |
| `group` | string | 所屬群組 |
| `default` | boolean | 是否為預設 State |
| `sequence` | float | 群組內排序（自動產生） |
| `project` | uuid | 所屬專案 |
| `workspace` | uuid | 所屬 workspace |

---

## 標準 States 規格（新專案初始化用）

標準 States 規格詳見 `plane-core/SKILL.md`。

### 批次建立所有 States

```bash
source ~/.cursor/skills/plane/plane-core/.env
PROJECT_ID="{PROJECT_ID}"

create_state() {
  curl -s -X POST \
    -H "X-API-Key: $PLANE_API_KEY" \
    -H "Content-Type: application/json" \
    "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/$PROJECT_ID/states/" \
    -d "$1" \
    | python3 -c "import sys,json; s=json.load(sys.stdin); print(s.get('name','?'), s.get('id','ERROR'))"
}

create_state '{"name":"Backlog",     "color":"#D9D9D9","group":"backlog",    "default":false}'
create_state '{"name":"Todo",        "color":"#D9D9D9","group":"unstarted",  "default":false}'
create_state '{"name":"In Progress", "color":"#F59E0B","group":"started",    "default":true}'
create_state '{"name":"Review",      "color":"#9900EF","group":"started",    "default":false}'
create_state '{"name":"Done",        "color":"#16A34A","group":"completed",  "default":false}'
create_state '{"name":"Cancelled",   "color":"#ABB8C3","group":"cancelled",  "default":false}'
```
