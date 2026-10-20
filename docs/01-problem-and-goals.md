# Problem Statement & Goals: Why Rewrite Aurora Edge Gateway in Rust?

## Executive Summary

**Aurora Edge Gateway** is a distributed industrial IoT platform responsible for real-time sensor data aggregation, edge-side computation, and device control on resource-constrained ARM edge hardware. The original Node.js implementation, while feature-complete, exhibited three critical operational weaknesses that threatened production reliability and scalability:

1. **Memory leaks** in long-running sensor pipelines (observable multi-day uptime degradation)
2. **High CPU overhead** during peak sensor-event processing (tail latencies > 100ms on 8-core ARM)
3. **Unpredictable crashes** after 7–14 days of continuous operation under load

This document outlines the business and technical rationale for a comprehensive rewrite in Rust, the measurable goals, and the trade-offs.

## Problem: The Node.js Bottleneck

### Symptom 1: Memory Leaks Under Continuous Load
- **Observation**: Edge gateways deployed in the field exhibited ~2% memory growth per day under normal operation
- **Impact**: A 256 MB device would become unstable after ~7 days; required manual restart
- **Root cause**: Circular references in event-stream aggregation loops; garbage collection pauses did not fully reclaim memory
- **Attempted fixes**: Switched GC strategies, added manual cleanup code → temporary relief, but fundamental issue remained

### Symptom 2: Tail Latency Spikes During Peak Load
- **Observation**: 95th-percentile event processing latency: **85–120 ms** (target: < 10 ms)
- **Impact**: Real-time sensor-driven triggers (e.g., equipment alarm thresholds) missed tight response deadlines
- **Root cause**: 
  - Node.js event loop contention during high-concurrency MQTT message bursts
  - V8 JIT compilation overhead on first-run code paths
  - Garbage collection pauses (100–500 ms) blocking sensor reads
- **Attempted fixes**: Worker threads, clustering, async/await refactoring → partial improvement, but architectural limit reached

### Symptom 3: Unexplained Process Crashes
- **Observation**: Gateways crash without logs after 7–14 days of continuous load
- **Root cause**: Likely stack overflow in deeply nested async call chains; difficult to diagnose in production
- **Impact**: 15–20 minute downtime per incident; loss of buffered sensor data

## Business Goals

### Primary Objectives
1. **Eliminate Memory Leaks**: Achieve zero unplanned restarts due to memory pressure over 30-day continuous operation
2. **Reduce Latency Tail**: 95th-percentile event processing latency < 10 ms, even during peak sensor bursts
3. **Improve Operational Stability**: Support 90+ day continuous uptime without manual intervention

### Secondary Objectives
1. **Reduce CPU Footprint**: Lower power consumption on edge devices (relevant for battery-backed or thermally constrained deployments)
2. **Simplify Debugging**: Deterministic behavior and compile-time checking reduce production incidents
3. **Enable Future Scale**: Prepare architecture for 10x sensor count growth without redesign

## Technical Goals

### Correctness & Safety
- **Memory safety**: Eliminate buffer overflows, use-after-free, and data races via Rust's type system
- **Determinism**: No undefined behavior; all control flow and resource management explicit
- **Behavioral parity**: New system must handle 100% of use cases of the old system (no silent feature loss)

### Performance Targets
| Metric | Node.js (Baseline) | Rust (Target) | Improvement |
|---|---|---|---|
| Avg event latency | 15 ms | < 2 ms | 7.5× |
| 95th-percentile latency | 100 ms | < 10 ms | 10× |
| Memory footprint | 80 MB (idle) | 25 MB (idle) | 3.2× |
| RSS growth (per day, under load) | ~2% | < 0.1% | 20× |
| Max GC pause | 500 ms | 0 ms (deterministic) | Unbounded |
| CPU usage (peak) | 85% on 8-core ARM | 25% on 8-core ARM | 3.4× |

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

### Phase B (Hardening & Optimization)
- ✓ Memory growth < 0.1% per day under continuous load over 30 days
- ✓ 95th-percentile event latency < 10 ms on full production workload
- ✓ Zero unplanned restarts over 90-day field trial
- ✓ CPU usage reduced by ≥ 50% vs. baseline at same throughput

### Operational
- ✓ Staged rollout completes without customer-blocking incidents
- ✓ Automated rollback triggers prevent sustained outages
- ✓ Team confident to maintain and extend Rust codebase (training, knowledge base, runbook)

## Scope & Timeline

### Scope
- **In**: Core gateway platform (all modules, all MQTT clients, all REST APIs, persistent storage, OTA updates)
- **Out**: Web UI frontend (stays Node.js for now), ML inference backend (stays Python)

### Estimated Timeline
- **Phase A (Faithful Port)**: 6–8 weeks (module-by-module port + golden-master testing)
- **Phase B (Hardening)**: 3–4 weeks (optimization, load testing, long-soak validation)
- **Staged Rollout**: 2–4 weeks (Stage 0 → 4, automated tests, customer validation)
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
