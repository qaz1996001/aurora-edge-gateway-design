# ADR-0006: Golden-Master Testing for Behavioral Parity

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Engineering team, QA team, architecture

---

## Context

Rewriting a system from scratch carries the risk of subtle behavioral changes: sensor calculations differ by 0.01%, MQTT message timing shifts by 5ms, REST response formats vary slightly. Unit and integration tests won't catch these — they test against expected behavior, not actual behavior.

We need to **validate that the Rust system produces identical outputs to the Node.js system on real-world traffic**, not just on synthetic test cases.

---

## Decision

Implement **golden-master testing** (characterization testing):
1. Capture 3+ weeks of production traffic from Node.js system
2. Record all outputs (REST responses, MQTT publishes, DB state) as "golden" baseline
3. Replay same traffic against Rust system
4. Verify all outputs match exactly
5. Use golden-master test result as mandatory gate for advancing to Stage 1

---

## Rationale

### Why Golden-Master vs. Unit/Integration Tests?

| Test Type | What It Tests | Coverage | Real-World Gaps |
|---|---|---|---|
| Unit Tests | Individual functions in isolation | Narrow | Doesn't catch integration bugs; doesn't test real data patterns |
| Integration Tests | Two modules together | Medium | Still synthetic; doesn't use real traffic patterns |
| Golden-Master Tests | Entire system on real production traffic | Broad | Captures actual use cases; behavioral edge cases |

**Example**: Sensor reading calculation
- **Unit test**: `assert_eq!(aggregate([1.0, 2.0, 3.0]), 2.0)`
- **Golden-master**: Replays the captured corpus of real sensor readings; verifies every output matches the Node.js baseline

---

### Why It's Necessary

1. **Floating-Point Precision**: Different languages/libraries compute `sqrt(2)` slightly differently; accumulates in chains
2. **Edge Cases**: Real data has unusual values (NaN, -0, very large numbers); unlikely in unit tests
3. **Timing Semantics**: Message ordering, timestamp precision, concurrency behavior differs between Node.js event loop and Rust async
4. **State Machine Bugs**: System behavior depends on historical state; real traffic exercises state in ways unit tests don't

---

## Implementation

### Phase 1: Traffic Capture (Weeks 0–2)

Instrument Node.js binary to capture:
```javascript
// Add logging hooks to every major function
app.get('/sensors', (req, res) => {
  LOGGER.info('REST_CALL', {
    method: 'GET',
    path: '/sensors',
    timestamp: Date.now(),
    params: req.query
  });
  // ... existing code ...
  LOGGER.info('REST_RESPONSE', {
    status: res.statusCode,
    body: res.body,
    timestamp: Date.now()
  });
});

mqtt.on('message', (topic, payload) => {
  LOGGER.info('MQTT_RECEIVED', {
    topic,
    payload: payload.toString(),
    timestamp: Date.now()
  });
  // ... existing code ...
});

db.run(sql, (err) => {
  LOGGER.info('DB_WRITE', {
    query: sql,
    affected: this.changes,
    timestamp: Date.now()
  });
  // ... existing code ...
});
```

**Output**: `golden-dataset-week1.json`, `golden-dataset-week2.json`, `golden-dataset-week3.json` (append-only logs)

### Phase 2: Golden Dataset Compilation (Week 2–3)

Merge 3 weeks of logs into single `golden-dataset.json`:
```json
{
  "rest_calls": [
    {
      "id": 1,
      "method": "GET",
      "path": "/sensors",
      "query": {"id": "sensor_1"},
      "expected_status": 200,
      "expected_body": {"id": "sensor_1", "value": 230.5, ...}
    },
    { ... }
  ],
  "mqtt_messages": [
    {
      "id": 1000,
      "topic": "gateway/sensor/reading",
      "payload": {"sensor_id": "s1", "voltage": 230.5},
      "timestamp": 1696416000000
    },
    { ... }
  ],
  "db_writes": [
    {
      "id": 5000,
      "query": "INSERT INTO readings (sensor_id, value) VALUES (?, ?)",
      "params": ["s1", 230.5]
    },
    { ... }
  ]
}
```

**Target corpus** (capture plan, not yet collected):
- ~1.2 million MQTT messages
- ~50,000 REST API calls
- ~10,000 database transactions
- Covers: normal ops (70%), peak load (15%), error cases (10%), edge cases (5%)

### Phase 3: Golden-Master Test Harness (Week 4 of Phase A)

Replay captured traffic against Rust system:

```rust
#[test]
fn golden_master_test() -> Result<()> {
    // Load golden dataset
    let golden: GoldenDataset = serde_json::from_str(&read_file("golden-dataset.json"))?;
    
    // Start Rust system under test
    let mut system = AuroraGateway::start().await?;
    
    // Replay REST calls
    for call in &golden.rest_calls {
        let response = system
            .http_client
            .get(&format!("http://localhost:3000{}", call.path))
            .query(&call.query)
            .send()
            .await?;
        
        assert_eq!(response.status(), call.expected_status, 
                   "Status mismatch for call {}", call.id);
        
        let body: serde_json::Value = response.json().await?;
        assert_json_eq!(body, call.expected_body,
                        "Response body mismatch for call {}", call.id);
    }
    
    // Replay MQTT messages
    for msg in &golden.mqtt_messages {
        system
            .mqtt_client
            .publish(msg.topic.clone(), msg.payload.clone())
            .await?;
        
        // Wait for processing + allow system to respond
        tokio::time::sleep(Duration::from_millis(50)).await;
        
        // Verify any resulting outputs match golden
        let outputs = system.capture_outputs();
        assert_eq!(outputs, msg.expected_outputs,
                   "Output mismatch for MQTT message {}", msg.id);
    }
    
    // Verify final database state matches golden
    system.verify_db_state(&golden.expected_db_state)?;
    
    Ok(())
}
```

**Execution**: `cargo test golden_master_test -- --ignored --nocapture`

**Output**: Pass/fail with detailed diffs for any mismatches

### Phase 4: Failure Resolution (Week 4–8 of Phase A)

If golden-master test fails:

1. **Identify mismatch** (e.g., sensor reading off by 0.1%)
2. **Debug in isolation** (write focused unit test for the specific calculation)
3. **Fix Rust code** (match Node.js behavior exactly)
4. **Re-run golden-master** (confirm fix doesn't break other outputs)
5. **Iterate** until 100% parity achieved

---

## Test Buckets & Incremental Validation

Don't wait for all 3 weeks of traffic to be captured; validate incrementally:

### Week 1 of Phase A
- Use Week 1 golden data (1/3 of dataset)
- Run golden-master test
- Fix low-hanging bugs

### Week 2 of Phase A
- Add Week 2 golden data (2/3 of dataset)
- Re-run golden-master test
- Fix remaining bugs

### Week 3 of Phase A
- Add Week 3 golden data (full dataset)
- Run full golden-master test
- Achieve 100% parity

---

## Acceptable Tolerance Ranges

Not all outputs can be identical (systems have inherent variations):

| Metric | Tolerance | Rationale |
|---|---|---|
| HTTP status code | 0% (must match exactly) | No ambiguity |
| REST response JSON | 0% (must match exactly) | Contracts are strict |
| MQTT message ordering | 0% (must match exactly) | Order matters for state |
| MQTT message content | ±0.1% (floating-point precision) | Inevitable in cross-language |
| Timestamps | ±5 ms | Clock skew acceptable |
| Response latency | Not tested (varies by system load) | Irrelevant for correctness |
| Database state | 0% (must match exactly) | ACID guarantees required |

---

## CI/CD Integration

### Pre-Merge (Code Review)
- Run against Week 1 golden data (fast feedback)
- Must pass before PR approval

### Post-Merge (Main Branch)
- Run against Week 1 + Week 2 + Week 3 golden data (full suite; 30 min)
- Failure = rollback PR immediately

### Nightly (Extended)
- Run golden-master + extended load tests
- Identify performance regressions
- Alert if latency percentiles worsen

---

## Maintenance & Evolution

### Golden Dataset Rotation
- **Quarterly**: Capture fresh 3-week golden dataset (customer behavior evolves)
- **Archive**: Keep historical datasets for regression testing
- **Sanitization**: Remove sensitive customer data before storing

### Golden-Master Coverage Gaps
If a customer reports a new edge case not in golden dataset:
1. Capture that specific scenario
2. Add to golden-master test
3. Ensure Rust system handles it correctly

---

## Validation Against Real Data

### Post-Phase A Validation
Run the Rust system **alongside Node.js on the test fleet**, replaying captured
traffic rather than splitting live traffic:
- Deploy Rust binaries to the test fleet (not production)
- Feed both stacks the same captured corpus and compare outputs
- If outputs diverge: alert; investigate; fix

Note: this is shadow comparison, not a percentage-based canary. Production
cutover is a binary switch with a manual rollback SOP — see
[ADR-0005](0005-staged-rollout-and-rollback.md).

---

## Monitoring & Observability

Each run generates a detailed mismatch report:
```json
{
  "passed": 1234567,
  "failed": 0,
  "summary": "✅ Golden-master test PASSED"
}
```

On failure:
```json
{
  "passed": 1234000,
  "failed": 567,
  "failures": [
    {
      "id": 42,
      "test_type": "rest_call",
      "expected": {"status": 200, "body": {...}},
      "actual": {"status": 200, "body": {...}},  // note subtle difference
      "diff": "response.sensors[0].voltage: expected 230.5, got 230.4"
    }
  ]
}
```

---

## Related Decisions
- **[ADR-0004](0004-ecosystem-crate-mapping.md)**: Technology choices that affect numerical behavior
- **Testing Strategy**: See `../04-testing-strategy.md` for full testing hierarchy

---

## Success Metrics

- ✓ 100% output parity across the full captured corpus
- ✓ Latency within ±5ms of Node.js baseline
- ✓ Zero silent correctness regressions discovered in Stage 1+
- ✓ Team confidence: "We know the Rust system behaves identically to Node.js"

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Engineering team, QA team  
**Implementation**: Traffic capture starts week 0; golden-master test runs weeks 4–8 of Phase A
