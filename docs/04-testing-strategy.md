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
- Memory profiling tests: Verify no leaks in drop chains; track allocation sizes

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

#### 4a. Load Testing (Representative & Peak Load)
```bash
# Exercise representative production traffic, then a peak-load profile,
# on the target 4-core ARM Cortex-A53 platform
load-generator \
  --profile representative-production \
  --duration 10800s \
  --target localhost:3000

load-generator \
  --profile peak \
  --duration 10800s \
  --target localhost:3000

# Measure results against the Phase B soak baseline (below)
# - CPU usage (soak baseline: 6.3% avg, 8.8% peak)
# - Memory usage (soak baseline: 206 MB full-stack resident)
# - Load average (soak baseline: 0.50 avg, 1.81 peak, 4-core)
# - MQTT message delivery rate (should be 100%)
```

**Target Metrics** (Phase B, anchored to the measured 7.8h soak baseline on 4-core ARM Cortex-A53):
- CPU: avg 6.3%, peak 8.8%, zero drift over the soak window — compare the Node.js baseline's idle 52% / 3h-active 64% on the same hardware
- Memory: full-stack resident 206 MB across 14 supervised services + Python subprocess — compare the Node.js baseline's 1538 MB active (~54% of 2.84 GB available)
- Load average (4-core): avg 0.50, peak 1.81 — compare the Node.js baseline's 3.30 avg
- Throughput: representative production load and peak load both sustained without dropping messages. We do not have a benchmarked maximum msg/sec ceiling for this platform; `--profile` exercises realistic traffic shapes rather than a specific rate, and establishing a hard throughput limit is tracked as separate follow-up work, not claimed here.
- Message loss: 0%
- Memory growth: no sustained positive RSS slope over the run (see the case study in 4b below for how this is measured and what it caught)

#### 4b. Memory Soak Testing (Measured: 7.8h and 10.6h Runs)
```bash
# Run continuous representative load and monitor memory, CPU, and service stability.
# Longest runs completed to date: 7.8h (initial baseline) and 10.6h (after the
# MALLOC_ARENA_MAX fix described in the case study below).

soak-test-runner \
  --duration 10.6h \
  --profile representative-production \
  --check-interval 1hour

# Output memory graph and stability report
```

**Results — initial 7.8h soak (full stack, 4-core ARM Cortex-A53)**:
- Full-stack resident memory: 206 MB (14 supervised services + Python subprocess)
- CPU: avg 6.3%, peak 8.8%, zero drift over the run
- Load average: avg 0.50, peak 1.81 (4-core)
- System-available memory: 6024 → 5885 MB (−139 MB over 7.8h)
- Restarts: 3 in the first 66 minutes (startup stabilization), 0 over the following 6.7h

**Success Criteria**:
- No sustained positive RSS slope per service over the run (not a fixed %/day figure — see the case study below for the slope methodology actually used)
- CPU stable, no degradation or drift (7.8h soak measured flat at 6.3% avg)
- Zero unplanned crashes (only deliberate kills for testing)
- Zero service restarts after the initial startup-stabilization window (0 over 6.7h in the 7.8h run; 10.6h run passed end-to-end — see case study)
- No GC pauses (Rust has no GC runtime)

#### 4b-i. Case Study: A Real Memory-Growth Bug Found and Fixed During Soak Testing

This is the single most useful result soak testing produced in Phase B, and it is worth documenting in full because it shows what soak testing is actually for.

**What we saw**: In the initial 7.8h soak, the message-bus process's RSS grew from 45.0 MB to 95.3 MB — a +112% increase, climbing at a roughly linear ~6.5 MB/h — while CPU usage for that same process stayed flat at 0.2% the entire time. A linear, CPU-flat growth curve is the signature of an allocator or runtime issue, not a request-driven leak in application logic.

**Root cause**: glibc's multi-threaded allocator was opening up to 64 separate memory arenas for the process. Each arena keeps its own free lists, so a multi-threaded binary under glibc can accumulate retained-but-unused memory across arenas with zero memory-safety bugs in the Rust code itself — this is documented glibc allocator behavior, not a Rust-side defect.

**Fix**: Set `MALLOC_ARENA_MAX=2` in the service's environment to cap the number of arenas glibc creates.

**Verification (10.6h soak, post-fix)**: RSS rose to roughly 24.5 MB, then arena reclamation brought it back down to 17.2 MB. The hourly growth slope across the run: first 2 hours +5.74 MB/h, middle of the run ~0.00 MB/h, last 2 hours −3.69 MB/h — the trend reverses and the process gives memory back by the end. Test passed.

**The insight**: Rewriting in Rust eliminates an entire class of memory-safety bugs — use-after-free, double-free, buffer overflows — at compile time. It does **not** make a process immune to memory growth. Allocator configuration and C runtime behavior sit below the language's safety guarantees, and only a soak test — not a unit test, not a code review — will surface that kind of growth. Soak testing stays a permanent part of the Phase B gate, not a milestone to clear once.

#### 4c. Stability & Determinism Validation (24h Monitoring)
```bash
# Monitor CPU, memory, and load for deterministic behavior over 24 hours.
# Builds on the 7.8h/10.6h soak results above; there is no GC-driven
# scheduling jitter to account for.

stability-monitor \
  --duration 86400s \
  --sample-interval 10s \
  --output stability-report.json

# Expected: CPU/memory/load as flat as observed in the completed soaks
```

**Success Criteria**:
- CPU stable around the measured 6.3% avg / 8.8% peak soak baseline, no sustained drift
- Memory stable at the 206 MB full-stack baseline; any sustained positive slope investigated using the method in the case study above
- Load average stable around the measured 0.50 avg / 1.81 peak baseline (4-core)
- No latency variance from garbage collection (Rust has no GC)
- Consistent performance across different times of day

---

### Category 5: Correctness & Verification Tests (Phase B)

#### 5a. Sensor Reading Accuracy
```bash
# Inject synthetic sensor data; verify calculations match baseline

synthetic-sensor-test \
  --test-data synthetic-readings.json \
  --compare-with node.js-baseline \
  --tolerance-pct 0.01  # 0.01% error allowed (floating point)

# Illustrative example only (not a measured result): if the baseline's
# aeg-metrics signal-RMS aggregation reports a reference value of, say,
# 230.5 for some channel, Rust must reproduce it within the tolerance band
# (230.5 ± 0.1 at the 0.01% setting above) to count as a golden-master match.
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

# Expected: 100% success rate; no process leaks; per-call latency recorded as a baseline
```

**Success Criteria**:
- All subprocess calls succeed
- Per-call response time recorded to establish a baseline (no target has been measured yet)
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
    └─ Basic load test (representative traffic shape, 1 min)
    ↓ [on failure: stop; notify team]
    ↓
Tier 3: Golden-Master Tests (30 min)
    ├─ Replay 1-week traffic sample against Rust
    ├─ Verify output parity
    └─ Generate diff report
    ↓ [on failure: alert code reviewer; may require manual review]
    ↓
Tier 4: Performance Tests (60 min, only on main branch)
    ├─ 1-hour load test (peak-load profile)
    ├─ Memory soak test (7.8h baseline run; see Category 4b)
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
