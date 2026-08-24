# Label API Reference

> Base URL: `$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}`
> Auth header: `X-API-Key: $PLANE_API_KEY`

---

## Endpoints

| Method | Path | 說明 |
|--------|------|------|
| `POST` | `/labels/` | 建立 Label |
| `GET` | `/labels/` | 列出所有 Labels |
| `GET` | `/labels/{label_id}` | 取得單一 Label |
| `PATCH` | `/labels/{label_id}` | 更新 Label |
| `DELETE` | `/labels/{label_id}` | 刪除 Label（不可復原） |

> ⚠️ **尾斜線不一致**：Create / List **有**尾斜線；Get / Update / Delete **沒有**。
> 這與 State（全部有斜線）不同，與 Project Update/Delete 的模式相同。

---

## Create Label

**POST** `/labels/`

```bash
curl -s -X POST \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/labels/" \
  -d '{
    "name": "bug",
    "color": "#eb5757",
    "parent": null
  }'
```

| 參數 | 必填 | 說明 |
|------|------|------|
| `name` | ✅ | Label 名稱 |
| `color` | — | Hex 色碼（選填，可為空字串） |
| `parent` | — | 父 Label UUID（支援巢狀），無則 `null` |

回傳：`201` + Label 物件（含 `id`）

---

## List Labels

**GET** `/labels/`

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/labels/" \
  | jq '.[] | {id, name, color, parent}'
```

回傳：陣列，依 `sort_order` 排列。

---

## Get Label

**GET** `/labels/{label_id}` （無尾斜線）

```bash
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/labels/{LABEL_ID}"
```

---

## Update Label

**PATCH** `/labels/{label_id}` （無尾斜線）

```bash
curl -s -X PATCH \
  -H "X-API-Key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/labels/{LABEL_ID}" \
  -d '{
    "name": "新名稱",
    "color": "#16A34A"
  }'
```

> ⚠️ 官方文件把 Update 的 `name` 標為 **required**，但語意上 PATCH 應該允許只傳要改的欄位。實際行為待驗證。

---

## Delete Label

**DELETE** `/labels/{label_id}` （無尾斜線）

```bash
curl -s -X DELETE \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/labels/{LABEL_ID}"
```

回傳：`204 No Content`

> ⚠️ 永久刪除，**不可復原**。

---

## Label 物件欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | uuid | Label ID |
| `name` | string | 名稱 |
| `color` | string | Hex 色碼（可為空字串） |
| `parent` | uuid / null | 父 Label（支援巢狀） |
| `sort_order` | float | 排序權重（自動產生） |
| `project` | uuid | 所屬專案 |
| `workspace` | uuid | 所屬 workspace |

---

## State 與 Label 的差異對照

| 面向 | State | Label |
|------|-------|-------|
| 必填建立欄位 | `name` + `color` + `group` | `name` 只有這一個 |
| `color` | 必填 | 選填 |
| 特有欄位 | `group`, `default`, `sequence` | `parent`（巢狀） |
| Get/Update/Delete 尾斜線 | **有** | **沒有** |
