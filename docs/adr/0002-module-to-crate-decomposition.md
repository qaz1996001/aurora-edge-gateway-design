# ADR-0002: Module → Crate Decomposition

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Architecture team, platform engineers

---

## Context

The original Node.js implementation already uses a process supervisor to manage multiple independent Node.js and Python processes, providing basic process-level fault isolation. However, individual processes still face limitations:
- **Per-process memory leaks**: Circular references in event handling cause gradual heap growth; no compile-time safety guarantees for module boundaries
- **GC pauses within processes**: Garbage collection runs frequently; unpredictable stop-the-world delays during peak load
- **Lack of type safety at IPC boundaries**: Module boundaries are enforced by convention, not by the compiler; runtime errors common in inter-process communication
- **Manual process coordination**: Dependencies between processes tracked manually; no declarative service management; restart ordering error-prone

The Rust rewrite opportunity: decompose the existing multi-process architecture into a cleaner architecture where each logical domain (sensors, LED control, OTA updates, etc.) becomes its own systemd-managed service with compile-time type guarantees at service boundaries. This strengthens the existing process isolation with language-level safety.

---

## Decision

**Map existing Node.js process boundaries into independent Rust crates**, with each crate compiled into its own systemd-managed binary (when user-facing) or a shared library (for internal abstractions). Strengthen the mapping with compile-time type safety and clearer domain boundaries.

**Decomposition Principles:**
1. **One logical module → one Rust crate** (e.g., sensor aggregation → `aeg-sensors` crate)
2. **Independent deployment**: Each crate/binary restarts independently (via systemd)
3. **Explicit dependencies**: Dependency graph is acyclic (DAG); tier-based organization
4. **Loose coupling**: Services communicate via Unix sockets or TCP; not shared memory

---

## Rationale

### Why Decompose?

**Fault Isolation & Compile-Time Safety**
- LED control crash doesn't kill sensor aggregation
- OTA firmware update can restart independently without touching database
- Service boundaries are verified at compile time (type system enforces correctness); runtime IPC errors caught early
- **Result**: System degrades gracefully; existing process-level isolation is strengthened by compile-time type safety

**Independent Scaling**
- Each service can be horizontally scaled (run multiple instances on multicore ARM)
- Services with different resource needs don't starve each other
- **Result**: Efficient resource usage; no single bottleneck

**Operational Clarity**
- `systemctl status aeg-sensors` shows if sensor service is healthy
- Logs are per-service (easier to debug issues)
- Each service has its own restart policy (some critical, some best-effort)
- **Result**: Simpler troubleshooting; faster incident response

**Development Velocity**
- Teams can work on different crates in parallel (less merge conflicts)
- One team can optimize sensor aggregation without breaking appserver routes
- Testing is per-crate (faster feedback loop)
- **Result**: Crate boundaries make parallel development and independent testing easier during the port

### Crate Organization (Dependency Tiers)

```
Tier 0: Data Structures (no dependencies)
  └─ aeg-config (TOML parsing, zero I/O)

Tier 1: Foundation (minimal dependencies)
  ├─ aeg-db (SQLite wrapper)
  └─ aeg-file-manager (file I/O)

Tier 2: Protocol Clients (Foundation only)
  ├─ aeg-mqtt-client (MQTT primitives)
  └─ aeg-python-bridge (Python IPC)

Tier 3: Domain Logic (Tier 0–2 only)
  ├─ aeg-sensors (aggregation)
  ├─ aeg-metrics (signal RMS aggregation)
  ├─ aeg-led (GPIO control)
  └─ aeg-ota (firmware updates)

Tier 4: Public Interfaces (all tiers)
  ├─ aeg-appserver (REST API)
  └─ aeg-main (systemd integration)
```

**Benefit**: Tier 0 can be tested in isolation; Tier 1 adds minimal dependencies; Tier 3 can't depend on Tier 4.

---

## Consequences

### Positive
1. **Fault isolation**: Crash in one service doesn't cascade
2. **Parallel development**: Teams work independently; fewer merge conflicts
3. **Faster testing**: Each crate has focused test suite
4. **Operational visibility**: `systemctl` and logs per-service
5. **Incremental optimization**: Can optimize one service without touching others

### Negative
1. **Increased IPC overhead**: Communication between processes (Unix sockets, TCP) slower than in-process function calls
   - **Mitigation**: Use efficient serialization (bincode, MessagePack); batch messages where possible
2. **Configuration complexity**: Each service needs its own config file; environment setup more complex
   - **Mitigation**: Shared configuration convention; template systemd units
3. **Testing complexity**: Integration tests must start multiple services
   - **Mitigation**: Docker or systemd test fixtures for multi-service tests
4. **Operational overhead**: More services to monitor and restart
   - **Mitigation**: Systemd health checks and automated restart policies

### Trade-Offs Accepted
- **Slight IPC latency overhead** (0.1–1 ms per inter-service call) for **fault isolation and operational clarity**
- **More complex deployment** (the actual decomposition landed at 25 crates, 19 binaries, and 14 systemd-supervised services) for **independent service management**

---

## Implementation Details

### Service Binaries
Services deployed as independent systemd units:

```ini
# /etc/systemd/system/aeg-appserver.service
[Unit]
Description=Aurora Edge Gateway - REST API Server
After=aeg-db.service

[Service]
Type=simple
ExecStart=/usr/local/bin/aeg-appserver --config /etc/aeg/appserver.toml
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Shared Libraries (Internal)
Internal crates (not user-facing) compiled as libraries and linked into binaries:

- `aeg-config`: Embedded in each binary; shared dependency
- `aeg-db`: Embedded in each binary that needs database access
- `aeg-mqtt-client`: Embedded in MQTT-dependent binaries

### Inter-Service Communication
- **SQLite**: Multiple services read/write via rusqlite; ACID guarantees ensure consistency
- **Event notifications**: Unix domain socket for low-latency notifications (sensor updated → LED should update)
- **REST APIs**: HTTP for high-level client communication (each service exposes port; loadbalancer or localhost routing)

---

## Service Interaction Example

```
REST Client → HTTP GET /sensors → aeg-appserver:3000
                                       ↓ (reads from shared DB)
                                    aeg-db.service
                                       ↓ (queries SQLite)
                                    /var/lib/aeg/data.db

MQTT Broker → MQTT message → aeg-mqtt:1883
                                ↓ (parses message)
                           (notifies via Unix socket)
                                ↓
                         aeg-sensors:9000
                                ↓ (aggregates; writes to DB)
                            aeg-db.service
                                ↓
                         /var/lib/aeg/data.db
                                ↓ (notifies LED service)
                                ↓
                         aeg-led:8000
                                ↓ (updates GPIO)
                          [RGB LED flashes]
```

---

## Related Decisions
- **[ADR-0003](0003-hybrid-runtime-keep-python-subprocess.md)**: How Python subprocess integrates
- **[ADR-0005](0005-staged-rollout-and-rollback.md)**: Independent service restart strategy
- **Architecture**: See `../02-architecture-overview.md` for full dependency diagram

---

## Monitoring & Observability

Each service logs to systemd journal:
```bash
# View all Aurora Edge Gateway logs
journalctl -u aeg-*.service

# View only sensor service
journalctl -u aeg-sensors.service -f

# View with timestamps and full detail
journalctl -u aeg-appserver.service -o verbose
```

Health checks per service:
```bash
# Is appserver responding?
curl localhost:3000/health

# Is sensor aggregation working?
curl localhost:9000/health

# Overall system health
systemctl status aeg-*.service
```

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Architecture team  
**Implementation**: Phase A (all crates created by week 2)
