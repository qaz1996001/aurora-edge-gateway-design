# Aurora Edge Gateway: Node.js → Rust Architectural Rewrite

A comprehensive design documentation portfolio showcasing the strategic migration of a production industrial IoT edge gateway platform from Node.js to Rust.

## Overview

This repository documents the architectural decisions, migration strategies, and engineering practices employed in rewriting **Aurora Edge Gateway**—a distributed industrial IoT edge device management platform—from Node.js to Rust.

The gateway serves as a critical data aggregation and processing layer for industrial sensors, managing real-time telemetry, device control, and edge-side computation. The rewrite was driven by:

- **Performance**: Lower CPU footprint and faster response latencies for millisecond-critical sensor data pipelines
- **Memory Safety**: Elimination of memory leaks and undefined behavior in long-running edge processes
- **Operational Reliability**: Validated via a 10.6-hour soak test on the target 4-core ARM Cortex-A53 platform (nominal 4 GB RAM, ~2.84 GB usable, no swap), with the full stack holding steady-state memory of 206 MB

## How to Navigate This Repository

### Quick Start
1. **First time here?** → Read [`docs/01-problem-and-goals.md`](docs/01-problem-and-goals.md) to understand the "why"
2. **Want the big picture?** → See [`docs/02-architecture-overview.md`](docs/02-architecture-overview.md) for Before/After system design
3. **Interested in the methodology?** → Jump to [`docs/03-migration-strategy.md`](docs/03-migration-strategy.md) for the staged rollout approach

### Full Document Index

#### Design & Strategy
- **[Problem Statement & Goals](docs/01-problem-and-goals.md)** — Why rewrite in Rust? Performance, safety, reliability metrics.
- **[Architecture Overview](docs/02-architecture-overview.md)** — System topology, module→crate decomposition, Before/After comparison with ASCII diagrams.
- **[Migration Strategy](docs/03-migration-strategy.md)** — Two-phase approach (Phase A: faithful port, Phase B: harden + optimize), staged rollout plan (Stage 0–4), rollback procedures.
- **[Testing Strategy](docs/04-testing-strategy.md)** — Golden-master / characterization testing methodology for validating behavioral parity.
- **[Review Process](docs/05-review-process.md)** — Multi-role structured audit framework (Rust expert, security, architecture, performance, code quality).
- **[Risk Register](docs/06-risk-register-template.md)** — Risk classification, example high/medium/low-risk scenarios, mitigation checklist.

#### Architecture Decision Records (ADRs)
Each ADR captures a single architectural decision with rationale, trade-offs, and implications:

- **[ADR-0001: Why Rewrite in Rust?](docs/adr/0001-why-rewrite-in-rust.md)** — Decision to rewrite; alternatives considered; reasoning.
- **[ADR-0002: Module → Crate Decomposition](docs/adr/0002-module-to-crate-decomposition.md)** — Granular module→Rust-crate structure; deployment as independent systemd units.
- **[ADR-0003: Hybrid Runtime (Keep Python Subprocess)](docs/adr/0003-hybrid-runtime-keep-python-subprocess.md)** — Rationale for keeping Python for DSP/ML, cross-language IPC design.
- **[ADR-0004: Ecosystem Crate Mapping](docs/adr/0004-ecosystem-crate-mapping.md)** — Technology choices: npm→Rust crate equivalents (express→axum, mqtt→rumqttc, better-sqlite3→rusqlite, etc.).
- **[ADR-0005: Staged Rollout & Rollback](docs/adr/0005-staged-rollout-and-rollback.md)** — Incremental deployment phases with binary rollback switch and manual SOP (rollback within 5 minutes per stage).
- **[ADR-0006: Golden-Master Testing](docs/adr/0006-golden-master-testing.md)** — Characterization-test approach for verifying behavioral parity.
- **[ADR-0007: Port Marker Conventions](docs/adr/0007-port-marker-convention.md)** — Traceability markers for deferred work and known issues during the port.

#### Visual Diagrams
Located in `/diagrams/`:
- `context-diagram.svg` — C4-style context diagram showing Aurora Edge Gateway and its external dependencies
- `module-crate-mapping.svg` — Node.js module → Rust crate → binary deployment mapping
- `deployment-topology.svg` — Edge device network topology (generic ARM edge, cloud backend)
- `staged-rollout-timeline.svg` — Stage 0–4 deployment timeline with go/no-go gates

## Key Takeaways

### Architecture Patterns
1. **Modular Decomposition**: One logical module → one Rust crate → one systemd-managed binary (loose coupling, independent upgrade).
2. **Hybrid Runtime**: Rust for systems programming (I/O, concurrency, memory-safety-critical paths); Python subprocess for scientific computing (DSP, ML inference).
3. **Two-Phase Port**: 
   - **Phase A**: Faithful behavioral port (preserve bugs, quirks) to de-risk testing.
   - **Phase B**: Refactor, harden, and optimize once parity is proven.
4. **Staged Rollout**: Stage-by-stage deployment (Stage 0: bootstrap; Stage 1–3: low/medium/high risk; Stage 4: cutover), ~22 weeks total, with a binary rollback switch and manual SOP (rollback within 5 minutes per stage).

### Process Practices
1. **Golden-Master Testing**: Characterization-test harness verifies old vs. new behavioral equivalence on real-world traffic fixtures.
2. **Marker Conventions**: `TODO(port)`, `FIXME(port)`, `PERF(port)`, `SAFETY:` markers maintain traceability of deferred decisions and known gaps.
3. **Multi-Role Code Review**: Structured 5-persona review (Rust specialist, security, architecture, performance, code quality) catches cross-cutting concerns.
4. **Risk Staging**: Categorize modules by risk (crypto/auth → high; sensor aggregation → medium; logging → low) to guide rollout order.

## Who Is This For?

- **Software Architects** evaluating legacy-to-Rust rewrite strategies
- **Platform Engineers** planning incremental system migrations
- **Technical Leads** designing governance for large port projects
- **Interview Candidates** showcasing systems thinking, risk management, and cross-functional coordination

## Contributing / Questions

This is a static design portfolio. For feedback, questions, or discussions about the architecture:
- Open an issue (if this becomes a public repo)
- Contact the repository owner

---

**Last Updated**: 2026-10-20  
**Format**: Markdown + ASCII diagrams inline + SVG/PNG diagrams in `/diagrams/` + Architecture Decision Records (ADR)  
**Audience**: Technical decision-makers, architects, engineering leads

---

## Repository Structure

```
aurora-edge-gateway-design/
├── README.md                           ← You are here
├── docs/
│   ├── 01-problem-and-goals.md        # Problem statement & rewrite rationale
│   ├── 02-architecture-overview.md    # System design, Before/After
│   ├── 03-migration-strategy.md       # Two-phase port, staged rollout
│   ├── 04-testing-strategy.md         # Golden-master testing
│   ├── 05-review-process.md           # Multi-role review framework
│   ├── 06-risk-register-template.md   # Risk taxonomy & examples
│   └── adr/                           # Architecture Decision Records
│       ├── 0001-why-rewrite-in-rust.md
│       ├── 0002-module-to-crate-decomposition.md
│       ├── 0003-hybrid-runtime-keep-python-subprocess.md
│       ├── 0004-ecosystem-crate-mapping.md
│       ├── 0005-staged-rollout-and-rollback.md
│       ├── 0006-golden-master-testing.md
│       └── 0007-port-marker-convention.md
└── diagrams/
    ├── context-diagram.svg / .png
    ├── module-crate-mapping.svg / .png
    ├── deployment-topology.svg / .png
    └── staged-rollout-timeline.svg / .png
```
