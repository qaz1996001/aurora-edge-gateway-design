# Review Process: Multi-Role Structured Code Review

## Overview

Code review for a large rewrite must catch cross-cutting concerns (security, performance, memory safety, architectural consistency) that no single reviewer can address. This document describes a **multi-role review framework** where each reviewer has a specific lens and responsibility.

See **[ADR-0007](adr/0007-port-marker-convention.md)** for marker conventions used during review.

---

## Review Roles & Responsibilities

### Role 1: Rust Expert (Language & Best Practices)

**Responsibility**: Ensure idiomatic Rust; catch memory-safety and lifetime issues; verify type-safety guarantees.

**Checklist**:
- [ ] Code uses Rust idioms (no C-style loops, manual lifetime management, unsafe blocks)
- [ ] All `unsafe` blocks are justified and marked with `SAFETY:` comment
- [ ] No unwrap() / panic!() in production code paths (use `?` operator or explicit error handling)
- [ ] Proper error types used (not `String`, not generic `anyhow::Error` when specific error is known)
- [ ] Lifetimes correct; no unnecessary `.clone()` or `.to_string()`
- [ ] Generic code is generic (not hardcoded for one type)
- [ ] Compiler warnings addressed (no `#![allow(...)]` surprises)
- [ ] Dependency versions locked and justified (no wildcard versions)

**Example Review Comment**:
```
🚨 Line 42: `user.posts.clone()` creates unnecessary deep copy.
   Consider returning `&[Post]` to avoid allocation.
   
   fn get_user_posts(user: &User) -> Vec<Post> {  // ❌
   fn get_user_posts(user: &User) -> &[Post] {   // ✅
```

---

### Role 2: Security Engineer

**Responsibility**: Identify security vulnerabilities (injection, buffer overflow, crypto misuse, privilege escalation).

**Checklist**:
- [ ] All user input validated before use (REST payload, MQTT topic, file path)
- [ ] No SQL injection (using parameterized queries, not string interpolation)
- [ ] No command injection (not passing user input to shell)
- [ ] Cryptographic operations use standard libraries (not custom crypto)
- [ ] Secrets not hardcoded (no API keys, passwords, private keys in code)
- [ ] Privilege handling correct (services run with minimal privileges; no unexpected sudo)
- [ ] Deserialization safe (no arbitrary code execution via JSON/TOML parsing)
- [ ] File operations safe (no path traversal; validate paths before access)

**Example Review Comment**:
```
⚠️ Line 156: SQL query with string concatenation.
   SELECT * FROM sensors WHERE id = " + input_id
   
   This is vulnerable to SQL injection if input_id is untrusted.
   Use parameterized query:
   
   let query = rusqlite::params![input_id];
   db.query("SELECT * FROM sensors WHERE id = ?1", query)?;
```

---

### Role 3: Systems Architect

**Responsibility**: Verify architectural consistency; ensure module boundaries are respected; check for cross-cutting issues.

**Checklist**:
- [ ] Module responsibilities clear; single concern per crate
- [ ] Dependencies follow acyclic dependency graph (Tier 0 → 4)
- [ ] Inter-service communication uses documented protocol (Unix socket, TCP port, format)
- [ ] Service discovery / service locator pattern is explicit (no magic)
- [ ] Concurrency model clear (tokio tasks, threads, locks, channels)
- [ ] Async/await boundaries explicit (no blocking calls in async context)
- [ ] Data ownership and passing is clear (move vs. borrow vs. clone)
- [ ] Error propagation consistent (Result types, error chain)

**Example Review Comment**:
```
🏗️ Line 78: `blocking::spawn_blocking()` inside async function.
   
   This is a red flag. If we're doing blocking I/O, we should:
   1. Either move to sync function, or
   2. Use async I/O (tokio::fs, tokio::net), or
   3. Use tokio::task::block_in_place() with clear documentation
   
   What's the blocking operation? Can we use async?
```

---

### Role 4: Performance Engineer

**Responsibility**: Identify performance bottlenecks; catch allocation/GC issues; verify latency targets.

**Checklist**:
- [ ] Hot paths use efficient data structures (no Vec when array/slice is appropriate)
- [ ] No unnecessary allocations in loops (`vec![]`, `String::from()`)
- [ ] Copying/cloning minimized (prefer borrowing; use move semantics where appropriate)
- [ ] Lock contention minimized (fine-grained locking; no lock-in-loop)
- [ ] Buffering strategy appropriate (unbounded queues can cause memory growth)
- [ ] Zero-copy patterns used where applicable (MMap for large files, byte slices)
- [ ] Profiling hooks in place for hot functions (tracing, timer)

**Example Review Comment**:
```
⚡ Line 203: Building Vec<Message> for every MQTT message.
   
   for msg in mqtt_stream {  // 100+ msg/sec
       let mut response = Vec::new();
       response.push(msg);
       // ... more work ...
   }
   
   This allocates a Vec for each message; high GC pressure.
   
   Solution: Use iterator combinator or reuse buffer:
   let mut buffer = Vec::with_capacity(100);
   for msg in mqtt_stream {
       buffer.clear();
       buffer.push(msg);
       // ... reuse buffer ...
   }
```

---

### Role 5: Code Reviewer (Testing & Maintainability)

**Responsibility**: Ensure code is testable, maintainable, readable; verify test coverage.

**Checklist**:
- [ ] Code is testable (no hard-to-mock dependencies; DI where appropriate)
- [ ] Unit tests present for all logic (70%+ coverage)
- [ ] Tests verify both happy path and error cases
- [ ] Test names are descriptive (not `test_1`, `test_2`)
- [ ] Comments explain *why*, not *what* (code is readable; comments add insight)
- [ ] No dead code (remove unused variables, functions, branches)
- [ ] Naming is clear (function names are verbs; variable names describe intent)
- [ ] Code duplication minimized (DRY principle)

**Example Review Comment**:
```
📝 Line 45: Function `process_sensor_data()` does 3 things.
   
   1. Validate input
   2. Query database
   3. Update LED state
   
   Hard to test in isolation; hard to understand; hard to change.
   
   Solution: Split into smaller functions with single responsibility:
   - fn validate_sensor_data(data: &SensorData) -> Result<()>
   - fn fetch_sensor_config(sensor_id: u32) -> Result<Config>
   - fn update_led_state(color: RGB) -> Result<()>
```

---

## Review Workflow

### Step 1: Self-Review (Developer)
Before submitting for review, the developer self-reviews against all 5 checklists:
- [ ] Rust Expert checklist (idiomatic; safe)
- [ ] Security checklist (no injection; no secrets)
- [ ] Architect checklist (module boundaries; async correctness)
- [ ] Performance checklist (efficient; no bottlenecks)
- [ ] Code reviewer checklist (testable; clear)

**Output**: Self-review checklist attached to PR or merge request.

### Step 2: Async Review (5 Reviewers in Parallel)
1. **Rust Expert** → Reviews for language/idioms (24h turnaround)
2. **Security Engineer** → Reviews for vulnerabilities (24h turnaround)
3. **Systems Architect** → Reviews for architectural consistency (24h turnaround)
4. **Performance Engineer** → Reviews for bottlenecks (48h turnaround, may require profiling)
5. **Code Reviewer** → Reviews for testing and maintainability (24h turnaround)

**Expectation**: All 5 reviewers provide feedback in parallel (not sequential).

**Output**: Each reviewer provides:
- Comments on specific lines (with line numbers)
- Severity level (🚨 must-fix; ⚠️ should-fix; 💡 nice-to-have)
- Actionable suggestion for fix

### Step 3: Developer Addresses Feedback
Developer reviews all feedback and makes changes:
- **🚨 Must-fix**: All blocking issues resolved
- **⚠️ Should-fix**: Either fixed or documented with explicit rationale
- **💡 Nice-to-have**: Fixed if trivial; deferred to Phase B with TODO(port) marker if complex

**Output**: Updated PR with responses to all comments (e.g., "Fixed in commit abc123").

### Step 4: Verification Review (Spot Check)
One reviewer (rotating role) spot-checks that feedback was actually addressed:
- [ ] 🚨 issues resolved
- [ ] ⚠️ issues resolved or documented
- [ ] Code change matches suggested fix

**Output**: Approval or request for further changes.

### Step 5: Merge & Regression Testing
- PR merged to main branch
- CI/CD pipeline runs (Tier 1–3 tests; see Testing Strategy)
- If tests fail: investigate; fix; re-test
- If tests pass: code is ready for Phase A integration

---

## Review Checklist Template

```markdown
## Code Review Checklist

### Rust Expert Review
- [ ] Code uses Rust idioms (no C-style loops)
- [ ] All `unsafe` blocks justified with SAFETY: comment
- [ ] No panics in production code
- [ ] Proper error types used (not String)
- [ ] No unnecessary clones or to_string()
- [ ] Generic code is truly generic
- [ ] Compiler warnings addressed

### Security Review
- [ ] User input validated before use
- [ ] No SQL injection (parameterized queries)
- [ ] No command injection
- [ ] Secrets not hardcoded
- [ ] File operations safe (no path traversal)
- [ ] Cryptography uses standard libraries

### Architecture Review
- [ ] Module boundaries respected
- [ ] Dependencies follow DAG (no cycles)
- [ ] Inter-service communication documented
- [ ] Concurrency model clear
- [ ] Async/await boundaries explicit
- [ ] Error propagation consistent

### Performance Review
- [ ] Hot paths efficient
- [ ] No unnecessary allocations
- [ ] Copying/cloning minimized
- [ ] Lock contention low
- [ ] Buffering strategy appropriate

### Code Quality Review
- [ ] Code is testable
- [ ] Unit tests present (70%+ coverage)
- [ ] Tests cover error cases
- [ ] Comments explain why, not what
- [ ] No dead code
- [ ] Naming is clear

---

**Reviewed By**: [Name], [Date]
**Approval**: ✅ Approved / ⏳ Needs Changes / ❌ Rejected
```

---

## Continuous Feedback Loop

### Weekly Code Review Sync
- **Tuesday 2 PM**: Review feedback trends (common issues, patterns)
- **Discussion**: What mistakes are recurring? What patterns should be enforced?
- **Output**: Refinements to team coding standards or review checklist

### Post-Phase A Retrospective
- Review all feedback collected during Phase A
- Identify top 10 most common issues
- Update team documentation and training
- Adjust review checklist for Phase B

---

## Tools & Automation

### Automated Checklist Items
These can be checked by CI/CD without human review:
- `cargo fmt` (code formatting)
- `cargo clippy` (linting)
- `cargo test` (unit tests)
- Compiler warnings (must be zero)
- Code coverage (must be ≥ 70%)
- Dependency audit (`cargo audit` — no known vulnerabilities)
- SAFETY comments present (regex check: all `unsafe {}` blocks must have SAFETY comment)

### Example GitHub Actions Workflow
```yaml
name: Code Review Automation
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: cargo test --all-features
      - run: cargo tarpaulin --out Xml  # Code coverage
      - uses: codecov/codecov-action@v2
  
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: cargo audit
  
  safety-comments:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: |
          unsafe_blocks=$(grep -r "unsafe {" src/ | wc -l)
          safety_comments=$(grep -r "SAFETY:" src/ | wc -l)
          if [ $unsafe_blocks -ne $safety_comments ]; then
            echo "❌ Not all unsafe blocks have SAFETY comments"
            exit 1
          fi
```

---

## Review SLAs (Service Level Agreements)

| Role | Turnaround | Example Scenario |
|---|---|---|
| Rust Expert | 24 hours | Code submitted Monday; feedback by Tuesday EOD |
| Security Engineer | 24 hours | Code review; security issues listed within 24h |
| Architect | 24 hours | Architectural feedback same day if submitted by 10 AM |
| Performance Engineer | 48 hours | May require profiling; non-blocking for merge (documented as Phase B optimization) |
| Code Reviewer | 24 hours | Testability feedback same day |

---

## Next Steps

1. **Risk Management** → See `06-risk-register-template.md` for how to categorize and track risks
2. **ADR on Marker Conventions** → See **[ADR-0007](adr/0007-port-marker-convention.md)** for `TODO(port)` / `FIXME(port)` standards

---

**Last Updated**: 2026-10-20  
**Status**: Review process approved  
**Next Review**: Week 2 of Phase A (first PRs submitted for review)
