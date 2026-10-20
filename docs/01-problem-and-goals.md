# Problem Statement & Goals: Why Rewrite Aurora Edge Gateway in Rust?

## Executive Summary

**Aurora Edge Gateway** is a distributed industrial IoT platform responsible for real-time sensor data aggregation, edge-side computation, and device control on resource-constrained ARM edge hardware. The original Node.js implementation, while feature-complete, exhibited three critical operational weaknesses that threatened production reliability and scalability:

1. **Memory leaks** in long-running sensor pipelines (observable multi-day uptime degradation)
2. **High CPU overhead** during peak sensor-event processing (tail latencies > 100ms on 8-core ARM)
3. **Unpredictable crashes** after 7–14 days of continuous operation under load

This document outlines the business and technical rationale for a comprehensive rewrite in Rust, the measurable goals, and the trade-offs.

## Problem: The Node.js Bottleneck

### Symptom 1: Memory Pressure Under Continuous Load
- **Observation**: Edge gateways exhibit memory growth over days; occasional GC-induced pauses cause latency spikes
- **Impact**: System degradation after 3–7 days; manual restart required to restore performance
- **Root cause**: 
  - Node.js garbage collection (100–500 ms pauses observed in testing)
  - Circular references in event-stream aggregation; GC doesn't fully reclaim memory
- **Production baseline**: ~1400 MB idle, ~1500–1600 MB under 3h active load on Genio 350 (4GB nominal, 2.84GB usable)

### Symptom 2: CPU Utilization & Load Variability
- **Observation**: 
  - Idle CPU: ~50–55%
  - Active CPU (3h test): ~64% average, occasional spikes to 100%
  - Load average: 3.3 on 4-core ARM (0.825/core); GC events cause transient load peaks
- **Impact**: Limited headroom for concurrent sensor processing; unpredictable response times during GC pauses
- **Root cause**: Node.js event loop + V8 JIT overhead + garbage collection timing unpredictable

### Symptom 3: Non-Deterministic Behavior
- **Observation**: GC pauses and event-loop scheduling make timing unpredictable
- **Impact**: Hard to diagnose performance issues; no compile-time safety guarantees
- **Root cause**: Dynamic runtime, implicit memory management, JavaScript semantics

## Business Goals

### Primary Objectives
1. **Eliminate Memory Bloat**: Achieve stable memory footprint over 30-day continuous operation (current: ~1500 MB under load; target: maintain < 1500 MB with no GC-induced growth)
2. **Reduce CPU Variability**: Deterministic CPU usage without GC pauses (current: 64% average + transient 100% spikes; target: consistent 50–65% with < 5% variability)
3. **Improve Operational Stability**: Support 90+ day continuous uptime without manual restart; no cascading failures from GC pauses

### Secondary Objectives
1. **Compile-Time Safety**: Eliminate memory-safety bugs (buffer overflow, use-after-free, data races) via language guarantees
2. **Predictable Performance**: No stop-the-world pauses; all resource cleanup deterministic (no garbage collection timing)
3. **Enable Future Scale**: Prepare architecture for 10x sensor count without increasing per-device resource footprint

## Technical Goals

### Correctness & Safety
- **Memory safety**: Eliminate buffer overflows, use-after-free, and data races via Rust's type system
- **Determinism**: No GC pauses; all resource cleanup explicit (deterministic drop semantics)
- **Behavioral parity**: New system must handle 100% of use cases of the old system (no silent feature loss)

### Performance Targets (Based on Real Measurements)
| Metric | Node.js (Baseline) | Rust (Target) | Notes |
|---|---|---|---|
| **CPU usage (3h active load)** | 64.1% avg on 4-core A53 | Maintain 50–65% avg | Eliminate transient 100% GC spikes |
| **Memory (idle)** | ~1400 MB | < 1400 MB | Tighter baseline |
| **Memory (3h active)** | ~1538 MB avg, growth stable | < 1500 MB, strictly stable | No GC-induced heap bloat |
| **Load average (3h)** | 3.30 (0.825/core headroom) | Maintain 3.0–3.5 (18% headroom) | Deterministic; no GC contention |
| **GC pauses** | 100–500 ms observed | 0 ms | Deterministic cleanup only |
| **Uptime (continuous)** | 3–7 days before degradation | 90+ days stable | No manual restart required |

### Deployment & Operations
- Support incremental rollout (Stage 0–4) with automatic rollback
- Maintain backward compatibility with existing MQTT clients and REST API consumers
- Reduce operational toil (monitoring, alerting, manual restarts)

## Constraints & Risks

### Constraints
1. **No Breaking API Changes**: Existing client code must continue to work unchanged
2. **Gradual Migration**: Cannot force all customers to upgrade simultaneously
3. **Team Expertise**: Team has limited Rust experience; ramp-up time required
4. **Maintenance Burden**: Will maintain both systems during transition (2–6 months)

### Known Risks
1. **Longer Development Timeline**: Rewrite will take 2–3 months vs. quick patches on Node.js
2. **New Tool Chain Overhead**: Build system, dependency management, cross-compilation for ARM edge devices
3. **Testing Burden**: Must validate behavioral parity exhaustively (golden-master tests, load testing, long-soak testing)

## Alternatives Considered

### Option A: "Throw More Memory" (Node.js + Cluster Mode)
- **Approach**: Run multiple Node.js processes with load balancer; add caching and memory limits
- **Pros**: Lower engineering effort; leverages existing codebase
- **Cons**: 
  - Masks the root cause; memory leaks still present per process
  - Increases per-device resource footprint (128 MB → 256 MB or more)
  - Does not solve tail-latency problem
- **Decision**: Rejected — temporary band-aid, not a solution

### Option B: "Go or C++" (Alternative Compiled Language)
- **Approach**: Rewrite in Go (easier learning curve) or C++ (maximum performance)
- **Pros**: Go offers faster ramp-up; C++ offers maximum optimization
- **Cons**: 
  - Go: Still has GC (doesn't solve memory-leak problem fully); concurrency complexity
  - C++: Steep learning curve, memory-safety burden remains on team
  - Both: Smaller ecosystem for IoT/industrial protocols than Rust
- **Decision**: Rejected — Rust's memory-safety guarantees and async ecosystem better aligned with goals

### Option C: Rewrite in Rust (Selected)
- **Approach**: Complete architectural rewrite in Rust with incremental staged rollout
- **Pros**: 
  - Eliminates memory leaks by design
  - Zero-cost abstractions for performance
  - Excellent async/concurrency primitives (tokio)
  - Rich IoT ecosystem (mqtt, serial protocols, cross-compilation support)
  - Learning opportunity for team
- **Cons**: 
  - Higher upfront engineering effort
  - Team ramp-up on new language
  - Longer development timeline
- **Decision**: Selected — best long-term bet for stability and performance

## Success Metrics

### Phase A (Faithful Port)
- ✓ 100% functional parity with Node.js (all routes, MQTT topics, data formats)
- ✓ Behavioral equivalence validated via golden-master tests on 3+ weeks of production traffic replays
- ✓ No new customer-facing bugs introduced
- ✓ Memory footprint stable (idle: < 1400 MB; active: < 1500 MB after warm-up)

### Phase B (Hardening & Optimization)
- ✓ CPU usage stable: 50–65% average on 3h active load (no transient 100% GC spikes)
- ✓ Memory growth rate < 0.01% per day over 30-day continuous operation
- ✓ Zero unplanned restarts over 90-day field trial
- ✓ Load average < 1/core (headroom maintained); GC pauses eliminated (0 ms)
- ✓ Deterministic timing: no latency variance from garbage collection

### Operational
- ✓ Staged rollout completes without customer-blocking incidents
- ✓ Automated rollback triggers prevent sustained outages
- ✓ Team confident to maintain and extend Rust codebase (training, knowledge base, runbook)

## Scope & Timeline

### Scope
- **In**: Core gateway platform (all modules, all MQTT clients, all REST APIs, persistent storage, OTA updates)
- **Out**: Web UI frontend (stays Node.js for now), ML inference backend (stays Python)

### Estimated Timeline
- **Phase A (Faithful Port)**: 6–8 weeks (module-by-module port + golden-master testing + memory/CPU profiling)
- **Phase B (Stabilization)**: 3–4 weeks (soak testing, GC/memory behavior validation, determinism verification)
- **Staged Rollout**: 2–4 weeks (Stage 0 → 4, performance monitoring, customer validation)
- **Total**: 3–4 months end-to-end

## Next Steps

1. Detailed architecture review → `02-architecture-overview.md`
2. Migration strategy & rollout plan → `03-migration-strategy.md`
3. Testing approach → `04-testing-strategy.md`
4. See also: **[ADR-0001: Why Rewrite in Rust?](adr/0001-why-rewrite-in-rust.md)** for formal decision record

---

**Status**: Approved  
**Decision Date**: 2026-Q3  
**Next Review**: Post-Phase A completion
