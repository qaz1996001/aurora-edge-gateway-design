# ADR-0007: Port Marker Conventions

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Engineering team, code review leads

---

## Context

During Phase A (faithful port), decisions are often deferred:
- "Should we refactor this error handling?" → Defer to Phase B
- "This logic has a known bug from Node.js; should we fix it?" → No; preserve behavior for now
- "This loop is inefficient." → We know; optimize in Phase B

Without a consistent way to mark these deferrals, the code becomes hard to understand and critical issues get forgotten.

---

## Decision

Adopt standardized **marker conventions** for tracking deferred work during the port:

| Marker | Usage | Duration | Example |
|---|---|---|---|
| `TODO(port)` | Deferred work; planned for Phase B | Phase A → Phase B | Refactoring, optimizations |
| `FIXME(port)` | Known bug replicated from Node.js; will fix in Phase B | Phase A → Phase B | Logic bug, quirk preserved for compatibility |
| `PERF(port)` | Performance opportunity deferred | Phase A → Phase B | Inefficient algorithm, unnecessary allocation |
| `SAFETY:` | Memory-safety audit marker | Phase A → present | `unsafe` block justification |

---

## Marker Details

### TODO(port): Deferred Work

Used when something should be improved but is deferred to Phase B.

**Format**:
```rust
// TODO(port): Refactor to use Result instead of panic
panic!("Database write failed");
```

**Scope**: Phase A → Phase B transition

**Tracking**: 
- CI gate: `grep -r "TODO(port)" . | wc -l` must decrease week-by-week
- Phase B goal: Zero remaining `TODO(port)` items
- Before Phase B, convert to real issues in project tracker if not addressed

---

### FIXME(port): Replicated Bugs

Used when replicating a known bug from Node.js (for behavioral parity), with plan to fix in Phase B.

**Format**:
```rust
// FIXME(port): RISK-003 — sensor reads drop 0.1% under peak load
// This bug exists in Node.js; we replicate it for parity. Fix in Phase B.
if event_queue.len() > MAX_QUEUE {
    events.pop();  // Silent drop (bug from Node.js); should alert instead
}
```

**Scope**: Phase A → Phase B transition

**Tracking**:
- Link to Risk Register (e.g., `RISK-003`)
- CI gate: Each `FIXME(port)` must have corresponding risk entry
- Phase B goal: Every `FIXME(port)` → either fixed or reclassified as lower-priority

**Example**:
```rust
// FIXME(port): RISK-007 — OTA timeout on slow network
// Node.js waits indefinitely; we replicate this.
// Phase B: Implement adaptive timeout + exponential backoff
firmware_download().await  // No timeout set (intentional bug replication)
```

---

### PERF(port): Performance Opportunities

Used when a performance optimization is identified but deferred (Phase A prioritizes correctness).

**Format**:
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

**Scope**: Phase A → Phase B

**Tracking**:
- Not a blocker (code is correct, just slow)
- Phase B goal: Address high-impact PERF items (> 1ms latency each)
- Low-impact items can stay (acceptable technical debt)

---

### SAFETY: Memory-Safety Audit Markers

Used to justify every `unsafe` block in code. Signals that this code has been explicitly audited for memory safety.

**Format**:
```rust
// SAFETY: Buffer is borrowed from a valid Vec; length is checked before access.
// No data race possible: this function runs in a tokio task; no other task
// can modify the buffer while we hold the reference.
unsafe {
    let ptr = buffer.as_ptr() as *mut u8;
    ptr.write_bytes(0, buffer.len());  // Zero-out sensitive data
}
```

**Scope**: Phase A → Forever (not deferred)

**Requirements**:
- Every `unsafe {}` block MUST have a `SAFETY:` comment
- Comment must explain: why unsafe is needed, what invariants are maintained, why it's safe
- CI gate: All unsafe blocks must have SAFETY comments (zero tolerance)

**Examples**:

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

**Non-Example** (BAD):
```rust
// Bad: No explanation; why is this safe?
unsafe { *(ptr as *mut u8) = 42; }
```

---

## CI/CD Enforcement

### Pre-Merge Gate: Safety Comment Check

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

### Weekly Marker Census

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

### Phase Transition Gates

**Before Phase B Starts**:
```
Checklist:
  ☑ All TODO(port) items → either addressed or documented with justification
  ☑ All FIXME(port) items → linked to Risk Register entry
  ☑ All PERF(port) items → assessed for Phase B inclusion
  ☑ All unsafe blocks → have SAFETY comments
  ☑ No new unsafe blocks without SAFETY (enforce in code review)
```

**Before Phase B Completes**:
```
Checklist:
  ☑ All FIXME(port) items → either fixed or reclassified
  ☑ All TODO(port) → resolved
  ☑ All high-impact PERF(port) → addressed
  ☑ Marker census run; report filed
```

---

## Code Review: Marker Expectations

### Reviewer Checklist (Role: Code Reviewer)

When reviewing Phase A code:
- [ ] Every `unsafe {}` has a `SAFETY:` comment (required for approval)
- [ ] Deferred work marked with `TODO(port)` or `FIXME(port)` (not vague comments)
- [ ] Each `FIXME(port)` references a RISK entry in the Risk Register
- [ ] Performance concerns marked with `PERF(port)` (not silently left)
- [ ] No markers for work that should be done now (all `TODO(port)` should be deferrable to Phase B)

---

## Marker Lifespan Example

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

**Week 4 (Phase A Complete)**:
- Code review: Approved (all markers present, SAFETY justified, FIXME linked)
- Risk Register: RISK-010 confirmed (needs Phase B fix)

**Week 10 (Phase B In Progress)**:
- `TODO(port)` → Addressed; replaced Vec with streaming iterator
- `FIXME(port)` → Investigated; found root cause of race condition; implemented fix
- `SAFETY:` → Remains (always present)

**Week 12 (Phase B Complete)**:
- All markers resolved or reclassified
- Code audit: No deferred work remaining

---

## Searching & Reporting

### Find All Deferred Work

```bash
# List all Phase A deferral markers
grep -r "TODO(port)\|FIXME(port)\|PERF(port)" src/ | sort

# Example output:
# src/sensors/aggregation.rs:42:    // TODO(port): Optimize aggregation loop
# src/ota/updater.rs:105:   // FIXME(port): RISK-007 — handle slow networks
# src/mqtt/client.rs:63:    // PERF(port): Use Arc<Mutex> instead of clone
```

### Link to Risk Register

```bash
# Find FIXME items without RISK reference (potential bug)
grep -r "FIXME(port)" src/ | grep -v "RISK-"
```

### Statistics & Trends

```bash
# Track progress week-by-week
echo "Marker census: $(date)"
echo "TODO(port):  $(grep -r 'TODO(port)' src/ | wc -l)"
echo "FIXME(port): $(grep -r 'FIXME(port)' src/ | wc -l)"
echo "PERF(port):  $(grep -r 'PERF(port)' src/ | wc -l)"
echo "SAFETY:      $(grep -r 'SAFETY:' src/ | wc -l)"
```

---

## Related Decisions
- **[ADR-0005](0005-staged-rollout-and-rollback.md)**: How markers inform rollout decisions
- **Risk Management**: See `../06-risk-register-template.md` for Risk ID format

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Engineering team, code review leads  
**Enforcement**: CI/CD gates + code review; begins week 1 of Phase A
