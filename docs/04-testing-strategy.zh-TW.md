# 測試策略：Golden-Master 測試與驗證

Golden-Master（特徵化）測試確保新系統在真實流量上的行為與舊系統完全等價。

參見 **[ADR-0006](adr/0006-golden-master-testing.md)** 了解決策理由。

---

## Golden-Master 測試方法論

### 核心原則
Golden-master 測試捕獲**原始系統**（Node.js）在實際輸入上的輸出，然後驗證**新系統**（Rust）在相同輸入上產生相同輸出。

**關鍵洞察**：我們測試的不是新系統是否「正確」（絕對意義），而是它是否**保留行為**。

---

## 測試分類與涵蓋範圍

### 分類 1：單元測試（Phase A & B）

**目的**：驗證個別函式在隔離環境中的正確性。

**範例**：
- `aeg-config` 測試：解析有效/無效 TOML；驗證預設值
- `aeg-db` 測試：SQL 插入/更新/刪除；驗證交易語義
- `aeg-sensors` 測試：聚合讀數；驗證濾波；檢查數值精確度
- 記憶體分析測試：驗證 drop chain 無洩漏

**最低覆蓋率**：70% 程式碼涵蓋率

**執行**：`cargo test`

### 分類 2：整合測試（Module Pairs，Phase A）

**目的**：驗證模組一起正常運作（真實 I/O、共享狀態）。

**涵蓋範圍**：所有面向使用者的程式碼路徑

**執行**：`cargo test --test integration_tests`

### 分類 3：Golden-Master 測試（完整系統，Phase A & B）

回放捕獲的流量，驗證輸出對等。

### 分類 4：性能與壓力測試（Phase B）

#### 4a. 負載測試

```bash
# 模擬活躍負載：500-1000 MQTT msg/sec + 100 REST req/sec for 3 hours
load-generator \
  --mqtt-rate 500 \
  --http-rate 100 \
  --duration 10800s \
  --target localhost:3000
```

**目標指標** (Phase B)：
- CPU 平均：50–65%（無持續 > 80%）
- CPU 高峰：< 100%（瞬間尖峰可接受）
- 記憶體基線：< 1400 MB（空閒）
- 記憶體活躍：< 1500 MB（持續）
- Load average：3.0–3.5 on 4-core（0.75–0.875/core）
- 訊息丟失率：0%
- 記憶體增長：< 0.01% per day

#### 4b. 記憶體 Soak 測試（30+ 天）

**成功標準**：
- 記憶體基線：穩定於 ~1400 MB（空閒）
- 記憶體活躍：穩定於 ~1500 MB（1h 暖機後）
- 記憶體增長：< 0.01% per day
- CPU：穩定 50–65%（無衰退或漂移）
- 零無計畫崩潰（只有刻意 kill）
- 零服務重啟（30 天內）
- 無 GC 暫停

#### 4c. 穩定性與確定性驗證（24h 監控）

**成功標準**：
- CPU 變異 < 10%（5 分鐘窗口內）
- 記憶體穩定（無增長 > 0.01%/hour）
- Load average 平滑（無 > 0.5/core 瞬間尖峰）
- 無垃圾回收暫停引起的延遲

---

## CI/CD 整合

測試執行管道：
1. Tier 1（5 min）：編譯 + 單元測試 + lint
2. Tier 2（15 min）：整合測試 + 基礎負載測試
3. Tier 3（30 min）：Golden-Master 測試（1 週樣本）
4. Tier 4（60 min）：性能測試（1 小時負載 + 6 小時 soak）
5. Tier 5（nightly，2 小時）：完整 golden-master + 擴展測試

---

## Golden-Master 測試工具（虛擬程式碼）

```python
class GoldenMasterHarness:
    def replay_mqtt_event(self, event):
        """注入 MQTT 訊息；驗證輸出"""
        published = self.capture_mqtt_publishes()
        expected = event.get('expected_publishes', [])
        assert published == expected
    
    def replay_rest_api_call(self, call):
        """注入 REST API 呼叫；驗證回應"""
        response = requests.request(...)
        expected = call['expected_response']
        assert response.status == expected['status']
        assert response.json() == expected['body']
    
    def generate_report(self):
        """列印摘要與失敗細節"""
        if self.results['failed']:
            return 1  # 失敗
        return 0  # 成功
```

---

**狀態**：✅ 測試策略已批准  
**下次審查**：Phase A 完成後（golden-master 測試結果）
