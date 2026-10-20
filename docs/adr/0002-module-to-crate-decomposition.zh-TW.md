# ADR-0002：模組 → Crate 分解

**狀態**：已接受  
**決策日期**：2026-Q3  
**利益相關者**：架構團隊、平台工程師

---

## 背景

原始 Node.js 實施已使用進程管理器來管理多個獨立的 Node.js 與 Python 進程，提供基本的進程級故障隔離。不過，個別進程仍面臨限制：
- **個別進程的記憶體洩漏**：事件處理中的循環參照導致堆積逐漸增長；模組邊界上無編譯期安全保證
- **進程內 GC 暫停**：垃圾收集頻繁運行；高峰負載期間不可預測的 stop-the-world 延遲
- **IPC 邊界上無型別安全**：模組邊界透過慣例強制執行，而非編譯器；進程間通訊中常見運行時錯誤
- **手動進程協調**：進程間依賴手動追蹤；無聲明式服務管理；重啟順序容易出錯

Rust 重寫的機會：將既有的多進程架構分解成更清晰的架構，其中每個邏輯領域（感測器、LED 控制、OTA 更新等）都成為自己的 systemd 管理服務，且服務邊界上有編譯期型別保證。這用語言級安全加強了既有的進程隔離。

---

## 決策

**將既有 Node.js 進程邊界對應至獨立的 Rust crate**，每個 crate 編譯成自己的 systemd 管理二進制檔（用戶面向時）或共享庫（內部抽象）。用編譯期型別安全和更清晰的領域邊界強化對應。

**分解原則：**
1. **一個邏輯模組 → 一個 Rust crate**（例如，感測器聚合 → `aeg-sensors` crate）
2. **獨立部署**：每個 crate/二進制檔獨立重啟（透過 systemd）
3. **明確依賴**：依賴圖無環；基於層級組織
4. **鬆散耦合**：服務透過 Unix socket 或 TCP 通訊；無共享記憶體

---

## 理由

### 為何分解？

**故障隔離與編譯期安全**
- LED 控制崩潰不會殺死感測器聚合
- OTA 韌體更新可獨立重啟而不涉及資料庫
- 服務邊界由編譯期驗證（型別系統強制正確性）；運行時 IPC 錯誤早期被捕捉
- **結果**：系統優雅降級；既有進程級隔離透過編譯期型別安全得到強化

**獨立擴展**
- 每個服務可水平擴展（在多核 ARM 上執行多個實例）
- 具有不同資源需求的服務不會互相餓死
- **結果**：有效率的資源使用；無單點瓶頸

**運營清晰度**
- `systemctl status aeg-sensors` 顯示感測器服務是否健康
- 日誌按服務區分（更容易除錯問題）
- 每個服務有自己的重啟政策（某些關鍵、某些盡力而為）
- **結果**：更簡單的故障排除；更快的事件回應

**開發速度**
- 團隊可並行在不同 crate 上工作（較少合併衝突）
- 一個團隊可優化感測器聚合而不破壞 appserver 路由
- 測試按 crate（更快反饋迴圈）
- **結果**：crate 邊界讓並行開發與獨立測試在搬遷期間更容易進行

### Crate 組織（依賴層級）

```
Tier 0: 資料結構（無依賴）
  └─ aeg-config（TOML 解析、零 I/O）

Tier 1: 基礎（最小依賴）
  ├─ aeg-db（SQLite 包裝函式庫）
  └─ aeg-file-manager（檔案 I/O）

Tier 2: 通訊協議用戶端（基礎僅限）
  ├─ aeg-mqtt-client（MQTT 原語）
  └─ aeg-python-bridge（Python IPC）

Tier 3: 領域邏輯（Tier 0–2 僅限）
  ├─ aeg-sensors（聚合）
  ├─ aeg-metrics（訊號 RMS 聚合）
  ├─ aeg-led（GPIO 控制）
  └─ aeg-ota（韌體更新）

Tier 4: 公開介面（所有層級）
  ├─ aeg-appserver（REST API）
  └─ aeg-main（systemd 整合）
```

**優點**：Tier 0 可獨立測試；Tier 1 新增最小依賴；Tier 3 無法依賴 Tier 4。

---

## 結果

### 正面
1. **故障隔離**：一個服務的崩潰不級聯
2. **並行開發**：團隊獨立工作；較少合併衝突
3. **快速測試**：每個 crate 有專注測試套件
4. **運營可見性**：每個服務 `systemctl` 與日誌
5. **漸進優化**：可優化一個服務而不涉及其他

### 負面
1. **IPC 開銷增加**：程序間通訊（Unix socket、TCP）比程序內函式呼叫慢
   - **緩解**：使用有效率序列化（bincode、MessagePack）；在可能處批量訊息
2. **配置複雜性**：每個服務需自己的配置檔；環境設定更複雜
   - **緩解**：共享配置慣例；範本 systemd 單位
3. **測試複雜性**：整合測試必須啟動多個服務
   - **緩解**：Docker 或 systemd 測試固件用於多服務測試
4. **運營開銷**：更多服務要監控與重啟
   - **緩解**：Systemd 健康檢查與自動重啟政策

### 接受的取捨
- **輕微 IPC 延遲開銷**（每個服務間呼叫 0.1–1 ms）換取**故障隔離與運營清晰度**
- **更複雜部署**（實際分解結果為 25 個 crate、19 個 binary、14 個 systemd 受監督服務）換取**獨立服務管理**

---

## 實施細節

### 服務二進制檔
服務部署為獨立 systemd 單位：

```ini
# /etc/systemd/system/aeg-appserver.service
[Unit]
Description=Aurora Edge Gateway - REST API Server
After=aeg-db.service

[Service]
Type=simple
ExecStart=/usr/local/bin/aeg-appserver --config /etc/aeg/appserver.toml
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### 共享庫（內部）
內部 crate（非用戶面向）編譯為庫並連結至二進制檔：

- `aeg-config`：嵌入每個二進制檔；共享依賴
- `aeg-db`：嵌入每個需資料庫存取的二進制檔
- `aeg-mqtt-client`：嵌入至 MQTT 依賴二進制檔

### 服務間通訊
- **SQLite**：多個服務透過 rusqlite 讀寫；ACID 保證確保一致性
- **事件通知**：Unix 域 socket 用於低延遲通知（感測器更新 → LED 應更新）
- **REST API**：HTTP 用於高階用戶端通訊（每個服務公開端口；負載平衡器或 localhost 路由）

### 服務交互範例

```
REST 客戶端 → HTTP GET /sensors → aeg-appserver:3000
                                      ↓（從共享 DB 讀取）
                                   aeg-db.service
                                      ↓（查詢 SQLite）
                                   /var/lib/aeg/data.db

MQTT Broker → MQTT 訊息 → aeg-mqtt:1883
                            ↓（解析訊息）
                       (透過 Unix socket 通知)
                            ↓
                     aeg-sensors:9000
                            ↓（聚合；寫至 DB）
                        aeg-db.service
                            ↓
                     /var/lib/aeg/data.db
                            ↓（通知 LED 服務）
                            ↓
                     aeg-led:8000
                            ↓（更新 GPIO）
                      [RGB LED 閃爍]
```

---

## 相關決策
- **[ADR-0003](0003-hybrid-runtime-keep-python-subprocess.md)**：Python subprocess 如何整合
- **[ADR-0005](0005-staged-rollout-and-rollback.md)**：獨立服務重啟策略
- **架構**：參見 `../02-architecture-overview.md` 了解完整依賴圖

---

## 監控與可觀察性

每個服務記錄至 systemd journal：
```bash
# 查看所有 Aurora Edge Gateway 日誌
journalctl -u aeg-*.service

# 僅查看感測器服務
journalctl -u aeg-sensors.service -f

# 含時間戳與完整細節查看
journalctl -u aeg-appserver.service -o verbose
```

每個服務的健康檢查：
```bash
# appserver 是否回應？
curl localhost:3000/health

# 感測器聚合是否工作？
curl localhost:9000/health

# 整體系統健康
systemctl status aeg-*.service
```

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：架構團隊  
**實施**：Phase A（第 2 週前所有 crate 建立）
