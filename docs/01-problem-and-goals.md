# Problem Statement & Goals: Why Rewrite Aurora Edge Gateway in Rust?

## Executive Summary

**Aurora Edge Gateway** is a distributed industrial IoT platform responsible for real-time sensor data aggregation, edge-side computation, and device control on resource-constrained ARM edge hardware (4-core ARM Cortex-A53, nominal 4 GB RAM with ~2.84 GB OS-usable, no swap). The original Node.js implementation, while feature-complete, exhibited three critical operational weaknesses:

1. **High baseline memory usage** relative to the device's available RAM, driven by a 16-service / 8-startup-stage process topology (measured: 1538 MB during a 3-hour active-load window, ~54% of ~2.84 GB OS-usable RAM)
2. **High CPU overhead** during peak sensor-event processing (measured: 52% idle, 64% during the same 3-hour active-load window, on 4 cores)
3. **High load average with little headroom**, driven by the same multi-process topology (measured: 3.30 average on 4 cores during the 3-hour active-load window)

This document outlines the business and technical rationale for a comprehensive rewrite in Rust, the measurable goals, and the trade-offs.

## Problem: The Node.js Bottleneck

### Symptom 1: Memory Pressure Relative to Available RAM
- **Observation**: The 16-service / 8-startup-stage Node.js topology (roughly 20 OS processes on the production device) holds 1538 MB resident during a 3-hour active-load window — about 54% of the ~2.84 GB OS-usable RAM on the 4-core ARM Cortex-A53 target platform (nominal 4 GB RAM, no swap). Boot to full service availability takes roughly 2 minutes 30 seconds.
- **Impact**: Little memory headroom for traffic spikes or future sensor-count growth; per-process overhead is duplicated across ~20 processes instead of being shared
- **Root cause**:
  - One OS process per Node.js service multiplies baseline runtime/heap overhead across the fleet
  - Node.js garbage collection adds further heap overhead on top of the per-process baseline
- **Current baseline (measured)**: 1538 MB during a 3-hour active-load window (4-core ARM Cortex-A53, ~54% of ~2.84 GB OS-usable RAM); see `02-architecture-overview.md` and `03-migration-strategy.md` for the Rust comparison (206 MB full-stack resident, 14 supervised services + Python subprocess)

### Symptom 2: CPU Utilization & Load Headroom
- **Observation**:
  - Idle CPU: 52% (4-core ARM Cortex-A53)
  - Active CPU under a 3-hour active-load window: 64% (4-core)
  - Load average: 3.30 on 4 cores during the same window — little headroom above the 4-core ceiling
- **Impact**: Limited headroom for concurrent sensor processing under traffic spikes or fleet growth
- **Root cause**: Node.js event loop + V8 JIT overhead, multiplied across the ~20-process topology, plus garbage-collection scheduling

### Symptom 3: Operational Complexity of the Multi-Process Topology
- **Observation**: 16 services across 8 startup stages (roughly 20 OS processes) must be sequenced, monitored, and restarted independently; garbage-collection timing adds further scheduling variability within each process
- **Impact**: Harder to diagnose cross-process performance issues; no compile-time memory-safety guarantees within each process
- **Root cause**: Dynamic runtime, implicit memory management, JavaScript semantics, and a process-per-service deployment model

## Business Goals

### Primary Objectives
1. **Reduce Memory Footprint**: Collapse the per-process Node.js topology (1538 MB, ~20 processes, 3h active load) into a small set of supervised Rust services with a much smaller resident footprint
2. **Reduce CPU & Load Average**: Lower steady-state CPU usage and load average versus the Node.js baseline (52% idle / 64% active, load average 3.30 on 4 cores)
3. **Improve Operational Stability**: Demonstrate stable, low-drift resource usage over soak testing (longest validated run to date: 10.6 hours); no cascading failures from GC pauses

### Secondary Objectives
1. **Compile-Time Safety**: Eliminate memory-safety bugs (buffer overflow, use-after-free, data races) via language guarantees
2. **Predictable Performance**: No stop-the-world pauses; all resource cleanup deterministic
3. **Enable Future Scale**: Prepare architecture for growth in sensor count without increasing per-device resource footprint

## Technical Goals

### Correctness & Safety
- **Memory safety**: Eliminate buffer overflows, use-after-free, and data races via Rust's type system
- **Determinism**: No GC pauses; all resource cleanup explicit (deterministic drop semantics)
- **Behavioral parity**: New system must handle 100% of use cases of the old system (no feature loss)

### Performance Targets
| Metric | Node.js Baseline (4-core ARM Cortex-A53, 3h active load) | Rust Measured (same 4-core ARM Cortex-A53 platform) | Notes |
|---|---|---|---|
| **Memory usage** | 1538 MB (~54% of ~2.84 GB OS-usable RAM) | 206 MB full-stack resident (14 supervised services + Python subprocess), measured over a 7.8h soak | Collapse per-process overhead into a small set of supervised services |
| **CPU usage** | 52% idle / 64% active | 6.3% average / 8.8% peak, zero drift over a 7.8h soak | No GC pause jitter |
| **Load average (4 cores)** | 3.30 average | 0.50 average / 1.81 peak over a 7.8h soak | Large headroom margin |
| **Per-second pipeline latency** | Not separately characterized | p95 56.4 ms, ~5.6% of the 1-second per-cycle budget, zero samples over 500 ms (285-sample measurement run) | Tail latency well inside budget |
| **Extended soak validation** | N/A | Longest continuous run to date: 10.6 hours, PASS | No 90-day or multi-month continuous-run data exists for this project |

### Deployment & Operations
- Support incremental rollout with a reliable rollback path (binary switch + manual SOP, rollback execution within 5 minutes overall, 2–4 minutes per stage)
- Maintain backward compatibility with existing MQTT clients and REST API consumers
- Reduce operational toil (monitoring, alerting, manual restarts)

## Constraints & Risks

### Constraints
1. **No Breaking API Changes**: Existing client code must continue to work unchanged
2. **Gradual Migration**: Cannot force all customers to upgrade simultaneously
3. **Team Expertise**: Team has limited Rust experience; ramp-up time required
4. **Maintenance Burden**: Will maintain both systems during transition

### Known Risks
1. **Longer Development Timeline**: Rewrite requires sustained engineering effort
2. **New Tool Chain Overhead**: Build system, dependency management, cross-compilation for ARM
3. **Testing Burden**: Must validate behavioral parity exhaustively (golden-master testing)

## Alternatives Considered

### Option A: "Throw More Memory" (Node.js + Cluster Mode)
- **Approach**: Run multiple Node.js processes with load balancer; add caching
- **Pros**: Lower engineering effort; leverages existing codebase
- **Cons**: 
  - Does not address the root cause; per-process runtime and heap overhead stays duplicated rather than shared
  - Increases per-device resource footprint
  - Does not solve latency problem
- **Decision**: Rejected — temporary band-aid, not a solution

### Option B: "Go or C++" (Alternative Compiled Language)
- **Approach**: Rewrite in Go (easier learning curve) or C++ (maximum performance)
- **Pros**: Go offers faster ramp-up; C++ offers maximum optimization
- **Cons**: 
  - Go: Still has GC (doesn't solve memory-leak problem fully)
  - C++: Steep learning curve, memory-safety burden remains on team
  - Both: Smaller IoT ecosystem than Rust
- **Decision**: Rejected — Rust's guarantees better aligned with goals

### Option C: Rewrite in Rust (Selected)
- **Approach**: Complete architectural rewrite in Rust with incremental staged rollout
- **Pros**: 
  - Eliminates use-after-free, double-free, and data races at compile time
  - Zero-cost abstractions for performance
  - Excellent async/concurrency primitives
  - Rich IoT ecosystem
  - Learning opportunity for team
- **Cons**: 
  - Higher upfront engineering effort
  - Team ramp-up on new language
  - Longer development timeline
- **Decision**: Selected — best long-term bet for stability

## Success Metrics

### Phase A (Faithful Port)
- ✓ 100% functional parity with Node.js (all routes, MQTT topics, data formats)
- ✓ Behavioral equivalence validated via golden-master tests
- ✓ No new customer-facing bugs introduced
- ✓ Memory and CPU profiling baseline captured for Phase B comparison

### Phase B (Hardening & Optimization)
- ✓ CPU usage measured at 6.3% average / 8.8% peak over a 7.8h soak (vs. 52% idle / 64% active on Node.js)
- ✓ Memory growth negligible over the soak window, with the one sustained-growth issue found during soak testing fixed (see `04-testing-strategy.md`)
- ✓ Longest continuous soak to date (10.6h) passes with zero restarts in the final stretch
- ✓ Latency measured at p95 56.4 ms (~5.6% of the 1-second budget), zero samples over 500 ms
- ✓ Team capable of independent maintenance and enhancement

### Operational
- ✓ Staged rollout completes without customer-blocking incidents
- ✓ Binary-switch rollback with a documented manual SOP completes within 5 minutes per stage, preventing sustained outages
- ✓ Team confident to maintain and extend Rust codebase

## Scope & Timeline

### Scope
- **In**: Core gateway platform (all modules, all MQTT clients, all REST APIs, persistent storage, OTA updates)
- **Out**: Web UI frontend (stays JavaScript), ML inference backend (stays Python)

### Estimated Timeline
- **Staged Rollout (Stage 0 → 4)**: ~22 weeks total, with Phase A (faithful port, module-by-module + golden-master testing) and Phase B (hardening, performance validation) as methodological lenses applied throughout — not separate time blocks before the rollout
- **Total**: ~22 weeks end-to-end (see `03-migration-strategy.md` for the stage-by-stage breakdown)

## Next Steps

1. Detailed architecture review → `02-architecture-overview.md`
2. Migration strategy & rollout plan → `03-migration-strategy.md`
3. Testing approach → `04-testing-strategy.md`
4. See also: **[ADR-0001: Why Rewrite in Rust?](adr/0001-why-rewrite-in-rust.md)** for formal decision record

---

**Status**: Approved  
**Decision Date**: 2026-Q3  
**Next Review**: Post-Phase A completion
