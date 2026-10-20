# Aurora Edge Gateway：Node.js → Rust 架構重寫

一份全面的設計文件 portfolio，展示工業 IoT 邊緣閘道器平台從 Node.js 遷移到 Rust 的策略性重新設計。

## 概述

本 Repository 記錄了 **Aurora Edge Gateway**（一個分散式工業 IoT 邊緣設備管理平台）從 Node.js 重寫到 Rust 的架構決策、遷移策略與工程實踐。

該閘道器是實時遙測和邊側計算的關鍵聚合層，負責管理工業感測器、設備控制和邊側資料處理。重寫的驅動力來自：

- **效能**：降低 CPU 佔用率與毫秒級感測器資料管道的回應延遲
- **記憶體安全**：消除長時間運行邊緣進程中的記憶體洩漏和未定義行為
- **運營可靠性**：在受限 ARM 邊緣硬體上實現多週不間斷運行（< 512 MB RAM）

## 如何導覽本 Repository

### 快速入門
1. **第一次來？** → 閱讀 [`docs/01-problem-and-goals.md`](docs/01-problem-and-goals.md) 了解「為什麼」
2. **想要全景概括？** → 參閱 [`docs/02-architecture-overview.md`](docs/02-architecture-overview.md) 查看 Before/After 系統設計
3. **對方法論有興趣？** → 跳至 [`docs/03-migration-strategy.md`](docs/03-migration-strategy.md) 了解分階段上線方案

### 完整文件索引

#### 設計與策略
- **[問題陳述與目標](docs/01-problem-and-goals.md)** — 為何用 Rust 重寫？效能、安全、可靠度指標
- **[架構概覽](docs/02-architecture-overview.md)** — 系統拓樸、模組→crate 分解、Before/After 對比（含 mermaid 圖表）
- **[遷移策略](docs/03-migration-strategy.md)** — 兩階段方案（Phase A：忠實移植，Phase B：強化+優化）、分階段上線計畫、回滾程序
- **[測試策略](docs/04-testing-strategy.md)** — Golden-master / 特徵化測試方法論，驗證行為對等性
- **[審查流程](docs/05-review-process.md)** — 多角色結構化審查框架（Rust 專家、安全、架構、效能、程式碼品質）
- **[風險登記](docs/06-risk-register-template.md)** — 風險分類、示例情境 high/medium/low 風險、緩解檢查表

#### 架構決策記錄（ADR）
每份 ADR 記錄一項架構決策、權衡取捨與影響：

- **[ADR-0001：為何用 Rust 重寫？](docs/adr/0001-why-rewrite-in-rust.md)** — 重寫決策；考量替代方案；理由分析
- **[ADR-0002：模組 → Crate 分解](docs/adr/0002-module-to-crate-decomposition.md)** — 細粒度模組→Rust-crate 結構；部署為獨立 systemd 單位
- **[ADR-0003：混合運行時（保留 Python Subprocess）](docs/adr/0003-hybrid-runtime-keep-python-subprocess.md)** — 保留 Python 用於 DSP/ML 的理由，跨語言 IPC 設計
- **[ADR-0004：生態系 Crate 對照](docs/adr/0004-ecosystem-crate-mapping.md)** — 技術選擇：npm→Rust crate 等價物（express→axum, mqtt→rumqttc, better-sqlite3→rusqlite 等）
- **[ADR-0005：分階段上線與回滾](docs/adr/0005-staged-rollout-and-rollback.md)** — 增量部署階段、自動化回滾閘門
- **[ADR-0006：Golden-Master 測試](docs/adr/0006-golden-master-testing.md)** — 行為對等性特徵化測試方案
- **[ADR-0007：移植標記慣例](docs/adr/0007-port-marker-convention.md)** — 可追蹤性標記系統（`TODO(port)`、`FIXME(port)`、`SAFETY:`）

#### 視覺圖表
位於 `/diagrams/`：
- `context-diagram.svg` — C4 風格的上下文圖，展示 Aurora Edge Gateway 與外部依賴
- `module-crate-mapping.svg` — Node.js 模組 → Rust crate → 二進位部署對照
- `deployment-topology.svg` — 邊緣設備網路拓樸（通用 ARM 邊緣、雲端後端）
- `staged-rollout-timeline.svg` — Stage 0–4 部署時間軸與 go/no-go 閘門

## 核心要點

### 架構模式
1. **模組化分解**：一個邏輯模組 → 一個 Rust crate → 一個 systemd 管理的二進位（鬆散耦合、獨立升級）
2. **混合運行時**：Rust 用於系統編程（I/O、並行、記憶體安全關鍵路徑）；Python subprocess 用於科學計算（DSP、ML 推理）
3. **兩階段移植**：
   - **Phase A**：忠實的行為移植（保留 bug、怪癖）以降低測試風險
   - **Phase B**：重構、強化、優化（確認對等性後）
4. **分階段上線**：Stage-by-Stage 部署（Stage 0：啟動；Stage 1–3：低/中/高風險；Stage 4：全面切換）含自動回滾觸發器

### 流程實踐
1. **Golden-Master 測試**：特徵化測試工具驗證舊系統 vs. 新系統在實際流量上的行為完全等價
2. **標記慣例**：`TODO(port)`、`FIXME(port)`、`PERF(port)`、`SAFETY:` 標記維持延遲決策與已知問題的可追蹤性
3. **多角色程式碼審查**：結構化 5 角色審查（Rust 專家、安全工程師、架構師、效能工程師、程式碼品質）捕捉跨領域問題
4. **風險分階**：依風險等級分類模組（加密/認證 → 高；感測器聚合 → 中；日誌 → 低）指導上線順序

## 適合誰看？

- **軟體架構師**：評估 Legacy-to-Rust 重寫策略
- **平台工程師**：規劃增量系統遷移
- **技術主管**：設計大型移植項目的治理
- **求職面試者**：展示系統思維、風險管理、跨功能協調能力

## 貢獻 / 疑問

本為靜態設計 portfolio。如有回饋、疑問或架構討論：
- 開啟 issue（如果日後成為公開 repo）
- 聯絡 Repository 擁有者

---

**最後更新**：2026-10-20  
**格式**：Markdown + mermaid 圖表 + 架構決策記錄（ADR）  
**受眾**：技術決策者、架構師、工程主管

---

## Repository 結構

```
aurora-edge-gateway-design/
├── README.md                           ← 英文版（在此）
├── README.zh-TW.md                     ← 繁體中文版
├── docs/
│   ├── 01-problem-and-goals.md
│   ├── 02-architecture-overview.md
│   ├── 03-migration-strategy.md
│   ├── 04-testing-strategy.md
│   ├── 05-review-process.md
│   ├── 06-risk-register-template.md
│   └── adr/                            # 架構決策記錄
│       ├── 0001-why-rewrite-in-rust.md
│       ├── 0002-module-to-crate-decomposition.md
│       ├── 0003-hybrid-runtime-keep-python-subprocess.md
│       ├── 0004-ecosystem-crate-mapping.md
│       ├── 0005-staged-rollout-and-rollback.md
│       ├── 0006-golden-master-testing.md
│       └── 0007-port-marker-convention.md
└── diagrams/
    ├── context-diagram.svg / .png
    ├── module-crate-mapping.svg / .png
    ├── deployment-topology.svg / .png
    └── staged-rollout-timeline.svg / .png
```
