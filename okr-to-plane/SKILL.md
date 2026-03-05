---
name: okr-to-plane
description: Import and structure OKR (Objectives and Key Results) from Markdown table format into Plane project management system. Use when user mentions "import OKRs to Plane", "create OKR structure", "setup quarterly OKRs", "建立 OKR 到 Plane", "匯入 OKR", or provides OKR markdown files to track in Plane. Supports table format with interactive splitting for cross-quarter objectives. Specifically designed for quarterly OKR tracking using Cycles.
metadata:
  author: Acme Platform Team
  version: 2.0.0
  mcp-server: plane
  category: project-management
  tags: [okr, project-management, quarterly-planning, table-format]
---

# OKR to Plane Importer (Table Format)

將 Markdown 表格格式的 OKR 清單匯入到 Plane 專案管理系統，使用 Cycles 進行季度追蹤。支援跨季度 Objective 的互動式拆分。

## 工作流程

### Phase 1: 準備與驗證

1. **確認專案**
   - 請使用者提供已存在的 Plane 專案 ID（或專案名稱）
   - 使用 `plane:list_projects` 查詢並確認專案存在
   - **注意：不要透過 MCP 創建新專案**，因為 MCP 創建的專案會缺少預設 State 設定，導致後續建立 Work Item 時失敗
   - 如果使用者尚未建立專案，請引導他們先到 Plane 介面手動建立專案，再回來提供專案 ID

2. **智能解析 Markdown OKR 文件**

   **支援的格式：**
   
   **格式 A - 表格格式（主要支援）：**
   ```markdown
   ### Objective 1：建立能力 (Q1-Q2)
   
   | KR | 關鍵成果 | 衡量指標 |
   | --- | --- | --- |
   | KR1 | 完成培訓 | 產出 2 個模型 |
   | KR2 | 建立工作流 | 產出 SOP 文件 |
   ```
   
   **格式 B - 表格格式（簡化）：**
   ```markdown
   ### Objective 1：建立能力 (1-3個月)
   
   | KR | 關鍵成果 Key Result |
   | --- | --- |
   | KR1 | 完成培訓並產出 2 個模型 |
   ```
   
   **格式 C - 組織級 OKR（含負責人欄位）：**
   ```markdown
   | KR | 關鍵成果 | 時程 | 負責人/組別 |
   | --- | --- | --- | --- |
   | KR1 | 完成系統規劃 | Q1 | 小美 |
   ```

   **解析邏輯：**
   
   a. **識別 Objective**：
      - 尋找 `###` 標題
      - 提取 Objective 標題文字
      - 從括號中提取時程：`(Q1-Q2)`, `(1-3個月)`, `(Q1-Q4)` 等
   
   b. **時程映射規則**：
      - `Q1`, `(1-3個月)` → Q1 (2026-01-01 ~ 2026-03-31)
      - `Q2`, `(4-6個月)` → Q2 (2026-04-01 ~ 2026-06-30)
      - `Q3`, `(7-9個月)` → Q3 (2026-07-01 ~ 2026-09-30)
      - `Q4`, `(10-12個月)` → Q4 (2026-10-01 ~ 2026-12-31)
      - `Q1-Q2` → 跨季度，需要互動拆分
      - `Q1-Q4`, `(1-12個月)` → 全年度，需要互動拆分
   
   c. **解析 Markdown 表格**：
      - 識別表格列：`| KR1 | ... |`
      - 提取 KR 編號、描述、衡量指標
      - 如果有負責人欄位，提取負責人資訊
   
   d. **負責人處理**：
      - 如果表格中有負責人欄位，使用該欄位
      - 否則，詢問使用者指定預設負責人
      - 或從使用者輸入的指令中推斷（例如："匯入 505 的 OKR，負責人是王大華"）

### Phase 1.5: 跨季度 Objective 互動式拆分 ⭐

**當遇到跨季度 Objective 時（如 Q1-Q2, Q1-Q4, 1-12個月），啟動互動式拆分流程：**

#### 步驟 1: 識別跨季度 Objective

檢測以下模式：
- `(Q1-Q2)` → 跨 2 個季度
- `(Q1-Q4)`, `(1-12個月)` → 全年度
- `(Q2-Q3)` → 跨 2 個季度

#### 步驟 2: 展示 Objective 並詢問拆分方式

**展示格式：**
```
🎯 發現跨季度 Objective：

Objective: AI 輔助開發能力（橫向支援 O1–O4，1–12 個月）
時程: Q1-Q4（全年度）
Key Results:
  - KR1: 學習使用 Skills 自動化工作流
  - KR2: 學習在本地自架 MCP 並導入日常開發流程
  - KR3: 學習 Claude Code Agent 的全自動開發流程

你希望如何拆分到各季度？

【選項 1】每季度重複相同的 Objective 和所有 KRs
  → 在 Q1, Q2, Q3, Q4 都創建完全相同的 Objective 和所有 KRs
  → 適用於：持續性學習、橫向支援、全年持續追蹤的目標
  
【選項 2】將 KRs 分配到不同季度
  → 我會逐個詢問每個 KR 應該在哪個季度完成
  → 適用於：有階段性里程碑、KRs 之間有先後順序的目標
  
【選項 3】在第一個季度創建，標註完整時程
  → 只在 Q1 創建 Objective，在描述中說明全年度追蹤
  → 適用於：不想拆分，在單一地方追蹤的目標
  
【選項 4】自訂拆分（由你描述如何拆分）

請選擇：1, 2, 3, 或 4
```

#### 步驟 3a: 選項 1 執行邏輯

**使用者選擇選項 1：**

在每個季度創建相同的 Objective：
```
Q1 Cycle:
  - [O] AI 輔助開發能力 (Q1/Q4)
    - [KR] 學習使用 Skills 自動化工作流
    - [KR] 學習在本地自架 MCP
    - [KR] 學習 Claude Code Agent 流程

Q2 Cycle:
  - [O] AI 輔助開發能力 (Q2/Q4)
    - [KR] 學習使用 Skills 自動化工作流
    - [KR] 學習在本地自架 MCP
    - [KR] 學習 Claude Code Agent 流程
    
（Q3, Q4 同理）
```

**Work Item 命名規範：**
- Objective: `[O] {標題} (Q{當前季度}/{總季度數})`
- 例如: `[O] AI 輔助開發能力 (Q1/Q4)`, `[O] AI 輔助開發能力 (Q2/Q4)`

#### 步驟 3b: 選項 2 執行邏輯（互動式分配）

**逐個詢問 KR 的季度分配：**

```
📋 KR1: 學習使用 Skills 自動化工作流

建議季度：Q1
  理由：基礎能力，適合早期建立
  
接受建議嗎？
  - 直接按 Enter 或輸入 "yes" 接受
  - 輸入季度編號調整 (例如: Q2, Q3, Q4)
  
你的選擇：
```

**使用者回應處理：**
- `[Enter]` 或 `yes` → 接受建議（Q1）
- `Q2` → 分配到 Q2
- `Q1,Q2` → 分配到 Q1 和 Q2（KR 在兩個季度都出現）

**持續詢問直到所有 KR 分配完成。**

**展示最終分配：**
```
✅ 分配完成：

Q1 Cycle:
  - [O] AI 輔助開發能力
    - [KR] 學習使用 Skills 自動化工作流
    - [KR] 學習在本地自架 MCP

Q3 Cycle:
  - [O] AI 輔助開發能力
    - [KR] 學習 Claude Code Agent 流程

確認要這樣創建嗎？(yes/no)
```

#### 步驟 3c: 選項 3 執行邏輯

只在第一個季度創建，描述中標註完整時程：

```
Q1 Cycle:
  - [O] AI 輔助開發能力 (全年度追蹤: Q1-Q4)
    - Description: 此 Objective 為全年度持續追蹤項目
    - [KR] 學習使用 Skills 自動化工作流
    - [KR] 學習在本地自架 MCP
    - [KR] 學習 Claude Code Agent 流程
```

#### 步驟 3d: 選項 4 執行邏輯

詢問使用者自訂拆分方式：
```
請描述你希望如何拆分這個 Objective：
例如：
- "KR1和KR2在Q1，KR3在Q2"
- "每個季度追蹤所有KRs，但只在Q4標記完成"
- "Q1-Q2一組，Q3-Q4一組"
```

根據使用者描述解析並執行。

#### 步驟 4: 記錄拆分決策

在 work item 的 description 或 comments 中記錄拆分邏輯：
```html
<p><strong>原始 Objective:</strong> AI 輔助開發能力（1-12個月）</p>
<p><strong>拆分方式:</strong> 選項 2 - KRs 分配到不同季度</p>
<p><strong>拆分邏輯:</strong></p>
<ul>
  <li>KR1, KR2 → Q1</li>
  <li>KR3 → Q3</li>
</ul>
```

這樣團隊成員可以了解為什麼某個 Objective 只有部分 KRs。

### Phase 2: 建立 Cycles（季度）

為每個包含 OKR 的季度創建 Cycle：

```
plane:create_cycle
project_id: [從 Phase 1 獲得]
name: "2026 Q1"
start_date: "2026-01-01"
end_date: "2026-03-31"
owned_by: [組長的 user_id]
```

**重要：**
- 在創建 Cycle 前，先用 `plane:list_cycles` 檢查是否已存在
- 如果已存在同名 Cycle，詢問使用者是否：
  - 使用現有 Cycle
  - 創建新的（需要不同名稱）
  - 更新現有 Cycle

### Phase 3: 建立 Objectives (Work Items)

**基於 Phase 1.5 的拆分決策，創建 work items：**

對每個季度的每個 Objective 創建 work item：

```
plane:create_work_item
project_id: [project_id]
name: "[O] {Objective 標題}"
description_html: "<p>{Objective 詳細描述}</p>
                   <p><strong>時程:</strong> {原始時程標註}</p>
                   <p><strong>衡量標準:</strong></p>
                   <ul>{KRs 總覽}</ul>"
assignees: [負責人的 user_id]
priority: "high"  # OKRs 通常是高優先級
```

**描述內容建議：**
- 包含原始的時程資訊（例如 Q1-Q2, 1-3個月）
- 如果是拆分後的，說明拆分邏輯
- 列出本季度的 KRs 總覽

創建後立即關聯到對應的 Cycle：
```
plane:add_work_items_to_cycle
project_id: [project_id]
cycle_id: [對應季度的 cycle_id]
issue_ids: [剛創建的 work_item_id]
```

**命名慣例：**
- 使用 `[O]` 前綴標示這是 Objective
- 如果是跨季度拆分，加上季度標記：
  - `[O] 建立 4DGS 能力 (Q1/Q2)` - 表示這是 Q1-Q2 計畫的 Q1 部分
  - `[O] AI 輔助開發 (Q2/Q4)` - 表示這是全年度計畫的 Q2 部分

### Phase 4: 新增 Key Results

**從表格解析的 KRs 創建為子 work items（推薦方式）：**

對每個 Objective 下的 Key Results，創建獨立的 work items：

```
plane:create_work_item
name: "[KR] {KR 描述}"
description_html: "<p><strong>衡量指標:</strong> {衡量指標}</p>"
parent: [Objective 的 work_item_id]
assignees: [相同的負責人]
priority: "medium"
```

**從表格提取資訊：**

**表格格式 A：**
```
| KR  | 關鍵成果              | 衡量指標          |
| --- | ------------------- | -------------- |
| KR1 | 完成 Blender 基礎建模培訓 | 產出 2 個 Demo 模型 |
```
→ 創建：
- name: `[KR] 完成 Blender 基礎建模培訓`
- description: `衡量指標: 產出 2 個 Demo 模型`

**表格格式 B（簡化版）：**
```
| KR  | 關鍵成果 Key Result                    |
| --- | ----------------------------------- |
| KR1 | 完成培訓並產出 2 個可用於 Developer Portal 的模型 |
```
→ 創建：
- name: `[KR] 完成培訓並產出 2 個可用於 Developer Portal 的模型`
- description: 從完整描述中提取

**表格格式 C（組織級，含時程）：**
```
| KR  | 關鍵成果        | 時程 | 負責人 |
| --- | ------------- | ---- | ----- |
| KR1 | 完成公司系統規劃書 | Q1   | 小美  |
```
→ 創建：
- name: `[KR] 完成公司系統規劃書`
- description: `目標完成時間: Q1`
- assignees: [小美的 user_id]

**處理加粗文字：**
如果 KR 描述中有 `**粗體**` 標記（如 `**50%**`, `**權限驗證 (Auth)**`），保留在描述中以強調關鍵指標。

**關聯到 Cycle：**
KR work items 會自動繼承父 Objective 的 Cycle 關聯，無需重複關聯。

**命名規範：**
- 使用 `[KR]` 前綴標示這是 Key Result
- 保持簡潔但完整：`[KR] 完成 Blender 基礎建模培訓`
- 避免過長：如果超過 80 字元，將詳細內容放在 description

### Phase 5: 驗證與報告

1. **驗證創建結果**
   - 列出所有創建的 Cycles：`plane:list_cycles`
   - 確認每個 Cycle 內的 work items：`plane:list_cycle_work_items`
   - 檢查父子關係是否正確

2. **生成詳細摘要報告**
   ```
   ✅ 專案: "2026 平台組 OKRs"
   
   ✅ Cycles 創建:
      📅 2026 Q1 (2026-01-01 ~ 2026-03-31)
         └─ 3 個 Objectives, 9 個 Key Results
      
      📅 2026 Q2 (2026-04-01 ~ 2026-06-30)
         └─ 2 個 Objectives, 6 個 Key Results
   
   ✅ 跨季度 Objective 處理:
      🔄 "AI 輔助開發能力" (Q1-Q4)
         - 拆分方式: 選項 2 (KRs 分配到不同季度)
         - Q1: KR1, KR2
         - Q3: KR3
   
   📊 總計:
      - Cycles: 2 個
      - Objectives: 5 個 (含拆分後的)
      - Key Results: 15 個
   
   🔗 Plane 連結: https://your-plane-instance/projects/{project_id}
   ```

3. **提供後續操作建議**
   - 建議設定 labels（如 `OKR-2026`, `Q1`, `Platform-Team`）
   - 建議設定 custom states（如 `Not Started`, `In Progress`, `At Risk`）
   - 提醒定期更新進度

## 支援的 Markdown 格式

此 skill 主要支援**表格格式**的 OKR：

### 格式範例 1：個人 OKR（3 欄表格）

```markdown
### Objective 1：建立 4DGS, 3D 資產製作能力 (Q1-Q2)

| KR  | 關鍵成果                   | 衡量指標                     |
| --- | ----------------------- | ------------------------ |
| KR1 | 完成 Blender 基礎建模培訓      | 產出 2 個可用於 Demo 的模型        |
| KR2 | 建立 Blender → Web 標準工作流 | 產出 SOP 文件，團隊可複製使用         |
| KR3 | 製作展示素材                 | 交付 3 組 .glb 範例資產          |

### Objective 2：支援 4DGS Viewer 前端開發 (Q2-Q3)

| KR  | 關鍵成果      | 衡量指標       |
| --- | --------- | ---------- |
| KR1 | 完成材質效果    | 實作至少 2 種顯示模式 |
| KR2 | 完成 R3F 整合 | 產出 SOP 文件   |
```

### 格式範例 2：簡化版（2 欄表格）

```markdown
### Objective 1：網站開發與部署 (1-3個月)

| KR  | 關鍵成果 Key Result                           |
| --- | ----------------------------------------- |
| KR1 | **完成專案基礎架構交付**：建立 repo，完成本機與 CF 兩種環境可啟動   |
| KR2 | **完成 D1 資料庫上線**：建立至少 3 張核心資料表，完成 migration |
```

### 格式範例 3：組織級 OKR（含負責人和時程）

```markdown
## 平台組 OKR

### Objective 1：完成公司系統數位化整合

| KR   | 關鍵成果 Key Result           | 時程  | 負責人/組別 |
| ---- | ------------------------- | --- | ------ |
| KR1  | Q1 完成公司系統規劃書與規格書交付        | Q1  | 小美     |
| KR2  | Q3 完成 ERP 3.0/EIP MVP 架構開發 | Q3  | 小美     |
| KR3  | 績效考核流程數位化                 | Q3  | 小美     |
```

### 時程標記支援

**在 Objective 標題中：**
- `(Q1)`, `(Q2)`, `(Q3)`, `(Q4)` - 單季度
- `(Q1-Q2)` - 跨兩個季度
- `(Q1-Q4)` - 全年度
- `(1-3個月)` - Q1
- `(4-6個月)` - Q2
- `(7-9個月)` - Q3
- `(10-12個月)` - Q4
- `(1-12個月)` - 全年度

**在表格的時程欄位中：**
- `Q1`, `Q2`, `Q3`, `Q4` - 直接指定季度

## 錯誤處理

### Plane 連線問題
如果 MCP 調用失敗：
1. 確認 Plane MCP 已連接：Settings > Extensions > Plane
2. 檢查 API token 是否有效
3. 驗證專案權限

### 重複創建問題
- 每次創建前都檢查是否已存在（專案、Cycle、work item）
- 使用 `plane:list_*` 工具進行檢查
- 詢問使用者如何處理重複項

### 負責人未找到
如果 Markdown 中的負責人在 Plane 中不存在：
1. 列出可用的專案成員：`plane:get_project_members`
2. 請使用者選擇或確認正確的成員
3. 或暫時不分配 assignee，稍後手動處理

## 使用範例

### 範例 1: 匯入個人 OKR（表格格式）

**使用者說：**
「請幫我把 505 的 OKR 匯入到 Plane，負責人是王小明」

*（提供包含表格的 Markdown 內容）*

**操作流程：**
1. 解析表格格式的 Markdown
2. 識別到 Objective A 是全年度 (Q1-Q4)
3. 觸發互動式拆分：
   ```
   🎯 發現跨季度 Objective：
   
   Objective A：AI 輔助開發能力（Q1-Q4）
   Key Results:
     - KR1: 學習使用 Skills 自動化工作流
     - KR2: 學習在本地自架 MCP
     - KR3: 學習 Claude Code Agent 流程
   
   你希望如何拆分？(選擇 1-4)
   ```
4. 使用者選擇選項 2（逐個分配）
5. 互動確認每個 KR 的季度
6. 創建 Cycles、Objectives、Key Results
7. 提供匯入摘要

### 範例 2: 匯入組織級 OKR（含負責人欄位）

**使用者說：**
「匯入平台組的 OKR 到 Plane」

**操作流程：**
1. 解析表格，發現有「負責人/組別」欄位
2. 自動從表格提取負責人資訊
3. 詢問是否需要查找 user_id：
   ```
   表格中的負責人是「小美」，
   需要我查找對應的 Plane user_id 嗎？
   ```
4. 使用 `plane:get_workspace_members` 找到對應的 user_id
5. 創建專案結構並分配負責人

### 範例 3: 更新現有 OKR

**使用者說：**
「Q2 有新的 Objective，幫我加到 Plane」

**操作流程：**
1. 檢查 Q2 Cycle 是否存在
2. 創建新的 Objective work item
3. 關聯到 Q2 Cycle
4. 創建對應的 Key Results

## 最佳實踐

### 命名規範
- Objectives: `[O] {標題}`
- Key Results: `[KR] {描述}` 或作為 checklist

### 優先級建議
- Objectives: `high` 或 `urgent`（核心目標）
- Key Results: `medium`（具體執行項目）

### Labels 建議
考慮創建以下 labels：
- `OKR-2026`
- `Q1`, `Q2`, `Q3`, `Q4`
- `Objective`, `KeyResult`

### 進度追蹤
- 定期更新 Key Results 的 state
- Cycle 會自動計算完成度
- 使用 Plane 的 Cycle 視圖查看季度進度

## Troubleshooting

### 問題：Cycle 日期重疊
**症狀：** 無法創建 Cycle，提示日期重疊
**解決：** 
- 檢查現有 Cycles 的日期範圍
- 調整新 Cycle 的日期
- 或使用現有 Cycle

### 問題：Work item 創建成功但未出現在 Cycle 中
**症狀：** Work item 存在但 Cycle 列表中沒有
**解決：**
- 使用 `plane:add_work_items_to_cycle` 明確關聯
- 驗證 cycle_id 和 work_item_id 正確

### 問題：無法找到組長的 user_id
**症狀：** 不知道如何獲取負責人的 user_id
**解決：**
```
1. 使用 plane:get_workspace_members 獲取所有成員
2. 根據名字或 email 找到對應的 user_id
3. 或使用 plane:get_project_members 獲取專案成員
```

## 技術細節

### 需要的 Plane MCP 工具
- `plane:list_projects` / `plane:create_project`
- `plane:list_cycles` / `plane:create_cycle`
- `plane:list_work_items` / `plane:create_work_item`
- `plane:add_work_items_to_cycle`
- `plane:get_workspace_members` / `plane:get_project_members`

### 日期格式
- 使用 ISO 8601 格式：`YYYY-MM-DD`
- 例如：`2026-01-01`

### User ID 獲取
- 從 `get_workspace_members` 或 `get_project_members` 的回應中取得
- 格式通常是 UUID：`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`
