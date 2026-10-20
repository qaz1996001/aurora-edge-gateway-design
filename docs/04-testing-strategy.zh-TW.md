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

#### 4a. 負載測試（代表性生產負載與峰值負載）

```bash
# 在目標 4 核 ARM Cortex-A53 平台上，先跑代表性生產負載，再跑峰值負載
load-generator \
  --profile representative-production \
  --duration 10800s \
  --target localhost:3000

load-generator \
  --profile peak \
  --duration 10800s \
  --target localhost:3000
```

**目標指標**（Phase B，錨定於實測的 7.8h soak 基線）：
- CPU：平均 6.3%、峰值 8.8%，soak 期間零漂移——對照 Node.js 基線同一硬體上的待機 52%／3h 活躍 64%
- 記憶體：全棧常駐 206 MB（14 個受監督服務 + Python 子行程）——對照 Node.js 基線的 1538 MB 活躍（約佔可用 2.84 GB 的 54%）
- Load average（4 核）：平均 0.50、峰值 1.81——對照 Node.js 基線的 3.30 平均
- 吞吐量：代表性生產負載與峰值負載皆能持續運行而不掉訊息。我們目前沒有針對此平台做過的吞吐量上限基準測試；`--profile` 只模擬真實流量形態，不代表某個具體 msg/sec 數字，建立明確吞吐量上限留作後續工作，本文件不作宣稱
- 訊息丟失率：0%
- 記憶體增長：測試期間無持續正斜率的 RSS 增長（斜率判定方法見下方 4b 案例研究）

#### 4b. 記憶體 Soak 測試（實測：7.8h 與 10.6h）

**結果（初始 7.8h soak，全棧，4 核 ARM Cortex-A53）**：
- 全棧常駐記憶體：206 MB（14 個受監督服務 + Python 子行程）
- CPU：平均 6.3%、峰值 8.8%，全程零漂移
- Load average：平均 0.50、峰值 1.81（4 核）
- 系統可用記憶體：6024 → 5885 MB（7.8h 內 −139 MB）
- 重啟：頭 66 分鐘內 3 次（啟動穩定期），之後 6.7 小時零重啟

**成功標準**：
- 每個服務在測試期間無持續正斜率的 RSS 增長（不是固定的「每天 %」數字——斜率判定方法見下方案例研究）
- CPU 穩定，無衰退或漂移（7.8h soak 實測平均 6.3% 持平）
- 零無計畫崩潰（只有刻意 kill）
- 啟動穩定期之後零服務重啟（7.8h 測試中啟動穩定期後 6.7 小時零重啟；10.6h 測試全程通過——見下方案例研究）
- 無 GC 暫停（Rust 無 GC）

#### 4b-i. 案例研究：soak 測試中發現並修復的真實記憶體增長問題

這是 Phase B soak 測試產出的最有價值的結果，值得完整記錄，因為它示範了 soak 測試真正的用途。

**觀察到的現象**：在初始 7.8h soak 中，訊息匯流排進程的 RSS 從 45.0 MB 增長到 95.3 MB——增幅 +112%，大致呈線性增長，速率約 6.5 MB/h，而該進程的 CPU 使用率全程恆定維持在 0.2%。線性且 CPU 持平的增長曲線，是配置器或執行環境層級問題的特徵，而不是應用邏輯驅動的洩漏。

**根本原因**：glibc 的多執行緒配置器為該進程開出多達 64 個記憶體 arena。每個 arena 都維護自己的 free list，因此即使 Rust 程式碼本身零記憶體安全錯誤，多執行緒二進制檔在 glibc 下仍可能在各 arena 間累積保留但未使用的記憶體——這是已知的 glibc 配置器行為，不是 Rust 側的缺陷。

**處置**：在該服務的環境變數中設定 `MALLOC_ARENA_MAX=2`，限制 glibc 建立的 arena 數量。

**驗證（修復後 10.6h soak）**：RSS 升至約 24.5 MB 後，arena 回收使其降至 17.2 MB。全程逐時增長斜率：前 2 小時 +5.74 MB/h，中段約 0.00 MB/h，末 2 小時 −3.69 MB/h——趨勢在測試後段反轉，進程把記憶體還回去了。測試通過。

**洞見**：改用 Rust 消除的是一整類語言層級的記憶體安全錯誤——use-after-free、double-free、緩衝區溢位——而且是在編譯期就排除。但這**不代表**一個進程就此對記憶體增長免疫。配置器設定與 C 執行環境層級的行為，位在語言安全保證之下的那一層，只有 soak 測試——不是單元測試，也不是程式碼審查——才能把這類問題浮現出來。Soak 測試應該是 Phase B 閘門中永久存在的一環，不是通過一次就結案的里程碑。

#### 4c. 穩定性與確定性驗證（24h 監控）

**成功標準**：
- CPU 穩定在實測的平均 6.3%／峰值 8.8% soak 基線附近，無持續漂移
- 記憶體穩定在 206 MB 全棧基線；任何持續正斜率都用上述案例研究的方法調查
- Load average 穩定在實測的平均 0.50／峰值 1.81 基線附近（4 核）
- 無垃圾回收暫停引起的延遲（Rust 無 GC）

---

## CI/CD 整合

測試執行管道：
1. Tier 1（5 min）：編譯 + 單元測試 + lint
2. Tier 2（15 min）：整合測試 + 基礎負載測試
3. Tier 3（30 min）：Golden-Master 測試（1 週樣本）
4. Tier 4（60 min）：性能測試（1 小時峰值負載 + 7.8 小時 soak 基線）
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
