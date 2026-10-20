# Risk Register: Taxonomy & Management

## Overview

This document defines how risks are identified, categorized, tracked, and mitigated during the Node.js → Rust migration. A shared risk taxonomy ensures consistent communication across the team.

---

## Risk Categories

### Category 1: Technical Risks (Quality, Correctness, Performance)
Risks related to code quality, architectural correctness, or performance targets.

**Examples:**
- Behavioral parity not achieved (Rust outputs differ from Node.js)
- Performance targets not met (e.g., end-to-end per-cycle latency grows well beyond the established baseline — p95 56.4 ms, ~5.6% of the 1-second per-cycle budget, measured on the 4-core ARM Cortex-A53 target platform — or resident memory grows well beyond the measured full-stack baseline of 206 MB across 14 supervised services + Python subprocess)
- Memory leaks in Rust (misuse of unsafe blocks or circular references in Arc/Rc)
- Database corruption (SQLite PRAGMA integrity_check fails)

---

### Category 2: Operational Risks (Deployment, Monitoring, Rollback)
Risks related to deployment, rollout, and operational readiness.

**Examples:**
- Rollback procedure doesn't work (can't quickly restore Node.js)
- Metrics/monitoring missing (can't detect issues in production)
- Automated safety gates fail (tests pass but real-world behavior wrong)
- Systemd service configuration incorrect (services won't start)

---

### Category 3: Knowledge Risks (Team Expertise, Documentation)
Risks related to team capability and knowledge transfer.

**Examples:**
- Team lacks Rust expertise (inexperienced with tokio, async/await, borrow checker)
- Key person leaves during migration
- Codebase lacks documentation (hard to debug issues later)
- Insufficient training (team not confident maintaining Rust code post-launch)

---

### Category 4: External Risks (Dependencies, Ecosystem, Third-Party)
Risks related to external factors outside team control.

**Examples:**
- Dependency vulnerabilities (rumqttc, tokio, rusqlite have security issues)
- Ecosystem immaturity (some crate is unmaintained or flaky)
- Customer feedback reveals unexpected use case (hidden behavior in Node.js)
- Hardware incompatibility (Rust binary doesn't work on specific ARM variant)

---

## Risk Severity Levels

### 🔴 CRITICAL
**Impact**: Production outage or data loss

**Example**: Database corruption during migration; cannot recover customer data

**Recovery Time**: > 2 hours

**Go/No-Go Gate**: Must be resolved before Stage 3 (medium-risk services deployment)

---

### 🟠 HIGH
**Impact**: Service unavailable or severe degradation

**Example**: Memory leak causes device to crash after 2 days; requires manual restart

**Recovery Time**: 1–2 hours

**Go/No-Go Gate**: Must be resolved before Stage 2 (stateful services deployment)

---

### 🟡 MEDIUM
**Impact**: Partial functionality loss or resource degradation

**Example**: CPU usage drifts to > 75% under sustained load, leaving < 10% headroom

**Recovery Time**: 30 min – 2 hours

**Go/No-Go Gate**: Should be resolved before Stage 1; if deferred, must have workaround

---

### 🟢 LOW
**Impact**: Minor user-facing issue or technical debt

**Example**: LED color slightly off RGB expected value; documentation incomplete

**Recovery Time**: < 30 minutes

**Go/No-Go Gate**: Can be addressed in Phase B; no block on rollout

---

## Risk Registry Template

### Example Risk: RISK-001

| Field | Value |
|---|---|
| **Risk ID** | RISK-001 |
| **Category** | Technical (Correctness) |
| **Severity** | 🔴 CRITICAL |
| **Title** | Behavioral Parity: Sensor Aggregation Off By > 1% |
| **Description** | Rust sensor aggregation algorithm produces different values than Node.js baseline (e.g., 230.5V vs. 234.2V). This violates the Phase A requirement of "100% behavioral equivalence." |
| **Probability** | 35% (moderate — floating-point precision is a known issue) |
| **Impact** | Complete failure of Phase A golden-master tests; project blocked. |
| **Mitigation** | <ul><li>Port algorithm line-by-line from Node.js to Rust (match operator order, data types)</li><li>Use same floating-point precision (f64) as Node.js</li><li>Unit test against golden dataset</li><li>Compare intermediate values (not just final output)</li></ul> |
| **Status** | 🟡 In Progress (week 4 of Phase A) |
| **Owner** | Jane (Sensors module lead) |
| **Created** | 2026-09-15 |
| **Last Updated** | 2026-10-05 |
| **Mitigation Completion Target** | 2026-10-10 |

---

### Example Risk: RISK-007

| Field | Value |
|---|---|
| **Risk ID** | RISK-007 |
| **Category** | Technical (Performance) |
| **Severity** | 🟡 MEDIUM |
| **Title** | OTA Firmware Update Timeout on Slow Network |
| **Description** | OTA firmware download times out if network latency > 500ms. Real field networks have latency up to 2s. Results in failed firmware updates, stranded devices. |
| **Probability** | 60% (high — field networks are unreliable) |
| **Impact** | Customers cannot update devices in low-bandwidth deployments (e.g., satellite uplinks). |
| **Mitigation** | <ul><li>Implement adaptive timeout (start with 30s, increase to 5 min if slow network detected)</li><li>Add exponential backoff (3 retries with backoff: 1s, 5s, 30s)</li><li>Log detailed timeout diagnostics to help debug field issues</li></ul> |
| **Status** | 🟢 Resolved (implemented in week 6) |
| **Owner** | Bob (OTA module lead) |
| **Created** | 2026-09-20 |
| **Last Updated** | 2026-10-10 |
| **Mitigation Completion Target** | 2026-10-08 |

---

### Example Risk: RISK-012

| Field | Value |
|---|---|
| **Risk ID** | RISK-012 |
| **Category** | Technical (Quality) |
| **Severity** | 🟢 LOW |
| **Title** | Indicator Lamp Serial-Port Timing/Enumeration Differences |
| **Description** | The indicator lamp is controlled over a USB serial port (9600 baud, 8N1), not GPIO. The Rust serial crate (`tokio-serial` / `serialport`) may enumerate the USB device in a different order than Node.js on boot, or exhibit slightly different open/write timing and retry behavior on first connect. This could cause a brief command-ack latency difference or a transient failure to find the correct device path, without affecting the actual lamp color/state once connected. |
| **Probability** | 40% (moderate — USB enumeration order is not guaranteed stable across reboots on either stack) |
| **Impact** | Possible flaky integration tests around first-connect timing; no sustained user-visible impact once the port is open. |
| **Mitigation** | <ul><li>Pin the device by stable identifier (serial number / udev symlink) rather than relying on enumeration order</li><li>Add a bounded retry/backoff on initial port open, matching the Node.js stack's observed retry behavior</li><li>Add a tolerance window in timing-based integration tests</li></ul> |
| **Status** | 🟠 Deferred to Phase B (documented in FIXME(port) marker) |
| **Owner** | Carol (LED module lead) |
| **Created** | 2026-10-01 |
| **Last Updated** | 2026-10-10 |
| **Mitigation Completion Target** | Phase B (week 10) |

---

### Example Risk: RISK-013

| Field | Value |
|---|---|
| **Risk ID** | RISK-013 |
| **Category** | Operational (Disk/Storage) |
| **Severity** | 🟠 HIGH |
| **Title** | Disk Growth from WAV Capture and Service Logs |
| **Description** | WAV capture accumulates at roughly +3.8 GB per 3 hours (≈30 GB/day) on the target platform (28 GB total disk). A single service log (message-bus) reached 914 MB within a 7.8-hour soak run, and a sensor-data service log reached 203 MB in the same window. Before a disk-cleanup fix, the cleanup job was triggering roughly every 1.3 hours (≈18 times/day), indicating the disk fills fast enough to need very frequent intervention. |
| **Probability** | 100% (observed — this happened during soak testing, not hypothetical) |
| **Impact** | Disk exhaustion halts WAV capture and/or logging, and on a 28 GB disk with ~30 GB/day WAV growth, unmanaged accumulation would exhaust free space within roughly a day without cleanup. |
| **Mitigation** | <ul><li>Tune disk-cleanup cadence and retention so cleanup runs on a predictable schedule rather than reactively (post-fix cadence observed at roughly every 7 hours, ≈3.4 times/day)</li><li>Cap or rotate the message-bus and sensor-data service logs</li><li>Track disk-free trend as a first-class soak-test metric, not just crash/no-crash</li></ul> |
| **Status** | 🟢 Resolved (cleanup cadence corrected; monitored during subsequent soak runs) |
| **Owner** | Platform team |
| **Created** | 2026-09-xx |
| **Last Updated** | 2026-10-xx |
| **Mitigation Completion Target** | Complete |

---

### Example Risk: RISK-014

| Field | Value |
|---|---|
| **Risk ID** | RISK-014 |
| **Category** | Technical (Memory, Runtime) |
| **Severity** | 🟠 HIGH |
| **Title** | glibc Multi-Threaded Arena Growth in the Message-Bus Process |
| **Description** | Even in Rust, the message-bus process showed a linear memory increase of roughly 6.5 MB/hour during a 7.8-hour soak test (45.0 MB → 95.3 MB, +112%). Root cause was glibc's multi-threaded malloc arena behavior (64 arenas), not a Rust-level leak (no `unsafe` misuse or `Arc`/`Rc` cycle involved). This is a reminder that "written in Rust" does not automatically rule out native-allocator-level memory growth. |
| **Probability** | 100% (observed directly in soak testing) |
| **Impact** | Left unmitigated, this growth pattern would eventually degrade or crash the message-bus process on long-running edge deployments; only a multi-hour soak test surfaces it — short functional tests do not. |
| **Mitigation** | <ul><li>Set `MALLOC_ARENA_MAX=2` to cap glibc arena count</li><li>Re-run soak test to confirm: a subsequent 10.6-hour soak passed, with the memory trend ending at roughly −3.69 MB/hour</li><li>Make multi-hour soak testing (not just short functional tests) a standard gate before any stage that introduces a new long-running service (see ADR-0005 Stage 3)</li></ul> |
| **Status** | 🟢 Resolved (fix verified via 10.6-hour soak test) |
| **Owner** | Platform team |
| **Created** | 2026-09-xx |
| **Last Updated** | 2026-10-xx |
| **Mitigation Completion Target** | Complete |

---

## Risk Management Process

### Step 1: Risk Identification (Ongoing)
- **During Code Review**: Reviewers flag potential issues (`⚠️ This looks risky...`)
- **During Testing**: Test failures or anomalies logged as risks
- **During Architecture Review**: Known hard problems identified proactively

### Step 2: Risk Assessment (Within 24 Hours)
Team lead (or risk owner) assesses:
- **Probability**: How likely is this risk to occur? (0–100%)
- **Impact**: How bad is the outcome if it occurs?
- **Severity**: Combine probability + impact to assign severity level (CRITICAL / HIGH / MEDIUM / LOW)

### Step 3: Risk Mitigation (Immediately After Assessment)
- Define concrete mitigation steps (not "be more careful" — specific actions)
- Assign owner and deadline
- Update risk registry

### Step 4: Risk Tracking (Weekly)
- **Tuesday Risk Sync** (30 min): Review all open risks
- **Status Updates**: Owner provides progress on mitigation
- **Escalation**: If deadline at risk, escalate to team lead

### Step 5: Risk Resolution (When Complete)
- Owner confirms mitigation complete (with evidence: test results, code review approval)
- Update status to ✅ Resolved
- Archive risk in historical register

---

## Risk Register at a Glance

| Risk ID | Title | Category | Severity | Status | Owner | Target Date |
|---|---|---|---|---|---|---|
| RISK-001 | Behavioral Parity: Sensor Aggregation | Technical | 🔴 CRITICAL | In Progress | Jane | 2026-10-10 |
| RISK-003 | Message Loss Under Peak Load | Technical | 🔴 CRITICAL | Resolved | Bob | 2026-10-05 |
| RISK-005 | Tokio Thread Pool Contention | Technical | 🟠 HIGH | In Progress | Alice | 2026-10-15 |
| RISK-007 | OTA Update Timeout on Slow Network | Technical | 🟡 MEDIUM | Resolved | Bob | 2026-10-08 |
| RISK-010 | Team Rust Experience | Knowledge | 🟡 MEDIUM | Ongoing | Lead | Continuous |
| RISK-012 | LED Color RGB Precision | Technical | 🟢 LOW | Deferred to Phase B | Carol | 2026-10-25 |
| RISK-013 | Disk Growth from WAV Capture and Service Logs | Operational | 🟠 HIGH | Resolved | Platform team | Complete |
| RISK-014 | glibc Arena Growth in Message-Bus Process | Technical | 🟠 HIGH | Resolved | Platform team | Complete |
| RISK-015 | rumqttc Dependency Vulnerability | External | 🟡 MEDIUM | Monitoring | Alice | TBD (dependency update) |

---

## Risk Escalation Procedure

### If Risk Is Unresolvable By Deadline

1. **Notify Team Lead** (same day as deadline miss)
2. **Reclassify Severity** if appropriate (might be less critical than initially thought)
3. **Choose Resolution Option**:
   - **Option A**: Defer to Phase B (document as TODO(port) in code)
   - **Option B**: Implement workaround (e.g., disable feature, add circuit breaker)
   - **Option C**: Halt Stage X; investigate root cause

**Example**: RISK-012 (LED color) could not be resolved by original deadline; decision was made to defer to Phase B because impact is cosmetic (LOW severity).

---

## Known Risks (Examples for Interview Context)

These are example risks for an **illustrative** Aurora Edge Gateway rewrite; they're not based on the real confidential project:

### Technical Example Risks
1. **Floating-Point Precision Loss**: Sensor readings (voltage, current) computed with slightly different precision in Rust vs. Node.js → affects 0.01% of readings → LOW severity; mitigated by accepting tolerance in golden-master tests

2. **Async/Await Deadlock**: Tokio task tries to acquire two locks in different order than Node.js event loop → potential deadlock under load → HIGH severity; mitigated by explicit lock ordering enforcement and stress testing

3. **Memory Fragmentation**: Long-running edge device allocates/deallocates memory; after 30 days heap is fragmented → out-of-memory crash → MEDIUM severity; mitigated by profiling allocator behavior and using jemalloc

### Operational Example Risks
1. **Rollback Tooling Missing**: No automated procedure to switch from Rust back to Node.js → manual recovery could take hours → HIGH severity; mitigated by testing rollback procedure weekly

2. **Monitoring Gaps**: Prometheus metrics missing for a new crate → cannot see performance issues until customer reports them → MEDIUM severity; mitigated by mandatory metrics in review checklist

### Knowledge Example Risks
1. **Rust Compiler Learning Curve**: Team takes 4+ weeks to understand borrow checker → development slower than estimated → MEDIUM severity; mitigated by pair programming, doc reviews, training sessions

2. **Key Person Dependency**: Only one engineer fully understands async/tokio architecture → if they leave, codebase at risk → HIGH severity; mitigated by documented architecture, multiple reviewers on async code

---

## Risk Acceptance & Sign-Off

### Before Each Stage Deployment

**Risk Review Board** (Tech Lead, Architect, Operations Lead) meets to:
1. Review all **CRITICAL** and **HIGH** severity risks → must be resolved or documented with explicit acceptance
2. Confirm **MEDIUM** severity risks have mitigation or acceptable workaround
3. Accept **LOW** severity risks or defer to Phase B
4. **Sign off**: "We accept the remaining risks and approve deployment to Stage X"

**Sign-Off Example**:
```
Risk Review for Stage 3 (Medium-Risk Services)
Date: 2026-10-15
Attendees: Jane (Tech Lead), Bob (Arch), Carol (Ops)

✅ RISK-001 (Sensor Parity): RESOLVED
✅ RISK-003 (Message Loss): RESOLVED
⏳ RISK-005 (Thread Contention): In Progress; mitigation on track
   → ACCEPTED with caveat: Must be resolved before Stage 4
✅ RISK-007 (OTA Timeout): RESOLVED
✅ RISK-010 (Team Expertise): ONGOING; training on track
   → ACCEPTED; team confident
⏸️ RISK-012 (LED Color): DEFERRED to Phase B
   → ACCEPTED; cosmetic only

DECISION: APPROVED for Stage 3 deployment
(Signed) Jane, Bob, Carol
```

---

## Continuous Risk Monitoring

- **Code commits**: Developers flag new risks in commit messages or PR comments
- **Test failures**: Failed tests reviewed for underlying risks
- **Production incidents**: Post-mortems add new risks to register
- **Team retrospectives**: Risk register reviewed quarterly

---

## Next Steps

1. **Marker Conventions** → See **[ADR-0007](adr/0007-port-marker-convention.md)** for how to mark risks in code
2. **ADRs** → See all ADRs for architectural decisions and associated risks

---

**Last Updated**: 2026-10-20  
**Status**: Risk register process approved  
**Next Review**: Weekly (Tuesday 2 PM sync)
