# 審查流程：多角色結構化程式碼審查

## 概覽

大型重寫的程式碼審查必須捕捉跨領域問題（安全性、效能、記憶體安全、架構一致性），這些問題不是單一審查者能全部顧到的。本文檔描述一個**多角色審查框架**，每個審查者有特定的視角和責任。

參見 **[ADR-0007](adr/0007-port-marker-convention.zh-TW.md)** 了解審查過程中使用的標記慣例。

---

## 審查角色與責任

### 角色 1：Rust 專家（語言與最佳實踐）

**責任**：確保慣用 Rust；捕捉記憶體安全和生命週期問題；驗證型別安全保證。

**檢查清單**：
- [ ] 程式碼使用 Rust 慣例（無 C 風格迴圈、手動生命週期管理、unsafe 區塊）
- [ ] 所有 `unsafe` 區塊都有正當理由，並以 `SAFETY:` 註釋標註
- [ ] 生產程式碼路徑中無 unwrap()/panic!()（使用 `?` 運算子或明確的錯誤處理）
- [ ] 使用正確的錯誤型別（已知具體錯誤時不用 `String`，不用泛用的 `anyhow::Error`）
- [ ] 生命週期正確；無不必要的 `.clone()` 或 `.to_string()`
- [ ] 泛型程式碼真正是泛型（不是為單一型別硬寫）
- [ ] 編譯器警告已處理（不會出現意外的 `#![allow(...)]`）
- [ ] 依賴版本已鎖定且有正當理由（無通配符版本）

**審查評論範例**：
```
🚨 Line 42: `user.posts.clone()` 產生不必要的深層複製。
   考慮回傳 `&[Post]` 以避免配置。
   
   fn get_user_posts(user: &User) -> Vec<Post> {  // ❌
   fn get_user_posts(user: &User) -> &[Post] {   // ✅
```

---

### 角色 2：安全工程師

**責任**：識別安全漏洞（injection、buffer overflow、密碼學濫用、權限提升）。

**檢查清單**：
- [ ] 所有使用者輸入在使用前都經過驗證（REST payload、MQTT topic、檔案路徑）
- [ ] 無 SQL injection（使用參數化查詢，而非字串拼接）
- [ ] 無命令 injection（不會把使用者輸入傳給 shell）
- [ ] 密碼學操作使用標準函式庫（非自訂加密）
- [ ] 祕密未硬編碼（程式碼中無 API 金鑰、密碼、私鑰）
- [ ] 權限處理正確（服務以最小權限運行；無預期外的 sudo）
- [ ] 反序列化安全（JSON/TOML 解析不會導致任意程式碼執行）
- [ ] 檔案操作安全（無路徑遍歷；存取前驗證路徑）

**審查評論範例**：
```
⚠️ Line 156: SQL 查詢使用字串拼接。
   SELECT * FROM sensors WHERE id = " + input_id
   
   若 input_id 不可信，這會導致 SQL injection。
   改用參數化查詢：
   
   let query = rusqlite::params![input_id];
   db.query("SELECT * FROM sensors WHERE id = ?1", query)?;
```

---

### 角色 3：系統架構師

**責任**：驗證架構一致性；確保模組邊界受到尊重；檢查跨領域問題。

**檢查清單**：
- [ ] 模組責任清晰；每個 crate 只負責單一關切點
- [ ] 依賴遵循無環依賴圖（Tier 0 → 4）
- [ ] 服務間通訊使用已記錄的協議（Unix socket、TCP port、格式）
- [ ] 服務發現／服務定位器模式明確（無魔法）
- [ ] 並行模型清晰（tokio tasks、threads、locks、channels）
- [ ] 非同步/等待邊界明確（async 上下文中無阻塞呼叫）
- [ ] 資料擁有權與傳遞方式清晰（move vs. borrow vs. clone）
- [ ] 錯誤傳播一致（Result 型別、錯誤鏈）

**審查評論範例**：
```
🏗️ Line 78: 非同步函式內出現 `blocking::spawn_blocking()`。
   
   這是個警示信號。如果在做阻塞式 I/O，我們應該：
   1. 移到同步函式，或
   2. 使用非同步 I/O（tokio::fs、tokio::net），或
   3. 使用 tokio::task::block_in_place() 並附上清楚的說明文件
   
   阻塞操作是什麼？能用非同步嗎？
```

---

### 角色 4：效能工程師

**責任**：識別效能瓶頸；捕捉配置/GC 問題；驗證延遲目標。

**檢查清單**：
- [ ] 熱路徑使用高效資料結構（不應使用 Vec 時用 array/slice 即可的場合用 Vec）
- [ ] 迴圈中無不必要的配置（`vec![]`、`String::from()`）
- [ ] 複製/複製最小化（優先使用借用；適當時使用 move 語義）
- [ ] 鎖競爭最小化（細粒度鎖定；無迴圈中鎖定）
- [ ] 緩衝策略適當（無上限的佇列會導致記憶體增長）
- [ ] 適用時使用零複製模式（大檔案用 MMap、byte slices）
- [ ] 熱路徑函式已設有效能剖析掛鉤（tracing、計時器）

**審查評論範例**：
```
⚡ Line 203: 每條 MQTT 訊息都建立一個 Vec<Message>。
   
   for msg in mqtt_stream {  // 每秒 100+ 條訊息
       let mut response = Vec::new();
       response.push(msg);
       // ... 更多工作 ...
   }
   
   這會為每條訊息配置一個 Vec；GC 壓力高。
   
   解法：使用迭代器組合子或重複使用緩衝區：
   let mut buffer = Vec::with_capacity(100);
   for msg in mqtt_stream {
       buffer.clear();
       buffer.push(msg);
       // ... 重複使用緩衝區 ...
   }
```

---

### 角色 5：程式碼審查者（測試與可維護性）

**責任**：確保程式碼可測試、可維護、可讀；驗證測試涵蓋率。

**檢查清單**：
- [ ] 程式碼可測試（無難以模擬的依賴；適當時使用依賴注入）
- [ ] 所有邏輯都有單元測試（70%+ 涵蓋率）
- [ ] 測試涵蓋 happy path 與錯誤情況
- [ ] 測試名稱具描述性（不是 `test_1`、`test_2`）
- [ ] 註釋解釋「為什麼」，而非「做什麼」（程式碼本身可讀；註釋補充洞見）
- [ ] 無死程式碼（移除未使用的變數、函式、分支）
- [ ] 命名清晰（函式名為動詞；變數名描述意圖）
- [ ] 程式碼重複最小化（DRY 原則）

**審查評論範例**：
```
📝 Line 45: 函式 `process_sensor_data()` 做了 3 件事。
   
   1. 驗證輸入
   2. 查詢資料庫
   3. 更新 LED 狀態
   
   難以獨立測試；難以理解；難以變更。
   
   解法：拆成各自單一責任的較小函式：
   - fn validate_sensor_data(data: &SensorData) -> Result<()>
   - fn fetch_sensor_config(sensor_id: u32) -> Result<Config>
   - fn update_led_state(color: RGB) -> Result<()>
```

---

## 審查工作流程

### 步驟 1：自我審查（開發者）
在提交審查前，開發者依照全部 5 份檢查清單自我審查：
- [ ] Rust 專家檢查清單（慣用；安全）
- [ ] 安全檢查清單（無 injection；無祕密外洩）
- [ ] 架構師檢查清單（模組邊界；非同步正確性）
- [ ] 效能檢查清單（高效；無瓶頸）
- [ ] 程式碼審查者檢查清單（可測試；清晰）

**輸出**：自我審查清單附加到 PR 或合併請求。

### 步驟 2：非同步審查（5 位審查者平行進行）
1. **Rust 專家** → 審查語言/慣例（24 小時周轉）
2. **安全工程師** → 審查漏洞（24 小時周轉）
3. **系統架構師** → 審查架構一致性（24 小時周轉）
4. **效能工程師** → 審查瓶頸（48 小時周轉，可能需要效能剖析）
5. **程式碼審查者** → 審查測試與可維護性（24 小時周轉）

**期望**：全部 5 位審查者平行（而非依序）提供回饋。

**輸出**：每位審查者提供：
- 針對特定行的評論（含行號）
- 嚴重性等級（🚨 必須修正；⚠️ 應該修正；💡 最好修正）
- 可實行的修正建議

### 步驟 3：開發者回應回饋
開發者檢視所有回饋並進行變更：
- **🚨 必須修正**：所有阻擋性問題都已解決
- **⚠️ 應該修正**：修正，或以明確理由記錄下來
- **💡 最好修正**：簡單的話就修；複雜的話延遲到 Phase B，並以 `TODO(port)` 標記

**輸出**：更新後的 PR，並對所有評論給出回應（例如「已在 commit abc123 修正」）。

### 步驟 4：驗證審查（抽查）
一位審查者（輪流擔任）抽查確認回饋確實已被處理：
- [ ] 🚨 問題已解決
- [ ] ⚠️ 問題已解決或已記錄
- [ ] 程式碼變更符合建議的修正方式

**輸出**：核准，或要求進一步修改。

### 步驟 5：合併與迴歸測試
- PR 合併到 main 分支
- CI/CD 管道執行（Tier 1–3 測試；見測試策略文件）
- 若測試失敗：調查；修正；重新測試
- 若測試通過：程式碼已準備好進入 Phase A 整合

---

## 審查檢查清單範本

```markdown
## Code Review Checklist

### Rust Expert Review
- [ ] Code uses Rust idioms (no C-style loops)
- [ ] All `unsafe` blocks justified with SAFETY: comment
- [ ] No panics in production code
- [ ] Proper error types used (not String)
- [ ] No unnecessary clones or to_string()
- [ ] Generic code is truly generic
- [ ] Compiler warnings addressed

### Security Review
- [ ] User input validated before use
- [ ] No SQL injection (parameterized queries)
- [ ] No command injection
- [ ] Secrets not hardcoded
- [ ] File operations safe (no path traversal)
- [ ] Cryptography uses standard libraries

### Architecture Review
- [ ] Module boundaries respected
- [ ] Dependencies follow DAG (no cycles)
- [ ] Inter-service communication documented
- [ ] Concurrency model clear
- [ ] Async/await boundaries explicit
- [ ] Error propagation consistent

### Performance Review
- [ ] Hot paths efficient
- [ ] No unnecessary allocations
- [ ] Copying/cloning minimized
- [ ] Lock contention low
- [ ] Buffering strategy appropriate

### Code Quality Review
- [ ] Code is testable
- [ ] Unit tests present (70%+ coverage)
- [ ] Tests cover error cases
- [ ] Comments explain why, not what
- [ ] No dead code
- [ ] Naming is clear

---

**Reviewed By**: [Name], [Date]
**Approval**: ✅ Approved / ⏳ Needs Changes / ❌ Rejected
```

---

## 持續回饋循環

### 每週程式碼審查同步會議
- **週二下午 2 點**：檢視回饋趨勢（常見問題、模式）
- **討論**：哪些錯誤反覆出現？哪些模式應該被強制要求？
- **輸出**：優化團隊程式碼規範或審查檢查清單

### Phase A 後回顧
- 檢視 Phase A 期間收集的所有回饋
- 找出最常見的前 10 大問題
- 更新團隊文件與培訓內容
- 為 Phase B 調整審查檢查清單

---

## 工具與自動化

### 自動化檢查項目
以下項目可由 CI/CD 自動檢查，無需人工審查：
- `cargo fmt`（程式碼格式化）
- `cargo clippy`（linting）
- `cargo test`（單元測試）
- 編譯器警告（必須為零）
- 程式碼涵蓋率（必須 ≥ 70%）
- 依賴稽核（`cargo audit` — 無已知漏洞）
- SAFETY 註釋是否存在（正規表示式檢查：所有 `unsafe {}` 區塊都必須有 SAFETY 註釋）

### GitHub Actions Workflow 範例
```yaml
name: Code Review Automation
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: cargo test --all-features
      - run: cargo tarpaulin --out Xml  # Code coverage
      - uses: codecov/codecov-action@v2
  
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: cargo audit
  
  safety-comments:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: |
          unsafe_blocks=$(grep -r "unsafe {" src/ | wc -l)
          safety_comments=$(grep -r "SAFETY:" src/ | wc -l)
          if [ $unsafe_blocks -ne $safety_comments ]; then
            echo "❌ Not all unsafe blocks have SAFETY comments"
            exit 1
          fi
```

---

## 審查 SLA（服務等級協議）

| 角色 | 周轉時間 | 範例情境 |
|---|---|---|
| Rust 專家 | 24 小時 | 週一提交程式碼；週二結束前給出回饋 |
| 安全工程師 | 24 小時 | 程式碼審查；安全問題於 24 小時內列出 |
| 架構師 | 24 小時 | 若上午 10 點前提交，當天即可得到架構回饋 |
| 效能工程師 | 48 小時 | 可能需要效能剖析；不阻擋合併（記錄為 Phase B 優化項目） |
| 程式碼審查者 | 24 小時 | 當天即可得到可測試性回饋 |

---

## 下一步

1. **風險管理** → 見 `06-risk-register-template.zh-TW.md` 了解如何分類與追蹤風險
2. **標記慣例的 ADR** → 參見 **[ADR-0007](adr/0007-port-marker-convention.zh-TW.md)** 了解 `TODO(port)` / `FIXME(port)` 標準

---

**最後更新**：2026-10-20  
**狀態**：審查流程已批准  
**下次審查**：Phase A 第 2 週（首批 PR 提交審查時）
