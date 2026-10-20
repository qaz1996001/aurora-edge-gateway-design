# ADR-0006：Golden-Master 測試用於行為對等

**狀態**：已接受  
**決策日期**：2026-Q3  
**利益相關者**：工程團隊、QA 團隊、架構

---

## 背景

從頭重寫系統面臨微妙行為變更風險：感測器計算相差 0.01%、MQTT 訊息時間偏移 5ms、REST 回應格式稍微變動。單元與整合測試無法捕捉這些 — 它們針對預期行為測試，非實際行為。

我們需要**驗證 Rust 系統在真實流量上產生與 Node.js 系統相同輸出**，不只在合成測試案例上。

---

## 決策

實施 **golden-master 測試**（特徵化測試）：
1. 從 Node.js 系統捕獲 3+ 週生產流量
2. 將所有輸出（REST 回應、MQTT 發佈、DB 狀態）記錄為「黃金」基線
3. 針對 Rust 系統重放相同流量
4. 驗證所有輸出完全符合
5. 使用 golden-master 測試結果作為推進 Stage 1 的強制閘門

---

## 理由

### 為何 Golden-Master vs. 單元/整合測試？

| 測試類型 | 測試內容 | 涵蓋 | 真實世界差距 |
|---|---|---|---|
| 單元測試 | 隔離環境中個別函式 | 窄 | 無法捕捉整合漏洞；無真實資料模式測試 |
| 整合測試 | 兩個模組一起 | 中 | 仍合成；無真實流量模式 |
| Golden-Master 測試 | 整個系統在真實生產流量上 | 寬 | 捕捉實際使用案例；行為邊緣案例 |

**範例**：感測器讀數計算
- **單元測試**：`assert_eq!(aggregate([1.0, 2.0, 3.0]), 2.0)`
- **Golden-master**：重放 100 萬個實際感測器讀數；驗證每個單一輸出符合 Node.js 基線

---

### 為何必須？

1. **浮點精度**：不同語言/庫計算 `sqrt(2)` 稍微不同；在鏈中累積
2. **邊緣案例**：真實資料有異常值（NaN、-0、非常大的數字）；單元測試中不太可能
3. **時間語義**：訊息排序、時間戳精度、並行行為在 Node.js 事件迴圈與 Rust 非同步間不同
4. **狀態機漏洞**：系統行為取決於歷史狀態；真實流量以單元測試不會的方式運用狀態

---

## 實施

### 階段 1：流量捕獲（第 0–2 週）

檢測 Node.js 二進制檔以捕獲：
```javascript
// 新增記錄掛鉤至每個主要函式
app.get('/sensors', (req, res) => {
  LOGGER.info('REST_CALL', {
    method: 'GET',
    path: '/sensors',
    timestamp: Date.now(),
    params: req.query
  });
  // ... 現有程式碼 ...
  LOGGER.info('REST_RESPONSE', {
    status: res.statusCode,
    body: res.body,
    timestamp: Date.now()
  });
});

// ... 類似記錄 MQTT、DB 操作
```

**輸出**：`golden-dataset-week1.json`、`golden-dataset-week2.json`、`golden-dataset-week3.json`（追加日誌）

### 階段 2：Golden 資料集編譯（第 2–3 週）

合併 3 週日誌至單一 `golden-dataset.json`；目標語料規模（擷取計畫，尚未收集）約 1.2 百萬 MQTT 訊息、50,000 REST 呼叫、10,000 DB 交易

### 階段 3：Golden-Master 測試套件（Phase A 第 4 週）

針對 Rust 系統重放捕獲流量；驗證所有輸出完全符合。見 04-testing-strategy.md 用於虛擬程式碼。

**執行**：`cargo test golden_master_test -- --ignored --nocapture`

### 階段 4：故障解決（Phase A 第 4–8 週）

若 golden-master 測試失敗：

1. **識別不符**（例如，感測器讀數偏差 0.1%）
2. **隔離除錯**（為特定計算寫入專注單元測試）
3. **修正程式碼**；重新測試
4. **驗證無衰退**（所有先前測試仍通過）

---

## 成功標準

- ✓ 100% REST API 回應符合黃金資料集
- ✓ 100% MQTT 訊息對等
- ✓ 100% DB 狀態符合
- ✓ 浮點公差 ± 0.1%（合理）
- ✓ 時間戳公差 ± 5ms
- ✓ 零隱藏行為變更

---

## 監控與可觀察性

每次運行時生成詳細不符報告：
```json
{
  "passed": 1234567,
  "failed": 0,
  "summary": "✅ Golden-master test PASSED"
}
```

若有故障：
```json
{
  "passed": 1234000,
  "failed": 567,
  "failures": [
    {
      "id": 42,
      "test_type": "rest_call",
      "expected": {"status": 200, "body": {...}},
      "actual": {"status": 200, "body": {...}},  // 注意細微差異
      "diff": "response.sensors[0].voltage: expected 230.5, got 230.4"
    }
  ]
}
```

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：工程團隊、QA 團隊  
**實施**：Phase A（從第 1 週開始流量捕獲）
