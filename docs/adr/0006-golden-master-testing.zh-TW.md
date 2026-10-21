# ADR-0006：用於行為對等的 Golden-Master 測試

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
- **Golden-master**：重放已擷取的真實感測器讀數語料；驗證每個輸出都符合 Node.js 基線

---

### 為何必須？

1. **浮點精度**：不同語言/庫計算 `sqrt(2)` 稍微不同；在鏈中累積
2. **邊緣案例**：真實資料有異常值（NaN、-0、非常大的數字）；單元測試中不太可能
3. **時間語義**：訊息排序、時間戳精度、並行行為在 Node.js 事件迴圈與 Rust 非同步間不同
4. **狀態機漏洞**：系統行為取決於歷史狀態；真實流量以單元測試不會的方式運用狀態

---

## 實施

### Phase 1：流量捕獲（第 0–2 週）

檢測 Node.js 二進制檔以捕獲：
```javascript
// 在每個主要函式上新增記錄掛鉤
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

mqtt.on('message', (topic, payload) => {
  LOGGER.info('MQTT_RECEIVED', {
    topic,
    payload: payload.toString(),
    timestamp: Date.now()
  });
  // ... 現有程式碼 ...
});

db.run(sql, (err) => {
  LOGGER.info('DB_WRITE', {
    query: sql,
    affected: this.changes,
    timestamp: Date.now()
  });
  // ... 現有程式碼 ...
});
```

**輸出**：`golden-dataset-week1.json`、`golden-dataset-week2.json`、`golden-dataset-week3.json`（僅追加日誌）

### Phase 2：Golden Dataset 編譯（第 2–3 週）

合併 3 週日誌至單一 `golden-dataset.json`：
```json
{
  "rest_calls": [
    {
      "id": 1,
      "method": "GET",
      "path": "/sensors",
      "query": {"id": "sensor_1"},
      "expected_status": 200,
      "expected_body": {"id": "sensor_1", "value": 230.5, ...}
    },
    { ... }
  ],
  "mqtt_messages": [
    {
      "id": 1000,
      "topic": "gateway/sensor/reading",
      "payload": {"sensor_id": "s1", "voltage": 230.5},
      "timestamp": 1696416000000
    },
    { ... }
  ],
  "db_writes": [
    {
      "id": 5000,
      "query": "INSERT INTO readings (sensor_id, value) VALUES (?, ?)",
      "params": ["s1", 230.5]
    },
    { ... }
  ]
}
```

**目標語料**（擷取計畫，尚未收集）：
- 約 120 萬則 MQTT 訊息
- 約 50,000 次 REST API 呼叫
- 約 10,000 筆資料庫交易
- 涵蓋：正常運作（70%）、尖峰負載（15%）、錯誤案例（10%）、邊緣案例（5%）

### Phase 3：Golden-Master 測試套件（Phase A 第 4 週）

針對 Rust 系統重放捕獲流量：

```rust
#[test]
fn golden_master_test() -> Result<()> {
    // 載入 golden dataset
    let golden: GoldenDataset = serde_json::from_str(&read_file("golden-dataset.json"))?;
    
    // 啟動受測的 Rust 系統
    let mut system = AuroraGateway::start().await?;
    
    // 重放 REST 呼叫
    for call in &golden.rest_calls {
        let response = system
            .http_client
            .get(&format!("http://localhost:3000{}", call.path))
            .query(&call.query)
            .send()
            .await?;
        
        assert_eq!(response.status(), call.expected_status, 
                   "Status mismatch for call {}", call.id);
        
        let body: serde_json::Value = response.json().await?;
        assert_json_eq!(body, call.expected_body,
                        "Response body mismatch for call {}", call.id);
    }
    
    // 重放 MQTT 訊息
    for msg in &golden.mqtt_messages {
        system
            .mqtt_client
            .publish(msg.topic.clone(), msg.payload.clone())
            .await?;
        
        // 等待處理完成，讓系統有時間回應
        tokio::time::sleep(Duration::from_millis(50)).await;
        
        // 驗證產生的任何輸出都符合 golden 資料
        let outputs = system.capture_outputs();
        assert_eq!(outputs, msg.expected_outputs,
                   "Output mismatch for MQTT message {}", msg.id);
    }
    
    // 驗證最終資料庫狀態符合 golden 資料
    system.verify_db_state(&golden.expected_db_state)?;
    
    Ok(())
}
```

**執行**：`cargo test golden_master_test -- --ignored --nocapture`

**輸出**：通過/失敗結果，並對任何不符之處附帶詳細差異

### Phase 4：故障解決（Phase A 第 4–8 週）

若 golden-master 測試失敗：

1. **識別不符**（例如，感測器讀數偏差 0.1%）
2. **隔離除錯**（為特定計算寫入專注單元測試）
3. **修正 Rust 程式碼**（使其精確符合 Node.js 行為）
4. **重新執行 golden-master**（確認修正不會破壞其他輸出）
5. **反覆迭代**直到達成 100% 對等

---

## 測試分桶與漸進式驗證

不必等待 3 週流量全部捕獲完成才開始；應漸進式驗證：

### Phase A 第 1 週
- 使用第 1 週 golden 資料（資料集的 1/3）
- 執行 golden-master 測試
- 修正容易發現的漏洞

### Phase A 第 2 週
- 加入第 2 週 golden 資料（資料集的 2/3）
- 重新執行 golden-master 測試
- 修正剩餘漏洞

### Phase A 第 3 週
- 加入第 3 週 golden 資料（完整資料集）
- 執行完整 golden-master 測試
- 達成 100% 對等

---

## 可接受的公差範圍

並非所有輸出都能完全一致（系統間存在固有差異）：

| 指標 | 公差 | 理由 |
|---|---|---|
| HTTP 狀態碼 | 0%（必須完全符合） | 無模糊空間 |
| REST 回應 JSON | 0%（必須完全符合） | 契約要求嚴格 |
| MQTT 訊息順序 | 0%（必須完全符合） | 順序影響狀態 |
| MQTT 訊息內容 | ±0.1%（浮點精度） | 跨語言中不可避免 |
| 時間戳 | ±5 ms | 可接受時鐘偏移 |
| 回應延遲 | 不測試（隨系統負載變化） | 與正確性無關 |
| 資料庫狀態 | 0%（必須完全符合） | 需要 ACID 保證 |

---

## CI/CD 整合

### Pre-Merge（程式碼審查）
- 針對第 1 週 golden 資料執行（快速回饋）
- 必須通過才能核准 PR

### Post-Merge（主分支）
- 針對第 1 週 + 第 2 週 + 第 3 週 golden 資料執行（完整套件；30 分鐘）
- 失敗 = 立即回退該 PR

### Nightly（延伸）
- 執行 golden-master + 延伸負載測試
- 找出效能衰退
- 若延遲百分位數惡化則發出告警

---

## 維護與演進

### Golden Dataset 輪替
- **每季**：重新捕獲新的 3 週 golden dataset（客戶行為會演變）
- **歸檔**：保留歷史資料集供回歸測試使用
- **去識別化**：儲存前移除敏感客戶資料

### Golden-Master 涵蓋缺口
若客戶回報 golden dataset 中未涵蓋的新邊緣案例：
1. 擷取該特定場景
2. 新增至 golden-master 測試
3. 確保 Rust 系統正確處理

---

## 針對真實資料的驗證

### Phase A 後驗證
在**測試機隊上讓 Rust 系統與 Node.js 並行運行**，重放已擷取的流量，而非分流真實流量：
- 將 Rust 二進制檔部署至測試機隊（非生產環境）
- 讓兩套堆疊處理相同的擷取語料並比對輸出
- 若輸出分歧：發出告警；調查；修正

注意：這是 shadow comparison（陰影比對），並非百分比灰度。正式切換是
二元切換，搭配人工回滾 SOP——見
[ADR-0005](0005-staged-rollout-and-rollback.zh-TW.md)。

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

## 相關決策
- **[ADR-0004](0004-ecosystem-crate-mapping.zh-TW.md)**：影響數值行為的技術選擇
- **測試策略**：完整測試層級請見 `../04-testing-strategy.zh-TW.md`

---

## 成功標準

- ✓ 整個捕獲語料達成 100% 輸出對等
- ✓ 延遲在 Node.js 基線的 ±5ms 內
- ✓ Stage 1 以上未發現任何無聲正確性衰退
- ✓ 團隊有信心：「我們知道 Rust 系統的行為與 Node.js 完全一致」

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：工程團隊、QA 團隊  
**實施**：流量捕獲自第 0 週開始；golden-master 測試於 Phase A 第 4–8 週執行
