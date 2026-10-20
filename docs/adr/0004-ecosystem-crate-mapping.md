# ADR-0004: Technology Choices – npm → Rust Ecosystem Mapping

**Status**: Accepted  
**Decision Date**: 2026-Q3  
**Stakeholders**: Platform engineers, architecture team

---

## Context

Migrating from Node.js to Rust requires selecting equivalent libraries (crates) for each npm dependency. Key criteria:
- **Maturity**: Crate is actively maintained with good API stability
- **Ecosystem Fit**: Crate designed for the use case (not generalist)
- **Community**: Large enough community to get help; good documentation
- **Async Support**: For I/O-bound crates, prefer async-first design

---

## Decision

Adopt the following npm → Rust crate mapping (see table below).

---

## Mapping Table

| npm (Node.js) | Rust (crate) | Domain | Rationale |
|---|---|---|---|
| **express** | **axum** | HTTP Server | Zero-cost abstractions; type-safe routing; excellent async integration with tokio |
| **mqtt** (rumba) | **rumqttc** | MQTT Client | Async-first MQTT library; excellent embedded/IoT support |
| **better-sqlite3** | **rusqlite** | SQLite Driver | Synchronous API is simpler for our use case; no async DB driver overhead needed |
| **socket.io** | **tokio-tungstenite** | WebSocket/Real-time | Lightweight WebSocket library; sufficient for our real-time needs |
| **winston** | **tracing** + **slog** | Logging | Structured logging with excellent async support; journald integration |
| JSON (native) | **serde_json** | JSON Serialization | Zero-copy; standard Rust serialization |
| **rpi.GPIO** (npm) | **rppal** | GPIO/PWM Control | Rust-native GPIO library; actively maintained; excellent ARM support |
| **request** (HTTP client) | **reqwest** | HTTP Client | Modern async HTTP; excellent error handling |
| **fs** (native) | **tokio::fs** | Async File I/O | Async file operations without blocking tokio runtime |
| **child_process** | **tokio::process** | Subprocess Spawning | Async subprocess management (for Python IPC) |
| **crypto** (native) | **ring** | Cryptography | Audited crypto library; better than Node.js `crypto` for security |
| **commander** (CLI) | **clap** | Command-line Parsing | Type-safe CLI argument parsing |

---

## Rationale per Choice

### HTTP Server: Express → Axum

**npm Choice**: Express.js (most popular Node.js HTTP server)

**Rust Alternative Candidates**:
- **Actix-web**: Very performant; less type-safe routing; heavier dependencies
- **Rocket**: More opinionated; powerful; heavier overhead
- **Warp**: Filter-based; functional style (learning curve)
- **Axum**: Minimal; composable; type-safe routing; excellent tokio integration ✅

**Selection Rationale**:
- Type safety on routing prevents 404 bugs
- Zero-cost layer on tokio (same async runtime we're using)
- Simpler mental model than filter-based alternatives
- Good documentation and examples

**Risk**: Axum is newer than Express; less mature ecosystem. **Mitigation**: Golden-master tests ensure behavioral parity.

---

### MQTT Client: Rumba → Rumqttc

**npm Choice**: mqtt (rumba package)

**Rust Alternative Candidates**:
- **paho-mqtt**: C library wrapper; works but heavier
- **rust-mqtt**: Experimental; not recommended for production
- **rumqttc**: Modern async-first Rust MQTT; excellent embedded support ✅

**Selection Rationale**:
- Designed for IoT/embedded (our use case)
- Async-first (integrates cleanly with tokio)
- Good API; handles reconnection and backpressure
- Actively maintained

---

### SQLite Driver: better-sqlite3 → Rusqlite

**npm Choice**: better-sqlite3 (synchronous SQLite bindings)

**Rust Alternative Candidates**:
- **sqlx**: Async; compile-time checked queries; heavier setup
- **diesel**: ORM; powerful but opinionated; heavier
- **rusqlite**: Simple synchronous wrapper; minimal overhead ✅
- **sqlite**: Pure Rust implementation; slower; not recommended

**Selection Rationale**:
- Our database workload is not async-heavy (simple queries, < 10ms latency)
- Synchronous API is simpler; no async overhead needed
- Excellent documentation; widely used
- No async complexity for database operations

**Trade-off**: Synchronous calls to rusqlite might block tokio tasks if DB is slow. **Mitigation**: Wrap DB calls in `tokio::task::spawn_blocking()` if needed.

---

### Logging: Winston → Tracing + Slog

**npm Choice**: winston (structured logging)

**Rust Alternative Candidates**:
- **log**: Lightweight; standard but less feature-rich
- **env_logger**: Simple; coupled to log crate
- **tracing**: Structured async logging; excellent async integration ✅
- **fern**: Flexible but heavier setup

**Selection Rationale**:
- Structured logging for machine-parsed alerts
- Async integration (no blocking in async context)
- journald support (direct logging to systemd journal)
- Excellent filtering and level control per module

---

### GPIO Control: npm rpi.GPIO → rppal (Rust)

**npm Choice**: npm module wrapping Python RPi.GPIO

**Rust Alternative Candidates**:
- **rppal**: Pure Rust GPIO; excellent ARM support ✅
- **sysfs_gpio**: Lower-level; more boilerplate; less safe
- C FFI to pigpio: Possible but unnecessary complexity

**Selection Rationale**:
- Pure Rust (no Python overhead; no interpreter start cost)
- Excellent ARM support (tested on Raspberry Pi, BeagleBone, etc.)
- Type-safe API; prevents invalid GPIO states

**Improvement**: Eliminates Python subprocess overhead for GPIO (Node.js called Python; Rust calls C directly).

---

## Crate Versions & Lock Strategy

### Dependency Lock File (Cargo.lock)
All dependencies version-pinned in `Cargo.lock` for reproducible builds:
```toml
[package]
name = "aeg-appserver"
version = "0.1.0"

[dependencies]
tokio = { version = "1.35", features = ["full"] }
axum = "0.7"
serde_json = "1.0"
```

**Policy**:
- Pin exact versions in Cargo.lock
- Review security advisories weekly (`cargo audit`)
- Update dependencies monthly (if no security issues)
- Test thoroughly after dependency updates

### Vulnerable Dependency Response
If a dependency has known vulnerability:
1. Patch released → Update crate to patched version (immediate)
2. No patch available → Pin to last safe version; find alternative crate (week-long sprint)

---

## Serialization Format: JSON vs. Bincode

For inter-process communication (Rust ↔ Python):
- **Format**: bincode (Rust) or msgpack/JSON for compatibility
- **Rationale**: 
  - bincode: Fast, compact (ideal for high-frequency messages)
  - JSON: Human-readable (easier debugging; good for less frequent messages)

**Decision**: Use bincode for high-frequency, low-latency IPC; JSON for config and human interaction.

---

## Build Target: Cargo Cross

For compiling Rust to ARM edge hardware:
```bash
# Compile for ARMv8 (64-bit ARM, typical edge device)
cargo build --release --target aarch64-unknown-linux-gnu

# Compile for ARMv7 (32-bit ARM, older devices)
cargo build --release --target armv7-unknown-linux-gnueabihf
```

**Tool**: `cargo-cross` for cross-compilation without native ARM toolchain locally.

---

## Validation

### Before Merging Any Code
```bash
# 1. Dependency audit
cargo audit  # Must pass (no known vulnerabilities)

# 2. Compile check
cargo check

# 3. Run tests
cargo test

# 4. Lint check
cargo clippy

# 5. Code coverage (Phase A)
cargo tarpaulin
```

---

## Related Decisions
- **[ADR-0002](0002-module-to-crate-decomposition.md)**: Each crate uses subset of these dependencies
- **[ADR-0003](0003-hybrid-runtime-keep-python-subprocess.md)**: tokio subprocess support for Python IPC

---

## Maintenance Plan

### Quarterly Dependency Review
- [ ] `cargo outdated` – identify stale dependencies
- [ ] `cargo audit` – check for vulnerabilities
- [ ] Evaluate major version upgrades (breaking changes)
- [ ] Test thoroughly if updates made

### SLA for Security Patches
- **Critical**: Deploy within 24 hours
- **High**: Deploy within 1 week
- **Medium**: Deploy within 1 month
- **Low**: Deploy on next scheduled release

---

**Status**: ✅ **ACCEPTED** (2026-Q3)  
**Ratified By**: Architecture team  
**Implementation**: Phase A (dependencies locked in Cargo.toml)
