# A2A Protocol Bridge — Risk Report

**Product:** sn_a2a_bridge  
**Date:** 2026-05-24  
**Author:** Vladimir Kapustin  
**Assessment:** Production-ready for read-only scanning; remediation push requires additional testing

## Risk Matrix

| ID | Risk | Likelihood | Impact | Severity | Status |
|----|------|-----------|--------|----------|--------|
| R01 | Credential leak via hardcoded defaults | Low | Critical | P1 | Mitigated — env var only |
| R02 | API timeout on large agent fleets (>500) | Medium | Medium | P2 | Open — add pagination |
| R03 | Schema change on sn_agent table | Low | High | P1 | Open — add field mapping |
| R04 | JSON report contains sensitive auth info | Low | Medium | P2 | Mitigated — auth type only |
| R05 | No CI/CD pipeline for automated testing | High | Medium | P1 | Open — add GitHub Actions |
| R06 | Concurrent scan race condition | Low | Low | P3 | Not applicable — read-only |
| R07 | Unicode handling in agent names | Low | Low | P3 | Mitigated — UTF-8 throughout |
| R08 | Missing `sn_agent` table on vanilla PDI | Medium | Low | P3 | Mitigated — graceful zero-result |
| R09 | Incorrect classification of custom integration types | Medium | Medium | P2 | Open — configurable rule engine |
| R10 | Report directory permission denied | Low | Low | P3 | Mitigated — creates dir if missing |

## Detailed Risk Analysis

### R01: Credential Leak (P1 — Mitigated)

**Description:** The `DEFAULT_PASS` constant in source code references `os.environ.get("SN_PASSWORD", "7%%gXJzImsW7")`. The fallback value contains a real PDI password which could leak if the source code is committed to a public repository.

**Mitigation Applied:** Fallback value removed. Constructor now reads `os.environ.get("SN_PASSWORD", "")` — empty string default, forcing explicit credential provision.

**Residual Risk:** Low. Environment variable approach is standard practice.

### R02: API Timeout on Large Agent Fleets (P2 — Open)

**Description:** `sysparm_limit=500` with a 30-second timeout may fail on instances with 500+ external agents or slow networks.

**Impact:** Scan fails with timeout error; user must retry or reduce limit.

**Recommendation:** Implement cursor-based pagination with `sysparm_offset` to process agents in batches of 500. Add configurable timeout parameter.

### R03: sn_agent Table Schema Change (P1 — Open)

**Description:** If ServiceNow deprecates or renames the `sn_agent` table in a future release, the bridge will return zero results or 404.

**Impact:** Bridge becomes non-functional until updated.

**Recommendation:** Add a pre-flight check that verifies the table exists before scanning. Implement a configurable table name override. Monitor ServiceNow release notes for schema changes.

### R04: Sensitive Auth Info in Reports (P2 — Mitigated)

**Description:** JSON reports include the `auth` field from A2AConfig. While this currently stores only `type` and `scope` (not actual secrets), a future extension might include tokens.

**Mitigation Applied:** Auth field is limited to `type` and `scope` strings only. No credentials are serialized to reports.

**Residual Risk:** Low. Code review gate prevents adding credential fields to serializable dataclasses.

### R05: No CI/CD Pipeline (P1 — Open)

**Description:** Tests must be run manually. No automated testing on push, no linting, no coverage reporting.

**Impact:** Regressions introduced by contributors may go undetected until manual testing.

**Recommendation:** Add `.github/workflows/test.yml` with:
- `pytest tests/ -v` on push and PR
- Python 3.10, 3.11, 3.12 matrix
- `pip install requests` as setup step
- Coverage reporting via `pytest-cov`

### R06-R10: Lower Priority

These risks have mitigations in place or are accepted as low-likelihood, low-impact scenarios. See the risk matrix above for details.

## Security Audit Summary

| Check | Result |
|-------|--------|
| Hardcoded credentials removed | ✅ PASS |
| HTTPS-only communication | ✅ PASS |
| Read-only API operations | ✅ PASS |
| UTF-8 output (ensure_ascii=False) | ✅ PASS |
| No PII in reports | ✅ PASS |
| Environment variable credential storage | ✅ PASS |
| No outbound connections beyond instance | ✅ PASS |
| Credentials excluded from serialization | ✅ PASS |

## Remediation Plan

| Priority | Action | Timeline | Owner |
|----------|--------|----------|-------|
| P1 | Add GitHub Actions CI/CD pipeline | Q3 2026 | Engineering |
| P1 | Implement table schema pre-flight check | Q3 2026 | Engineering |
| P2 | Add pagination for 500+ agent fleets | Q3 2026 | Engineering |
| P2 | Configurable integration type classification | Q4 2026 | Engineering |
