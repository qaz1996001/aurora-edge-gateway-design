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

**Deploy in five stages (Stage 0–4) over 4 weeks**, with automated health checks and manual go/no-go gates at each stage. Each stage increases fleet coverage and risk; rollback window decreases as confidence increases.

---

## Stage Breakdown

### Stage 0: Bootstrap & Infrastructure (Week 12, 5–10 devices)

**Objective**: Verify Rust binaries load, systemd services start, basic connectivity works.

**Scope**: Deploy aeg-config, aeg-db (foundational services only)

**Rollback Window**: 10 minutes (operator kills Rust services; Node.js resumes)

**Success Criteria**:
- ✓ All services start without error
- ✓ Health checks pass (REST `/health` endpoints respond)
- ✓ Memory usage stable (< 50 MB idle)
- ✓ Zero crashes over 7 days

---

### Stage 1: Low-Risk Services (Week 13, 20–30 devices)

**Objective**: Deploy stateless services (GPIO control, configuration) with no data dependencies.

**Scope**: Add aeg-led, aeg-config

**Rollback Window**: 10 minutes

**Success Criteria**:
- ✓ LED control works (colors match expectations)
- ✓ No GPIO conflicts with Node.js legacy code
- ✓ Memory stable; no leaks
- ✓ 7-day uptime without crashes

---

### Stage 2: Medium-Risk Services (Week 14, 40–50 devices)

**Objective**: Deploy stateful services (database, sensors) with I/O side effects.

**Scope**: Add aeg-sensors, aeg-vrms (database writers)

**Rollback Window**: 1 hour (more complex; DB consistency check required)

**Success Criteria**:
- ✓ Sensor readings 100% accurate (≤ 1% error vs. Node.js)
- ✓ Database integrity maintained (PRAGMA integrity_check passes)
- ✓ Zero lost writes (all 360k writes persisted)
- ✓ Latency targets met (p95 < 20ms during peak load)

---

### Stage 3: High-Risk Services (Week 15, 60–70 devices, 2-week duration)

**Objective**: Deploy critical message pipeline (MQTT, firmware updates) with high uptime requirement.

**Scope**: Add aeg-mqtt, aeg-ota

**Rollback Window**: 2 hours (firmware update safety checks required)

**Success Criteria**:
- ✓ 100% MQTT message delivery (zero loss under peak load)
- ✓ OTA firmware updates 100% successful
- ✓ Cloud connectivity stable (no unexpected disconnects)
- ✓ 95th-percentile latency < 20ms sustained

**Extended Validation**:
- Run load tests: 1000 MQTT msg/sec + 100 REST req/sec for 24 hours
- Run soak tests: 7-day continuous operation (memory stable, no crashes)
- Run chaos tests: network faults, disk full, subprocess crashes (graceful recovery)

---

### Stage 4: Full Cutover (Week 16, 100% fleet)

**Objective**: Complete migration; decommission Node.js binary.

**Scope**: Deploy aeg-appserver (REST API); disable Node.js process

**Rollback Window**: 48 hours (emergency window; can revert to Node.js within 2 days)

**Success Criteria**:
- ✓ All 10 REST API endpoints respond < 10ms @ p95
- ✓ MQTT heartbeat maintained (no disruption to cloud)
- ✓ Zero unplanned service restarts over 24 hours
- ✓ Memory stable; CPU < 25% peak load
- ✓ Customer-facing metrics (uptime, latency, message delivery) match or exceed Node.js baseline

**Post-Cutover Monitoring**:
- Continuous monitoring for 2 weeks (extended observation)
- If all green: Decommission Node.js binary; declare migration complete
- If issues: Extend rollback window; investigate; potentially revert entire Stage 4

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
│         ┌─ Golden-Master Tests (1-week sample) ─┐
│         │  ├─ Output parity ✓                     │
│         │  ├─ Performance baseline captured ✓    │
│         │  └─ No regressions ✓                   │
│         └─────────────────────────────────────────┘
│                       ↓
│         ┌─ Extended Load/Soak Tests ─┐
│         │  (Only for Stage 3 & 4)    │
│         │  ├─ 1-hour load test ✓     │
│         │  ├─ 7-day soak test ✓      │
│         │  └─ Chaos tests ✓          │
│         └────────────────────────────┘
│                       ↓
│  [Automated Gates Passed] → Ready for Manual Deployment
│                       ↓
│  Stage 0–4 Deployment (Manual Approval Required)
│                       ↓
│  Field Monitoring (7–14 days per stage)
│                       ↓
│  Go/No-Go Decision (Operations + Tech Lead)
│
└─────────────────────────────────────────────────────────
```

---

## Rollback Procedure

### Automated Rollback (Stages 0–3)

```bash
# Automated trigger: If systemd health check fails 3 times in 5 min
systemctl --failed | grep aeg  # Check if Rust services failed

if [ <3 failures detected> ]; then
  echo "🚨 ROLLBACK TRIGGERED"
  
  # Stage 0–1: Simple service stop
  systemctl stop aeg-led aeg-config
  systemctl start node-gateway  # Resume Node.js
  
  # Stage 2: DB consistency check before rollback
  sqlite3 /var/lib/aeg/data.db "PRAGMA integrity_check"
  if [ $? -ne 0 ]; then
    # DB corrupted; restore from backup
    cp /var/backups/aeg/data.db.backup /var/lib/aeg/data.db
  fi
  systemctl stop aeg-sensors aeg-vrms
  systemctl start node-gateway
  
  # Stage 3: Firmware check before rollback
  fw_verify /var/lib/aeg/firmware/*
  systemctl stop aeg-mqtt aeg-ota
  systemctl start node-gateway
  
  # Alert team
  mail -s "🚨 Aurora Edge Gateway Rollback Triggered" team@example.com \
    < <(systemctl status aeg-*.service)
fi
```

### Manual Rollback (Stage 4, Emergency)

```bash
# Operator-initiated rollback within 48-hour window
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

| Stage | Duration | Fleet Size | Team Load | Rollback Window |
|---|---|---|---|---|
| 0 | 1 week | 5–10 | 1 on-call | 10 min |
| 1 | 1 week | 20–30 | 1 on-call | 10 min |
| 2 | 1 week | 40–50 | 2 on-call | 1 hour |
| 3 | 2 weeks | 60–70 | 2 on-call | 2 hours |
| 4 | 1 week | 100% | 3 on-call | 48 hours |
| **Total** | **~6 weeks** | **Progressive** | **Varies** | **Decreasing** |

---

## Communication Plan

### Internal (Team)
- **Daily standup** during Stages 0–4
- **Weekly risk review** (Tuesday 2 PM)
- **Post-stage retrospective** (identify issues; plan next stage)

### External (Customers)
- **Pre-announcement**: "Infrastructure improvements coming"
- **Stage 2 onward**: Real-time dashboard showing rollout progress
- **Incident notification**: If rollback triggered, brief message within 5 min
- **Post-cutover**: Release notes documenting improvements (performance, reliability)

---

## Go/No-Go Criteria per Stage

### Stage 0 → 1 Decision (7 days post-Stage 0)
- [ ] Zero crashes
- [ ] Memory stable (< 50 MB)
- [ ] Health checks 100% pass rate
- **Decision**: **GO** proceed to Stage 1

### Stage 1 → 2 Decision (7 days post-Stage 1)
- [ ] LED control 100% accurate
- [ ] Zero crashes; memory stable
- [ ] 7-day uptime achieved
- **Decision**: **GO** proceed to Stage 2

### Stage 2 → 3 Decision (7 days post-Stage 2)
- [ ] Sensor readings ≤ 1% error vs. baseline
- [ ] DB integrity checks pass
- [ ] Latency targets met (p95 < 20ms)
- [ ] Zero data loss incidents
- **Decision**: **GO** proceed to Stage 3

### Stage 3 → 4 Decision (14 days post-Stage 3)
- [ ] 100% MQTT message delivery
- [ ] OTA updates 100% successful
- [ ] 7-day soak test: no crashes, memory stable
- [ ] Stress test (1000 msg/sec + 100 HTTP req/sec): latency targets met
- [ ] Chaos tests (network faults, disk full, subprocess crashes): all handled gracefully
- **Decision**: **GO** proceed to Stage 4

### Stage 4 → Decommission Node.js Decision (14 days post-Stage 4)
- [ ] All REST API endpoints < 10ms @ p95
- [ ] Zero unplanned restarts
- [ ] Customer-reported SLI (99.9% availability, < 50ms latency): met or exceeded
- [ ] No significant bugs reported by customers
- **Decision**: **COMPLETE** migration; decommission Node.js

---

## Monitoring & Alerting

### Per-Stage Metrics
```yaml
stage_0:
  alerts:
    - service_crash_rate > 0
    - memory_usage > 50MB
    - health_check_fail_rate > 5%

stage_1:
  alerts:
    - service_crash_rate > 0
    - memory_growth > 0.1%/day
    - led_control_error_rate > 0.1%

stage_2:
  alerts:
    - database_integrity_fail
    - sensor_error_rate > 1%
    - latency_p95 > 50ms

stage_3:
  alerts:
    - mqtt_message_loss > 0.01%
    - ota_update_failure_rate > 0.1%
    - cloud_connectivity_drop > 5s

stage_4:
  alerts:
    - service_crash_rate > 0
    - api_latency_p95 > 20ms
    - memory_growth > 0.01%/day
```

---

## Related Decisions
- **[ADR-0006](0006-golden-master-testing.md)**: How golden-master tests feed into stage gates
- **[ADR-0002](0002-module-to-crate-decomposition.md)**: How independent services enable staged deployment

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Operations, product management, engineering  
**Implementation**: Stages 0–4 execute weeks 12–16 post-Phase A
