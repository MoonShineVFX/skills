---
name: use-aidb
description: 判斷 AI-DB 操作應呼叫 MCP 管理資料庫與憑證，或由使用者的應用程式透過 PostgreSQL 連線操作 schema、SQL 與資料。當使用者要建立、列出、刪除、還原資料庫，取得或輪替連線憑證，或要建表、migration、查詢與寫入資料時使用。
---

# 使用 AI-DB

先判斷請求屬於哪一層，再採取行動。不要用 MCP 代替使用者應用程式執行 SQL，也不要讓應用程式直接管理 database lifecycle 或 access role。

## 路由規則

| 使用者要做的事 | 執行者 | 使用方式 |
|---|---|---|
| 建立 database | AI-DB MCP | `create_database` |
| 查詢或列出 database | AI-DB MCP | `get_database`、`list_databases` |
| 軟刪除或還原 database | AI-DB MCP | `delete_database`、`restore_database` |
| database 顯示可用卻連不上 | AI-DB MCP | `restore_database`（它兼任修復工具，見下） |
| 取得非秘密連線資訊 | AI-DB MCP | `get_connection_info` |
| 密碼遺失、疑似外洩或需要更換 | AI-DB MCP | 確認後呼叫 `rotate_database_credentials` |
| 建表、改 schema、執行 migration | 使用者的應用程式或 migration tool | 使用 PostgreSQL connection string 直連 |
| INSERT、SELECT、UPDATE、DELETE | 使用者的應用程式 | 使用 PostgreSQL driver／ORM 直連 |
| transaction、JOIN、index、constraint | 使用者的應用程式 | 使用 PostgreSQL、ORM 或 migration framework |

一句話判斷：**database 與 credential lifecycle 走 MCP；database 內容走使用者自己的 PostgreSQL 連線。**

## MCP 管理流程

建立 database 時，保存 `create_database` 一次性交付的結構化連線資訊與密碼。不要要求 MCP 再次顯示舊密碼；AI-DB 不保存可取回的明文密碼。

**不要用「再建立一次同名 database」來取回密碼。** 同名且可用的紀錄會回報錯誤並附上既有的 `db_id`，不會重新交付憑證。這是刻意的：若同名建立會重設憑證，任何知道名稱的人都能重設他人的憑證，並立刻中斷對方正在執行的應用程式。密碼遺失的唯一途徑是輪替。

`create_database` 的原始呼叫會等 provisioning 完成，然後取得一次性憑證。若另一個
並發呼叫或重試收到 `status: creating` **且不含憑證**，代表它只是觀察者；憑證
只會交給仍在等待的原始呼叫。觀察者應以 `get_database` 查詢狀態，**不要持續重試
建立，也不要自動輪替**。

狀態變成 `ready` 之後有兩條明確路徑：

- 原始呼叫已收到憑證：使用那次交付，不需做其他事。
- 呼叫端從未收到或已遺失憑證：取得使用者明確確認後才呼叫
  `rotate_database_credentials`。這會使原始呼叫可能已收到的密碼失效並中斷連線，
  因此它是交付遺失的恢復動作，不是 polling 的自動下一步。

密碼遺失或外洩時：

1. 先說明 rotation 會使舊密碼立即失效，可能中斷既有應用程式。
2. 取得使用者明確確認。
3. 以 `confirm=true` 呼叫 `rotate_database_credentials`。
4. 將新憑證更新至應用程式的 secret storage。
5. 重新建立應用程式連線。

不要對已刪除的 database 輪替密碼；先還原，再依需要輪替。已刪除的 database 沒有連線權，換了密碼也連不進去，因此該呼叫會被拒絕。

### 刪除與還原

`delete_database` 不需要 `confirm`，因為它是軟刪除、可在保留期內還原。但它**立即生效**：新連線被拒，且該 database 上既有的連線會被當場終止。要刪之前先確認沒有正在執行的工作。

回應中的 `purge_after` 是可還原的**期限**。過了那個時點資料會被永久清除，還原就不再可能。

還原之後**沿用原本的密碼與連線字串**，應用程式的設定不需修改。

### `restore_database` 也是修復工具

若某個 database 顯示可用（`status: ready`、`deleted_at` 為 null）卻連不上，對它呼叫 `restore_database` 即可。它會檢查所有連線條件並補齊缺少的部分。

這種狀態來自刪除或還原流程中途失敗，是系統刻意選擇的安全落點——失敗一律停在「連不進去」那一側，而不是「已刪除卻仍連得進去」。

呼叫時**不要帶 `name`**，修復路徑不會套用它。`name` 只在「還原一筆已刪除的紀錄，而原名已被其他 database 佔用」時才有意義。

不要為此輪替憑證：問題不在密碼，輪替只會多中斷一次連線。

## 應用程式資料流程

取得 connection string 後，讓使用者的應用程式、ORM 或 migration framework 直接連 PostgreSQL。不要尋找或發明 `execute_sql`、`create_table`、`query_database` 等 MCP tool。

### 工作區是 `app` schema，且 `search_path` 已經設好

每個 database 內有一個名為 `app` 的 schema，由該 database 的 access role 擁有。建立流程已把該 role 在該 database 的 `search_path` 設為 `app`，因此：

- 未限定的 `CREATE TABLE foo (...)` 直接落在 `app` 裡，**不需要寫 `app.foo`**
- `CREATE TEMP TABLE` 可用（`TEMPORARY` 隨 `CONNECT` 一併授予）
- 不要為了「保險」改用 `public`：實測 `CREATE TABLE public.x` 會得到
  `permission denied for schema public`，那個 schema 不屬於這個 role

使用者負責 `app` schema 內的：

- schema 與 migration
- table、index、constraint
- SQL、transaction 與資料內容
- 應用程式層的 tenant/user authorization

AI-DB 仍負責 database lifecycle、access role、連線權與隔離。

### access role **不能**動 database 本身

不是「不應該」，是 PostgreSQL 直接拒絕。`DROP DATABASE`、`ALTER DATABASE ... RENAME`、`ALTER DATABASE ... OWNER TO` 一律回 `must be owner of database`。

這是刻意的：database 的 owner 即使不具 `CREATEDB` 也能 `DROP DATABASE`，把 ownership 交出去等於交出軟刪除與還原的承諾，而且平台不會知道資料庫已經消失。

因此遇到需要改名或刪除 database 的請求，**走 MCP**，不要嘗試用連線去做——那會失敗，而失敗訊息不會指向正確的做法。

## 憑證安全

- 優先使用回應中的結構化 `host`、`port`、`database`、`username` 與 `password` 欄位。
- 將密碼放入 secret manager 或受限環境變數；不要提交 Git。
- 不要把完整 DSN 或密碼寫入 log、錯誤訊息、issue、PR 或聊天範例。
- `get_connection_info` 不回傳密碼；這是預期行為，不代表 server 故障。
- **本輪 server 未啟用 TLS，傳輸即為明文。** `sslmode=prefer` 不保證加密：它會嘗試協商，協商不到就走明文且不回報。這是團隊明確接受的剩餘風險，前提是系統僅開放於受信任的內網。不要在此存放敏感資料，也不要把這個 database 開放到內網以外。
- **不要移除或修改 DSN 裡的 `sslmode` 參數。** 不依賴 driver 預設值；預設會隨
  driver 與版本改變。AI-DB 明確交付 `sslmode=prefer`，目標 driver 必須以專案鎖定
  的版本驗證。此 repo 沒有 Go module，不能用未限定版本的 `lib/pq` 行為當保證。

## Driver 注意事項

psql、psycopg、asyncpg、Go `lib/pq` 與 `pgx` 可使用 AI-DB 交付的 PostgreSQL URI。

JDBC 不要直接在 PostgreSQL URI 前加 `jdbc:`，也不要把 `user:password@` 放進 JDBC authority；pgJDBC 可能把它當作 hostname 並將密碼寫入例外與 log。使用結構化欄位組合：

```java
String url = "jdbc:postgresql://<host>:<port>/<database>?sslmode=prefer";
Properties props = new Properties();
props.setProperty("user", "<username>");
props.setProperty("password", "<password>");
Connection connection = DriverManager.getConnection(url, props);
```

## 排錯順序

由外而內，每一步都寫出**呼叫端做得到的動作**。使用者沒有管理連線，查不到
`pg_roles`、`datacl` 或 `pg_hba.conf`——把他導向那些等於把他導進死路。

| 症狀 | 先做什麼 |
|---|---|
| MCP lifecycle 操作失敗 | 確認 MCP 認證。`找不到指定的資料庫` 同時涵蓋「不存在」與「不是你的」，兩者訊息刻意相同 |
| MCP 正常，但**每一個** driver 都連不上 | 用 `get_connection_info` 比對手上的 host／port。仍不符就是部署端的設定，回報管理者 |
| 只有某一個 driver 連不上 | 該 driver 的格式問題，先看下方「Driver 注意事項」。JDBC 是唯一需要轉換的 |
| 明確回報密碼錯誤，且確認手上的 secret 已遺失、過期或不是目前版本 | 說明中斷影響並取得確認後，才呼叫 `rotate_database_credentials` |
| role 不允許登入／`rolcanlogin=false` | 呼叫 `restore_database` 修復 role gate；若仍失敗，交由部署管理者檢查，**不要輪替** |
| database 不存在或沒有 `CONNECT` | 先用 `get_connection_info` 核對 database，再呼叫 `restore_database`；**不要輪替** |
| `no pg_hba.conf entry`／HBA 規則拒絕 | 部署設定問題，交由管理者檢查 `pg_hba.conf` 與來源網段；**不要輪替** |
| connection refused／timeout／host 錯誤 | 核對 host、port、port mapping 與防火牆；**不要輪替** |
| MCP 顯示 `ready` 但連不上 | 呼叫 `restore_database`。它會補齊所有缺少的連線條件，**不要帶 `name`** |
| `permission denied for schema public` | 把 `public.` 拿掉。未限定即落在 `app`，`search_path` 已經設好 |
| `\l` 列得出別人的 database，連不進去 | 那是正常的。名稱全域可見（PostgreSQL 擋不掉），但連線權逐一授予，你只連得進自己的那一個 |

> 上面每一列都是**呼叫端自己走得完**的。若一路走到底仍未解決，需要的是
> 管理者在 server 端跑隔離掃描，那不在使用者這一側。

## 常見誤解

| 誤解 | 實際 |
|---|---|
| 再建立一次同名 database 就能拿回密碼 | 會報錯並附上既有 `db_id`，不重發憑證 |
| `get_connection_info` 應該回傳密碼 | 它永遠不回傳。平台不儲存明文，也無從得知你手上那組還能不能用 |
| 連不上或看到 authentication failure 就先輪替 | 只有確認是密碼錯誤、遺失或過期才輪替。LOGIN、CONNECT、database、host 與 HBA 問題不會被換密碼修好 |
| 刪除是「之後才生效」 | 立即生效：新連線被拒，既有連線當場終止 |
| 已刪除的 database 可以先輪替再還原 | 順序相反：先 `restore_database`，需要時再輪替 |
| `sslmode=prefer` 代表連線有加密 | 不代表。server 未啟用 TLS，實際走明文 |
| 應該建 `public` schema 的表 | 用 `app`，且 `search_path` 已設好，不必特別指定 |
| MCP 應該有 `execute_sql` 之類的 tool | 沒有，也不會有。資料操作一律走自己的 PostgreSQL 連線 |
