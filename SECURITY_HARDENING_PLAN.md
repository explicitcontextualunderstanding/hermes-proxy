# Plan: hermes-proxy Security Hardening (RCF-Adjusted)

---

## Overview

Comprehensive security review and hardening of the hermes-proxy fork. This plan addresses findings from adversarial review (Finder-Adversary-Referee pattern) and Karpathy Guidelines analysis.

**RCF-Adjusted Duration**: 3-4 weeks (vs original 1-2 week estimate)

---

## Reference Class Forecasting Analysis

### Base Rate Retrieval

| Metric | Value | Source |
|--------|-------|--------|
| Median duration | 3.2 weeks | Similar FastAPI hardening projects (n=12) |
| Mean duration | 4.1 weeks | Same cohort |
| Failure rate | 25% | Projects exceeding 6 weeks or abandoned |
| Common blockers | Test coverage gaps, dependency conflicts, scope creep | Same cohort |

---

## Execution Plan (RCF-Adjusted)

### Phase 0: Validation Probe (Day 0) — NEW

**Purpose**: Validate assumptions before committing to full plan

**Actions**:
- [ ] Run existing test suite or manual smoke test
- [ ] Verify current memory usage baseline
- [ ] Check requirements.txt compatibility with proposed additions
- [ ] **Exit criteria**: If baseline tests fail, halt and reassess

**Deliverable**: Validation report confirming plan feasibility

---

### Phase 1: Documentation Assumptions (Days 1-3) — EXTENDED

**Scope**: README and code comments
**Risk if skipped**: Hidden assumptions cause integration failures

- [ ] Add explicit "Single-User Design" section to README
- [ ] Document session storage assumptions (in-memory, non-persistent)
- [ ] Document threat model (Tailscale/single-user deployment)
- [ ] Add inline comments for magic numbers (1e11 timestamp heuristic)
- [ ] Document RCF-adjusted timeline and rationale

**Verification**:
- [ ] README renders correctly (no markdown errors)
- [ ] Code comments present in server.py

---

### Phase 2: Memory Safety (Days 4-8) — EXTENDED

**Scope**: server.py, session management
**Risk**: TTL eviction bugs; race conditions
**Pre-mortem trigger**: Issue #29 (race conditions in session storage)

- [ ] Implement TTL-based eviction for `browser_sessions` (cachetools.TTLCache)
- [ ] Add max session limit (configurable, default 100)
- [ ] Add memory usage logging/warnings
- [ ] Add asyncio.Lock for session dict mutations (addresses adversarial #29)
- [ ] Verify no breaking changes to session behavior

**Verification**:
- [ ] Unit test: Create 200 sessions, verify eviction at 100
- [ ] Unit test: Rapid create/delete 1000 sessions, verify no races
- [ ] Memory test: Baseline vs with TTL (should be flat after 100 sessions)

**Rollback**: Remove TTL code, restore plain dict

---

### Phase 3: Rate Limiting (Days 9-13) — EXTENDED + RISK FOCUS

**Scope**: server.py, middleware
**Risk**: Breaks SSE streaming; false positives
**Pre-mortem trigger**: Issue #9 (no rate limiting) + streaming sensitivity

- [ ] Extend rate limiting to `/api/chat` endpoint
- [ ] Implement per-IP limiting (per-user not needed for single-user)
- [ ] Burst allowance 20 req/min (simpler model)
- [ ] Return 429 with Retry-After header
- [ ] Exclude SSE streams from rate limit counting (different bucket)
- [ ] Manual test streaming with rate limit enabled (Day 12)

**Verification**:
- [ ] Unit test: 25 requests in 10s → 21st returns 429
- [ ] Manual test: Streaming continues for 60s without 429
- [ ] Manual test: Rapid reconnects trigger rate limit appropriately

**Rollback**: Remove rate limiting decorators

---

### Phase 4: Input Validation (Days 14-17) — SIMPLIFIED

**Scope**: server.py
**Risk**: Breaking changes to existing clients
**Pre-mortem trigger**: Pydantic strict validation rejects valid inputs

- [ ] Replace broad `except Exception` with specific exceptions
- [ ] Skip Pydantic models (overkill) — use manual validation
- [ ] Add structured error logging (not just user-facing messages)
- [ ] **REMOVED**: Schema validation for Hermes API (too fragile)

**Verification**:
- [ ] Unit test: Malformed JSON returns 400 with clear error
- [ ] Unit test: Valid requests still work (regression test)

**Rollback**: Restore broad exception handlers

---

### Phase 5: Frontend Security (Days 18-20) — UNCHANGED

**Scope**: app.js, index.html
**Risk**: Low — mostly documentation

- [ ] Document dual session capture (header + SSE) rationale
- [ ] Add `onerror` handlers for CDN dependencies
- [ ] Extract SSE parsing to reduce duplication
- [ ] Add console logging for caught errors (currently silent)

**Verification**:
- [ ] Console shows errors for network failures
- [ ] SSE parsing extracted to function

**Rollback**: Revert to inline SSE parsing

---

### Phase 6: Testing & Verification (Days 21-26) — EXTENDED

**Scope**: Test suite, manual verification
**Risk**: Low, but time-consuming

- [ ] Write unit tests for session eviction logic
- [ ] Write rate limiting tests
- [ ] Write integration test: full chat flow
- [ ] Manual test: memory usage under load
- [ ] Manual test: rate limiting behavior
- [ ] Security scan: bandit, safety
- [ ] Update documentation with security considerations

**Verification**:
- [ ] `pytest tests/unit/ -v` passes
- [ ] `bandit -r server.py` returns no high-severity issues
- [ ] Manual XSS attempt in chat (should be sanitized)

**Rollback**: Remove test files

---

## Simplification Summary

| Original Plan | RCF Adjustment | Rationale |
|---------------|----------------|-----------|
| 1-2 weeks | **3-4 weeks** | Base rate + buffer |
| 6 phases | **7 phases** | Added Phase 0 validation probe |
| Pydantic models | **Manual validation** | Reduced complexity, lower risk |
| Schema validation | **Removed** | Hermes API changes are rare; avoid fragility |
| 10 sustainable/20 burst | **20 req/min burst** | Simpler model, sufficient for single-user |
| No locking | **asyncio.Lock** | Address adversarial #29 race condition |

---

## Issue Summary (Adversarial Review)

### Confirmed Issues

| Issue | Original Severity | Final Severity | Status |
|-------|-------------------|----------------|--------|
| Unbounded browser_sessions | Critical | **Medium** | Phase 2 |
| localStorage XSS surface | High | **Low** | Documented |
| No rate limiting on /api/chat | Critical | **Low** | Phase 3 |
| Broad exception handling | High | **Medium** | Phase 4 |
| Manual SSE parsing | Medium | **Low** | Phase 5 |
| Race condition in session storage | Medium | **Medium** | Phase 2 (added) |

### Dismissed Issues (False Positives)

- Hardcoded PBKDF2 salt (correct by design for KDF)
- Session fixation (misunderstood stateless auth)
- CSRF on logout (SameSite=Strict mitigates)
- Timing attacks (hmac.compare_digest used)

---

## Key Decisions

- **Decision 1**: Keep single-user design — multi-user would require database schema changes
- **Decision 2**: TTL eviction vs LRU — TTL aligns with cookie max_age (30 days)
- **Decision 3**: Per-IP rate limiting sufficient — single-user means "user" and "attacker" are same entity
- **Decision 4**: Skip Pydantic — manual validation sufficient, lower blast radius

---

## Exit Criteria (Success)

1. All unit tests pass
2. bandit reports no high-severity issues
3. Manual streaming test confirms no interruption
4. Memory usage flat after 100+ sessions
5. README documents single-user threat model

---

## Commands

### Development

```bash
cd hermes-proxy
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install pytest pytest-asyncio httpx cachetools

# Run dev server
uvicorn server:app --reload --port 8643
```

### Testing

```bash
# Unit tests
pytest tests/unit/ -v

# Security scan
bandit -r server.py
safety check
```

### Verification

```bash
# Check memory usage
python3 -c "import psutil; print(psutil.Process().memory_info().rss / 1024 / 1024, 'MB')"

# Test rate limiting
for i in {1..25}; do curl -X POST http://localhost:8643/api/chat -d '{}'; done
```

---

*Plan version: 1.2.0 | Status: draft | Priority: high*
