# ADR-0001: Why Rewrite Aurora Edge Gateway in Rust?

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Engineering team, product management, operations  

---

## Context

Aurora Edge Gateway is an industrial IoT data aggregation platform offered in two hardware configurations. The high-spec variant ships with 8-core ARM aarch64 and 8 GB RAM; the low-spec variant ships with 4-core ARM Cortex-A53, nominally 4 GB RAM but with only ~2.84 GB actually available to user-space processes after the OS, no swap configured, and 28 GB of disk. Node.js implementation runs acceptably on the high-spec variant but continuously hits memory limits on the low-spec variant. Analysis confirmed that a Rust rewrite is the root-cause fix — this mismatch is the real trigger for this decision.

### Hardware Reality Gap

A 3-hour active-load measurement of the Node.js baseline on the 4-core product platform showed active-process memory at **1538 MB — roughly 54% of the ~2.84 GB actually available** to user space (no swap). CPU usage was 52% idle, rising to 64% after 3 hours active, with a load average of 3.30 against 4 cores. The baseline ran as 16 services across 8 startup phases (about 20 OS processes), with boot taking roughly 2 minutes 30 seconds end to end.

That 54%-of-available-memory floor is the core of the rewrite decision: roughly 1.5 GB of the ~2.84 GB available is consumed at steady state, with no swap to fall back on. The remaining headroom has to absorb traffic spikes, accommodate future sensor-count growth, and survive the memory behaviour of any single misbehaving service — and under those conditions, a service growing by even a few MB per hour is hours, not days, away from tripping the per-service 500 MB cap and being force-restarted.

The decision therefore rests on the baseline resource envelope itself, not on any single incident. The Node.js multi-process topology consumed more than half the available memory at idle, leaving no room to absorb the allocator- and runtime-level memory growth that any long-running edge service eventually exhibits. That last point is not hypothetical, and it is worth being precise about the direction of the argument: the rewritten Rust system hit exactly that class of growth during its own soak testing, and the rewrite is what gave us the headroom to diagnose it instead of absorbing another forced restart (see "Memory Growth Found During Soak Testing" below).

### Methodology Confidence

Large-scale language migration feasibility is validated by recent industry precedent. In 2026, the Bun team executed a substantial codebase rewrite from Zig to Rust using AI-assisted workflows (see [Anthropic's official case study](https://claude.com/resources/articles/ai-code-migration), [GitHub PR #30412](https://github.com/oven-sh/bun/pull/30412), and [InfoQ coverage](https://www.infoq.com/news/2026/09/bun-AI-rewrite-zig-rust-4-months/)), migrating ~535K lines of Zig into ~750K lines of Rust, with a 99.8% existing test pass rate in 11 days from first commit to merge. This validates:
- Structured Phase A/B methodology (faithful port, then optimize) scales to large codebases
- Golden-master testing (ADR-0006) reliably detects behavioral drift where unit tests alone fail
- Multi-agent parallel workflows accelerate migration while reducing review bottlenecks

This precedent informed our confidence in the project timeline (Stage 0–4, ~22 weeks) and verification strategy.

---

## Decision

**Rewrite Aurora Edge Gateway in Rust** using incremental Phase A/B strategy with staged production rollout (see ADR-0005, ADR-0006).

---

## Rationale

### Why Rust?

**Memory Safety by Design**
- Rust's type system eliminates use-after-free, double-free, and data races at compile time. It does **not** eliminate memory growth — reference cycles still leak, and allocator-level growth sits below the language entirely (see the soak-testing section below)
- No garbage collector → no GC pauses blocking sensor reads
- Explicit resource management via drop trait → deterministic cleanup at a known point
- **Result**: memory behaviour becomes attributable — growth has a locatable cause instead of being absorbed by restarts

**Deterministic Performance**
- No stop-the-world GC pauses → no tail-latency spikes under peak load
- All concurrency explicit (no magic event loop hiding issues)
- Compiler forces error handling explicitly
- **Result**: Latency becomes bounded and analyzable; peak performance predictable

**Exceptional Performance Headroom**
- Zero-cost abstractions (compiled to machine code, no interpreter overhead)
- Async/await with minimal runtime overhead (tokio)
- Fine-grained allocation control (no hidden allocations in standard library)
- **Result**: Efficient resource utilization on constrained hardware; CPU headroom for future features

**Excellent IoT/Systems Ecosystem**
- `tokio`: Industry-standard async runtime for high-concurrency workloads
- `rumqttc`: Mature MQTT client library (async-first, IoT-optimized)
- `rusqlite`: Synchronous SQLite bindings (sufficient for edge use cases without async complexity)
- `axum`: Modern HTTP framework (zero-cost routing, type-safe)
- `serde`: Zero-copy serialization
- **Result**: All required technologies mature and well-integrated

---

### Alternatives Considered

#### Option A: "Throw More Memory" (Node.js + Clustering)
- **Approach**: Run multiple Node.js processes; add caching; increase memory limits
- **Pros**: Lower engineering effort (no rewrite); leverages existing code
- **Cons**: 
  - Does not address the root cause (per-process runtime and heap overhead stays duplicated rather than shared)
  - Increases per-device footprint (infeasible on constrained hardware)
  - Does not solve tail-latency or stability issues
- **Decision**: Rejected — temporary band-aid; doesn't address fundamental constraints

#### Option B: Go (Compiled Alternative)
- **Approach**: Rewrite in Go (easier learning curve than Rust)
- **Pros**: Faster team ramp-up; compiled; good concurrency (goroutines)
- **Cons**: 
  - Still has garbage collection (doesn't solve memory-leak problem fundamentally)
  - Concurrency debugging harder than Rust's explicit model
  - Smaller IoT/edge ecosystem than Rust
- **Decision**: Rejected — doesn't eliminate root memory issue; less ideal for constrained hardware

#### Option C: C++ (Maximum Performance)
- **Approach**: Rewrite in C++ (maximum control, maximum performance)
- **Pros**: Absolute maximum performance; fine-grained memory control
- **Cons**: 
  - Steep learning curve; memory-safety burden remains on team
  - Higher risk of introducing new vulnerabilities
  - Build system complexity
  - Slower development velocity
- **Decision**: Rejected — unnecessary complexity; Rust achieves safety goals more cleanly

#### Option D: Rust (Selected) ✅
- **Approach**: Full rewrite in Rust with phased rollout
- **Pros**: 
  - Eliminates use-after-free, double-free, and data races at compile time
  - Performance equivalent to C++ without safety burden
  - Excellent async/concurrency primitives
  - Rich IoT ecosystem
  - Team learning of valuable systems language
  - Strong compile-time guarantees reduce production issues
- **Cons**: 
  - Higher upfront engineering effort (~22 weeks across Stage 0–4)
  - Team ramp-up on new language (2–3 weeks)
  - Longer initial development timeline
- **Decision**: Selected — best long-term investment; addresses memory leak and determinism issues

---

## Consequences

### Positive
1. **Memory footprint collapsed**: on the product hardware baseline comparison, full-stack resident memory went from **1538 MB** (Node.js, 3h active load) to **206 MB** (Rust, full stack of 14 supervised services + Python subprocess) — leaving real headroom against the ~2.84 GB available on the 4-core product platform.
2. **CPU and load dropped sharply**: Node.js measured **64% CPU** after 3h active load with load average **3.30/4 cores**; the Rust rewrite measured **6.3% average / 8.8% peak CPU** and **0.50 average / 1.81 peak load average** over a 7.8-hour soak test — comparable duration, same product-class hardware class.
3. **Latency became bounded and small**: a per-second pipeline sample of 285 cycles measured p95 **56.4 ms** and p99 **57.7 ms**, consuming roughly **5.6% of the 1-second budget** at p95, with **zero** samples exceeding 500 ms.
4. **Restart pattern stabilized early**: in soak testing, 3 restarts were concentrated in the first 66 minutes, followed by **6.7 hours with zero restarts**.
5. **Reduced Maintenance Burden**: simpler alerting rules; clearer root causes when issues do occur
6. **Team Growth**: Engineers gain systems programming skill; transferable to future projects

### Memory Growth Found During Soak Testing — and Its Real Root Cause
Rust's compile-time guarantees eliminate a specific class of bugs (use-after-free, data races, dangling pointers) — they do not make a system immune to memory growth in general, and soak testing on the rewritten system proved that distinction in practice.

A 7.8-hour soak test caught the message-bus process growing from **45.0 MB to 95.3 MB (+112%)** over the run — a linear trend of roughly **6.5 MB/h**, the kind of slope that would have looked exactly like a leak on a dashboard.

Root-cause investigation traced the growth not to a Rust memory-safety bug but to **glibc's multi-threaded allocator arena behavior**: with per-thread arenas allowed to grow up to 64 by default, the message-bus process's threading and allocation pattern caused its total mapped arena memory to expand over time, independent of any leaked Rust-level allocation.

The fix was a one-line environment change — **`MALLOC_ARENA_MAX=2`** — capping the glibc arena count. A follow-up **10.6-hour soak test** confirmed the fix: RSS rose to ~24.5 MB then settled to **17.2 MB**, with the growth slope moving from **+5.74 MB/h** (first 2 hours) to **0.00 MB/h** (mid-run) to **−3.69 MB/h** (last 2 hours) — **PASS**.

**The lesson this validates**: rewriting in Rust removes language-level memory-safety bugs, but it does not automatically remove memory growth introduced at the allocator or runtime-configuration layer. That class of issue is invisible to the type system and only surfaces under sustained soak testing — which is exactly why soak testing stays part of the validation plan after the rewrite, not just something we needed during the Node.js era.

### Negative
1. **Development Timeline**: ~22-week rewrite and staged rollout (Stage 0–4) (vs. ongoing hotfixes on Node.js)
2. **Team Learning Curve**: Rust borrow checker, lifetimes, async/await require ramp-up (2–3 weeks)
3. **Dual Maintenance**: Must support both systems during rollout (~22 weeks of engineering overhead)
4. **Rollout Risk**: Rewrite always carries risk of behavioral divergence (mitigated by golden-master testing, staged rollout)

### Trade-Offs Accepted
- **Short-term engineering effort** (~22-week development and rollout, 2–3 week training) for **long-term operational peace of mind**
- **Development complexity** (learning new language) for **elimination of memory-related production issues**

---

## Validation

### How We Know This Decision Works
1. **Phase A** (Faithful Port): Golden-master tests verify behavioral equivalence against the Node.js baseline
2. **Phase B** (Hardening): Soak testing on product-class hardware surfaces and closes issues like the glibc arena growth above
3. **Staged Rollout** (Stage 0–4, ~22 weeks total): Each stage validates on production devices; rollback is available within 5 minutes overall (2–4 minutes per stage) if issues are detected
4. **Production Observation**: the longest continuous soak test completed to date is **10.6 hours** (the MALLOC_ARENA_MAX fix-verification run), which passed; we have not yet accumulated 90-day field data and do not claim it

### Success Metrics (Measured Results)
- ✓ Full-stack resident memory: **206 MB** vs Node.js **1538 MB** (3h active load, same product-class hardware)
- ✓ CPU: **6.3% average / 8.8% peak** (7.8h soak) vs Node.js **64%** (3h active); load average **0.50 average / 1.81 peak** vs **3.30**
- ✓ Per-second pipeline latency: p95 **56.4 ms** / p99 **57.7 ms**, ~**5.6%** of the 1-second budget at p95, **0** samples over 500 ms (285 samples)
- ✓ Longest soak test passed to date: **10.6 hours**, post `MALLOC_ARENA_MAX=2` fix, slope −3.69 MB/h in the final 2 hours
- ✓ Staged rollout plan (Stage 0–4, ~22 weeks) with rollback achievable within 5 minutes
- ✓ Team capable of independent maintenance and enhancement after Phase A

---

## Related Decisions
- **[ADR-0002](0002-module-to-crate-decomposition.md)**: How to map existing process boundaries into Rust crates
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
