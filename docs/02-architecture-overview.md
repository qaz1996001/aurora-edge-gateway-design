# Architecture Overview: Before & After

## System Context

**Aurora Edge Gateway** is an industrial IoT data aggregation and control platform deployed on edge devices (4-core ARM Cortex-A53 single-board computers, nominal 4 GB RAM with ~2.84 GB OS-usable, no swap). It serves as a bridge between:
- **Upstream**: Cloud backend (data ingestion, fleet management, analytics)
- **Downstream**: Industrial sensors and equipment — USB Audio (vibration capture), Modbus RTU over USB-RS485 (current/power), USB serial (temperature), and serial (beacon / indicator lights)

### C4 Context Diagram

![System context: cloud, edge gateway, field hardware](../diagrams/context-diagram.png)

<sub>Vector source: [`context-diagram.svg`](../diagrams/context-diagram.svg)</sub>

## Before: Node.js Architecture

### System Topology (Original)

![Deployment topology, Node.js baseline](../diagrams/before-topology.png)

<sub>Vector source: [`before-topology.svg`](../diagrams/before-topology.svg)</sub>

### Problems

While the multi-process architecture provides basic fault isolation, individual processes face limitations:

1. **Memory Footprint**: Measured in production. The 16-service / 8-startup-stage topology (~20 OS processes) holds 1538 MB resident during a 3-hour active-load window — about 54% of the ~2.84 GB OS-usable RAM on the 4-core product platform, with no swap configured. Per-process runtime and heap overhead is duplicated across every service rather than shared, and the remaining headroom has to absorb traffic spikes, sensor-count growth, and any single service's memory behaviour at the same time
2. **GC Pauses**: Unpredictable stop-the-world delays under peak load, with duration scaling with heap size, causing downstream timeouts
3. **Lack of Compile-Time Safety**: No static type checking on module boundaries; runtime errors in IPC and data validation
4. **Complex Coordination**: Manual orchestration of process startup order, dependency tracking, and graceful shutdown
5. **CPU Overhead**: Garbage collection runs frequently; the process supervisor adds overhead across ~20 processes; measured CPU sits at 52% idle and rises to 64% under a 3-hour active load on 4 cores, with load average 3.30 — limited optimization opportunities within the runtime

---

## After: Rust Architecture

### System Topology (Rust Rewrite)

![Deployment topology, Rust rewrite](../diagrams/deployment-topology.png)

<sub>Vector source: [`deployment-topology.svg`](../diagrams/deployment-topology.svg)</sub>

### Advantages
1. **Modular**: Each crate/binary is independently restartable; failure isolation
2. **Memory Safe**: No undefined behavior, no use-after-free, no data races — enforced at compile time. Memory *growth* is a separate concern and is covered by soak testing
3. **Deterministic**: No GC; all resource cleanup is explicit (drop semantics)
4. **Scalable**: Async/await via tokio; efficient concurrency without threads
5. **Maintainable**: Strong typing catches refactoring errors early
6. **Observable**: Each service logs to systemd journald; easy centralized monitoring

---

## Module → Crate → Binary Mapping

![Node.js service to Rust crate mapping](../diagrams/module-crate-mapping.png)

<sub>Vector source: [`module-crate-mapping.svg`](../diagrams/module-crate-mapping.svg)</sub>

### Example Mapping

| Original Node.js Module | Rust Crate | Supervised Binary | Dependencies |
|---|---|---|---|
| appserver (Express routes) | `aeg-appserver` | `aeg-appserver` | axum, tokio, serde, aeg-db |
| mqtt-client | `aeg-mqtt-client` | (shared lib) | rumqttc, tokio |
| sensor-data | `aeg-sensors` | `aeg-sensors` | aeg-db, aeg-mqtt-client, tokio |
| beacon-control (serial port) | `aeg-beacon` | `aeg-beacon` | tokio-serial, tokio |
| file-manager | `aeg-file-manager` | (shared lib) | tokio, anyhow |
| ota-updater | `aeg-ota` | `aeg-ota` | sha2, aeg-file-manager |
| metrics-engine (signal RMS aggregation) | `aeg-metrics` | `aeg-metrics` | aeg-sensors, aeg-db |
| configuration | `aeg-config` | (build-time) | serde, toml |
| database | `aeg-db` | (shared lib) | rusqlite, tokio |
| python-bridge | `aeg-python-bridge` | (IPC daemon, child process) | tokio, serde |

Each binary above runs as a child process under the single `aeg-supervisor` systemd unit, not as its own systemd unit (14 supervised services in total).

---

## Technology Choices: npm → Rust

See **[ADR-0004: Ecosystem Crate Mapping](adr/0004-ecosystem-crate-mapping.md)** for detailed rationale.

| Domain | Node.js (npm) | Rust (crates.io) | Rationale |
|---|---|---|---|
| HTTP server | express | axum | Zero-cost abstractions; type-safe routing |
| MQTT | mqtt (npm) | rumqttc | Async-first; better embedded use case support |
| SQLite | better-sqlite3 | rusqlite | Synchronous API sufficient for edge use; no GC |
| Async runtime | node (native) | tokio | Industry-standard; excellent concurrency |
| Logging | winston | tracing + slog | Structured logging; journald integration |
| Serialization | JSON (native) | serde | Zero-cost serialization |
| Serial communication | serialport (npm) | tokio-serial / serialport | Async serial I/O; matches actual hardware (USB serial beacon control — there is no GPIO header on this platform) |
| Cross-compilation | n/a | cargo (built-in) | Native support for ARM targets |

---

## Before vs. After Comparison

| Characteristic | Node.js (Baseline) | Rust (Target) | Notes |
|---|---|---|---|
| **Process Architecture** | 16 process-manager-managed services across 8 startup phases (~20 processes observed in production) | 14 services supervised as child processes under one `aeg-supervisor` systemd unit | Clearer fault boundaries; compile-time type safety |
| **Memory Usage** | 1538 MB during 3h active load on the 4-core production platform (~54% of ~2.84 GB OS-usable) | 206 MB full stack (14 supervised services + Python subprocess, 1 vibration sensor active) | Explicit cleanup, no GC heap growth |
| **CPU Usage** | 52% idle / 64% during 3h active load (4-core) | 6.3% average / 8.8% peak over a 7.8h soak test, zero overnight drift | Deterministic performance, no GC pause jitter |
| **Load Average** | 3.30 / 4 cores (3h active load) | 0.50 average / 1.81 peak over a 7.8h soak test (always < 2 cores) | Large headroom margin |
| **Compile-Time Safety** | Runtime type checking on IPC boundaries | Full compile-time verification | Catches boundary errors before deployment |
| **Process Coordination** | Manual process-manager orchestration with per-service memory caps (200 MB–2 GB, auto-restart on breach) and dependency tracking | Single supervisor process launches and monitors 14 child-process services; IPC over an internal message bus (not systemd socket activation) | Simpler, standard OS-level management; existing per-service restart behavior is preserved, not lost |
| **Restart Impact** | Process manager can restart an individual service independently (e.g., on a memory-cap breach) without affecting its siblings | Supervisor restarts affected services independently; 3 restarts observed in the first 66 minutes of a soak test, then 6.7h with zero restarts | Both architectures isolate restarts to the affected service — this was not a Node.js weakness |
| **Uptime Predictability** | Steady-state consumption at 54% of available RAM leaves the per-service memory cap as the only buffer between any sustained growth and a forced restart | The 7.8h soak surfaced allocator-level growth, which was diagnosed and fixed (`MALLOC_ARENA_MAX=2`); the 10.6h re-run ended on a negative slope | Growth becomes diagnosable instead of being absorbed by restarts |
| **Memory Safety** | Runtime checks only; undefined behavior possible | Compile-time guaranteed; no undefined behavior | Eliminates entire class of memory corruption bugs |
| **API Breaking Changes** | None | None (backward compatible) | Drop-in replacement |
| **Implementation Effort** | Baseline (already deployed) | ~22 weeks (Stage 0–4 staged rollout, with Phase A porting work concentrated in Stage 0–2) | Justified by compile-time safety & reliability gains |

---

## Crate Architecture Diagram

See `../diagrams/module-crate-mapping.svg` for the full crate dependency graph (DAG structure).

### Dependency Layers

![Crate dependency tiers, Tier 0 to Tier 4](../diagrams/crate-dependency-tiers.png)

<sub>Vector source: [`crate-dependency-tiers.svg`](../diagrams/crate-dependency-tiers.svg)</sub>

---

## Deployment Topology

See `../diagrams/deployment-topology.svg` for edge-to-cloud topology.

### Edge Device Systemd Unit

```
[aeg-supervisor]
ExecStart=/usr/local/bin/aeg-supervisor --config /etc/aeg/supervisor.toml
Restart=always
RestartSec=5

# aeg-supervisor launches and monitors 14 services as child
# processes, including:
#   aeg-appserver, aeg-sensors, aeg-metrics, aeg-ota, aeg-beacon,
#   aeg-db, aeg-file-manager, aeg-python-bridge, aeg-mqtt-client,
#   aeg-config, plus log/rotation, watchdog, and threshold-sync
#   helpers.
```

There is a single systemd unit on the device. Underneath it:
- The supervisor can restart an individual child service independently, without affecting its siblings
- Each service has a configurable memory cap, enforced by the supervisor (auto-restart on breach)
- Services log via the supervisor to the systemd journal
- Services communicate over the internal message bus (local MQTT broker), not raw Unix sockets or TCP

---

## Data Flow & Concurrency

![Per-second data flow, Node.js versus Rust](../diagrams/data-flow-pipeline.png)

<sub>Vector source: [`data-flow-pipeline.svg`](../diagrams/data-flow-pipeline.svg)</sub>

**Measured server-side pipeline** (285 one-second samples): median 55.5 ms / p95 56.4 ms / p99 57.7 ms / max 59.5 ms — roughly 5.5–5.6% of the 1-second budget, with zero samples over 500 ms. Server-side breakdown: JSON deserialise 18.1 ms + RMS calculation across 3 axes 3.7 ms + diagnostics 13.7 ms = 35.5 ms.

---

## Next Steps

1. **Migration Strategy** → See `03-migration-strategy.md` for the two-phase port approach
2. **Testing & Validation** → See `04-testing-strategy.md` for golden-master testing
3. **Decision Rationale** → See **[ADR-0002](adr/0002-module-to-crate-decomposition.md)** and **[ADR-0003](adr/0003-hybrid-runtime-keep-python-subprocess.md)** for architectural decisions
4. **Visual Diagrams** → See `../diagrams/` directory

---

**Last Updated**: 2026-10-20  
**Status**: Baseline architecture approved  
**Next Review**: Post-Phase A (golden-master test results)
