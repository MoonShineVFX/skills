---
name: plane-api
description: 使用 curl + x-api-key 操作 Plane 專案管理系統。當使用者要在 Plane 建立、查詢、更新 work items/issues，或提到「建立 Plane 任務」、「新增 work item」、「更新 Plane issue」時使用此 skill。不需要 MCP，直接透過 REST API 操作。
---

# Plane API via curl

不使用 MCP，直接以 curl 呼叫 Plane REST API。

## 連線資訊

**使用前先讀取** `~/.cursor/skills/plane-api/.env`，取得以下變數：

```
PLANE_BASE_URL   # Plane 實例網址
PLANE_API_KEY    # x-api-key 的值
PLANE_WORKSPACE  # workspace slug
```

所有 curl 指令均以這三個變數組成 URL 和 Header：

```bash
# 以 shell 載入（執行 curl 前先 source）
source ~/.cursor/skills/plane-api/.env

# 或直接 export 供後續使用
export $(cat ~/.cursor/skills/plane-api/.env | xargs)
```

## 已知專案

| 專案 | Project ID |
|------|-----------|
| LED 牆 | `08593f37-0c1f-4014-9437-26491832b083` |

### LED 牆 States

| 狀態 | ID |
|------|---|
| Backlog | `37eb9789-1031-4d30-bf53-f4f9df9dc2dd` |
| Todo | `05d56210-9990-4220-8560-265b91a3145a` |
| In Progress | `4dd5c4fd-3dcb-402c-81c4-7a0f426e752c` |
| Done | `4c0ec082-b09f-480e-b853-4b9a55eeb287` |
| Cancelled | `00900430-befb-4e32-af21-4e98fa29b6ab` |

## 常用 curl 指令

> 假設已 source .env，變數 `$PLANE_BASE_URL`, `$PLANE_API_KEY`, `$PLANE_WORKSPACE` 已可用。

### 列出 Issues

```bash
curl -s \
  -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/issues/" \
  | jq '.results[] | {id, sequence_id, name, state, priority, parent}'
```

### 建立 Issue

```bash
curl -s -X POST \
  -H "x-api-key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/issues/" \
  -d '{
    "name": "Issue 標題",
    "description_html": "<p>詳細描述</p>",
    "state": "{STATE_ID}",
    "priority": "high",
    "parent": null
  }'
```

回傳 JSON 中的 `id` 即為新 issue 的 UUID。

### 建立子 Issue

```bash
-d '{ ..., "parent": "{PARENT_ISSUE_ID}" }'
```

### 更新 Issue

```bash
curl -s -X PATCH \
  -H "x-api-key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/issues/{ISSUE_ID}/" \
  -d '{ "state": "{STATE_ID}" }'
```

### 查詢專案列表

```bash
curl -s \
  -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/" \
  | jq '.[] | {id, name, identifier}'
```

### 查詢 States（新專案）

```bash
curl -s \
  -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/" \
  | jq '.[] | {id, name, group}'
```

### Archive / Unarchive Issue（非公開內部 API）

> ⚠️ **此為 Plane UI 內部使用的非公開 API**（官方文件未收錄），透過 DevTools Network 監聽確認。
> - 路徑為 `/api/workspaces/`（**不含 `v1`**），與其他公開 API 不同
> - **完全不支援 x-api-key 認證**，必須使用瀏覽器 session cookie
>
> **🔑 使用前必做：直接詢問使用者提供目前的 session token**
>
> 請用以下提示詢問使用者：
> > 「請開啟 Plane，在 DevTools → Application → Cookies 中複製目前的 `csrftoken` 和 `session-id` 的值給我。」

#### 取得 Cookie（每次 session 過期需重新取得）

1. 登入 Plane，開啟 DevTools → Application → Cookies
2. 複製 `csrftoken` 和 `session-id` 的完整值

#### 執行 Archive（POST）

```bash
CSRF="<csrftoken 的值>"
SESSION="<session-id 的值>"

curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "X-CSRFToken: $CSRF" \
  -H "Cookie: csrftoken=$CSRF; session-id=$SESSION" \
  -H "Referer: $PLANE_BASE_URL/" \
  "$PLANE_BASE_URL/api/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/issues/{ISSUE_ID}/archive/" \
  -d '{}'
```

回傳 `{"archived_at":"YYYY-MM-DD"}` 表示成功。

#### 執行 Unarchive（DELETE）

> Archive 的逆操作：對同一個 `/archive/` 端點發送 **DELETE** 請求。

```bash
CSRF="<csrftoken 的值>"
SESSION="<session-id 的值>"

curl -s -X DELETE \
  -H "X-CSRFToken: $CSRF" \
  -H "Cookie: csrftoken=$CSRF; session-id=$SESSION" \
  -H "Referer: $PLANE_BASE_URL/" \
  "$PLANE_BASE_URL/api/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/issues/{ISSUE_ID}/archive/"
```

回傳 HTTP `204 No Content` 表示成功。

#### 列出所有已 Archive 的 Issue（內部 API）

```bash
curl -s \
  -H "X-CSRFToken: $CSRF" \
  -H "Cookie: csrftoken=$CSRF; session-id=$SESSION" \
  -H "Referer: $PLANE_BASE_URL/" \
  "$PLANE_BASE_URL/api/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/archived-issues/?per_page=100" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
results = data.get('results', [])
print(f'Total archived: {data.get(\"total_count\", len(results))}')
for item in results:
    print(f'{item[\"id\"]}|{item.get(\"sequence_id\",\"\")}|{item[\"name\"]}')
"
```

#### 批次 Archive 多個 Issue

```bash
source ~/.cursor/skills/plane-api/.env
CSRF="<csrftoken 的值>"
SESSION="<session-id 的值>"
PROJECT_ID="{PROJECT_ID}"

# 取得目標 issue IDs 後逐一 archive
for ID in uuid1 uuid2 uuid3; do
  RESP=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
    -H "Content-Type: application/json" \
    -H "X-CSRFToken: $CSRF" \
    -H "Cookie: csrftoken=$CSRF; session-id=$SESSION" \
    -H "Referer: $PLANE_BASE_URL/" \
    "$PLANE_BASE_URL/api/workspaces/$PLANE_WORKSPACE/projects/$PROJECT_ID/issues/$ID/archive/" \
    -d '{}')
  echo "HTTP $RESP  $ID"
done
```

#### 批次 Unarchive 所有已封存的 Issue

```bash
source ~/.cursor/skills/plane-api/.env
CSRF="<csrftoken 的值>"
SESSION="<session-id 的值>"
PROJECT_ID="{PROJECT_ID}"

# 先取得所有 archived items
ITEMS=$(curl -s \
  -H "X-CSRFToken: $CSRF" \
  -H "Cookie: csrftoken=$CSRF; session-id=$SESSION" \
  -H "Referer: $PLANE_BASE_URL/" \
  "$PLANE_BASE_URL/api/workspaces/$PLANE_WORKSPACE/projects/$PROJECT_ID/archived-issues/?per_page=100" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data.get('results', []):
    print(item['id'] + '|' + str(item.get('sequence_id','')) + '|' + item['name'])
")

COUNT=0
while IFS='|' read -r ID SEQ NAME; do
  [ -z "$ID" ] && continue
  RESP=$(curl -s -o /dev/null -w "%{http_code}" -X DELETE \
    -H "X-CSRFToken: $CSRF" \
    -H "Cookie: csrftoken=$CSRF; session-id=$SESSION" \
    -H "Referer: $PLANE_BASE_URL/" \
    "$PLANE_BASE_URL/api/workspaces/$PLANE_WORKSPACE/projects/$PROJECT_ID/issues/$ID/archive/")
  [ "$RESP" = "204" ] && echo "OK  [$SEQ] $NAME" || echo "FAIL ($RESP)  [$SEQ] $NAME"
  COUNT=$((COUNT + 1))
done <<< "$ITEMS"
echo "Done: $COUNT items processed"
```

### 查詢 Workspace 成員

```bash
curl -s \
  -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/members/" \
  | jq '.[] | {id, display_name, email}'
```

## 欄位規範

| 欄位 | 可用值 |
|------|--------|
| `priority` | `urgent` / `high` / `medium` / `low` / `none` |
| `description_html` | HTML 字串（`<p>`, `<ul>`, `<li>`, `<strong>` 等） |
| `state` | State UUID |
| `assignees` | `["user-uuid"]`（陣列） |
| `parent` | Issue UUID 或 `null` |

## 已知限制

- **Relations API 不支援**：self-hosted CE 版本無 `/relations/` endpoint，以 description 的「前置條件」欄位代替

## 建立新專案的標準流程

### 1. 建立專案

```bash
curl -s -X POST \
  -H "x-api-key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/" \
  -d '{
    "name": "專案名稱",
    "identifier": "PROJ",
    "network": 2
  }'
```

記錄回傳的 `id` 作為 `PROJECT_ID`。

### 2. 初始化 States

建立新專案後，**必須**依序建立以下標準 states（先查詢並刪除預設 states，再建立）：

#### 查詢現有 States

```bash
curl -s \
  -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/" \
  | python3 -c "import sys,json; [print(s['id'], s['group'], s['name']) for s in json.load(sys.stdin)]"
```

#### 建立 State 指令

```bash
curl -s -X POST \
  -H "x-api-key: $PLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$PLANE_BASE_URL/api/v1/workspaces/$PLANE_WORKSPACE/projects/{PROJECT_ID}/states/" \
  -d '{
    "name": "{STATE_NAME}",
    "color": "{HEX_COLOR}",
    "group": "{GROUP}",
    "default": false
  }'
```

#### 標準 States 規格

| 名稱 | group | color | default |
|------|-------|-------|---------|
| Backlog | `backlog` | `#D9D9D9` | false |
| Todo | `unstarted` | `#D9D9D9` | false |
| In Progress | `started` | `#F59E0B` | **true** |
| Review | `started` | `#9900EF` | false |
| Done | `completed` | `#16A34A` | false |
| Cancelled | `cancelled` | `#ABB8C3` | false |

#### 批次建立所有 States

```bash
source ~/.cursor/skills/plane-api/.env
PROJECT_ID="{PROJECT_ID}"

create_state() {
  curl -s -X POST \
    -H "x-api-key: $PLANE_API_KEY" \
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

## 建立 Work Items 的標準流程

1. `source ~/.cursor/skills/plane-api/.env`
2. 列出既有 issues，確認目前最大的 sequence_id
3. 建立 parent issue，記錄回傳的 `id`
4. 建立子 issues，填入 `"parent": "{parent_id}"`
5. 再次列出 issues 驗證父子關係

## description_html 模板（AI Agent 用）

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
  <li>依賴 {LED-N}：{說明}</li>
</ul>

<h3>驗收條件</h3>
<ul>
  <li>{驗收項目 1}</li>
</ul>

<h3>相關資源</h3>
<ul>
  <li>Repo: /home/noflame/interactive-photo-wall</li>
  <li>Tech stack: Vite + TypeScript + Firebase</li>
</ul>
```
