# Testing Strategy: Golden-Master Testing & Verification

## Overview

Testing a major rewrite requires validating not just that the new system works, but that it behaves **identically** to the old system on all observable outputs. This is the purpose of **golden-master** (or **characterization**) testing.

See **[ADR-0006](adr/0006-golden-master-testing.md)** for the decision rationale.

---

## Golden-Master Testing Methodology

### Core Principle
A golden-master test captures the outputs of the **original system** (Node.js) on realistic inputs, then verifies the **new system** (Rust) produces identical outputs on the same inputs.

**Key insight**: We're not testing whether the system is "correct" in an absolute sense; we're testing whether it preserves behavior.

### Process

#### Phase 1: Traffic Capture (2 weeks before Phase A)
1. Instrument the production Node.js gateway with logging hooks:
   - Record every MQTT message received (topic, payload, timestamp)
   - Record every REST API call (endpoint, method, payload, query params, timestamp)
   - Record every database operation (SQL query, parameters, row count affected)
   - Record sensor reading values and timing

2. Run instrumented system in production for 3+ weeks
   - Capture ≥ 1 million MQTT events
   - Capture ≥ 50,000 REST API calls
   - Capture ≥ 10,000 database transactions
   - Ensure traffic includes both normal operations and edge cases

3. Save all captured data to a structured JSON log (golden dataset)

#### Phase 2: Golden Dataset Analysis (Week 1 of Phase A)
1. Replay captured traffic through Node.js system with instrumentation:
   - Capture internal state at key decision points (before/after sensor aggregation, before/after DB writes)
   - Capture all observable outputs (REST responses, MQTT publishes, DB state)
   - Store as "ground truth" reference

2. Bucket traffic by scenario:
   - **Normal operations** (70% of traffic): typical sensor reads, REST queries
   - **Peak load** (15% of traffic): high-frequency MQTT bursts, concurrent REST requests
   - **Error cases** (10% of traffic): invalid inputs, network timeouts, DB errors
   - **Edge cases** (5% of traffic): boundary conditions, race conditions, data anomalies

#### Phase 3: Golden-Master Test Suite (Weeks 2–8 of Phase A)
As each Rust module is ported, test it against golden dataset:

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

## Test Categories & Coverage

### Category 1: Unit Tests (Per-Module, Phase A & B)

**Purpose**: Verify individual functions behave correctly in isolation.

**Examples:**
- `aeg-config` tests: Parse valid/invalid TOML; verify defaults; verify override precedence
- `aeg-db` tests: SQL insert/update/delete; verify transaction semantics; test constraint violations
- `aeg-sensors` tests: Aggregate readings; verify filtering; check numerical accuracy

**Minimum Coverage**: 70% code coverage (excluding error paths that are difficult to trigger)

**Execution**: `cargo test` (runs automatically in CI)

---

### Category 2: Integration Tests (Module Pairs, Phase A)

**Purpose**: Verify modules work correctly together (real I/O, shared state).

**Examples:**
- `aeg-db` + `aeg-sensors`: Write sensor readings to DB; verify correct SQL; verify DB state matches expectations
- `aeg-mqtt-client` + `aeg-sensors`: Inject MQTT messages; verify sensor aggregation; verify outputs
- `aeg-config` + `aeg-appserver`: Load config; verify appserver routes initialized; verify REST endpoints respond

**Minimum Coverage**: All user-facing code paths (REST endpoints, MQTT topics, DB transactions)

**Execution**: `cargo test --test integration_tests` (manual trigger in CI)

---

### Category 3: Golden-Master Tests (Full System, Phase A & B)

**Purpose**: Verify entire system produces identical outputs to Node.js baseline on production-like traffic.

**Scope**: 
- 1+ million MQTT events
- 50,000+ REST API calls
- 10,000+ database transactions
- 3+ weeks of continuous operation captured in golden dataset

**Execution**:
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

**Success Criteria**:
- All REST API responses match golden dataset (status code, response body)
- All MQTT published messages match (topic, payload, ordering, timing ± 5ms)
- All database state matches (rows, column values)
- All error cases handled identically (error message, HTTP status)

**Failure Handling**:
- Any mismatch → immediately halt; investigate root cause; fix code; re-test
- Acceptable tolerance: ± 5ms timestamp differences; ± 0.1% floating-point precision loss

---

### Category 4: Performance & Stress Tests (Phase B)

**Purpose**: Validate performance targets and reliability under extreme conditions.

#### 4a. Load Testing (Peak Throughput)
```bash
# Simulate peak load: 1000 MQTT msg/sec + 100 REST req/sec for 1 hour
load-generator \
  --mqtt-rate 1000 \
  --http-rate 100 \
  --duration 3600s \
  --target localhost:3000

# Measure results
# - HTTP response latency (p50, p95, p99)
# - MQTT message delivery rate (should be 100%)
# - CPU usage (should stay < 50%)
# - Memory usage (should stay < 40MB)
```

**Target Metrics** (Phase B):
- p50 latency: < 2 ms
- p95 latency: < 10 ms
- p99 latency: < 20 ms
- Throughput: ≥ 2000 MQTT msg/sec + 200 REST req/sec without dropping
- Message loss: 0%
- CPU: < 50% on 8-core ARM
- Memory: < 40 MB sustained

#### 4b. Memory Soak Testing (30+ Days)
```bash
# Run continuous load (realistic, not peak) for 30+ days
# Monitor memory usage, CPU usage, and service stability

soak-test-runner \
  --duration 30days \
  --mqtt-rate 200 \
  --http-rate 20 \
  --check-interval 1hour

# Output memory graph and stability report
# Expected: < 0.01% memory growth per day; zero crashes; zero restarts
```

**Success Criteria**:
- Memory growth < 0.01% per day (< 4 KB per day on 40 MB baseline)
- Zero unplanned crashes (only deliberate kills for testing)
- Zero service restarts over 30 days
- All metrics stable (no degradation over time)

#### 4c. Latency Tail Analysis (p99 Stability)
```bash
# Capture latency histogram over 24 hours of peak load
latency-profiler \
  --duration 86400s \
  --bucket-size 1ms \
  --output histogram.json

# Plot histogram
# Expected: p99 latency stable; no spikes; no degradation over time
```

**Success Criteria**:
- p99 latency < 20 ms (consistent)
- No latency spikes > 50 ms (except during GC, which shouldn't happen in Rust)
- Latency stable across different times of day (no scheduling artifacts)

---

### Category 5: Correctness & Verification Tests (Phase B)

#### 5a. Sensor Reading Accuracy
```bash
# Inject synthetic sensor data; verify calculations match baseline

synthetic-sensor-test \
  --test-data synthetic-readings.json \
  --compare-with node.js-baseline \
  --tolerance-pct 0.01  # 0.01% error allowed (floating point)

# Example: If baseline computes VRMS as 230.5V, Rust must compute 230.5±0.1V
```

**Success Criteria**:
- All calculated values match baseline to within tolerance
- No systematic bias (not consistently high or low)

#### 5b. Data Consistency Under Concurrent Writes
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

**Success Criteria**:
- Database passes `PRAGMA integrity_check` after test
- No lost writes (all 100 * 3600 = 360k writes persisted)
- All reads see consistent snapshots (ACID guarantees maintained)

#### 5c. Python Subprocess Bridge
```bash
# Verify correct IPC to Python subprocess; measure overhead

python-bridge-test \
  --test-cases signal-processing,ml-inference \
  --throughput 100calls/sec \
  --duration 3600s

# Expected: 100% success rate; latency < 50ms per call; no process leaks
```

**Success Criteria**:
- All subprocess calls succeed
- Response time < 50 ms (including Python startup time)
- Python processes cleaned up (no zombie processes)
- Subprocess crashes handled gracefully (automatic restart + alert)

---

### Category 6: Chaos & Fault Injection Tests (Phase B)

#### 6a. Network Faults
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

**Expected Behavior**:
- MQTT client automatically reconnects (backoff strategy)
- REST API requests timeout gracefully (with retry logic)
- No permanent loss of connection
- Service continues operating (degraded mode acceptable)

#### 6b. Disk Full
```bash
# Simulate disk full; verify graceful handling

fault-injection-test \
  --fault-type disk-full \
  --reserve-mb 0 \
  --duration 300s

# Expected: Writes fail with proper error; service alerts but doesn't crash; can recover when space freed
```

#### 6c. Database Corruption
```bash
# Inject corrupted database; verify detection and recovery

fault-injection-test \
  --fault-type corrupt-db \
  --corruption-type bad-magic-bytes \
  --duration 60s

# Expected: Service detects corruption; refuses to start; can restore from backup
```

---

## Test Execution Pipeline (CI/CD)

```
Code Commit
    ↓
Tier 1: Compile & Unit Tests (5 min)
    ├─ cargo build
    ├─ cargo test (all unit tests)
    └─ cargo clippy (lint)
    ↓ [on failure: stop; notify developer]
    ↓
Tier 2: Integration Tests (15 min)
    ├─ cargo test --test '*' (all integration tests)
    └─ Basic load test (100 MQTT msg/sec for 1 min)
    ↓ [on failure: stop; notify team]
    ↓
Tier 3: Golden-Master Tests (30 min)
    ├─ Replay 1-week traffic sample against Rust
    ├─ Verify output parity
    └─ Generate diff report
    ↓ [on failure: alert code reviewer; may require manual review]
    ↓
Tier 4: Performance Tests (60 min, only on main branch)
    ├─ 1-hour load test (1000 MQTT msg/sec + 100 HTTP req/sec)
    ├─ Memory soak test (6 hours)
    └─ Latency histogram
    ↓ [on failure: alert performance team]
    ↓
Tier 5: Extended Golden-Master (nightly, 2 hours)
    └─ Replay 3-week production traffic; verify 100% parity
    ↓ [on failure: alert team; investigate; possible rollback needed]
    ↓
Deploy Candidate
```

---

## Golden-Master Test Harness (Pseudocode)

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

## Test Data Management

### Golden Dataset Lifecycle
1. **Capture**: 3+ weeks of production traffic (1+ million events)
2. **Version**: Store in git (or git-lfs for large files); tag with date
3. **Archive**: Keep historical datasets for regression testing
4. **Rotate**: Keep latest 3 datasets; archive older ones

### Privacy & Sanitization
- Remove sensitive data (real IP addresses, usernames, customer-specific values)
- Replace with generic equivalents (192.0.2.0/24 test addresses, "Customer A", etc.)
- Encrypt golden dataset in git (if stored)

---

## Next Steps

1. **Review Process** → See `05-review-process.md` for code review framework
2. **Risk Management** → See `06-risk-register-template.md` for risk taxonomy
3. **Decision Rationale** → See **[ADR-0006](adr/0006-golden-master-testing.md)** for the why and how

---

**Last Updated**: 2026-10-20  
**Status**: Testing strategy approved  
**Next Review**: Week 2 of Phase A (golden dataset analysis complete)
