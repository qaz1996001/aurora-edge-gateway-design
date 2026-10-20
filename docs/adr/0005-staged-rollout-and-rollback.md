# ADR-0005: Staged Rollout & Rollback Strategy

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Operations, product management, engineering leads

---

## Context

Deploying a major rewrite to production is high-risk. A full-fleet cutover could cascade failures across all edge devices simultaneously. We need a strategy to:
1. Validate the Rust system on real hardware before full rollout
2. Detect issues early (on small population) before they affect all customers
3. Rollback quickly if anything goes wrong
4. Maintain high availability throughout the transition

---

## Decision

**Deploy in five stages (Stage 0–4) over approximately 22 weeks** (Stage 0: ~1 week; Stage 1: weeks 1–5; Stage 2: weeks 6–9; Stage 3: weeks 10–16; Stage 4: weeks 17–22), with manual go/no-go gates at each stage. Each stage increases fleet coverage and risk.

**Important — actual rollout mechanism**: The real cutover mechanism used on this project is a **binary switch** (the old stack is fully stopped and the new stack fully started on a device) combined with a **manual, documented rollback SOP** — not an automated percentage-based canary and not a monitoring-threshold-triggered automatic rollback. The automated canary and threshold-triggered rollback described later in this document (Automated Safety Gates, per-stage alert thresholds) are **proposed future-state mechanisms**, not things that have been implemented. They are included here as the direction we intend to evolve toward — automation reduces the time and judgment calls required from an on-call operator — but today, every rollback on this project is a person following the documented SOP in the Rollback Procedure section below.

---

## Stage Breakdown

### Stage 0: Bootstrap & Infrastructure (~1 week)

**Objective**: Verify Rust binaries load, systemd services start, basic connectivity works.

**Scope**: Deploy aeg-config, aeg-db (foundational services only)

**Risk Level**: Low (no field data dependency; internal validation only)

**Rollback Mechanism**: Binary switch — operator stops the Rust services and starts the Node.js process via the manual SOP. No automated canary or threshold-triggered rollback exists; see the note under Decision above.

**Success Criteria**:
- ✓ All services start without error
- ✓ Health checks pass (REST `/health` endpoints respond)
- ✓ No crashes observed during the bring-up window

---

### Stage 1: Low-Risk Services (Weeks 1–5)

**Objective**: Deploy stateless/low-dependency services with no field sensor data dependencies.

**Scope**: Add aeg-led, aeg-config

**Risk Level**: Low

**Rollback Commitment**: Full rollback to Node.js in **under 2 minutes** (binary switch + manual SOP)

**Success Criteria**:
- ✓ Indicator lamp control works as expected
- ✓ No conflicts with Node.js legacy code
- ✓ No crashes observed during the stage window

---

### Stage 2: Medium-Risk Services (Weeks 6–9)

**Objective**: Deploy stateful services (database, sensors) with I/O side effects.

**Scope**: Add aeg-sensors, aeg-metrics (database writers)

**Risk Level**: Medium

**Rollback Mechanism**: Binary switch + manual SOP; includes a database integrity check (`PRAGMA integrity_check`) before the Rust stack is stopped, since this stage introduces stateful writers.

**Success Criteria**:
- ✓ Sensor readings consistent with the Node.js baseline (parity validated via golden-master comparison; see ADR-0006)
- ✓ Database integrity maintained (PRAGMA integrity_check passes)
- ✓ No writes lost during the observation window

---

### Stage 3: High-Risk Services (Weeks 10–16)

**Objective**: Deploy the services with the highest field-data risk — vibration, temperature, and other sensor-data services — plus the critical message pipeline (MQTT, firmware updates).

**Scope**: Add aeg-mqtt, aeg-ota, and the vibration/temperature/sensor-data services

**Risk Level**: High — this stage carries the project's highest risk level, because it touches the sensor-data services whose correctness most directly affects the customer-facing product and the firmware-update path.

**Rollback Commitment**: Full rollback to Node.js in **3–4 minutes** (binary switch + manual SOP, including firmware-update safety checks)

**Success Criteria**:
- ✓ MQTT message delivery consistent with the Node.js baseline under observed load
- ✓ OTA firmware updates succeed reliably
- ✓ Cloud connectivity stable (no unexpected disconnects)
- ✓ Per-second processing pipeline stays within budget: p95 end-to-end latency 56.4 ms, representing ~5.6% of the 1-second per-cycle budget, with zero cycles exceeding 500 ms (285-sample measurement run)

**Extended Validation**:
- Longest continuous validation run on this project to date: **10.6-hour soak test, PASS** (there is no 90-day or similarly long continuous-run validation on this project)
- Chaos-style checks: network faults, disk full, subprocess crashes — verify graceful recovery

---

### Stage 4: Full Cutover (Weeks 17–22)

**Objective**: Complete migration; decommission Node.js binary.

**Scope**: Deploy aeg-appserver (REST API) — the single largest migration unit in this rollout; disable Node.js process

**Risk Level**: Medium — lower than Stage 3 in terms of per-cycle data correctness risk, but the largest single migration unit, so operational/rollback risk is non-trivial.

**Rollback Commitment**: Full rollback to Node.js in **2–3 minutes** (binary switch + manual SOP)

**Success Criteria**:
- ✓ REST API latency within the measured per-second processing budget: p95 56.4 ms, ~5.6% of the per-cycle budget, zero cycles over 500 ms (see ADR-0005 latency measurements, 285-sample run on the 4-core ARM Cortex-A53 target platform)
- ✓ MQTT heartbeat maintained (no disruption to cloud connectivity)
- ✓ No unplanned service restarts observed during the stage window
- ✓ Resident memory and CPU usage remain near the measured full-stack baseline (206 MB resident across 14 supervised services + Python subprocess; 6.3% average / 8.8% peak CPU over a 7.8-hour soak)
- ✓ Customer-facing metrics match or exceed the Node.js baseline

**Post-Cutover Monitoring**:
- Continued monitoring after cutover
- If all green: Decommission Node.js binary; declare migration complete
- If issues: Follow the manual rollback SOP; investigate; potentially revert the entire Stage 4 change

---

## Automated Safety Gates

Each stage has automated tests that must pass before manual go/no-go decision:

```
┌─ Code Review & Merge ─┐
│                       ↓
│              CI Pipeline (Tier 1–3)
│              ├─ Compile ✓
│              ├─ Unit Tests ✓
│              ├─ Integration Tests ✓
│              └─ Marker Census ✓
│                       ↓
│         ┌─ Golden-Master Tests ─┐
│         │  ├─ Output parity ✓                     │
│         │  └─ No regressions ✓                   │
│         └─────────────────────────────────────────┘
│                       ↓
│         ┌─ Extended Soak Test ─┐
│         │  (Stage 3: 10.6-hour soak, PASS) │
│         └────────────────────────────┘
│                       ↓
│  [Gates Passed] → Ready for Manual Deployment
│                       ↓
│  Stage 0–4 Deployment (Manual Approval Required; binary switch per device)
│                       ↓
│  Field Monitoring (per the Stage 0–4 week schedule above)
│                       ↓
│  Go/No-Go Decision (Operations + Tech Lead)
│
└─────────────────────────────────────────────────────────
```

**Note on automation**: The diagram above includes aspirational automated gating. Today, every gate below "Golden-Master Tests" is evaluated and actuated by a person: there is no automated canary percentage ramp and no monitoring-threshold-triggered automatic rollback in production. We list the target-state automation here because it is the direction we intend to build toward — reducing on-call toil and reaction time — not because it exists today.

---

## Rollback Procedure

**Actual mechanism (today)**: Every rollback on this project is a **binary switch** — stop the Rust services, start the Node.js process — carried out by an operator following a documented manual SOP. There is no systemd-health-triggered automatic rollback in production today. The commands below describe that manual SOP.

**Overall rollback commitment**: Full rollback to the Node.js stack in **under 5 minutes** for any stage. Measured/targeted rollback times per stage:
- Stage 1: under 2 minutes
- Stage 3: 3–4 minutes
- Stage 4: 2–3 minutes

### Manual Rollback SOP (Stages 0–3)

```bash
# Operator-initiated; no automatic trigger. Operator observes a problem
# (via dashboards/alerts) and runs the documented rollback SOP.

echo "Rollback initiated by operator"

# Stage 0–1: Simple service stop
systemctl stop aeg-led aeg-config
systemctl start node-gateway  # Resume Node.js

# Stage 2: DB consistency check before rollback
sqlite3 /var/lib/aeg/data.db "PRAGMA integrity_check"
if [ $? -ne 0 ]; then
  # DB corrupted; restore from backup
  cp /var/backups/aeg/data.db.backup /var/lib/aeg/data.db
fi
systemctl stop aeg-sensors aeg-metrics
systemctl start node-gateway

# Stage 3: Firmware check before rollback
fw_verify /var/lib/aeg/firmware/*
systemctl stop aeg-mqtt aeg-ota
systemctl start node-gateway

# Notify team
mail -s "Aurora Edge Gateway Rollback Executed (manual SOP)" team@example.com \
  < <(systemctl status aeg-*.service)
```

### Manual Rollback (Stage 4)

```bash
# Operator-initiated rollback; target commitment 2-3 minutes
systemctl stop aeg-*
systemctl start node-gateway

# Verify fallback working
curl localhost:3000/health  # Should respond
mosquitto_sub -t 'gateway/#'  # Should see MQTT traffic

# Notify customers
status-page update "System recovering; brief interruption expected"

# Post-mortem: Why did Stage 4 fail?
# - Revert commit causing issue
# - Fix bug in Rust code
# - Re-test with golden-master
# - Plan re-deployment
```

---

## Timeline & Resource Allocation

| Stage | Duration | Risk Level | Rollback Commitment |
|---|---|---|---|
| 0 | ~1 week | Low | Binary switch + manual SOP |
| 1 | Weeks 1–5 | Low | < 2 minutes |
| 2 | Weeks 6–9 | Medium | Binary switch + manual SOP (DB integrity check) |
| 3 | Weeks 10–16 | High | 3–4 minutes |
| 4 | Weeks 17–22 | Medium (largest single migration unit) | 2–3 minutes |
| **Total** | **~22 weeks** | **Progressive** | **Within 5 minutes, every stage** |

All rollback is a manual, SOP-driven binary switch (old stack fully stopped, new stack fully started) — there is no automated percentage-based canary and no automatic threshold-triggered rollback in production today.

---

## Communication Plan

### Internal (Team)
- **Daily standup** during Stages 0–4
- **Weekly risk review** (Tuesday 2 PM)
- **Post-stage retrospective** (identify issues; plan next stage)

### External (Customers)
- **Pre-announcement**: "Infrastructure improvements coming"
- **Stage 2 onward**: Real-time dashboard showing rollout progress
- **Incident notification**: If a manual rollback is executed, brief message within 5 min
- **Post-cutover**: Release notes documenting improvements (performance, reliability)

---

## Go/No-Go Criteria per Stage

### Stage 0 → 1 Decision (end of Stage 0, ~week 1)
- [ ] Zero crashes during bring-up
- [ ] Health checks 100% pass rate
- **Decision**: **GO** proceed to Stage 1

### Stage 1 → 2 Decision (end of week 5)
- [ ] Indicator lamp control works as expected
- [ ] Zero crashes
- [ ] Rollback rehearsed within the < 2 minute target
- **Decision**: **GO** proceed to Stage 2

### Stage 2 → 3 Decision (end of week 9)
- [ ] Sensor readings consistent with Node.js baseline (golden-master comparison)
- [ ] DB integrity checks pass
- [ ] Zero data loss incidents observed
- **Decision**: **GO** proceed to Stage 3

### Stage 3 → 4 Decision (end of week 16)
- [ ] MQTT message delivery consistent with baseline
- [ ] OTA updates succeed reliably
- [ ] Soak test passed (10.6-hour continuous run, no crashes, memory stable after applying the `MALLOC_ARENA_MAX` fix described in the Monitoring & Alerting section)
- [ ] Per-cycle latency within budget: p95 56.4 ms (~5.6% of the 1-second budget), zero cycles over 500 ms
- [ ] Rollback rehearsed within the 3–4 minute target
- **Decision**: **GO** proceed to Stage 4

### Stage 4 → Decommission Node.js Decision (end of week 22)
- [ ] REST API latency within the measured per-second processing budget (p95 56.4 ms, ~5.6% of budget, zero cycles over 500 ms)
- [ ] Resident memory and CPU near the measured full-stack baseline (206 MB resident; 6.3% average / 8.8% peak CPU over a 7.8-hour soak)
- [ ] Zero unplanned restarts
- [ ] Rollback rehearsed within the 2–3 minute target
- [ ] No significant bugs reported by customers
- **Decision**: **COMPLETE** migration; decommission Node.js

---

## Monitoring & Alerting

**Current state**: We do not have automated threshold-triggered rollback wired to these alerts in production. Per-stage metrics below are watched manually by the on-call operator; an alert is a prompt for a human to decide whether to run the manual rollback SOP, not an automatic trigger. We list concrete thresholds here as the target state to build toward.

One real finding from soak testing that motivates this section: during a 7.8-hour soak test, the message-bus process showed a linear memory increase of roughly 6.5 MB/hour (45.0 MB → 95.3 MB, +112%), root-caused to glibc multi-threaded arena growth (64 arenas). It was fixed by setting `MALLOC_ARENA_MAX=2`; a subsequent 10.6-hour soak test passed, with the memory trend ending at roughly −3.69 MB/hour. This is exactly the kind of regression a long-running soak test catches that short functional tests miss — reinforcing why Stage 3's extended soak validation matters.

### Per-Stage Metrics (target state; thresholds to be tuned as real field telemetry accumulates)
```yaml
stage_0:
  watch:
    - service_crash_rate > 0
    - health_check_fail_rate > 5%

stage_1:
  watch:
    - service_crash_rate > 0
    - memory_growth_rate  # watch for arena-growth-style leaks; see MALLOC_ARENA_MAX note above

stage_2:
  watch:
    - database_integrity_fail
    - sensor_reading_divergence_from_baseline

stage_3:
  watch:
    - mqtt_message_loss
    - ota_update_failure_rate
    - cloud_connectivity_drop
    - pipeline_latency_p95  # baseline: 56.4 ms, ~5.6% of the 1s budget

stage_4:
  watch:
    - service_crash_rate > 0
    - api_latency_p95  # baseline: 56.4 ms, ~5.6% of the 1s budget
    - resident_memory   # baseline: 206 MB full stack
    - cpu_usage          # baseline: 6.3% avg / 8.8% peak over 7.8h soak
```

---

## Related Decisions
- **[ADR-0006](0006-golden-master-testing.md)**: How golden-master tests feed into stage gates
- **[ADR-0002](0002-module-to-crate-decomposition.md)**: How independent services enable staged deployment

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Operations, product management, engineering  
**Implementation**: Stages 0–4 execute over ~22 weeks (Stage 0 ~1 week; Stage 1 weeks 1–5; Stage 2 weeks 6–9; Stage 3 weeks 10–16; Stage 4 weeks 17–22)
