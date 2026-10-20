# Risk Register: Taxonomy & Management

## Overview

This document defines how risks are identified, categorized, tracked, and mitigated during the Node.js → Rust migration. A shared risk taxonomy ensures consistent communication across the team.

---

## Risk Categories

### Category 1: Technical Risks (Quality, Correctness, Performance)
Risks related to code quality, architectural correctness, or performance targets.

**Examples:**
- Behavioral parity not achieved (Rust outputs differ from Node.js)
- Performance targets not met (latency > 10ms, memory > 40MB)
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
**Impact**: Partial functionality loss or performance degradation

**Example**: Sensor reads delayed 50ms under peak load (target is < 10ms)

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
| **Title** | LED Color Rendering Slightly Different (RGB Precision) |
| **Description** | Rust GPIO library (rppal) reports slightly different RGB values than Node.js rpi.GPIO (e.g., 255 vs. 254 for full red). Visually imperceptible but fails strict color-match tests. |
| **Probability** | 75% (very likely — GPIO libraries have different precision) |
| **Impact** | Test failures in color verification tests; no real user impact (colors look identical). |
| **Mitigation** | <ul><li>Accept ±1 tolerance in color tests (allow 255 ≈ 254)</li><li>Or: Normalize values on both sides to same scale before comparing</li></ul> |
| **Status** | 🟠 Deferred to Phase B (documented in FIXME(port) marker) |
| **Owner** | Carol (LED module lead) |
| **Created** | 2026-10-01 |
| **Last Updated** | 2026-10-10 |
| **Mitigation Completion Target** | Phase B (week 10) |

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
