# Migration Strategy: Two-Phase Approach & Staged Rollout

## Overview

The migration from Node.js to Rust is executed as a **~22-week staged rollout (Stage 0–4)**, with binary rollback switches and manual SOPs at every stage (rollback execution completes within 5 minutes overall, 2–4 minutes per stage). Phase A (Faithful Port) and Phase B (Harden & Optimize) are not separate time blocks bolted onto the front of this schedule — they are the two methodological lenses applied *within* the Stage 0–4 timeline: each module is first ported faithfully (Phase A) before the stage that deploys it goes live, and hardening/optimization work (Phase B) continues alongside later stages once parity is proven. This approach minimizes risk while maintaining behavioral compatibility.

### Two-Phase Philosophy

**Phase A (Faithful Port)**: Replicate behavior exactly, including bugs and quirks, to establish a high-confidence baseline.  
**Phase B (Harden & Optimize)**: Refactor, fix known issues, and optimize once parity is proven.

This separation of concerns is critical: mixing refactoring with porting introduces orthogonal variables and makes testing intractable.

---

## Phase A: Faithful Port (Primarily Stage 0–2)

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
- `aeg-sensors`, `aeg-metrics` (signal RMS aggregation), `aeg-led`, `aeg-ota`
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

## Phase B: Harden & Optimize (Primarily Stage 2–4)

### Goal
Fix known bugs, optimize performance, and improve reliability. All changes validated against Phase A baseline to prevent regression.

### Approach

#### 1. Process All `TODO(port)` and `FIXME(port)` Markers
- `FIXME(port)` → Fix known bugs that were intentionally replicated (e.g., RISK-003)
- `TODO(port)` → Implement deferred refactorings and optimizations

**Example Fixes:**
- **RISK-003 (Sensor Drop Under Load)**: Implement priority queue + backpressure to prevent message loss during MQTT bursts
- **RISK-007 (OTA Timeout)**: Add network timeout with exponential backoff instead of infinite wait
- **RISK-012 (Indicator Lamp Serial-Port Enumeration)**: Pin the lamp to a stable device identifier (USB serial number / udev symlink) instead of relying on enumeration order; add retry-with-backoff on first open

#### 2. Performance Optimization
Target metrics from Problem Statement:

| Metric | Phase A (Node.js baseline, 4-core ARM Cortex-A53, 3h active load) | Phase B (Rust, measured on 7.8h soak, same hardware) | Method |
|---|---|---|---|
| **Memory (active)** | 1538 MB active (~54% of 2.84 GB available RAM); ~20 processes across 16 services / 8 startup stages | 206 MB full-stack resident (14 supervised services + Python subprocess) | Collapse the per-process Node.js footprint into a small set of supervised Rust services |
| **CPU** | idle 52%, 3h active 64% | avg 6.3%, peak 8.8%, zero drift over the full soak | Remove event-loop/GC contention; profile hot paths |
| **Load average (4-core)** | 3.30 avg (3h active) | avg 0.50, peak 1.81 (7.8h soak) | Maintain headroom; no GC pauses (Rust has no GC) |
| **System memory headroom** | not separately tracked; the per-service 500 MB cap left hours, not days, of margin against any sustained growth | system-available memory 6024 → 5885 MB over 7.8h (−139 MB); see `04-testing-strategy.md` for the sustained-growth issue found and fixed during soak testing | Soak testing catches allocator/runtime-level growth that unit tests cannot |
| **Service restarts / startup stability** | per-service memory cap (200 MB–2 GB) triggers auto-restart; cold boot to full service availability ≈ 2m30s | 3 restarts in the first 66 minutes (startup stabilization), then 0 restarts over the following 6.7h; longest clean soak to date is 10.6h | Supervised startup; see `04-testing-strategy.md` |
| **GC pauses** | 0 (V8 GC exists but was not the bottleneck here) | 0 (no GC in Rust) | N/A |

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
- ✓ Performance targets met: CPU avg 6.3% / peak 8.8% (7.8h soak, 4-core ARM Cortex-A53); full-stack resident memory 206 MB across 14 supervised services + Python subprocess; load average avg 0.50 / peak 1.81 (4-core)
- ✓ Longest continuous soak to date (10.6h) passes with the memory-growth issue found during soak testing fixed and the slope trending negative in the final 2h (see `04-testing-strategy.md`). Note: "90 days" elsewhere in this project refers only to the certificate rotation cycle, not an uptime requirement.
- ✓ Phase A golden-master parity maintained (no regressions)
- ✓ Team fully trained and confident in Rust codebase

---

## Staged Rollout: Stage 0–4 (~22 Weeks)

### Philosophy
Incremental deployment with automated safety gates; rolling a stage back takes 2–4 minutes of execution time (see ADR-0005). This is distinct from the *rollback decision window* in the timeline table below — how long a stage stays reversible before the next one begins.

![Stage 0 to Stage 4 rollout timeline](../diagrams/staged-rollout-timeline.png)

<sub>Vector source: [`staged-rollout-timeline.svg`](../diagrams/staged-rollout-timeline.svg)</sub>

---

### Stage 0: Bootstrap & Infrastructure (Week 0, Low Risk)

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

### Stage 1: Low-Risk Services (Weeks 1–5, Low Risk)

**Objective**: Deploy independent services with no upstream dependencies.

**Scope:**
- Deploy: `aeg-led` (GPIO control) — isolated; no data dependencies
- Deploy: `aeg-config` (configuration service)
- Test: Can these services run alongside Node.js? Are there resource conflicts?

**Rollback Trigger:**
- Service crashes repeatedly
- Memory leaks detected (sustained positive RSS growth slope observed during monitoring, as distinct from normal startup transients — see `04-testing-strategy.md` for the soak-testing method used to detect this)
- GPIO control fails (LED not responding)

**Rollback Procedure:**
```
systemctl stop aeg-led aeg-config
# Node.js resumes control
```

**Duration**: ~4 weeks (Weeks 1–5); deployed to 20–30% of fleet  
**Go/No-Go Gate**: All Stage 1 devices stable; LED control 100% functional; zero memory leaks

---

### Stage 2: Medium-Risk Services (Weeks 6–9, Medium Risk)

**Objective**: Deploy event-streaming and firmware-update services; test message delivery and OTA reliability.

**Scope:**
- Deploy: `aeg-mqtt` (MQTT client, event streaming)
- Deploy: `aeg-ota` (firmware updates)
- Test: Message delivery under load? OTA update reliability? Cloud connectivity?

**Rollback Trigger:**
- MQTT message loss > 0.1% (serious data loss)
- OTA update fails or leaves device in bad state
- Cloud API connection fails
- Per-second pipeline latency exceeds the 500 ms budget ceiling observed during soak testing (Phase B measured p95 is 56.4 ms, 5.6% of the 1-second budget, across a 285-sample run — see ADR-0005), or shows a sustained upward trend versus that baseline

**Rollback Procedure:**
```
systemctl stop aeg-mqtt aeg-ota
# Verify no firmware corruption
# Restore MQTT session from checkpoint
# Node.js resumes
# Notify cloud backend to resume polling (fallback mode)
```

**Duration**: ~4 weeks (Weeks 6–9); deployed to 40–50% of fleet  
**Go/No-Go Gate**: 100% message delivery, zero failed OTA updates, 100% cloud connectivity, zero data loss

---

### Stage 3: High-Risk Services — Vibration, Temperature & Sensor Data (Weeks 10–16, High Risk)

**Objective**: Deploy stateful vibration/temperature/sensor-data services; test persistence and concurrency. This is the highest-risk stage in the rollout.

**Scope:**
- Deploy: `aeg-sensors` (sensor aggregation, DB writes)
- Deploy: `aeg-metrics` (signal RMS aggregation, e.g. meter-reading calculations)
- Test: DB write conflicts? Sensor reading accuracy? Data consistency?

**Rollback Trigger:**
- Sensor reading divergence from Node.js baseline (> 1% error)
- Database corruption or query failures
- Per-second pipeline latency exceeds the 500 ms budget ceiling, or any sample drifts materially above the Phase B soak baseline (p95 56.4 ms / 5.6% of the 1-second budget; see ADR-0005)

**Rollback Procedure:**
```
systemctl stop aeg-sensors aeg-metrics
# Verify DB is not corrupted
sqlite3 /var/lib/aeg/data.db "PRAGMA integrity_check"
# Rollback DB to checkpoint if needed
# Node.js resumes
```

**Duration**: ~7 weeks (Weeks 10–16); deployed to 60–70% of fleet  
**Go/No-Go Gate**: Sensor readings 100% accurate; zero DB corruption incidents; latency within the measured budget

---

### Stage 4: Cutover & Full Migration — REST App Server (Weeks 17–22, Medium Risk, Rollback Window = 48h)

**Objective**: Full fleet migration; Node.js binary decommissioned.

**Scope:**
- Deploy: `aeg-appserver` (REST API, replace Node.js appserver process)
- Disable: Node.js process (process supervisor)
- Test: Full feature parity; all API clients functional

**Rollback Trigger (48h Window):**
- Any service crashes
- Per-second processing pipeline exceeds the 500 ms budget ceiling, or p95 drifts materially above the Phase B soak baseline (56.4 ms / 5.6% of the 1-second budget — see ADR-0005); `aeg-appserver` is the single largest migration unit in the rollout, so this is the most consequential gate
- Memory usage trending upward with no return to baseline (full-stack resident baseline is 206 MB; see soak-slope methodology in `04-testing-strategy.md`)
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

**Duration**: ~6 weeks (Weeks 17–22); 48-hour rollback window  
**Go/No-Go Gate**: 
- Processing pipeline stays within the measured budget: p95 ≤ 56.4 ms (5.6% of the 1-second budget; 285-sample Phase B measurement), zero samples exceeding 500 ms
- MQTT heartbeat interval maintained (no message loss)
- Zero service restarts over 24 hours
- Memory stable at or near the 206 MB full-stack baseline; no sustained upward slope

**Post-Cutover (Week 23+):**
- Monitor fleet for 2 weeks
- If stable: Decommission Node.js binary; retire process supervisor
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
        # Performance targets — thresholds anchored to the Phase B soak baseline
        # (p95 56.4 ms / 5.6% of the 1-second budget; 206 MB full-stack resident)
        perf-benchmark > results.json
        if results.json latency_p95_ms > 500; then exit 1; fi  # budget ceiling; 0 samples exceeded this in the 285-sample baseline run
        if results.json memory_slope_mb_per_hour > 0; then exit 1; fi  # flag any sustained positive slope — this is how the arena-growth issue was caught (see 04-testing-strategy.md)
        
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
        - Run MQTT message loss test (expect 0 loss)
        - Run OTA update simulation
        - Cloud connectivity health check
        
  - stage-3-deploy:
      script: |
        - Verify sensor readings match Node.js baseline (< 1% error)
        - Check DB integrity post-write
        - Golden-master test on sensor sample data
        
  - stage-4-deploy:
      script: |
        - Processing pipeline latency test (p95 within the 56.4 ms / 5.6%-of-budget baseline)
        - 24-hour load test (no crashes)
        - Memory soak test (flag any sustained positive RSS slope per service; see 04-testing-strategy.md)
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

| Stage | Duration | Fleet Size | Rollback Decision Window | Team Load |
|---|---|---|---|---|
| Stage 0 | ~1 week (Week 0) | 5–10 devices | 10 minutes | 1 on-call engineer |
| Stage 1 | ~4 weeks (Weeks 1–5) | 20–30 devices | 10 minutes | 1 on-call engineer |
| Stage 2 | ~4 weeks (Weeks 6–9) | 40–50 devices | 1 hour | 2 on-call engineers |
| Stage 3 | ~7 weeks (Weeks 10–16, high risk) | 60–70 devices | 2 hours | 2 on-call engineers |
| Stage 4 | ~6 weeks (Weeks 17–22) | 100% fleet | 48 hours | 3 on-call engineers |
| **Project total** | **~22 weeks (Stage 0–4)** | **Progressive** | **Decreasing then 48h** | **Varies** |

Phase A (Faithful Port) and Phase B (Harden & Optimize) are not additional time on top of this — they describe *what kind of work* is happening inside Stage 0–2 (porting + golden-master validation) versus Stage 2–4 (hardening + performance tuning), respectively. They run concurrently with, not in sequence before, the staged rollout.

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
