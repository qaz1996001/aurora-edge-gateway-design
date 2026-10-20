# ADR-0001: Why Rewrite Aurora Edge Gateway in Rust?

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Engineering team, product management, operations  

---

## Context

Aurora Edge Gateway is an industrial IoT data aggregation platform deployed on constrained ARM edge devices (256 MB–1 GB RAM). The original Node.js implementation achieved feature parity but exhibited three critical production issues:

1. **Memory leaks**: 2% daily growth; devices required manual restart every 7–14 days
2. **Tail latency**: 95th-percentile event processing latency ~100 ms (target: < 10 ms)
3. **Unpredictable crashes**: Process died after 7–14 days under continuous load; root cause difficult to diagnose

These issues limit customer deployments to short-lived edge scenarios and prevent scaling to high-sensor-count deployments.

---

## Decision

**Rewrite Aurora Edge Gateway in Rust** using incremental Phase A/B rollout strategy (see ADR-0005, ADR-0006).

---

## Rationale

### Why Rust?

**Memory Safety by Design**
- Rust's type system eliminates undefined behavior, memory leaks, and data races at compile time
- No garbage collector → no GC pauses blocking sensor reads
- Explicit resource management via drop trait → guaranteed cleanup
- **Result**: Zero unplanned restarts due to memory pressure

**Exceptional Performance**
- Zero-cost abstractions (compiled to machine code, no interpreter overhead)
- Async/await with minimal overhead (tokio runtime)
- Fine-grained control over allocations (no hidden allocations in standard library)
- **Result**: 95th-percentile latency < 10 ms; CPU footprint 3× lower

**Deterministic Behavior**
- No stop-the-world GC pauses
- All concurrency explicit (no magic event loop hiding issues)
- Compiler forces handling of errors explicitly
- **Result**: Predictable, debuggable behavior in production

**Excellent IoT/Systems Ecosystem**
- `tokio`: Industry-standard async runtime for high-concurrency workloads
- `rumqttc`: Mature MQTT client library (async-first)
- `rusqlite`: Synchronous SQLite bindings (no complex async DB driver needed)
- `axum`: Modern HTTP framework (zero-cost routing, type-safe)
- `serde`: Zero-copy serialization
- **Result**: All required technologies available with good maturity

---

### Alternatives Considered

#### Option A: "Throw More Memory" (Node.js + Clustering)
- **Approach**: Run multiple Node.js processes; add caching; increase memory limits
- **Pros**: Lower engineering effort (no rewrite); leverages existing code
- **Cons**: 
  - Masks the root cause (memory leaks still present per process)
  - Increases per-device footprint (128 MB → 256+ MB); infeasible on 256 MB devices
  - Does not solve tail-latency or stability problems
- **Decision**: Rejected — temporary band-aid; doesn't address fundamental issues

#### Option B: Go (Compiled Alternative)
- **Approach**: Rewrite in Go (easier learning curve than Rust)
- **Pros**: Faster team ramp-up; compiled; good concurrency (goroutines)
- **Cons**: 
  - Still has garbage collection (doesn't solve memory-leak problem fully)
  - Concurrency debugging harder than Rust's explicit model
  - Smaller IoT/edge ecosystem than Rust
- **Decision**: Rejected — doesn't eliminate memory leaks; less ideal for constrained hardware

#### Option C: C++ (Maximum Performance)
- **Approach**: Rewrite in C++ (maximum control, maximum performance)
- **Pros**: Absolute maximum performance; fine-grained memory control
- **Cons**: 
  - Steep learning curve; memory-safety burden remains on team
  - Higher risk of introducing new vulnerabilities
  - Build system complexity
  - Slower development velocity
- **Decision**: Rejected — overkill complexity for this workload; Rust achieves same goals more safely

#### Option D: Rust (Selected) ✅
- **Approach**: Full rewrite in Rust with phased rollout
- **Pros**: 
  - Eliminates memory leaks by design
  - Zero-cost abstractions (performance on par with C++)
  - Excellent async/concurrency primitives
  - Rich IoT ecosystem
  - Team learning opportunity (valuable skill)
  - Strong compile-time guarantees reduce production issues
- **Cons**: 
  - Higher upfront engineering effort (6–8 weeks Phase A)
  - Team ramp-up on new language
  - Longer initial development timeline
- **Decision**: Selected — best long-term investment; addresses all three production issues

---

## Consequences

### Positive
1. **Operational Reliability**: 90+ day uptime achievable (vs. 7–14 days with Node.js)
2. **Performance**: 95th-percentile latency < 10 ms (vs. 100 ms); 3× lower CPU footprint
3. **Reduced Maintenance Burden**: Zero unplanned restarts; simpler alerting
4. **Team Growth**: Engineers learn systems programming language; value for future projects
5. **Future Scalability**: Can support 10× sensor count without redesign

### Negative
1. **Development Time**: 3–4 month rewrite (vs. quick patches on Node.js)
2. **Team Learning Curve**: Rust borrow checker, lifetime, async/await unfamiliar (2–3 week ramp-up)
3. **Dual Maintenance**: Must support both systems during rollout (3–4 months overhead)
4. **Risk**: Rewrite always carries risk of introducing new bugs (mitigated by golden-master testing)

### Trade-Offs Accepted
- **Short-term pain** (3–4 month development, 2–3 week team training) for **long-term reliability** (90+ day uptime, 3× lower resource usage)
- **Upfront effort** for **operational peace of mind** and reduced MTTR on edge devices

---

## Validation

### How We Know This Decision Works
1. **Phase A** (Faithful Port): Golden-master tests verify 100% behavioral equivalence; zero new bugs introduced
2. **Phase B** (Hardening): Performance targets met; memory stable over 30 days
3. **Stage 0–4 Rollout**: Each stage validates on production devices; automated rollback if issues detected
4. **90-Day Trial**: Post-cutover, devices run unattended for 90 days without restarts

### Success Metrics
- ✓ Zero unplanned restarts due to memory pressure (over 90 days)
- ✓ 95th-percentile latency < 10 ms under peak load
- ✓ Memory footprint < 40 MB sustained
- ✓ CPU usage < 25% on 8-core ARM at full throughput
- ✓ Team confident to maintain and extend Rust codebase

---

## Related Decisions
- **[ADR-0002](0002-module-to-crate-decomposition.md)**: How to decompose the monolith into crates
- **[ADR-0003](0003-hybrid-runtime-keep-python-subprocess.md)**: Why keep Python for DSP/ML
- **[ADR-0004](0004-ecosystem-crate-mapping.md)**: Technology choices (npm → Rust crate equivalents)
- **[ADR-0005](0005-staged-rollout-and-rollback.md)**: How to safely roll out changes to production

---

## References
- Problem Statement: See `../01-problem-and-goals.md`
- Architecture: See `../02-architecture-overview.md`
- Rust Website: https://www.rust-lang.org/
- Tokio (async runtime): https://tokio.rs/
- OWASP Memory Safety: https://owasp.org/www-project-secure-coding-practices/

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Engineering leads, product management  
**Implementation Target**: Phase A completes by 2026-11-15
