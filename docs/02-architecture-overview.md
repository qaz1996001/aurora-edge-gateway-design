# Architecture Overview: Before & After

## System Context

**Aurora Edge Gateway** is an industrial IoT data aggregation and control platform deployed on edge devices (4-core ARM Cortex-A53 single-board computers, nominal 4 GB RAM with ~2.84 GB OS-usable, no swap). It serves as a bridge between:
- **Upstream**: Cloud backend (data ingestion, fleet management, analytics)
- **Downstream**: Industrial sensors and equipment — USB Audio (vibration capture), Modbus RTU over USB-RS485 (current/power), USB serial (temperature), and serial (beacon / indicator lights)

### C4 Context Diagram
See `../diagrams/context-diagram.svg` for a visual overview of external dependencies and data flows.

## Before: Node.js Architecture

### System Topology (Original)

```
    ┌─────────────────────────────────────────────────────┐
    │ Cloud Backend (REST API, data ingestion)             │
    └─────────────────────────────────────────────────────┘
                     ↕ REST (heartbeat & batch sync)
                     ┄ optional MQTT uplink (factory-disabled)
┌─ Edge Device (ARM Linux, 4-core Cortex-A53) ────────────────┐
│                                                              │
│  Process Supervisor (process manager) managing 16 Node.js/  │
│  Python services across 8 startup phases (~20 processes     │
│  observed in production, incl. supervisor, threshold sync,  │
│  log rotation, and dynamic sensor processes):                │
│                                                              │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │ appserver (Node.js)  │  │ mqtt-client (Node.js)│         │
│  │ (Express HTTP)       │  │ (mqtt client)        │         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                              │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │ sensor-data (Node.js)│  │ beacon-control        │         │
│  │ (aggregation, DSP)   │  │ (Node.js, serial port)│         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                              │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │ file-manager (Node.js)│ │ ota-updater (Node.js)│         │
│  │ (file I/O)           │  │ (firmware updates)   │         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                              │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │ metrics-engine        │  │ python-bridge        │         │
│  │ (Node.js, signal RMS  │  │ (Python subprocess)  │         │
│  │  aggregation)         │  │                      │         │
│  └──────────────────────┘  └──────────────────────┘         │
│                                                              │
│  Internal message bus (MQTT, in-process event emitters)     │
│  Database file (SQLite3, shared mmap)                       │
│  Config & state files (TOML, JSON)                           │
│  Per-service memory caps: 200 MB – 2 GB, enforced by the     │
│  process manager (auto-restart on breach)                    │
│                                                              │
│  Total memory (3h active load): 1538 MB (~54% of ~2.84 GB    │
│  OS-usable); CPU: 52% idle / 64% 3h-active; load avg          │
│  3.30 / 4 cores; boot → all services online: ~2m30s           │
└──────────────────────────────────────────────────────────────┘
                     ↕ Serial / USB
    ┌─────────────────────────────────────────────────────┐
    │ Industrial Sensors: USB Audio (vibration), Modbus     │
    │ RTU over USB-RS485 (current/power), USB serial         │
    │ (temperature), serial (beacon / indicator lights)      │
    └─────────────────────────────────────────────────────┘
```

### Problems

While the multi-process architecture provides basic fault isolation, individual processes face limitations:

1. **Memory Leaks**: Observed in production. A CSV field-expansion leak grew RSS at +13 MB/h, tripping the 500 MB per-service memory cap (auto-restart) after ~14 hours; steady-state CPU p90 rose from ~140% to 169.5% (4-core) as the leak progressed. A larger incident on the 8-core validation rig showed a peak growth rate of +1887 MB over 13 minutes, exhausting memory and forcing a watchdog hardware reset — 98 resets in a single day (roughly one every 15 minutes) before the leak was isolated
2. **GC Pauses**: Unpredictable stop-the-world delays under peak load, with duration scaling with heap size, causing downstream timeouts
3. **Lack of Compile-Time Safety**: No static type checking on module boundaries; runtime errors in IPC and data validation
4. **Complex Coordination**: Manual orchestration of process startup order, dependency tracking, and graceful shutdown
5. **CPU Overhead**: Garbage collection runs frequently; a log-flood incident (20,000 log lines, 19,997 of them reordering messages) drove CPU to 66.7%; process supervisor adds overhead; limited optimization opportunities

---

## After: Rust Architecture

### System Topology (Rust Rewrite)

```
    ┌─────────────────────────────────────────────────────┐
    │ Cloud Backend (primary uplink via REST)               │
    └─────────────────────────────────────────────────────┘
                     ↕ REST (heartbeat & batch sync)
                     ┄ optional MQTT acceleration uplink
                       (factory-disabled; opt-in via config)
┌─ Edge Device (ARM Linux, 4-core Cortex-A53) ────────────────┐
│                                                              │
│  Single systemd unit (aeg-supervisor) launches and          │
│  supervises 14 services as child processes:                 │
│                                                              │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ aeg-appserver   │  │ aeg-sensors  │  │ aeg-beacon   │   │
│  │ (Axum REST API) │  │ (aggregation)│  │ (serial port │   │
│  │                 │  │              │  │  control)    │   │
│  └─────────────────┘  └──────────────┘  └──────────────┘   │
│                                                              │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐     │
│  │ aeg-ota      │  │ aeg-config    │  │ aeg-metrics  │     │
│  │ (firmware)   │  │ (TOML parse)  │  │ (signal RMS  │     │
│  │              │  │               │  │  aggregation)│     │
│  └──────────────┘  └───────────────┘  └──────────────┘     │
│                                                              │
│  ... plus 8 further supervised services (aeg-db,            │
│  aeg-file-manager, aeg-python-bridge, aeg-mqtt-client,       │
│  log/rotation, watchdog, threshold sync, and others)         │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Internal Message Bus (local MQTT broker):          │    │
│  │  - Services publish/subscribe for inter-process     │    │
│  │    events (sensor updates, config changes)          │    │
│  │  - NOT cloud uplink; separate from REST API          │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  Shared Infrastructure:                                     │
│  - aeg-db (rusqlite SQLite wrapper, compiled into binaries) │
│  - aeg-python-bridge (local IPC → Python subprocess)         │
│  - inter-service IPC via the internal message bus above     │
│    (not systemd socket activation)                          │
│  - structured logging via journald                          │
│                                                              │
│  Python Subprocess (DSP/ML):                                │
│  - signal-processing (Fourier, filtering)                    │
│  - ml-inference (lightweight models)                        │
│  - Spawned on-demand; killed after task completion           │
│                                                              │
│  Total memory footprint (14 supervised services + Python     │
│  subprocess, 1 vibration sensor active): 206 MB               │
│  CPU (7.8h soak): 6.3% avg / 8.8% peak, zero overnight drift  │
│  Load avg (7.8h soak): 0.50 avg / 1.81 peak (always < 2       │
│  cores); 3 restarts in first 66 min, then 6.7h zero restarts │
└───────────────────────────────────────────────────────────────┘
                     ↕ Serial / USB
    ┌─────────────────────────────────────────────────────┐
    │ Industrial Sensors: USB Audio (vibration), Modbus      │
    │ RTU over USB-RS485 (current/power), USB serial          │
    │ (temperature), serial (beacon / indicator lights)        │
    └─────────────────────────────────────────────────────┘
```

### Advantages
1. **Modular**: Each crate/binary is independently restartable; failure isolation
2. **Memory Safe**: No undefined behavior; Rust compiler prevents leaks at compile time
3. **Deterministic**: No GC; all resource cleanup is explicit (drop semantics)
4. **Scalable**: Async/await via tokio; efficient concurrency without threads
5. **Maintainable**: Strong typing catches refactoring errors early
6. **Observable**: Each service logs to systemd journald; easy centralized monitoring

---

## Module → Crate → Binary Mapping

See `../diagrams/module-crate-mapping.svg` for a detailed breakdown.

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
| **Uptime Predictability** | Known leak scenarios require per-service auto-restart (e.g., a CSV field-expansion leak triggers a restart after ~14h at +13 MB/h) | No leaks observed in a 7.8h soak test; uptime limited by hardware, not by GC/heap behavior | Reduces reliance on the per-service memory cap as a safety net |
| **Memory Safety** | Runtime checks only; undefined behavior possible | Compile-time guaranteed; no undefined behavior | Eliminates entire class of memory corruption bugs |
| **API Breaking Changes** | None | None (backward compatible) | Drop-in replacement |
| **Implementation Effort** | Baseline (already deployed) | ~22 weeks (Stage 0–4 staged rollout, with Phase A porting work concentrated in Stage 0–2) | Justified by compile-time safety & reliability gains |

---

## Crate Architecture Diagram

See `../diagrams/module-crate-mapping.svg` for the full crate dependency graph (DAG structure).

### Dependency Layers

```
Tier 0 (Pure Data Structures):
  └─ aeg-config (TOML parsing, no external I/O)

Tier 1 (Foundation Services):
  ├─ aeg-db (SQLite wrapper, stateless)
  └─ aeg-file-manager (file I/O primitives)

Tier 2 (Protocol Clients):
  ├─ aeg-mqtt-client (MQTT client primitives)
  └─ aeg-python-bridge (IPC to Python subprocess)

Tier 3 (Domain Logic):
  ├─ aeg-sensors (sensor aggregation, uses aeg-db + aeg-python-bridge)
  ├─ aeg-metrics (signal RMS aggregation, uses aeg-db + aeg-sensors)
  ├─ aeg-beacon (serial port control, no shared dependencies)
  └─ aeg-ota (firmware updates, uses aeg-file-manager)

Tier 4 (Public Interfaces):
  ├─ aeg-appserver (REST API, uses all Tier 3)
  └─ aeg-supervisor (single systemd unit; launches and supervises
     all 14 services as child processes)
```

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

### Original (Node.js, Synchronous Pipeline)

```
Cloud MQTT Broker
       ↓ (1 event per ~100ms)
MQTT.on('message') [callback in event loop]
       ↓
Parse message + validate
       ↓
Database write (blocking, then async)
       ↓
Trigger sensor aggregation [async/await]
       ↓
Compute signal RMS → beacon update
       ↓ (potentially blocked by GC or worker thread contention)
REST client polling (separate thread) hangs
```

### New (Rust, Async Pipeline with Isolation)

```
Cloud MQTT Broker
       ↓ (1 event per ~100ms)
rumqttc async::stream::next()
       ↓ (tokio task, non-blocking)
Parse message + validate
       ↓ (spawns task)
aeg-db async write (async sqlite wrapper)
       ↓
Emit "sensor-updated" event on the internal message bus
       ↓ (aeg-sensors subscribes to the bus)
aeg-sensors aggregates & computes signal RMS
       ↓
aeg-beacon updates indicator (independent service, no blocking)
       ↓ (all parallel, no contention)
REST client polling (appserver, async, no blocking)
       ✓ Measured server-side pipeline (285 one-second samples):
         median 55.5 ms / p95 56.4 ms / p99 57.7 ms / max 59.5 ms
         — ~5.5–5.6% of the 1-second budget, 0 samples over 500 ms
         (breakdown: JSON deserialize 18.1 ms + RMS calc ×3 axes
         3.7 ms + diagnostics 13.7 ms = 35.5 ms server-side)
```

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
