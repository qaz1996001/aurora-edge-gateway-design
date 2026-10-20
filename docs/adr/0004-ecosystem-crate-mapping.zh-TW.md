# ADR-0004：技術選擇 — npm → Rust 生態系對映

**狀態**：已接受  
**決策日期**：2026-Q3  
**利益相關者**：平台工程師、架構團隊

---

## 背景

從 Node.js 遷移至 Rust 需選擇等價庫（crate）用於每個 npm 依賴。關鍵標準：
- **成熟度**：Crate 主動維護，具良好 API 穩定性
- **生態系適配**：Crate 為使用案例設計（非通用）
- **社群**：社群足夠大以取得幫助；文件良好
- **非同步支援**：對於 I/O 綁定 crate，偏好 async-first 設計

---

## 決策

採用下表所示的 npm → Rust crate 對映。

---

## 對映表

| npm（Node.js） | Rust（crate） | 領域 | 理由 |
|---|---|---|---|
| **express** | **axum** | HTTP 伺服器 | 零成本抽象；型別安全路由；卓越 tokio 非同步整合 |
| **mqtt** | **rumqttc** | MQTT 用戶端 | Async-first MQTT 庫；卓越嵌入式/IoT 支援 |
| **better-sqlite3** | **rusqlite** | SQLite 驅動程式 | 同步 API 對我們使用案例更簡單；無需非同步 DB 驅動程式開銷 |
| **socket.io** | **tokio-tungstenite** | WebSocket/即時 | 輕量 WebSocket 庫；足以滿足我們即時需求 |
| **winston** | **tracing** + **slog** | 日誌 | 結構化日誌搭配卓越非同步支援；journald 整合 |
| JSON（原生） | **serde_json** | JSON 序列化 | 零複製；標準 Rust 序列化 |
| **serialport**（npm） | **tokio-serial** / **serialport** | 指示燈（序列埠控制） | Rust 原生序列埠庫；指示燈硬體透過 USB 序列埠（9600 8N1）控制，非 GPIO |
| **request**（HTTP 用戶端） | **reqwest** | HTTP 用戶端 | 現代非同步 HTTP；卓越錯誤處理 |
| **fs**（原生） | **tokio::fs** | 非同步檔案 I/O | 非同步檔案操作而不阻擋 tokio 執行時 |
| **child_process** | **tokio::process** | 子程序生成 | 非同步子程序管理（用於 Python IPC） |
| **crypto**（原生） | **ring** | 密碼學 | 稽核密碼庫；比 Node.js `crypto` 更安全 |
| **commander**（CLI） | **clap** | 命令列解析 | 型別安全 CLI 引數解析 |

---

## 每個選擇的理由

### HTTP 伺服器：Express → Axum

**npm 選擇**：Express.js（最熱門 Node.js HTTP 伺服器）

**Rust 替代候選項**：
- **Actix-web**：非常高效；型別安全路由較少；依賴較重
- **Rocket**：更見解；強大；開銷較重
- **Warp**：過濾器基礎；函式式風格（學習曲線）
- **Axum**：最小；可組合；型別安全路由；卓越 tokio 整合 ✅

**選擇理由**：
- 路由型別安全防止 404 漏洞
- Tokio 上零成本層（我們使用的相同非同步執行時）
- 比過濾器替代方案更簡單心智模型
- 良好文件與範例

**風險**：Axum 比 Express 新；生態系成熟度較低。**緩解**：Golden-master 測試確保行為對等。

---

### MQTT 用戶端：mqtt → rumqttc

**npm 選擇**：mqtt

**Rust 替代候選項**：
- **paho-mqtt**：C 庫包裝函式；有效但較重
- **rust-mqtt**：實驗性；不建議生產
- **rumqttc**：現代 async-first Rust MQTT；卓越嵌入式支援 ✅

**選擇理由**：
- 為 IoT/嵌入式設計（我們的使用案例）
- Async-first（乾淨整合 tokio）
- 良好 API；處理重連與背壓
- 主動維護

**在新架構中的主要角色**：
在 Rust 架構中，`rumqttc` 充當**內部服務訊息匯流排**（類似原始系統），而非主要雲端上行。服務發佈/訂閱本機 MQTT 主題用於服務間事件協調（例如，感測器更新觸發 LED 更新）。雲端通訊透過 **REST API** 處理（心跳與批次同步）。MQTT 雲端上行是一個選配、工廠預設關閉的加速功能；它不是預設上行路徑。

---

### SQLite 驅動程式：better-sqlite3 → Rusqlite

**npm 選擇**：better-sqlite3（同步 SQLite 綁定）

**Rust 替代候選項**：
- **sqlx**：非同步；編譯時檢查查詢；設定較重
- **diesel**：ORM；強大但見解傾向；較重
- **rusqlite**：簡單同步包裝函式；最小開銷 ✅
- **sqlite**：純 Rust 實施；較慢；不建議

**選擇理由**：
- 我們的資料庫工作負載查詢負載輕、非 async 密集
- 同步 API 更簡單；無需非同步開銷
- 卓越文件；廣泛使用
- 資料庫操作無非同步複雜性

**取捨**：若 DB 慢，rusqlite 同步呼叫可能阻擋 tokio 任務。**緩解**：若需要，將 DB 呼叫包裝在 `tokio::task::spawn_blocking()` 中。

---

### 日誌：Winston → Tracing + Slog

**npm 選擇**：winston（結構化日誌）

**Rust 替代候選項**：
- **log**：輕量；標準但功能較少
- **env_logger**：簡單；與 log crate 耦合
- **tracing**：結構化非同步日誌；卓越非同步整合 ✅
- **fern**：靈活但設定較重

**選擇理由**：
- 機器解析的結構化日誌用於告警
- 非同步整合（無非同步上下文中阻擋）
- journald 支援（直接記錄至 systemd journal）
- 卓越每個模組過濾與層級控制

---

### 指示燈控制：npm serialport → tokio-serial / serialport（Rust）

指示燈是透過 USB 序列埠（9600 baud、8N1）控制的——並未接到 GPIO。因此正確的對映目標是序列埠 crate，而非 GPIO crate。

**npm 選擇**：透過 USB 序列埠（9600 8N1）與燈具通訊的 Node.js 序列埠庫

**Rust 替代候選項**：
- **tokio-serial**：與 tokio 整合的非同步序列埠 I/O ✅
- **serialport**：同步、跨平台序列埠庫；對低頻率控制指令較簡單 ✅
- 直接透過 `std::fs` 存取 `/dev/ttyUSB*`：可行但需要不必要地重新實作線路規程處理

**選擇理由**：
- 符合實際硬體介面（USB 序列埠，非 GPIO）
- 若燈具指令需要非同步處理，`tokio-serial` 可與既有 tokio 執行時整合；考量指令頻率低，`serialport` 是較簡單的同步選項
- 型別安全的連接埠設定（鮑率、同位、停止位）

**改善**：消除先前橋接至序列裝置的 Python 子程序（Node.js 呼叫 Python 處理序列 I/O；Rust 直接與序列埠通訊）。

---

## Crate 版本與鎖策略

### 依賴鎖檔（Cargo.lock）
所有依賴在 `Cargo.lock` 中版本固定用於可重現建置：
```toml
[package]
name = "aeg-appserver"
version = "0.1.0"

[dependencies]
tokio = { version = "1.35", features = ["full"] }
axum = "0.7"
serde_json = "1.0"
```

**政策**：
- 在 Cargo.lock 中固定精確版本
- 每週審查安全公告（`cargo audit`）
- 每月更新依賴（若無安全問題）
- 依賴更新後徹底測試

### 易受攻擊依賴回應
若依賴有已知漏洞：
1. 補丁發佈 → 更新 crate 至修補版本（立即）
2. 無補丁可用 → 固定至上一個安全版本；尋找替代 crate（週度衝刺）

---

## 序列化格式：JSON vs. Bincode

用於程序間通訊（Rust ↔ Python）：
- **格式**：bincode（Rust）或 msgpack/JSON 用於相容性
- **理由**：
  - bincode：快速、緊湊（適合高頻訊息）
  - JSON：人類可讀（更容易除錯；適合較少頻繁訊息）

**決策**：用 bincode 用於高頻、低延遲 IPC；JSON 用於配置與人類互動。

---

## 驗證

### 合併任何程式碼前
```bash
# 1. 依賴稽核
cargo audit  # 必須通過（無已知漏洞）

# 2. 編譯檢查
cargo check

# 3. 執行測試
cargo test

# 4. Lint 檢查
cargo clippy

# 5. 程式碼涵蓋（Phase A）
cargo tarpaulin
```

---

## 相關決策
- **[ADR-0002](0002-module-to-crate-decomposition.md)**：每個 crate 使用這些依賴的子集
- **[ADR-0003](0003-hybrid-runtime-keep-python-subprocess.md)**：Python IPC 的 tokio 子程序支援

---

## 維護計畫

### 季度依賴審查
- [ ] `cargo outdated` – 識別過時依賴
- [ ] `cargo audit` – 檢查漏洞
- [ ] 評估主要版本升級（破壞變更）
- [ ] 若進行更新則徹底測試

### 安全補丁 SLA
- **關鍵**：24 小時內部署
- **高**：1 週內部署
- **中**：1 個月內部署
- **低**：下一次排程發佈時部署

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：架構團隊  
**實施**：Phase A（依賴在 Cargo.toml 中鎖定）
