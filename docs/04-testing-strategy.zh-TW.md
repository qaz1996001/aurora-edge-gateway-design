# 測試策略：Golden-Master 測試與驗證

## 概觀

測試一次大型重寫，不只要驗證新系統「能動」，還要驗證它在所有可觀察輸出上的行為與舊系統**完全一致**。這正是 **golden-master**（又稱**特徵化**，characterization）測試的目的。

參見 **[ADR-0006](adr/0006-golden-master-testing.zh-TW.md)** 了解決策理由。

---

## Golden-Master 測試方法論

### 核心原則
Golden-master 測試捕獲**原始系統**（Node.js）在實際輸入上的輸出，然後驗證**新系統**（Rust）在相同輸入上是否產生相同輸出。

**關鍵洞察**：我們測試的不是新系統在絕對意義上是否「正確」；我們測試的是它是否**保留了行為**。

### 流程

#### 第一階段：流量捕獲（Phase A 前 2 週）
1. 在生產環境的 Node.js gateway 上安裝記錄 hook：
   - 記錄每一則收到的 MQTT 訊息（topic、payload、timestamp）
   - 記錄每一次 REST API 呼叫（endpoint、method、payload、query 參數、timestamp）
   - 記錄每一次資料庫操作（SQL 查詢、參數、影響的行數）
   - 記錄感測器讀數值與時序

2. 讓已安裝記錄 hook 的系統在生產環境中運行 3 週以上
   - 捕獲 ≥ 100 萬筆 MQTT 事件
   - 捕獲 ≥ 50,000 筆 REST API 呼叫
   - 捕獲 ≥ 10,000 筆資料庫交易
   - 確保流量涵蓋正常操作與邊界情境

3. 將所有捕獲的資料存為結構化的 JSON log（golden dataset）

#### 第二階段：Golden Dataset 分析（Phase A 第 1 週）
1. 以記錄 hook 回放捕獲的流量到 Node.js 系統：
   - 在關鍵決策點捕獲內部狀態（感測器聚合前/後、資料庫寫入前/後）
   - 捕獲所有可觀察的輸出（REST 回應、MQTT 發佈、DB 狀態）
   - 存為「ground truth」參考基準

2. 依情境將流量分桶：
   - **正常操作**（70% 流量）：一般感測器讀取、REST 查詢
   - **峰值負載**（15% 流量）：高頻 MQTT 爆發、並發 REST 請求
   - **錯誤情境**（10% 流量）：無效輸入、網路逾時、DB 錯誤
   - **邊界情境**（5% 流量）：邊界條件、競態條件、資料異常

#### 第三階段：Golden-Master 測試套件（Phase A 第 2–8 週）
每移植一個 Rust 模組，就對照 golden dataset 測試：

```
For each bucket in traffic:
  1. Replay traffic to Rust system
  2. Capture outputs (REST responses, MQTT publishes, DB state)
  3. Compare outputs with golden dataset
     - Response status codes must match exactly
     - Response body JSON must match exactly (ignoring timestamps if appropriate)
     - Database state must match exactly (same rows, same values)
     - Error messages must match (or be explicitly equivalent)
  4. If mismatch found:
     - Log detailed diff (what was expected vs. what was observed)
     - Halt testing; investigate and fix Rust code
     - Re-test

Test result: 100% output parity achieved ✓
```

---

## 測試分類與涵蓋範圍

### 分類 1：單元測試（依模組，Phase A & B）

**目的**：驗證個別函式在隔離環境中的行為是否正確。

**範例：**
- `aeg-config` 測試：解析有效/無效 TOML；驗證預設值；驗證覆寫優先順序
- `aeg-db` 測試：SQL 插入/更新/刪除；驗證交易語義；測試違反限制條件的情況
- `aeg-sensors` 測試：聚合讀數；驗證濾波；檢查數值精確度
- 記憶體分析測試：驗證 drop chain 無洩漏；追蹤配置大小

**最低覆蓋率**：70% 程式碼涵蓋率（排除難以觸發的錯誤路徑）

**執行**：`cargo test`（CI 中自動執行）

---

### 分類 2：整合測試（模組配對，Phase A）

**目的**：驗證模組之間能正確協同運作（真實 I/O、共享狀態）。

**範例：**
- `aeg-db` + `aeg-sensors`：將感測器讀數寫入 DB；驗證 SQL 正確；驗證 DB 狀態符合預期
- `aeg-mqtt-client` + `aeg-sensors`：注入 MQTT 訊息；驗證感測器聚合；驗證輸出
- `aeg-config` + `aeg-appserver`：載入設定；驗證 appserver 路由已初始化；驗證 REST endpoint 有回應

**最低覆蓋率**：所有面向使用者的程式碼路徑（REST endpoint、MQTT topic、DB 交易）

**執行**：`cargo test --test integration_tests`（CI 中手動觸發）

---

### 分類 3：Golden-Master 測試（完整系統，Phase A & B）

**目的**：驗證整個系統在近似生產流量上，產生與 Node.js 基準完全一致的輸出。

**範圍**：
- 100 萬以上 MQTT 事件
- 50,000 以上 REST API 呼叫
- 10,000 以上資料庫交易
- golden dataset 中捕獲了 3 週以上的連續運行

**執行**：
```bash
# Replay golden dataset against Rust system
cargo build --release --target aarch64-unknown-linux-gnu

# Run Rust system
./target/aarch64-unknown-linux-gnu/release/aeg-appserver &
./target/aarch64-unknown-linux-gnu/release/aeg-mqtt &
# ... (start all services)

# Replay golden traffic
python3 golden-master-harness.py \
  --golden-dataset production-traffic-3weeks.json \
  --compare-outputs true \
  --tolerance-pct 0.1  # Allow 0.1% numerical error (floating point)

# Check result
if [ $? -eq 0 ]; then
  echo "✓ Golden-master test PASSED"
else
  echo "✗ Golden-master test FAILED"
  # Review detailed diff log
  cat golden-master-diff.json | jq .
fi
```

**成功標準**：
- 所有 REST API 回應與 golden dataset 相符（狀態碼、回應內容）
- 所有已發佈的 MQTT 訊息相符（topic、payload、順序、時序誤差在 ± 5ms 內）
- 所有資料庫狀態相符（行數、欄位值）
- 所有錯誤情境處理方式一致（錯誤訊息、HTTP 狀態碼）

**失敗處理**：
- 一旦發現不符 → 立即暫停測試；調查根本原因；修正程式碼；重新測試
- 可接受的容差：± 5ms 時間戳差異；± 0.1% 浮點精度損失

---

### 分類 4：性能與壓力測試（Phase B）

**目的**：驗證性能目標，以及系統在極端條件下的可靠性。

#### 4a. 負載測試（代表性負載與峰值負載）
```bash
# Exercise representative production traffic, then a peak-load profile,
# on the target 4-core ARM Cortex-A53 platform
load-generator \
  --profile representative-production \
  --duration 10800s \
  --target localhost:3000

load-generator \
  --profile peak \
  --duration 10800s \
  --target localhost:3000

# Measure results against the Phase B soak baseline (below)
# - CPU usage (soak baseline: 6.3% avg, 8.8% peak)
# - Memory usage (soak baseline: 206 MB full-stack resident)
# - Load average (soak baseline: 0.50 avg, 1.81 peak, 4-core)
# - MQTT message delivery rate (should be 100%)
```

**目標指標**（Phase B，錨定於在 4 核 ARM Cortex-A53 上實測的 7.8h soak 基線）：
- CPU：平均 6.3%、峰值 8.8%，soak 期間零漂移——對照 Node.js 基線在同一硬體上的待機 52%／3h 活躍 64%
- 記憶體：全棧常駐 206 MB（14 個受監督服務 + Python 子行程）——對照 Node.js 基線的 1538 MB 活躍（約佔可用 2.84 GB 的 54%）
- Load average（4 核）：平均 0.50、峰值 1.81——對照 Node.js 基線的 3.30 平均
- 吞吐量：代表性生產負載與峰值負載皆可持續運行而不掉訊息。我們目前沒有針對此平台建立過的吞吐量上限基準測試；`--profile` 模擬的是真實的流量形態,而非某個具體速率,而建立明確的吞吐量上限被列為另外的後續工作,本文件不作此宣稱
- 訊息丟失率：0%
- 記憶體增長：運行期間無持續正斜率的 RSS 增長（斜率量測方法與其曾揪出的問題，見下方 4b 的案例研究）

#### 4b. 記憶體 Soak 測試（實測：7.8h 與 10.6h 運行）
```bash
# Run continuous representative load and monitor memory, CPU, and service stability.
# Longest runs completed to date: 7.8h (initial baseline) and 10.6h (after the
# MALLOC_ARENA_MAX fix described in the case study below).

soak-test-runner \
  --duration 10.6h \
  --profile representative-production \
  --check-interval 1hour

# Output memory graph and stability report
```

**結果——初始 7.8h soak（全棧，4 核 ARM Cortex-A53）**：
- 全棧常駐記憶體：206 MB（14 個受監督服務 + Python 子行程）
- CPU：平均 6.3%、峰值 8.8%，全程零漂移
- Load average：平均 0.50、峰值 1.81（4 核）
- 系統可用記憶體：6024 → 5885 MB（7.8h 內 −139 MB）
- 重啟次數：頭 66 分鐘內 3 次（啟動穩定期），之後 6.7 小時零重啟

**成功標準**：
- 運行期間每個服務無持續正斜率的 RSS 增長（不是一個固定的每日 % 數字——斜率量測方法實際採用的方式見下方案例研究）
- CPU 穩定，無衰退或漂移（7.8h soak 實測平均值 6.3% 持平）
- 零無計畫崩潰（僅有刻意為測試而發起的 kill）
- 啟動穩定期後零服務重啟（7.8h 測試中啟動穩定期後的 6.7 小時內零重啟；10.6h 測試全程通過——見案例研究）
- 無 GC 暫停（Rust 沒有 GC runtime）

#### 4b-i. 案例研究：在 Soak 測試中發現並修復的一個真實記憶體增長問題

這是 Phase B soak 測試迄今產出的最具價值的單一結果，值得完整記錄，因為它正好示範了 soak 測試真正的用途所在。

**觀察到的現象**：在初始 7.8h soak 中，訊息匯流排（message-bus）進程的 RSS 從 45.0 MB 增長到 95.3 MB——增幅 +112%，大致呈線性攀升，速率約 6.5 MB/h——而該進程的 CPU 使用率全程維持在恆定的 0.2%。線性且 CPU 持平的增長曲線，正是配置器（allocator）或 runtime 層級問題的特徵，而不是應用邏輯驅動的洩漏。

**根本原因**：glibc 的多執行緒配置器為該進程開出了多達 64 個獨立的記憶體 arena。每個 arena 都維護自己的 free list，因此即使 Rust 程式碼本身零記憶體安全錯誤，多執行緒二進位檔在 glibc 下仍可能在各 arena 之間累積保留但未使用的記憶體——這是已被記錄在案的 glibc 配置器行為，不是 Rust 側的缺陷。

**修復方式**：在該服務的環境變數中設定 `MALLOC_ARENA_MAX=2`，限制 glibc 建立的 arena 數量上限。

**驗證（修復後的 10.6h soak）**：RSS 升至約 24.5 MB 後，arena 回收使其回落到 17.2 MB。全程逐時增長斜率為：前 2 小時 +5.74 MB/h，運行中段約 0.00 MB/h，末 2 小時 −3.69 MB/h——趨勢在運行後段反轉，進程把記憶體還了回去。測試通過。

**啟示**：改寫為 Rust 在編譯期就消除了一整類語言層級的記憶體安全錯誤——use-after-free、double-free、緩衝區溢位。但這**並不代表**一個進程就此對記憶體增長免疫。配置器設定與 C 執行環境的行為，位於語言安全保證之下的那一層，只有 soak 測試——不是單元測試，也不是程式碼審查——才能把這類增長問題浮現出來。Soak 測試應該是 Phase B 閘門中永久存在的一環，不是通過一次就結案的里程碑。

#### 4c. 穩定性與確定性驗證（24h 監控）
```bash
# Monitor CPU, memory, and load for deterministic behavior over 24 hours.
# Builds on the 7.8h/10.6h soak results above; there is no GC-driven
# scheduling jitter to account for.

stability-monitor \
  --duration 86400s \
  --sample-interval 10s \
  --output stability-report.json

# Expected: CPU/memory/load as flat as observed in the completed soaks
```

**成功標準**：
- CPU 穩定在實測的平均 6.3%／峰值 8.8% soak 基線附近，無持續漂移
- 記憶體穩定在 206 MB 全棧基線；任何持續正斜率都用上述案例研究中的方法調查
- Load average 穩定在實測的平均 0.50／峰值 1.81 基線附近（4 核）
- 無來自垃圾回收造成的延遲變異（Rust 沒有 GC）
- 一天內不同時段的表現保持一致

---

### 分類 5：正確性與驗證測試（Phase B）

#### 5a. 感測器讀數精確度
```bash
# Inject synthetic sensor data; verify calculations match baseline

synthetic-sensor-test \
  --test-data synthetic-readings.json \
  --compare-with node.js-baseline \
  --tolerance-pct 0.01  # 0.01% error allowed (floating point)

# Illustrative example only (not a measured result): if the baseline's
# aeg-metrics signal-RMS aggregation reports a reference value of, say,
# 230.5 for some channel, Rust must reproduce it within the tolerance band
# (230.5 ± 0.1 at the 0.01% setting above) to count as a golden-master match.
```

**成功標準**：
- 所有計算值都在容差範圍內與基準相符
- 沒有系統性偏差（不會一致偏高或偏低）

#### 5b. 並發寫入下的資料一致性
```bash
# Inject 100 concurrent sensor writes + 10 concurrent REST reads
# Verify database consistency and no lost updates

concurrent-write-test \
  --concurrent-sensors 100 \
  --concurrent-http-reads 10 \
  --duration 3600s \
  --verify-db-integrity true

# Expected: Zero DB corruption; all reads see consistent data; no lost writes
```

**成功標準**：
- 測試後資料庫通過 `PRAGMA integrity_check`
- 沒有寫入遺失（全部 100 * 3600 = 36 萬筆寫入皆已持久化）
- 所有讀取都看到一致的快照（維持 ACID 保證）

#### 5c. Python 子行程橋接
```bash
# Verify correct IPC to Python subprocess; measure overhead

python-bridge-test \
  --test-cases signal-processing,ml-inference \
  --throughput 100calls/sec \
  --duration 3600s

# Expected: 100% success rate; no process leaks; per-call latency recorded as a baseline
```

**成功標準**：
- 所有子行程呼叫都成功
- 記錄每次呼叫的回應時間以建立基準（目前尚無已量測的目標值）
- Python 行程皆被正確清理（無殭屍行程）
- 子行程崩潰時能被優雅處理（自動重啟 + 告警）

---

### 分類 6：混沌與故障注入測試（Phase B）

#### 6a. 網路故障
```bash
# Simulate network issues: packet loss, latency, disconnects

fault-injection-test \
  --fault-type packet-loss \
  --packet-loss-rate 5% \
  --duration 3600s

fault-injection-test \
  --fault-type latency \
  --latency-ms 500 \
  --duration 3600s

fault-injection-test \
  --fault-type disconnect \
  --disconnect-every 60s \
  --duration 3600s
```

**預期行為**：
- MQTT client 自動重連（採用 backoff 策略）
- REST API 請求優雅地逾時（具重試邏輯）
- 不會發生永久性的連線中斷
- 服務持續運作（可接受降級模式）

#### 6b. 磁碟滿
```bash
# Simulate disk full; verify graceful handling

fault-injection-test \
  --fault-type disk-full \
  --reserve-mb 0 \
  --duration 300s

# Expected: Writes fail with proper error; service alerts but doesn't crash; can recover when space freed
```

#### 6c. 資料庫損毀
```bash
# Inject corrupted database; verify detection and recovery

fault-injection-test \
  --fault-type corrupt-db \
  --corruption-type bad-magic-bytes \
  --duration 60s

# Expected: Service detects corruption; refuses to start; can restore from backup
```

---

## 測試執行管道（CI/CD）

```
Code Commit
    ↓
Tier 1：編譯與單元測試（5 min）
    ├─ cargo build
    ├─ cargo test（所有單元測試）
    └─ cargo clippy（lint）
    ↓ [失敗時：停止；通知開發者]
    ↓
Tier 2：整合測試（15 min）
    ├─ cargo test --test '*'（所有整合測試）
    └─ 基礎負載測試（代表性流量形態，1 min）
    ↓ [失敗時：停止；通知團隊]
    ↓
Tier 3：Golden-Master 測試（30 min）
    ├─ 回放 1 週流量樣本到 Rust 系統
    ├─ 驗證輸出對等
    └─ 產生 diff 報告
    ↓ [失敗時：通知程式碼審查者；可能需要人工審查]
    ↓
Tier 4：性能測試（60 min，僅在 main 分支執行）
    ├─ 1 小時負載測試（峰值負載 profile）
    ├─ 記憶體 soak 測試（7.8h 基線運行；見分類 4b）
    └─ 延遲 histogram
    ↓ [失敗時：通知性能團隊]
    ↓
Tier 5：擴展 Golden-Master（每夜，2 小時）
    └─ 回放 3 週生產流量；驗證 100% 對等
    ↓ [失敗時：通知團隊；調查；可能需要回滾]
    ↓
Deploy Candidate
```

---

## Golden-Master 測試工具（虛擬程式碼）

```python
class GoldenMasterHarness:
    def __init__(self, golden_dataset_json):
        self.golden = json.load(golden_dataset_json)
        self.results = {"passed": [], "failed": []}
    
    def replay_mqtt_event(self, event):
        """Inject MQTT message into Rust system; capture output."""
        self.mqtt_client.publish(event['topic'], event['payload'])
        # Wait for any resulting REST API calls or DB writes
        time.sleep(0.1)  # Give system time to process
        
        # Capture any resulting MQTT publishes
        published = self.mqtt_client.get_published_messages()
        expected = event.get('expected_publishes', [])
        
        if published == expected:
            self.results["passed"].append(event['id'])
        else:
            self.results["failed"].append({
                'id': event['id'],
                'expected': expected,
                'actual': published,
                'diff': compute_diff(expected, published)
            })
    
    def replay_rest_api_call(self, call):
        """Inject REST API call into Rust system; verify response."""
        response = requests.request(
            method=call['method'],
            url=f"http://localhost:3000{call['path']}",
            json=call.get('body')
        )
        
        expected_response = call['expected_response']
        
        if (response.status_code == expected_response['status'] and
            response.json() == expected_response['body']):
            self.results["passed"].append(call['id'])
        else:
            self.results["failed"].append({
                'id': call['id'],
                'expected_status': expected_response['status'],
                'actual_status': response.status_code,
                'expected_body': expected_response['body'],
                'actual_body': response.json()
            })
    
    def replay_all(self):
        """Replay entire golden dataset."""
        for event in self.golden['mqtt_events']:
            self.replay_mqtt_event(event)
        
        for call in self.golden['rest_api_calls']:
            self.replay_rest_api_call(call)
        
        # Verify database state
        self.verify_db_state()
        
        # Generate report
        self.generate_report()
    
    def generate_report(self):
        """Print summary and detailed failures."""
        print(f"Passed: {len(self.results['passed'])}")
        print(f"Failed: {len(self.results['failed'])}")
        
        if self.results['failed']:
            print("\nFailed Tests:")
            for failure in self.results['failed']:
                print(json.dumps(failure, indent=2))
            return 1  # Non-zero exit code
        
        return 0  # Success
```

---

## 測試資料管理

### Golden Dataset 生命週期
1. **捕獲**：3 週以上的生產流量（100 萬以上事件）
2. **版本控管**：存入 git（大檔案用 git-lfs）；以日期標記 tag
3. **封存**：保留歷史 dataset 以供迴歸測試
4. **輪替**：保留最近 3 份 dataset；較舊的予以封存

### 隱私與資料清洗
- 移除敏感資料（真實 IP 位址、使用者名稱、客戶專屬數值）
- 以通用值取代（192.0.2.0/24 測試位址、「Customer A」等）
- golden dataset 若存入 git，須加密

---

## 下一步

1. **審查流程** → 見 `05-review-process.zh-TW.md` 了解程式碼審查框架
2. **風險管理** → 見 `06-risk-register-template.zh-TW.md` 了解風險分類
3. **決策理由** → 見 **[ADR-0006](adr/0006-golden-master-testing.zh-TW.md)** 了解原因與作法

---

**最後更新**：2026-10-20  
**狀態**：測試策略已批准  
**下次審查**：Phase A 第 2 週（golden dataset 分析完成時）
