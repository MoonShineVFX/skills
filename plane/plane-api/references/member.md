# Member API Reference

> Auth header: `X-API-Key: $PLANE_API_KEY`

Members API 為**唯讀**，只有兩個 GET endpoint，無法透過 API 新增/移除成員。

---

## Endpoints

| Method | Path | 說明 |
|--------|------|------|
| `GET` | `/api/v1/workspaces/{workspace_slug}/members/` | 取得 Workspace 所有成員 |
| `GET` | `/api/v1/workspaces/{workspace_slug}/projects/{project_id}/members/` | 取得 Project 所有成員 |

兩者皆有尾斜線。

---

## Get Workspace Members

**GET** `/api/v1/workspaces/{workspace_slug}/members/`

```bash
# 取得所有成員的 id + display_name 對照表
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/members/" \
  | jq '[.[] | {id, display_name, email, role}]'

# 依 display_name 找 id（用於 assignees 欄位）
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/members/" \
  | jq '.[] | select(.display_name == "noflame") | .id'
```

Scope：`workspaces.members:read`

---

## Get Project Members

**GET** `/api/v1/workspaces/{workspace_slug}/projects/{project_id}/members/`

```bash
# 取得特定 project 的成員（subset of workspace members）
curl -s \
  -H "X-API-Key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/members/" \
  | jq '[.[] | {id, display_name, role}]'
```

Scope：`projects.members:read`

> 使用場景：確認某位成員是否已加入該 project，再 assign work item 給他。  
> 若成員不在 project 中，assign 時 API 會返回錯誤。

---

## Member 物件欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| `id` | uuid | **Work Item `assignees` 欄位需要的值** |
| `display_name` | string | 顯示名稱（UI 中看到的） |
| `first_name` | string | 名字 |
| `last_name` | string | 姓氏 |
| `email` | string | 電子郵件 |
| `avatar` | string | avatar 圖片 ref |
| `avatar_url` | string / null | avatar 公開 URL |
| `role` | integer | 角色權限等級（見下方） |

### Role 數值對照

| role 值 | 權限等級 |
|---------|----------|
| `5` | Guest |
| `10` | Viewer |
| `15` | Member |
| `20` | Admin |

> Workspace Owner 不會透過此 API 回傳 role 值，或可能以更高數值表示。

---

## 實用 jq 查詢

```bash
BASE="$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE"
MEMBERS=$(curl -s -H "X-API-Key: $PLANE_API_KEY" "$BASE/members/")

# 建立 display_name → id 對照表
echo $MEMBERS | jq '[.[] | {key: .display_name, value: .id}] | from_entries'

# 找出所有 Admin 以上成員
echo $MEMBERS | jq '[.[] | select(.role >= 20) | {display_name, role}]'

# 取得指定 email 的 user id
echo $MEMBERS | jq '.[] | select(.email == "user@moonshine.com") | .id'
```

---

## 與 `plane-core.md` 的關係

`plane-core.md` 裡的 Member Lookup 區塊即基於本 API。  
建立 Work Item 時 `assignees` 欄位需傳入 **Member 的 `id`**（UUID），  
而非 `display_name` 或 `email`。
