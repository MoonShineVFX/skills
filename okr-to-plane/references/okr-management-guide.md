# Plane 中的 OKR 管理最佳實踐

## 結構設計

### 推薦的層級結構

```
Project: 2026 平台組 OKRs
  │
  ├─ Cycle: 2026 Q1 (季度時間盒)
  │   ├─ Work Item: [O] Objective 1
  │   │   ├─ Work Item: [KR] Key Result 1.1
  │   │   ├─ Work Item: [KR] Key Result 1.2
  │   │   └─ Work Item: [KR] Key Result 1.3
  │   └─ Work Item: [O] Objective 2
  │       └─ ...
  │
  └─ Cycle: 2026 Q2
      └─ ...
```

### 為什麼使用 Cycles？

1. **時間盒特性**：Cycles 有明確的開始和結束日期，完美對應季度
2. **進度追蹤**：自動計算 Cycle 內任務的完成百分比
3. **視覺化**：Plane 提供 Cycle 專屬的看板和圖表
4. **焦點管理**：當前季度 = 當前活躍的 Cycle

## 命名規範

### Objectives
- 格式：`[O] {動詞} + {目標}`
- 範例：
  - ✅ `[O] 優化平台穩定性與可觀測性`
  - ✅ `[O] 提升開發者體驗與生產力`
  - ❌ `平台穩定性` (缺少動詞，不夠明確)

### Key Results
- 格式：`[KR] {可量化的結果}`
- 範例：
  - ✅ `[KR] 系統 uptime 達到 99.9%`
  - ✅ `[KR] 部署時間減少 50%`
  - ❌ `[KR] 改善系統性能` (無法量化)

### Cycles
- 格式：`{年度} Q{季度}`
- 範例：`2026 Q1`, `2026 Q2`

## Work Item 屬性設定

### Priority（優先級）
- **Objectives**: `high` 或 `urgent`
  - 理由：OKRs 是組織的核心目標
- **Key Results**: `medium` 或 `high`
  - 根據實際重要性調整

### Type（類型）
建議創建自訂的 Work Item Types：
- `Objective`：用於 Objectives
- `Key Result`：用於 Key Results

或使用現有的：
- `Task`：適用於兩者
- `Epic`：如果 Objective 很大

### State（狀態）
建議的狀態流程：
1. `Not Started`：尚未開始
2. `In Progress`：進行中
3. `At Risk`：有風險，需要關注
4. `Completed`：已完成
5. `Deferred`：延後到下個季度

### Labels（標籤）
建議創建的標籤：
- `OKR-2026`：標示年度
- `Q1`, `Q2`, `Q3`, `Q4`：標示季度
- `Objective`, `KeyResult`：標示類型
- `Platform-Team`：標示所屬團隊

## 進度追蹤流程

### 週會檢查
每週檢查 Key Results 的進度：
1. 更新 work item 的狀態
2. 在 comments 中記錄進度更新
3. 標記有風險的項目（使用 `At Risk` 狀態）

### 月度回顧
每月檢查 Objectives 的達成狀況：
1. 計算 Key Results 的完成率
2. 更新 Objective 的描述（加入當前進度）
3. 調整策略或資源分配

### 季度總結
季度結束時：
1. 將所有完成的 KRs 標記為 `Completed`
2. 未完成的項目標記原因：
   - `Deferred`：延後
   - 關閉並註明原因（如：優先級變更）
3. 在 Cycle 中生成總結報告

## 使用 Plane 功能

### Cycle View
- 查看當前季度所有 Objectives 和 KRs
- 追蹤完成百分比
- 識別阻礙和風險

### Gantt Chart
- 視覺化 Objectives 的時間線
- 看出依賴關係和資源衝突

### Dashboard
創建自訂 Dashboard：
- OKR 完成率儀表板
- 各季度完成狀況對比
- 團隊成員工作負載

## 常見場景處理

### 場景 1：季度中新增 Objective
1. 創建新的 work item
2. 關聯到當前 Cycle
3. 在描述中說明為何新增

### 場景 2：Key Result 需要調整
1. 不要刪除原有的 KR
2. 在 comments 中記錄調整原因
3. 更新 KR 的描述和目標值

### 場景 3：Objective 跨季度
1. 在第一個季度創建 Objective
2. 在後續季度創建相關的 work items 並連結
3. 使用 Relations 功能建立關聯

### 場景 4：追蹤依賴關係
使用 Work Item Relations：
- `blocking`：這個 Objective 阻礙另一個
- `blocked_by`：被另一個阻礙
- `relates_to`：相關的 Objectives

## 報告和可視化

### 建議的定期報告
1. **週報**：更新 Key Results 進度
2. **月報**：Objectives 達成狀況
3. **季報**：Cycle 總結和下季規劃

### 匯出選項
- PDF 報告：用於高層匯報
- CSV 匯出：用於數據分析
- Gantt Chart 截圖：用於時間線展示

## 與其他系統整合

### 與 Google Drive 整合
- 在 work item 中附加相關文件連結
- 季度總結報告存放在 Drive

### 與 Slack 整合
- 設定 Plane 通知到 Slack
- 重要 milestone 達成時自動通知

### 與 GitHub 整合
- Key Results 對應的開發任務關聯到 GitHub issues
- 在 commit 中引用 Plane work item ID

## 常見錯誤避免

### ❌ 不要
- 設定無法量化的 Key Results
- 在一個 Cycle 中塞入太多 Objectives（建議 3-5 個）
- 季度結束後不做回顧
- 忽略 `At Risk` 狀態的項目

### ✅ 要
- 每個 Key Result 都要可量化
- 定期更新進度（至少每週一次）
- 使用 comments 記錄討論和決策
- 慶祝達成的 milestones

## 範例 Dashboard 設定

```
Dashboard: 2026 平台組 OKR 儀表板

小工具 1: Cycle Progress
- 顯示當前季度完成百分比
- 按 Objective 分組

小工具 2: Burndown Chart
- 顯示 Key Results 隨時間的完成趨勢

小工具 3: At Risk Items
- 列出所有狀態為 "At Risk" 的項目

小工具 4: Completed This Month
- 顯示本月完成的 Key Results

小工具 5: Next Quarter Preview
- 預覽下個季度規劃的 Objectives
```

## 總結

有效的 OKR 管理需要：
1. **清晰的結構**：Project → Cycles → Objectives → Key Results
2. **定期更新**：週會、月會、季度回顧
3. **透明溝通**：使用 comments 和 links 記錄上下文
4. **持續改進**：根據實際執行調整流程

記住：OKR 是用來對齊目標和追蹤進度的工具，不是用來懲罰的 KPI。保持彈性，專注在持續改進。
