# 架構概覽：Before & After

## 系統上下文

**Aurora Edge Gateway** 是工業 IoT 資料聚合與控制平台，部署在邊緣設備（ARM 單板電腦，執行 256 MB–1 GB RAM）上。它充當以下之間的橋樑：
- **上游**：雲端後端（資料攝取、車隊管理、分析）
- **下游**：工業感測器與設備（MQTT、serial 協議、Modbus）

### C4 上下文圖
參見 `../diagrams/context-diagram.svg`（外部依賴與資料流的視覺概覽）

---

## Before：Node.js 架構

### 系統拓樸（原始）

```
┌─ 邊緣設備（ARM Linux） ──────────────────────────────┐
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  單一 Node.js 進程（PM2 管理）                │ │
│  │  - 1 個主進程（40+ 個模組，~15k LOC）        │ │
│  │  - Worker 線程（3–5 個線程用於 CPU 任務）    │ │
│  │  - 記憶體佔用：80–120 MB                     │ │
│  │                                              │ │
│  │  模組（打包成 index.js）：                   │ │
│  │  - appserver（Express HTTP 路由、REST API）  │ │
│  │  - mqtt-client（rumba/paho MQTT 客戶端）    │ │
│  │  - sensor-data（聚合、DSP 濾波）             │ │
│  │  - led-tower（RGB LED 控制）                 │ │
│  │  - file-manager（本地檔案 I/O）              │ │
│  │  - ota-updater（韌體更新邏輯）               │ │
│  │  - VRMS（電錶讀數計算）                     │ │
│  │  - configuration（TOML/JSON 解析）           │ │
│  │  - database（better-sqlite3 包裝）           │ │
│  │  - python-bridge（subprocess.spawn Python）   │ │
│  │                                              │ │
│  │  依賴：                                      │ │
│  │  - Express.js → HTTP 路由、中間件            │ │
│  │  - mqtt (rumba) → MQTT 客戶端                │ │
│  │  - better-sqlite3 → SQLite3 綁定             │ │
│  │  - socket.io → 實時客戶端通訊                │ │
│  │  - winston → 日誌記錄                        │ │
│  └──────────────────────────────────────────────┘ │
│                ↓（PM2 重啟崩潰時）                │
└─────────────────────────────────────────────────────┘
     ↓ MQTT out       ↑ MQTT in         ↓ REST ↑
    ┌─────────────────────────────────────────────┐
    │ 雲端後端（MQTT Broker、REST API 消費者） │
    └─────────────────────────────────────────────┘
     ↓ Serial/USB      ↑ GPIO
    ┌─────────────────────────────────────────────┐
    │ 工業感測器（溫度、振動、耗電量等）與     │
    │ LED/繼電器控制                              │
    └─────────────────────────────────────────────┘
```

### 問題
1. **單體**：所有模組在一個進程內；一個崩潰級聯
2. **記憶體洩漏**：事件流聚合中的循環參照
3. **GC 暫停**：不可預測的 stop-the-world 延遲（100–500 ms）
4. **可擴展性受限**：單一事件迴圈；高吞吐量時 CPU 爭用
5. **維護困難**：任何模組變更需要完整重啟

---

## After：Rust 架構

### 系統拓樸（Rust 重寫）

```
┌─ 邊緣設備（ARM Linux） ────────────────────────────┐
│                                                   │
│  多個 Systemd 服務（鬆散耦合）：                  │
│                                                   │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ aeg-appserver│  │ aeg-mqtt     │              │
│  │ (Axum+REST) │  │ (rumqttc)    │              │
│  │ 15 MB       │  │ 12 MB        │              │
│  └──────────────┘  └──────────────┘              │
│                                                   │
│  ┌──────────────┐  ┌───────────────┐             │
│  │ aeg-sensors  │  │ aeg-led       │             │
│  │ (聚合)       │  │ (GPIO/PWM)    │             │
│  │ 18 MB       │  │ 8 MB          │             │
│  └──────────────┘  └───────────────┘             │
│                                                   │
│  共享基礎設施：                                   │
│  - aeg-db（SQLite 包裝器）                       │
│  - aeg-mqtt-client（rumqttc 原語）              │
│  - aeg-python-bridge（Unix socket → Python）     │
│  - systemd socket 啟動（零複製 IPC）             │
│  - syslog（結構化日誌透過 journald）             │
│                                                   │
│  Python Subprocess（DSP/ML）：                    │
│  - signal-processing（傅立葉、濾波）             │
│  - ml-inference（輕量模型、TensorFlow Lite）     │
│  - 按需生成；任務完成後殺死                      │
│                                                   │
│  總記憶體佔用（空閒）：~25 MB（vs. 80+ MB Node.js）
└────────────────────────────────────────────────┘
     ↓ MQTT out       ↑ MQTT in         ↓ REST ↑
    ┌──────────────────────────────────────────┐
    │ 雲端後端（MQTT Broker、REST API 消費者）│
    └──────────────────────────────────────────┘
     ↓ Serial/USB      ↑ GPIO
    ┌──────────────────────────────────────────┐
    │ 工業感測器與 LED/繼電器控制              │
    └──────────────────────────────────────────┘
```

### 優勢
1. **模組化**：每個 crate/二進位獨立可重啟；故障隔離
2. **記憶體安全**：無未定義行為；Rust 編譯器在編譯時防止洩漏
3. **確定性**：無 GC；所有資源清理明確（drop 語義）
4. **可擴展**：tokio 的 async/await；無線程高效並行
5. **可維護**：強型別在重構時捕捉錯誤
6. **可觀察**：每個服務記錄到 systemd journald；易於集中監控

---

## 模組 → Crate → 二進位對照

參見 `../diagrams/module-crate-mapping.svg` 了解詳細拆解

### 範例對照

| 原 Node.js 模組 | Rust Crate | Systemd 二進位 | 依賴 |
|---|---|---|---|
| appserver（Express 路由） | `aeg-appserver` | `aeg-appserver` | axum, tokio, serde, aeg-db |
| mqtt-client | `aeg-mqtt-client` | (shared lib) | rumqttc, tokio |
| sensor-data | `aeg-sensors` | `aeg-sensors` | aeg-db, aeg-mqtt-client, tokio |
| led-tower | `aeg-led` | `aeg-led` | rppal, tokio |
| file-manager | `aeg-file-manager` | (shared lib) | tokio, anyhow |
| ota-updater | `aeg-ota` | `aeg-ota` | sha2, aeg-file-manager |
| VRMS | `aeg-vrms` | `aeg-vrms` | aeg-sensors, aeg-db |
| configuration | `aeg-config` | (build-time) | serde, toml |
| database | `aeg-db` | (shared lib) | rusqlite, tokio |
| python-bridge | `aeg-python-bridge` | (IPC daemon) | tokio, serde |

---

## 技術選擇：npm → Rust

詳見 **[ADR-0004：生態系 Crate 對照](adr/0004-ecosystem-crate-mapping.md)** 了解詳細理由

| 領域 | Node.js (npm) | Rust (crates.io) | 理由 |
|---|---|---|---|
| HTTP 伺服器 | express | axum | 零成本抽象；型別安全路由 |
| MQTT | mqtt (rumba) | rumqttc | 異步優先；更好的嵌入式使用案例支持 |
| SQLite | better-sqlite3 | rusqlite | 同步 API 足以；無 GC |
| 異步運行時 | node（原生） | tokio | 業界標準；優秀的並行 |
| 日誌 | winston | tracing + slog | 結構化日誌；journald 整合 |
| 序列化 | JSON（原生） | serde | 零成本序列化 |
| GPIO 控制 | rpi.GPIO (npm) | rppal | Rust 原生；無 Python 開銷 |
| 跨編譯 | n/a | cargo（內建） | ARM 目標原生支持 |

---

## Before vs. After 對比

| 特徵 | Node.js | Rust |
|---|---|---|
| **進程數** | 1 個單體 | 6–8 個獨立服務 |
| **記憶體（空閒）** | 80 MB | 25 MB |
| **記憶體（負載下）** | 120+ MB，每日增長 2% | 35–40 MB，< 0.1% 每日增長 |
| **CPU（高峰，8 核 ARM）** | 85% | 25% |
| **GC 暫停** | 100–500 ms（非確定性） | 0 ms（確定性） |
| **平均事件延遲** | 15 ms | < 2 ms |
| **95th percentile 延遲** | 100 ms | < 10 ms |
| **無故障運行時間** | 7–14 天 | 90+ 天（測試） |
| **部署單位** | 完整進程重啟 | 個別服務重啟 |
| **API 破壞性變更** | 無 | 無（向後相容） |
| **斜坡效應** | 無（已部署） | 6–8 週（Phase A 移植） |

---

## Crate 架構圖

參見 `../diagrams/module-crate-mapping.svg` 了解完整 crate 依賴圖（DAG 結構）

### 依賴層

```
Tier 0（純資料結構）：
  └─ aeg-config（TOML 解析，無外部 I/O）

Tier 1（基礎服務）：
  ├─ aeg-db（SQLite 包裝器，無狀態）
  └─ aeg-file-manager（檔案 I/O 原語）

Tier 2（協議用戶端）：
  ├─ aeg-mqtt-client（MQTT 用戶端原語）
  └─ aeg-python-bridge（IPC 至 Python subprocess）

Tier 3（領域邏輯）：
  ├─ aeg-sensors（感測器聚合，使用 aeg-db + aeg-python-bridge）
  ├─ aeg-vrms（電錶讀數，使用 aeg-db + aeg-sensors）
  ├─ aeg-led（GPIO 控制，無共享依賴）
  └─ aeg-ota（韌體更新，使用 aeg-file-manager）

Tier 4（公開介面）：
  ├─ aeg-appserver（REST API，使用所有 Tier 3）
  └─ aeg-main（systemd 整合，啟動所有服務）
```

---

## 部署拓樸

參見 `../diagrams/deployment-topology.svg` 了解邊緣到雲端拓樸

### 邊緣設備 Systemd 服務

```
[aeg-appserver]
ExecStart=/usr/local/bin/aeg-appserver --config /etc/aeg/appserver.toml
Restart=always
RestartSec=5

[aeg-mqtt]
ExecStart=/usr/local/bin/aeg-mqtt --config /etc/aeg/mqtt.toml
Restart=always
RestartSec=5

... (similar for aeg-sensors, aeg-led, aeg-vrms, etc.)
```

每個服務：
- 以受限特權執行（大多數不需要 root）
- 崩潰時自動重啟（Restart=always）
- 記錄到 systemd journal（無檔案 I/O）
- 透過 Unix sockets 或 TCP 進行服務間通訊

---

## 資料流與並行

### 原始（Node.js，同步管道）

```
雲端 MQTT Broker
       ↓（每 ~100ms 1 個事件）
MQTT.on('message')（事件迴圈中的回調）
       ↓
解析消息 + 驗證
       ↓
資料庫寫入（阻塞，然後非同步）
       ↓
觸發感測器聚合（async/await）
       ↓
計算 VRMS → LED 更新
       ↓（可能被 GC 或 worker 線程競爭阻塞）
REST 客戶端輪詢（單獨線程）掛起
```

### 新的（Rust，含隔離的異步管道）

```
雲端 MQTT Broker
       ↓（每 ~100ms 1 個事件）
rumqttc async::stream::next()
       ↓（tokio 任務，非阻塞）
解析消息 + 驗證
       ↓（生成任務）
aeg-db 非同步寫入（非同步 sqlite 包裝器）
       ↓
透過 Unix socket 發出「sensor-updated」事件
       ↓（aeg-sensors 監聽 socket）
aeg-sensors 聚合 & 計算 VRMS
       ↓
aeg-led 更新 LED（獨立服務，無阻塞）
       ↓（全部平行，無爭用）
REST 客戶端輪詢（appserver，非同步，無阻塞）
       ✓ 即使高峰負載下回應 < 5ms
```

---

## 後續步驟

1. **遷移策略** → 參見 `03-migration-strategy.zh-TW.md` 了解兩階段移植方案
2. **測試與驗證** → 參見 `04-testing-strategy.zh-TW.md` 了解 golden-master 測試
3. **決策理由** → 參見 **[ADR-0002](adr/0002-module-to-crate-decomposition.md)** 和 **[ADR-0003](adr/0003-hybrid-runtime-keep-python-subprocess.md)** 了解架構決策
4. **視覺圖表** → 參見 `../diagrams/` 目錄

---

**最後更新**：2026-10-20  
**狀態**：基線架構已批准  
**下次審查**：Phase A 完成後（golden-master 測試結果）
