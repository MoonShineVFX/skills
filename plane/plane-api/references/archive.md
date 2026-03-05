# Archive / Unarchive Issue（非公開內部 API）

> ⚠️ **此為 Plane UI 內部使用的非公開 API**（官方文件未收錄），透過 DevTools Network 監聽確認。
> - 路徑為 `/api/workspaces/`（**不含 `v1`**），與其他公開 API 不同
> - **完全不支援 x-api-key 認證**，必須使用瀏覽器 session cookie
>
> **🔑 使用前必做：直接詢問使用者提供目前的 session token**
>
> 請用以下提示詢問使用者：
> > 「請開啟 Plane，在 DevTools → Application → Cookies 中複製目前的 `csrftoken` 和 `session-id` 的值給我。」

## 取得 Cookie（每次 session 過期需重新取得）

1. 登入 Plane，開啟 DevTools → Application → Cookies
2. 複製 `csrftoken` 和 `session-id` 的完整值

## 執行 Archive（POST）

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

## 執行 Unarchive（DELETE）

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

## 列出所有已 Archive 的 Issue

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

## 批次 Archive 多個 Issue

```bash
source ~/.cursor/skills/plane/plane-core/.env
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

## 批次 Unarchive 所有已封存的 Issue

```bash
source ~/.cursor/skills/plane/plane-core/.env
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
