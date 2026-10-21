# ADR-0007：搬遷標記慣例

**狀態**：已接受  
**決策日期**：2026-Q3  
**利益相關者**：工程團隊、程式碼審查主管

---

## 背景

於 Phase A（忠實搬遷）期間，決策經常延遲：
- 「應該重構此錯誤處理嗎？」→ 延遲至 Phase B
- 「此邏輯來自 Node.js 的已知漏洞；應該修正嗎？」→ 不；暫時保留行為
- 「此迴圈效率低。」→ 我們知道；Phase B 時優化

沒有一致方法標記這些延遲，程式碼變得難以理解，關鍵問題被遺忘。

---

## 決策

採用標準化**標記慣例**用於搬遷期間追蹤延遲工作：

| 標記 | 用途 | 持續時間 | 範例 |
|---|---|---|---|
| `TODO(port)` | 延遲工作；計畫於 Phase B | Phase A → Phase B | 重構、優化 |
| `FIXME(port)` | 來自 Node.js 的已知漏洞複製；將於 Phase B 修正 | Phase A → Phase B | 邏輯漏洞、怪癖保留相容性 |
| `PERF(port)` | 效能機會延遲 | Phase A → Phase B | 低效演算法、不必要配置 |
| `SAFETY:` | 記憶體安全稽核標記 | Phase A → 現在 | `unsafe` 區塊理由 |

---

## 標記細節

### TODO(port)：延遲工作

於某些應改進但延遲至 Phase B 時使用。

**格式**：
```rust
// TODO(port): Refactor to use Result instead of panic
panic!("Database write failed");
```

**範圍**：Phase A → Phase B 轉換

**追蹤**：
- CI 閘門：`grep -r "TODO(port)" . | wc -l` 必須週週減少
- Phase B 目標：零剩餘 `TODO(port)` 項目
- Phase B 之前，若未處理則轉換為專案追蹤系統中的正式議題（issue）

---

### FIXME(port)：複製漏洞

於複製來自 Node.js 的已知漏洞（行為對等）、計畫於 Phase B 修正時使用。

**格式**：
```rust
// FIXME(port): RISK-003 — sensor reads drop 0.1% under peak load
// This bug exists in Node.js; we replicate it for parity. Fix in Phase B.
if event_queue.len() > MAX_QUEUE {
    events.pop();  // Silent drop (bug from Node.js); should alert instead
}
```

**範圍**：Phase A → Phase B 轉換

**追蹤**：
- 連結至風險登記（例如，`RISK-003`）
- CI 閘門：每個 `FIXME(port)` 必須有對應風險項目
- Phase B 目標：每個 `FIXME(port)` → 已修正，或重新分類為較低優先級

**範例**：
```rust
// FIXME(port): RISK-007 — OTA timeout on slow network
// Node.js waits indefinitely; we replicate this.
// Phase B: Implement adaptive timeout + exponential backoff
firmware_download().await  // No timeout set (intentional bug replication)
```

---

### PERF(port)：效能機會

於識別效能優化但延遲時使用（Phase A 優先正確性）。

**格式**：
```rust
// PERF(port): Use circular buffer instead of Vec for event queue
// Current: allocates new Vec for each batch
let mut events = Vec::new();
for msg in mqtt_stream {
    events.push(msg);
    if events.len() > 1000 {
        process_batch(&events);
        events.clear();
    }
}
```

**範圍**：Phase A → Phase B

**追蹤**：
- 非阻擋（程式碼正確，只是慢）
- Phase B 目標：解決高衝擊 PERF 項目（每項 > 1ms 延遲）
- 低衝擊項目可保留（可接受的技術債）

---

### SAFETY：記憶體安全稽核標記

用於為每個 `unsafe` 區塊正當化。信號此程式碼已明確稽核記憶體安全。

**格式**：
```rust
// SAFETY: Buffer is borrowed from a valid Vec; length is checked before access.
// No data race possible: this function runs in a tokio task; no other task
// can modify the buffer while we hold the reference.
unsafe {
    let ptr = buffer.as_ptr() as *mut u8;
    ptr.write_bytes(0, buffer.len());  // Zero-out sensitive data
}
```

**範圍**：Phase A → 永遠（非延遲）

**需求**：
- 每個 `unsafe {}` 區塊必須有 `SAFETY:` 註釋
- 註釋必須解釋：為何需要 unsafe、維持哪些不變數、為何安全
- CI 閘門：所有 unsafe 區塊必須有 SAFETY 註釋（零容忍）

**範例**：

```rust
// SAFETY: We own this Vec; freeing it is guaranteed safe. Drop trait ensures cleanup.
// No data races possible because we're in single-threaded context.
unsafe {
    let _ = Vec::<u8>::from_raw_parts(ptr, len, capacity);
}

// SAFETY: FFI call to MQTT library. Function signature guaranteed to not
// modify pointers passed to it. Input validated before call (not null, aligned).
unsafe {
    mqtt_publish(topic_ptr, payload_ptr, payload_len);
}
```

**反例**（不良）：
```rust
// Bad: No explanation; why is this safe?
unsafe { *(ptr as *mut u8) = 42; }
```

---

## CI/CD 實施

### 預先合併閘門：安全註釋檢查

```bash
# Check: All unsafe blocks have SAFETY comments
unsafe_count=$(grep -r "unsafe {" src/ | wc -l)
safety_count=$(grep -r "SAFETY:" src/ | wc -l)

if [ $unsafe_count -ne $safety_count ]; then
  echo "❌ Error: $unsafe_count unsafe blocks but only $safety_count SAFETY comments"
  echo "Every unsafe block must have a SAFETY: comment explaining why it's safe"
  exit 1
fi
```

### 週度標記普查

```bash
#!/bin/bash
# Generate weekly report on marker status

echo "=== Phase A Marker Census ==="
echo ""
echo "TODO(port) items:"
grep -r "TODO(port)" src/ | wc -l
echo ""
echo "FIXME(port) items (should link to Risk Register):"
grep -r "FIXME(port)" src/ | wc -l
echo ""
echo "PERF(port) items:"
grep -r "PERF(port)" src/ | wc -l
echo ""
echo "SAFETY comments:"
grep -r "SAFETY:" src/ | wc -l
echo ""
echo "Unsafe blocks:"
grep -r "unsafe {" src/ | wc -l

# Check: FIXME(port) items must reference a RISK-N
fixme_without_risk=$(grep -r "FIXME(port)" src/ | grep -v "RISK-" | wc -l)
if [ $fixme_without_risk -gt 0 ]; then
  echo ""
  echo "⚠️  Warning: $fixme_without_risk FIXME(port) items don't reference a risk:"
  grep -r "FIXME(port)" src/ | grep -v "RISK-"
fi
```

### Phase 轉換閘門

**Phase B 開始前**：
```
Checklist:
  ☑ All TODO(port) items → either addressed or documented with justification
  ☑ All FIXME(port) items → linked to Risk Register entry
  ☑ All PERF(port) items → assessed for Phase B inclusion
  ☑ All unsafe blocks → have SAFETY comments
  ☑ No new unsafe blocks without SAFETY (enforce in code review)
```

**Phase B 完成前**：
```
Checklist:
  ☑ All FIXME(port) items → either fixed or reclassified
  ☑ All TODO(port) → resolved
  ☑ All high-impact PERF(port) → addressed
  ☑ Marker census run; report filed
```

---

## 程式碼審查：標記期望

### 審查者檢查清單（角色：程式碼審查者）

審查 Phase A 程式碼時：
- [ ] 每個 `unsafe {}` 都有 `SAFETY:` 註釋（核准必要條件）
- [ ] 延遲工作標記為 `TODO(port)` 或 `FIXME(port)`（非含糊的註釋）
- [ ] 每個 `FIXME(port)` 皆連結至風險登記中的一筆風險項目
- [ ] 效能疑慮標記為 `PERF(port)`（非默默留下）
- [ ] 沒有把本應立即處理的工作用標記帶過（所有 `TODO(port)` 都應可延遲至 Phase B）

---

## 標記生命週期範例

```rust
// Week 1 of Phase A: Port database module
impl Database {
    fn open(path: &str) -> Result<Self> {
        // SAFETY: FFI to rusqlite library. Function signature guarantees
        // the returned database handle is valid and thread-safe.
        unsafe {
            let db = sqlite3_open(path.as_ptr(), &mut db_ptr);
            Ok(Database { db })
        }
    }
    
    fn query(&self, sql: &str) -> Result<Vec<Row>> {
        // TODO(port): Replace Vec with streaming iterator
        // Current: allocates all rows upfront; should stream
        let mut rows = Vec::new();
        while let Some(row) = self.next_row() {
            rows.push(row);
        }
        Ok(rows)
    }
    
    fn delete_old_records(&self, days: u32) -> Result<()> {
        // FIXME(port): RISK-010 — Race condition on concurrent deletes
        // This bug exists in Node.js; we replicate for parity.
        // Phase B: Implement row-level locking
        self.execute("DELETE FROM logs WHERE timestamp < ?", &[days])
    }
}
```

**第 4 週（Phase A 完成）**：
- 程式碼審查：已核准（所有標記齊全、SAFETY 已說明理由、FIXME 已連結）
- 風險登記：RISK-010 已確認（需要 Phase B 修正）

**第 10 週（Phase B 進行中）**：
- `TODO(port)` → 已處理；以串流迭代器取代 Vec
- `FIXME(port)` → 已調查；找出競爭條件根因；已實作修正
- `SAFETY:` → 保留（永遠存在）

**第 12 週（Phase B 完成）**：
- 所有標記皆已解決或重新分類
- 程式碼稽核：無剩餘延遲工作

---

## 搜尋與報告

### 找出所有延遲工作

```bash
# List all Phase A deferral markers
grep -r "TODO(port)\|FIXME(port)\|PERF(port)" src/ | sort

# Example output:
# src/sensors/aggregation.rs:42:    // TODO(port): Optimize aggregation loop
# src/ota/updater.rs:105:   // FIXME(port): RISK-007 — handle slow networks
# src/mqtt/client.rs:63:    // PERF(port): Use Arc<Mutex> instead of clone
```

### 連結至風險登記

```bash
# Find FIXME items without RISK reference (potential bug)
grep -r "FIXME(port)" src/ | grep -v "RISK-"
```

### 統計與趨勢

```bash
# Track progress week-by-week
echo "Marker census: $(date)"
echo "TODO(port):  $(grep -r 'TODO(port)' src/ | wc -l)"
echo "FIXME(port): $(grep -r 'FIXME(port)' src/ | wc -l)"
echo "PERF(port):  $(grep -r 'PERF(port)' src/ | wc -l)"
echo "SAFETY:      $(grep -r 'SAFETY:' src/ | wc -l)"
```

---

## 相關決策
- **[ADR-0005](0005-staged-rollout-and-rollback.zh-TW.md)**：標記如何影響推出決策
- **風險管理**：Risk ID 格式請見 `../06-risk-register-template.zh-TW.md`

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：工程團隊、程式碼審查主管  
**實施**：CI/CD 閘門 + 程式碼審查；自 Phase A 第 1 週開始
