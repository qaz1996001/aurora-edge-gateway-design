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
// TODO(port)：重構使用 Result 而非 panic
panic!("Database write failed");
```

**範圍**：Phase A → Phase B 轉換

**追蹤**：
- CI 閘門：`grep -r "TODO(port)" . | wc -l` 必須週週減少
- Phase B 目標：零剩餘 `TODO(port)` 項目

---

### FIXME(port)：複製漏洞

於複製來自 Node.js 的已知漏洞（行為對等）、計畫於 Phase B 修正時使用。

**格式**：
```rust
// FIXME(port)：RISK-003 — 感測器讀數於尖峰負載下掉 0.1%
// 此漏洞存在於 Node.js；我們複製以對等。Phase B 修正。
if event_queue.len() > MAX_QUEUE {
    events.pop();  // 無聲掉落（Node.js 漏洞）；應警告
}
```

**範圍**：Phase A → Phase B 轉換

**追蹤**：
- 連結至風險登記（例如，`RISK-003`）
- CI 閘門：每個 `FIXME(port)` 必須有對應風險項目

---

### PERF(port)：效能機會

於識別效能優化但延遲時使用（Phase A 優先正確性）。

**格式**：
```rust
// PERF(port)：用循環緩衝取代 Vec 用於事件隊列
// 目前：每個批次配置新 Vec
let mut events = Vec::new();
```

**範圍**：Phase A → Phase B

**追蹤**：
- 非阻擋（程式碼正確，只是慢）
- Phase B 目標：解決高衝擊 PERF 項目（> 1ms 延遲各）

---

### SAFETY：記憶體安全稽核標記

用於為每個 `unsafe` 區塊正當化。信號此程式碼已明確稽核記憶體安全。

**格式**：
```rust
// SAFETY：緩衝從有效 Vec 借用；長度於存取前檢查。
// 無資料競爭可能：此函式於 tokio 任務中執行；無其他任務
// 可於我們持有參考時修改緩衝。
unsafe {
    let ptr = buffer.as_ptr() as *mut u8;
    ptr.write_bytes(0, buffer.len());  // 零化敏感資料
}
```

**範圍**：Phase A → 永遠（非延遲）

**需求**：
- 每個 `unsafe {}` 區塊必須有 `SAFETY:` 註釋
- 註釋必須解釋：為何需要 unsafe、維持哪些不變數、為何安全
- CI 閘門：所有 unsafe 區塊必須有 SAFETY 註釋（零容忍）

---

## CI/CD 實施

### 預先合併閘門：安全註釋檢查

```bash
# 檢查：所有 unsafe 區塊有 SAFETY 註釋
unsafe_count=$(grep -r "unsafe {" src/ | wc -l)
safety_count=$(grep -r "SAFETY:" src/ | wc -l)

if [ $unsafe_count -ne $safety_count ]; then
  echo "❌ 錯誤：$unsafe_count unsafe 區塊但僅 $safety_count SAFETY 註釋"
  exit 1
fi
```

### 週度標記普查

```bash
#!/bin/bash
# 生成標記狀態週度報告

echo "=== Phase A 標記普查 ==="
echo "TODO(port) 項目："
grep -r "TODO(port)" src/ | wc -l
echo ""
echo "FIXME(port) 項目（應連結至風險登記）："
grep -r "FIXME(port)" src/ | wc -l
echo ""
echo "PERF(port) 項目："
grep -r "PERF(port)" src/ | wc -l
```

---

## Phase B 轉換

當進入 Phase B 時：

1. 每個 `TODO(port)` → 應被解決或明確推遲至 Phase B+
2. 每個 `FIXME(port)` → 應被解決或重新分類為接受的技術債
3. 每個 `PERF(port)` → 應被解決（若高衝擊）或接受為低優先項目
4. 每個 `SAFETY:` → 移到永遠保存（無時間限制）

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：工程團隊  
**實施**：Phase A 從第 1 週開始（所有新程式碼須遵守）
