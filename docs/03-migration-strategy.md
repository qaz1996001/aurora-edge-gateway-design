# Migration Strategy: Two-Phase Approach & Staged Rollout

## Overview

The migration from Node.js to Rust is executed in **two sequential phases**, followed by a carefully staged rollout with automated safety gates. This approach minimizes risk while maintaining behavioral compatibility.

### Two-Phase Philosophy

**Phase A (Faithful Port)**: Replicate behavior exactly, including bugs and quirks, to establish a high-confidence baseline.  
**Phase B (Harden & Optimize)**: Refactor, fix known issues, and optimize once parity is proven.

This separation of concerns is critical: mixing refactoring with porting introduces orthogonal variables and makes testing intractable.

---

## Phase A: Faithful Port (Weeks 0–8)

### Goal
Achieve 100% **behavioral equivalence** with the Node.js baseline, verified via golden-master tests on production traffic replays.

### Approach

#### 1. Module-by-Module Analysis & Port
Process modules in dependency order (Tier 0 → 4; see Architecture Overview):

**Tier 0: Foundation**
- `aeg-config` — TOML parsing and configuration schema
- Target: Exact same config format; same defaults; same error messages

**Tier 1: Base Services**
- `aeg-db` — SQLite wrapper
- `aeg-file-manager` — File I/O primitives
- Target: Same SQL queries; same file paths; same error handling

**Tier 2: Protocol Clients**
- `aeg-mqtt-client` — MQTT primitives (async wrapper around rumqttc)
- `aeg-python-bridge` — Inter-process communication to Python subprocess
- Target: Same topic subscriptions; same message formats; same Python command-line arguments

**Tier 3: Domain Logic**
- `aeg-sensors`, `aeg-vrms`, `aeg-led`, `aeg-ota`
- Target: Exact same sensor-reading values; same LED color/brightness; same OTA logic

**Tier 4: Public Interfaces**
- `aeg-appserver` — REST API routes and HTTP response formats
- Target: Identical REST endpoints, status codes, response JSON schemas

#### 2. Marker Conventions for Traceability
Use consistent markers to flag deferred decisions and known issues **during the port**:

| Marker | Usage | Example |
|---|---|---|
| `TODO(port)` | Work deferred to Phase B | `TODO(port): Refactor error handling per ADR-0005` |
| `FIXME(port)` | Known bug in Node.js that we replicate here | `FIXME(port): RISK-003 — sensor reads drop 0.1% under peak load` |
| `PERF(port)` | Optimization opportunity deferred to Phase B | `PERF(port): Replace Vec with circular buffer for lower allocation` |
| `SAFETY:` | Memory-safety or security audit marker | `SAFETY: Bounds-checked input; no buffer overflow possible` |

All markers are tracked in a searchable marker census (CI gate: `grep -r "TODO(port)"` must have < 100 matches before Phase B).

#### 3. Golden-Master Testing
See **[ADR-0006](adr/0006-golden-master-testing.md)** and `04-testing-strategy.md` for full methodology.

**High-Level Process:**
1. Record 3+ weeks of production traffic (MQTT messages, REST API calls, sensor readings, timestamps)
2. Instrument Node.js binary to capture all internal state at key points (before/after DB writes, sensor reads, etc.)
3. Run same traffic against Rust binary; compare outputs at same key points
4. **Target**: 100% match on all observable outputs (REST responses, MQTT messages, file writes, DB state)

#### 4. Risk Disposition
Every module flagged during code review has an assigned risk level:

| Risk Level | Characteristics | Example | Mitigation |
|---|---|---|---|
| **LOW** | Simple data transformation; no side effects | JSON serialization, config parsing | Unit tests sufficient |
| **MEDIUM** | Some I/O or state mutation; well-defined semantics | Database queries, file writes | Golden-master tests on sample data |
| **HIGH** | Complex concurrency, cryptography, or business logic | MQTT event aggregation, firmware update | Extensive golden-master tests + stress testing |

### Phase A Success Criteria
- ✓ All modules ported; 0 `FIXME(port)` items remain unresolved
- ✓ Golden-master test suite passes on ≥ 3 weeks of production traffic
- ✓ No new bugs introduced (issue count < Node.js baseline)
- ✓ Behavioral equivalence established on 100% of REST API endpoints
- ✓ Memory and CPU profiling baseline captured for Phase B comparison

---

## Phase B: Harden & Optimize (Weeks 8–12)

### Goal
Fix known bugs, optimize performance, and improve reliability. All changes validated against Phase A baseline to prevent regression.

### Approach

#### 1. Process All `TODO(port)` and `FIXME(port)` Markers
- `FIXME(port)` → Fix known bugs that were intentionally replicated (e.g., RISK-003)
- `TODO(port)` → Implement deferred refactorings and optimizations

**Example Fixes:**
- **RISK-003 (Sensor Drop Under Load)**: Implement priority queue + backpressure to prevent message loss during MQTT bursts
- **RISK-007 (OTA Timeout)**: Add network timeout with exponential backoff instead of infinite wait
- **RISK-012 (LED Flicker)**: Debounce GPIO updates; add frame-rate limiting

#### 2. Performance Optimization
Target metrics from Problem Statement:

| Metric | Phase A (Baseline) | Phase B (Target) | Method |
|---|---|---|---|
| **CPU (3h active)** | ~60% avg | 50–65% avg, stable | Eliminate GC contention; profile hot paths |
| **Memory (idle)** | < 1400 MB | < 1350 MB | Reduce allocation overhead |
| **Memory (active, 3h)** | < 1500 MB | < 1500 MB sustained | Verify no GC-induced bloat; audit drop trait |
| **Memory growth (per day)** | < 0.1% | < 0.01% | Ensure deterministic cleanup |
| **Load average (4-core)** | 3.3 avg | 3.0–3.5 avg | Maintain headroom; no GC pauses |
| **GC pauses** | 0 (expected) | 0 (verified) | Confirm no unexpected allocations |

#### 3. Hardening & Reliability
- **Error handling**: Replace panics with graceful degradation (retry, fallback, alerting)
- **Timeouts**: Add comprehensive timeout handling for all I/O operations
- **Monitoring**: Add per-service prometheus metrics; configure alerting
- **Systemd integration**: Add health checks (ping endpoints, DB connectivity, MQTT broker connectivity)

#### 4. Testing Evolution
- Phase A tests remain unchanged (prevent regression)
- Add new unit tests for refactored code
- Run extended stress tests (48+ hour load testing, memory soak testing)
- Validate Phase B fixes don't break Phase A parity

### Phase B Success Criteria
- ✓ All `TODO(port)` and `FIXME(port)` markers resolved or reclassified
- ✓ Performance targets met (CPU 50–65% avg, Memory < 1500 MB sustained, Growth < 0.01%/day)
- ✓ 90-day continuous operation test passes without crashes or restarts
- ✓ Load average stable (3.0–3.5 on 4-core, no GC-induced spikes)
- ✓ Phase A golden-master parity maintained (no regressions)
- ✓ Team fully trained and confident in Rust codebase

---

## Staged Rollout: Stage 0–4 (Weeks 12–16)

### Philosophy
Incremental deployment with automated safety gates; each stage can be rolled back in < 1 minute.

See `../diagrams/staged-rollout-timeline.svg` for visual timeline.

---

### Stage 0: Bootstrap & Infrastructure (Week 12, Low Risk)

**Objective**: Deploy Rust binaries and systemd infrastructure to test fleet; verify no service startup issues.

**Scope:**
- Deploy aeg-config (config parser, no runtime changes)
- Deploy aeg-db (database layer, no API changes)
- Test: Can systemd start and manage Rust binaries? Do binaries log correctly?

**Rollback Trigger:**
- Any service fails to start
- Systemd health checks fail
- Memory usage exceeds 50 MB idle

**Rollback Procedure:**
```
# On test device
systemctl stop aeg-*
# Replace Rust binaries with Node.js binary
systemctl start node-gateway
# Verify REST API and MQTT traffic resumed
```

**Duration**: 1 week; deployed to 5–10% of fleet (low-risk test devices)  
**Go/No-Go Gate**: All test devices stable for 7 days; zero service restarts

---

### Stage 1: Low-Risk Services (Week 13, Medium Risk)

**Objective**: Deploy independent services with no upstream dependencies.

**Scope:**
- Deploy: `aeg-led` (GPIO control) — isolated; no data dependencies
- Deploy: `aeg-config` (configuration service)
- Test: Can these services run alongside Node.js? Are there resource conflicts?

**Rollback Trigger:**
- Service crashes repeatedly
- Memory leaks detected (growing > 0.1% per day)
- GPIO control fails (LED not responding)

**Rollback Procedure:**
```
systemctl stop aeg-led aeg-config
# Node.js resumes control
```

**Duration**: 1 week; deployed to 20–30% of fleet  
**Go/No-Go Gate**: All Stage 1 devices stable; LED control 100% functional; zero memory leaks

---

### Stage 2: Medium-Risk Services (Week 14, Medium-High Risk)

**Objective**: Deploy stateful services; test persistence and concurrency.

**Scope:**
- Deploy: `aeg-sensors` (sensor aggregation, DB writes)
- Deploy: `aeg-vrms` (meter reading calculations)
- Test: DB write conflicts? Sensor reading accuracy? Data consistency?

**Rollback Trigger:**
- Sensor reading divergence from Node.js baseline (> 1% error)
- Database corruption or query failures
- Event processing latency > 100ms (worse than Phase A)

**Rollback Procedure:**
```
systemctl stop aeg-sensors aeg-vrms
# Verify DB is not corrupted
sqlite3 /var/lib/aeg/data.db "PRAGMA integrity_check"
# Rollback DB to checkpoint if needed
# Node.js resumes
```

**Duration**: 1 week; deployed to 40–50% of fleet  
**Go/No-Go Gate**: Sensor readings 100% accurate; zero DB corruption incidents; latency targets met

---

### Stage 3: High-Risk Services (Week 15, High Risk)

**Objective**: Deploy critical services; full feature parity required.

**Scope:**
- Deploy: `aeg-mqtt` (MQTT client, event streaming)
- Deploy: `aeg-ota` (firmware updates)
- Test: Message delivery under load? OTA update reliability? Cloud connectivity?

**Rollback Trigger:**
- MQTT message loss > 0.1% (serious data loss)
- OTA update fails or leaves device in bad state
- Cloud API connection fails
- Latency tail (95th percentile) > 20ms (worse than Phase A)

**Rollback Procedure:**
```
systemctl stop aeg-mqtt aeg-ota
# Verify no firmware corruption
# Restore MQTT session from checkpoint
# Node.js resumes
# Notify cloud backend to resume polling (fallback mode)
```

**Duration**: 2 weeks; deployed to 60–70% of fleet  
**Go/No-Go Gate**: 100% message delivery, zero failed OTA updates, 100% cloud connectivity, zero data loss

---

### Stage 4: Cutover & Full Migration (Week 16, Rollback Window = 48h)

**Objective**: Full fleet migration; Node.js binary decommissioned.

**Scope:**
- Deploy: `aeg-appserver` (REST API, replace Node.js monolith)
- Disable: Node.js process (PM2)
- Test: Full feature parity; all API clients functional

**Rollback Trigger (48h Window):**
- Any service crashes
- API response time > 50ms (95th percentile)
- Memory usage growth > 0.1% per day
- Data corruption incidents

**Rollback Procedure (Emergency):**
```
# Quick restart with Node.js (systemd ExecStart=...)
systemctl stop aeg-appserver
systemctl set-environment NODE_GATEWAY_ENABLED=1
systemctl start node-gateway
# All Rust services remain running (they don't conflict)
# Cloud backend auto-detects failover via health checks
```

**Duration**: 1 week active rollout; 48-hour rollback window  
**Go/No-Go Gate**: 
- All 10 REST API endpoints respond < 10ms @ 95th percentile
- MQTT heartbeat interval maintained (no message loss)
- Zero service restarts over 24 hours
- Memory stable (< 0.01% growth per day)

**Post-Cutover (Week 17+):**
- Monitor fleet for 2 weeks
- If stable: Decommission Node.js binary; retire PM2
- If issues: Extend monitoring and debug

---

## Automated Safety Gates (CI/CD Integration)

At each stage, automated tests must pass before deployment proceeds:

```yaml
# Pseudo-code for CI pipeline
stages:
  - build:
      script: cargo build --release --target aarch64-unknown-linux-gnu
      
  - test-phase-a:
      script: |
        # Run golden-master tests on production traffic
        ./golden-master-harness < production-traffic.json
        if [ $? -ne 0 ]; then exit 1; fi
        
  - test-phase-b:
      script: |
        # Performance targets
        perf-benchmark > results.json
        if results.json latency_p95 > 10ms; then exit 1; fi
        if results.json memory_growth > 0.01%; then exit 1; fi
        
  - marker-census:
      script: |
        # Ensure all markers accounted for
        todo_count=$(grep -r "TODO(port)" . | wc -l)
        fixme_count=$(grep -r "FIXME(port)" . | wc -l)
        if [ $todo_count -gt 0 ]; then exit 1; fi
        
  - stage-0-deploy:
      script: |
        # Stage 0 pre-check
        - Verify systemd units load without error
        - Check for library conflicts
        - Validate config file format
        
  - stage-1-deploy:
      script: |
        - Verify LED control works (GPIO integration)
        - Check memory footprint < 50MB
        
  - stage-2-deploy:
      script: |
        - Verify sensor readings match Node.js baseline (< 1% error)
        - Check DB integrity post-write
        - Golden-master test on sensor sample data
        
  - stage-3-deploy:
      script: |
        - Run MQTT message loss test (expect 0 loss)
        - Run OTA update simulation
        - Cloud connectivity health check
        
  - stage-4-deploy:
      script: |
        - Full API endpoint test (10 endpoints)
        - 24-hour load test (no crashes)
        - Memory soak test (no growth > 0.01%)
```

---

## Rollback Decision Tree

```
Incident Detected
    ↓
Automatic Health Check Failed?
    ├─ YES → Automated Rollback (< 1 min)
    │         ├─ Kill Rust service
    │         ├─ Restore Node.js/fallback
    │         └─ Alert team
    │
    └─ NO (Manual Intervention Required)
         ├─ Is this a data consistency issue?
         │   └─ YES → Restore DB from checkpoint; roll back
         │
         ├─ Is this a temporary network glitch?
         │   └─ YES → Wait 5 min; observe; decide
         │
         └─ Is this a fundamental incompatibility?
             └─ YES → Roll back to prior stage
```

---

## Timeline & Resource Requirements

| Stage | Duration | Fleet Size | Rollback Window | Team Load |
|---|---|---|---|---|
| Phase A | 8 weeks | N/A (lab testing) | N/A | 2–3 FTE engineers |
| Phase B | 4 weeks | N/A (lab + extended testing) | N/A | 1–2 FTE engineers |
| Stage 0 | 1 week | 5–10 devices | 10 minutes | 1 on-call engineer |
| Stage 1 | 1 week | 20–30 devices | 10 minutes | 1 on-call engineer |
| Stage 2 | 1 week | 40–50 devices | 1 hour | 2 on-call engineers |
| Stage 3 | 2 weeks | 60–70 devices | 2 hours | 2 on-call engineers |
| Stage 4 | 1 week | 100% fleet | 48 hours | 3 on-call engineers |
| **Total** | **~16 weeks** | **Progressive** | **Decreasing** | **Varies** |

---

## Communication Plan

### Internal (Team)
- Daily standup during Stages 0–4
- Weekly rollout readiness review
- Post-stage incident review (blameless)

### External (Customers)
- Pre-announcement: "Planned performance improvements coming"
- Stage 2 onward: Monitoring metrics dashboard (transparent progress)
- Post-cutover: Release notes describing improvements

---

## Next Steps

1. **Testing Methodology** → See `04-testing-strategy.md` for golden-master test design
2. **Review Process** → See `05-review-process.md` for multi-role code review
3. **Decision Rationale** → See **[ADR-0005](adr/0005-staged-rollout-and-rollback.md)** for full decision record

---

**Last Updated**: 2026-10-20  
**Status**: Migration strategy approved  
**Next Review**: Week 8 (Phase A completion)
