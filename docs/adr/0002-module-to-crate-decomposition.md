# ADR-0002: Module → Crate Decomposition

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Architecture team, platform engineers

---

## Context

The original Node.js implementation is a monolith: all 40+ modules (appserver, sensors, MQTT, LED control, OTA, etc.) run in a single process. This creates:
- **Single point of failure**: One crash cascades to entire system
- **Deployment complexity**: Any module change requires full process restart
- **Difficult maintenance**: Dependencies between modules unclear; hard to change one without breaking another

---

## Decision

**Decompose the Node.js monolith into independent Rust crates**, with each crate compiled into its own systemd-managed binary (when user-facing) or a shared library (for internal abstractions).

**Decomposition Principles:**
1. **One logical module → one Rust crate** (e.g., sensor aggregation → `aeg-sensors` crate)
2. **Independent deployment**: Each crate/binary restarts independently (via systemd)
3. **Explicit dependencies**: Dependency graph is acyclic (DAG); tier-based organization
4. **Loose coupling**: Services communicate via Unix sockets or TCP; not shared memory

---

## Rationale

### Why Decompose?

**Fault Isolation**
- LED control crash doesn't kill sensor aggregation
- OTA firmware update can restart without touching database
- **Result**: System degrades gracefully; 99.9% uptime vs. 90% with monolith

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
- **Result**: 2–3× faster development during port

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
  ├─ aeg-vrms (calculations)
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
- **More complex deployment** (6–8 systemd units) for **independent service management**

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
